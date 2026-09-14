# RHEL Metering via the OpenTelemetry Collector

**Jira:** [TRACING-6702](https://redhat.atlassian.net/browse/TRACING-6702) (Epic — "RHEL telemetry using OpenTelemetry collector")
**Feature draft:** RHELBU-4218 (REQ-SC-01…07), parent Outcome RHELBU-3325
**Support Level:** Technology Preview (`[PLANNED: TRACING-6702] TP`)
**Scope:** Bootstrap shell script + stock OpenTelemetry Collector components
**Repo:** `redhat-opentelemetry-collector`
**Product spec:** `.ai/spec/what/rhel-metering.md` (authoritative behavioral rules)

## Overview

Red Hat needs first-party, billing-grade measurement of RHEL deployments — CPU capacity
units plus a software/entitlement fingerprint — emitted on a recurring cadence from RHEL
hosts, independent of hyperscaler reporting and independent of whether the host is
registered to a Red Hat account.

The feature's driving question: *can the OpenTelemetry Collector — which already ships as a
RHEL package — collect, package, and transmit this metering data instead of a bespoke
metering agent?* This design answers it after validating the requirements against the actual
component inventory of the Red Hat collector distribution.

**Conclusion:** Yes, the collector is a viable vehicle — but **not with stock components
alone.** It supplies transport, durability, scheduling, and a subset of the host facts. The
metering-specific facts are gathered by a **bootstrap shell script** run from a systemd unit
(as root) and handed to the collector as **resource attributes**. No collector fork is
required — the only custom component is the script.

MVP scope: streamed/connected **cloud images** (Console and Marketplace) on **RHEL 8, 9,
10**, in **RPM** and **image-mode** deployments. On-premise and disconnected/batched
collection are designed-for but out of MVP; RHEL 7 is out of scope entirely.

## Architecture

```
+--------------------------------------------------+     +---------------------------------------------+
| systemd: rhel-metering-bootstrap (root)          |     | systemd: otelcol                            |
|                                                  |     |                                             |
|  On first boot:                                  |     |  Receivers                                  |
|    - generate + persist instance_id (UUIDv4)     |     |    +- hostmetrics                           |
|  Every start / refresh:                          |     |         +- cpu: logical.count, physical.count|
|    - gather machine_id (/etc/machine-id)         |     |         +- system: system.uptime            |
|    - gather product_uuid (DMI system UUID)       | env |  Processors                                 |
|    - gather cpu_sockets (lscpu / dmidecode)      | --> |    +- resourcedetection (host.arch, cloud.*)|
|    - detect deployment_model (bootc/rpm-ostree)  |file |    +- resource (inject script attrs from env)|
|    - read RHEL NVR (/etc/redhat-release + RPM)   |     |    +- batch                                 |
|    - read subscription-manager facts             |     |  Exporters                                  |
|    - set delivery_channel                        |     |    +- prometheusremotewrite --> Observatorium|
|    - write OTEL_RESOURCE_ATTRIBUTES env file     |     |  Extensions                                 |
|    - probe persistent storage -> pick queue      |     |    +- filestorage (persistent profile only) |
|      profile (persistent vs in-memory)           |     |                                             |
+--------------------------------------------------+     +---------------------------------------------+
```

## Feasibility investigation (requirement → component)

Validated against `redhat-opentelemetry-collector/manifest.yaml` (distro v0.158.0) and the
contrib component metadata.

### Satisfied by stock components

| Requirement / field | Component | Notes |
|---|---|---|
| `logical_cpus` | `hostmetrics` cpu scraper → `system.cpu.logical.count` | On by default |
| `physical_cores` | `hostmetrics` cpu scraper → `system.cpu.physical.count` | Opt-in; must enable |
| `architecture` | `resourcedetection` system detector → `host.arch` | Off by default; enable |
| `cloud_provider`/`cloud_region`/`cloud_instance_id` | `resourcedetection` ec2/azure/gcp → `cloud.provider`, `cloud.region`, `host.id` | On by default |
| `system_uptime` | `hostmetrics` system scraper → `system.uptime` | On by default |
| Cadence ≥1/hr, 10/hr | `collection_interval` (default 1m) | Config to 6m |
| Durability / no loss on disconnect | `filestorage` + persistent `sending_queue` + `retry_on_failure` | Un-sent requests only |
| Batching | `batch` processor | |
| Transport (TLS, remote write) | `prometheusremotewrite` exporter | Present in distro |

### Gaps — no stock source, handled by the bootstrap script

- **Stable identity** — `instance_id` (UUIDv4, first-boot persistence), `machine_id`,
  `product_uuid`.
- **Capacity** — `cpu_sockets` (confirmed: zero socket references in the hostmetrics cpu
  scraper), `is_virtual_machine`.
- **RHEL version** — `rhel_major/minor/full_version` matched to `/etc/redhat-release` + RPM
  NVR (system detector's `os.version` is insufficient).
- **Deployment model** — `deployment_model ∈ {rpm,image}` via bootc/rpm-ostree indicators.
- **Entitlement** — `installed_products`, `subscription_manager_id`, org ID.
- **Commercial tagging** — `delivery_channel`.
- **Payload framing** — `schema_version`, `collection_mode`, `collection_time`,
  `interval_start`/`interval_end`, assembled and validated by a shared JSON Schema.

Roughly 6 of ~26 payload fields come from stock components; the rest are the script's job.

## Key design decisions

### Bootstrap script over a custom Go receiver

The initial instinct was a custom Go "metering receiver" compiled into the collector.
Instead, a **bootstrap shell script** run from a systemd unit gathers the gap facts and
hands them to the collector as resource attributes (via an `OTEL_RESOURCE_ATTRIBUTES` env
file consumed by the `resource` processor). This turns ~20 "custom Go" gaps into one shell
script plus stock components — cheaper, maintainable, and naturally per-major adaptable
(which the requirements explicitly permit). The script runs as root because `product_uuid`,
`dmidecode`, and subscription-manager facts require it.

### Signal modeling: metrics with resource attributes

The MVP destination is **Observatorium (RHOB)**, which ingests **Prometheus remote write** —
metrics only. So each sample is modeled as **metrics with resource attributes**: CPU
capacity units are metric values; the fingerprint (identity, version, deployment model,
entitlement, delivery channel, cloud, arch) rides as resource attributes. OTLP-logs modeling
was considered but rejected because it does not match the remote-write transport.

### Identity: three fields, three jobs

| Field | Source | Purpose |
|---|---|---|
| `instance_id` | Persisted UUIDv4 (first boot) | Stable, account-independent identity |
| `machine_id` | `/etc/machine-id` | systemd install id; regenerated on proper clone |
| `product_uuid` | DMI system UUID | Per-VM hypervisor id; reveals replication |

**Clone detection**: a golden image clone may duplicate `instance_id`, but `product_uuid` is
unique per VM. Downstream detects replicated instances by finding identical `instance_id`
across hosts with differing `product_uuid` — satisfying the "clone must receive a new ID"
requirement even before a factory-reset regenerates the persisted id.

### Durability: split configuration, storage-conditional

The collector config splits into a **base config** (default in-memory queue) plus an
**optional persistent-storage overlay** (adds `filestorage` + persistent `sending_queue`).
At startup the bootstrap script probes for a writable persistent volume and merges the
overlay only when one exists; otherwise the collector runs in-memory. This handles ephemeral
/ read-only image-mode rootfs cleanly. The exact merge mechanism is left to implementation.

### Uptime as a gap hint

`system.uptime` (stock) is captured to help downstream reconstruct missed intervals. It
distinguishes reboot vs. no-reboot: a sample gap with uptime reset ⇒ reboot; a gap with
uptime still climbing ⇒ host up but collector missed ticks.

**Known limitation:** it cannot separate "collector crashed" from "VM paused/resumed" — a VM
paused for a day then resumed looks identical to a crashed collector (gap, no uptime reset).
That disambiguation is a downstream data-analysis concern, not something the collector
resolves.

## Resolved / non-issues

- **PQC** — solved: default in the RHEL 9/10 crypto policy; RHEL 8 does not require it. Not a
  gap.
- **Socket-pair translation / `legacy_unit_mapping_version`** — out of scope: the script
  emits raw `cpu_sockets`; any socket-pair translation is a downstream business rule.
- **Label cardinality** (`instance_id`, `cloud_instance_id` per-host) — a downstream
  data-storage concern, not a collector concern.

## Open items (pending stakeholder input)

- **Local retention duration** — ≥72h is a requirements proposal; final value pending
  platform confirmation. Configurable, must not default below the agreed minimum.
- **HBI / inventory-of-record path** — whether metering also lands in Host Based Inventory
  (via the Insights pipeline, RHIN-1997) alongside Observatorium is pending stakeholder
  input. MVP ships to Observatorium; HBI is the entitlement-reconciliation path, gated on
  RHIN-1997.
- **Authoritative clone / factory-reset procedure** — to be documented (relies on
  `product_uuid` detection + cloud-init/sysprep clearing `/etc/machine-id`).

## References

- Product spec: `.ai/spec/what/rhel-metering.md`
- Requirements brief: `14-09-2026-key-requirements.md`, `14-09-2026-brainstorming-requierements.md`
- Distro component inventory: `redhat-opentelemetry-collector/manifest.yaml`
