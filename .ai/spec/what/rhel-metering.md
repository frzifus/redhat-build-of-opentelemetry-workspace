# RHEL Metering via the OpenTelemetry Collector

RHEL metering delivers first-party, billing-grade measurement of RHEL deployments — CPU
capacity units plus a software/entitlement fingerprint — emitted on a recurring cadence
from RHEL hosts, independent of hyperscaler reporting and independent of whether the host
is registered to a Red Hat account.

This spec answers the feature's core question: *can the OpenTelemetry Collector — which
already ships as a RHEL package — collect, package, and transmit this metering data instead
of a bespoke metering agent?* The answer is **yes, as the vehicle** — the collector supplies
transport, durability, scheduling, and a subset of the host facts through stock components —
**but not with stock components alone.** The metering-specific facts (stable identity,
socket capacity, RHEL product/subscription fingerprint, deployment-model, delivery-channel)
are gathered by a **bootstrap shell script** run from a systemd unit and handed to the
collector as **resource attributes**. No collector fork is required.

The entire feature is **[PLANNED: TRACING-6702] TP (Technology Preview)**.

## Scope

**MVP (this phase):**
- Streamed / connected **cloud images** — **Console** (hyperscaler is seller of record) and
  **Marketplace** (Red Hat is seller of record).
- **RHEL 8, 9, 10**, in both **RPM** and **image-mode** (bootc/rpm-ostree) deployments.
- Delivered as updated cloud images or a separately downloadable RPM package set.

**Designed-for but not MVP** (the design must not preclude these): on-premise footprints;
disconnected / batched (offline) collection; Satellite-mediated forward.

**Out of scope:** RHEL 7 (and RHEL 6) — not supported, extended, or backported; downstream
daily/monthly/yearly aggregation (platform scope); billing/invoice/marketplace integration
(consumers of this telemetry); UI/portal presentation.

Per-major implementation variance is permitted (separate packages, collectors, or bootstrap
scripts per RHEL major) provided every implementation emits payloads that validate against
the same `schema_version`.

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

The bootstrap script closes every gap that stock components cannot; the collector reuses
stock components for everything else and ships the result to Observatorium (RHOB) via
Prometheus remote write.

## Behavioral Rules

### Signal modeling

1. **[PLANNED: TRACING-6702] TP**: Each sample is modeled as **metrics with resource
   attributes**. CPU capacity units (`logical_cpus`, `physical_cores`) are metric values;
   the metering fingerprint (identity, RHEL version, deployment model, entitlement, delivery
   channel, cloud metadata, arch) rides as resource attributes. This modeling follows the
   MVP transport: Observatorium ingests **Prometheus remote write**, which is metrics-only.
2. Capacity units that stock components do not expose (e.g. `cpu_sockets`) may be emitted
   either as metric values or as resource-attribute labels; the mechanism is left to
   implementation.

### Instance identity

3. On first boot the bootstrap script generates and persists a random **UUIDv4
   `instance_id`**, independent of customer account, subscription state, and
   `delivery_channel`. It is present and non-null in every emitted sample and is never gated
   on Subscription Manager registration, Insights enrollment, or Red Hat account linkage.
   (REQ-SC-05, -06; Cross-cutting — Instance identity)
4. `instance_id` is stable across reboots; it is regenerated only by a documented
   factory-reset procedure.
5. The identity set carries three distinct fields, each with its own job:

   | Field | Source | Purpose |
   |---|---|---|
   | `instance_id` | Persisted UUIDv4 (script, first boot) | Stable, account-independent identity |
   | `machine_id` | `/etc/machine-id` | systemd install id; regenerated on a proper clone (cloud-init/sysprep) |
   | `product_uuid` | DMI system UUID (`/sys/class/dmi/id/product_uuid`) | Per-VM hypervisor id; reveals replication |

6. **Clone / replication detection**: a clone built from a golden image may carry a
   duplicated `instance_id`, but its `product_uuid` is unique per VM. Downstream reconciles
   the "clone must receive a new ID" requirement by comparing `instance_id` against
   `product_uuid`: identical `instance_id` across hosts with differing `product_uuid`
   indicates replicated instances. Reading `product_uuid` requires root, reinforcing the
   root/systemd-unit requirement for the bootstrap script.

### Facts gathered by stock components

7. `hostmetrics` cpu scraper provides `logical_cpus` (`system.cpu.logical.count`, on by
   default) and `physical_cores` (`system.cpu.physical.count`, opt-in — must be enabled).
   (REQ-SC-02)
8. `resourcedetection` provides CPU architecture (`host.arch`, system detector — must be
   enabled), and cloud metadata (`cloud.provider`, `cloud.region`, and the cloud instance id
   as `host.id`) via the ec2/azure/gcp detectors. Cloud fields are null (not omitted) on
   footprints where no detector matches. (REQ-SC-02, -05)
9. `hostmetrics` system scraper provides `system.uptime` (on by default), captured to help
   downstream reconstruct missed intervals. See the uptime hint under Notes.

### Facts gathered by the bootstrap script

10. **[PLANNED: TRACING-6702] TP**: The bootstrap script gathers everything stock components
    cannot and exposes it to the collector as resource attributes via an
    `OTEL_RESOURCE_ATTRIBUTES` env file:
    - `cpu_sockets` (from `lscpu` / `dmidecode`) — socket count. (REQ-SC-03)
    - `rhel_major`, `rhel_minor`, `rhel_full_version` — matched to `/etc/redhat-release` and
      the `redhat-release` RPM NVR. (REQ-SC-01)
    - `deployment_model ∈ {rpm, image}` — image mode detected via bootc/rpm-ostree indicators
      (`/run/ostree-booted`, `rpm-ostree status`, or an image-mode metapackage); RPM
      otherwise. (REQ-SC-07)
    - `installed_products` — Red Hat product fingerprint equivalent to the Subscription
      Manager system profile (product IDs/names, architecture, base OS version).
      (Cross-cutting — Subscription/entitlement parity)
    - `subscription_manager_id` and organization ID — populated when the host is registered;
      null otherwise. (Cross-cutting — Subscription/entitlement parity)
    - `delivery_channel ∈ {console, marketplace, on_prem, other}` — the commercial model the
      instance was provisioned under. (REQ-SC-05)
11. Mandatory fields are present in both RPM and image-mode deployments and are identical
    across Console and Marketplace images; only `delivery_channel` and seller-of-record
    metadata differ. (REQ-SC-05, -07)
12. A valid payload with a non-null `instance_id` is produced **without any Red Hat account
    association** — on Console, Marketplace, and on-prem footprints. (REQ-SC-05, -06)

### Cadence and durability

13. Default cadence is **≥ 1 sample/hour, defaulting to 10/hour (every 6 minutes)**,
    configurable. Samples carry `interval_start` and `interval_end` (UTC, RFC3339).
    (REQ-SC-04)
14. A systemd timer/service enables the collector in the default connected profile.
15. Durability uses stock components: the `batch` processor batches in memory before export;
    the exporter's persistent `sending_queue` (backed by the `filestorage` extension) plus
    `retry_on_failure` persist un-sent requests across restart and disconnect, so no sample
    is lost while upload is unavailable.
16. **Split configuration for storage**: the collector config is split into a **base config**
    (default in-memory queue) plus an **optional persistent-storage overlay** that adds the
    `filestorage` extension and persistent `sending_queue`. At startup the bootstrap script
    probes for a writable persistent volume and merges the overlay **only when one exists**;
    otherwise the collector runs in-memory. Mechanism (config merge) is left to
    implementation.

### Transport

17. **[PLANNED: TRACING-6702] TP**: The MVP transport is **Prometheus remote write to
    Observatorium (RHOB)** via the `prometheusremotewrite` exporter over TLS 1.2+.
18. Transport must not break when the system crypto policy is FIPS. PQC is the default in the
    RHEL 9/10 crypto policy and is used for key exchange where applicable; RHEL 8 does not
    require PQC. (Cross-cutting — Transport & security)

### Opt-out

19. An administrator can disable collection via supported config; disabling stops new samples
    within one interval. (REQ-SC-06)

## Payload / Resource-Attribute Contract

A single versioned schema (`schema_version`) with an identical mandatory field set and types
across RHEL 8/9/10 and across RPM and image-mode hosts. Schema conformance is enforced by a
shared JSON Schema checked in CI for each major's collector. (REQ-SC-01, -07)

| Field | Source | Notes |
|---|---|---|
| `schema_version` | script | Single versioned contract |
| `collection_mode` | script | `streamed` (MVP) |
| `delivery_channel` | script | `console` \| `marketplace` \| `on_prem` \| `other` |
| `instance_id` | script (persisted) | UUIDv4, non-null always |
| `machine_id` | script | `/etc/machine-id` |
| `product_uuid` | script | DMI system UUID; replication signal |
| `collection_time`, `interval_start`, `interval_end` | script / pipeline | UTC RFC3339 |
| `rhel_major`, `rhel_minor`, `rhel_full_version` | script | Matches `/etc/redhat-release` + RPM NVR |
| `deployment_model` | script | `rpm` \| `image` |
| `logical_cpus` | hostmetrics | `system.cpu.logical.count` |
| `physical_cores` | hostmetrics | `system.cpu.physical.count` (opt-in) |
| `cpu_sockets` | script | `lscpu` / `dmidecode` |
| `architecture` | resourcedetection | `host.arch` (system detector) |
| `is_virtual_machine` | script | virtual vs physical |
| `system_uptime` | hostmetrics | `system.uptime`; see Notes |
| `cloud_provider`, `cloud_region`, `cloud_instance_id` | resourcedetection | ec2/azure/gcp; null on-prem |
| `installed_products` | script | Subscription Manager system profile |
| `subscription_manager_id`, org id | script | Null when unregistered |

## Component Mapping (feasibility summary)

| Requirement / field | Satisfied by | Custom work |
|---|---|---|
| `logical_cpus`, `physical_cores` | `hostmetrics` cpu scraper | Enable `physical.count` |
| `architecture`, cloud metadata | `resourcedetection` | Enable system detector attrs |
| `system_uptime` | `hostmetrics` system scraper | None |
| Cadence (≥1/hr, 10/hr) | `collection_interval` | Config |
| Durability, no loss on disconnect | `filestorage` + persistent `sending_queue` + `retry_on_failure` | Split-config selection |
| Batching | `batch` processor | Config |
| Transport (TLS, remote write) | `prometheusremotewrite` | Endpoint/auth config |
| Identity (`instance_id`, `machine_id`, `product_uuid`) | — | Bootstrap script |
| `cpu_sockets` | — (no stock source) | Bootstrap script |
| RHEL NVR, `deployment_model` | — | Bootstrap script |
| Entitlement fingerprint, `subscription_manager_id` | — | Bootstrap script |
| `delivery_channel`, payload framing (`schema_version`, intervals) | — | Bootstrap script + JSON Schema |

## Configuration Surface

### Bootstrap script / systemd unit

| Concern | Detail |
|---|---|
| Privilege | Runs as root (or with capabilities) — `product_uuid`, `dmidecode`, subscription-manager facts require it |
| Output | `OTEL_RESOURCE_ATTRIBUTES` env file consumed by the collector's `resource` processor |
| Storage probe | Selects persistent vs in-memory collector config profile at startup |
| Identity persistence | `instance_id` written to a persistent path on first boot |

### Collector (sketch)

```yaml
receivers:
  hostmetrics:
    collection_interval: 6m   # 10 samples/hour, configurable
    scrapers:
      cpu:
        metrics:
          system.cpu.physical.count: { enabled: true }
      system:                 # system.uptime enabled by default
processors:
  resourcedetection:
    detectors: [system, ec2, azure, gcp]
    system:
      resource_attributes:
        host.arch: { enabled: true }
  resource: {}               # injects OTEL_RESOURCE_ATTRIBUTES from the bootstrap script
  batch: {}
exporters:
  prometheusremotewrite:
    endpoint: <observatorium-remote-write-url>
    tls: {}
# Persistent-storage overlay (merged only when a persistent volume exists):
# extensions: { file_storage: {} }
# exporters.prometheusremotewrite.sending_queue: { enabled: true, storage: file_storage }
```

## Observability

The collector's own internal telemetry (queue length, export success/failure, retry counts)
covers pipeline health. Downstream verifies delivery by confirming the sample reaches
Observatorium/HBI within the agreed SLA. (Cross-cutting — Backend delivery)

## Constraints

1. All transport crypto must be FIPS-compliant; PQC is default on RHEL 9/10, not required on
   RHEL 8.
2. Per-major implementation variance is permitted; all implementations must emit payloads
   that validate against the same `schema_version` (enforced in CI).
3. No hard dependency on a single cloud SDK in the core collector; cloud metadata is
   best-effort via `resourcedetection` detectors.
4. The bootstrap script is the only custom component; the collector itself is unmodified
   stock (no fork).

## Notes / Hints

- **Uptime hint**: `system.uptime` distinguishes **reboot vs. no-reboot** — a sample gap with
  uptime reset to a small value means the host rebooted; a gap with uptime still climbing
  means the host stayed up but the collector missed ticks. It **cannot** separate "collector
  crashed" from "VM paused/resumed": a VM paused for a day then resumed looks identical to a
  crashed collector (gap, no uptime reset). That disambiguation is a downstream
  data-analysis concern, not something the collector resolves.
- **Cardinality note**: per-host label cardinality (`instance_id`, `cloud_instance_id` are
  unique per host by design) is a downstream data-storage concern, not a collector concern.

## Open Items (pending stakeholder input)

- **Local retention duration** — the ≥72h target is a requirements proposal; final value
  pending platform/stakeholder confirmation. Retention duration is configurable and must not
  default below the agreed minimum.
- **HBI / inventory-of-record path** — whether metering also lands in Host Based Inventory
  (via the Insights pipeline, RHIN-1997) in addition to Observatorium is pending stakeholder
  input. MVP ships to Observatorium via remote write; HBI is the inventory-of-record path for
  entitlement reconciliation, gated on RHIN-1997.
- **Clone → new instance_id** — expected behavior documented via `product_uuid` replication
  detection (rule 6) and cloud-init/sysprep clearing `/etc/machine-id`; the authoritative
  factory-reset/clone procedure is to be documented.
