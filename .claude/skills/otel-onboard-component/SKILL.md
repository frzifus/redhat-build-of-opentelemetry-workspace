---
name: otel-onboard-component
description: >
  Use when adding a new collector component (receiver, processor, exporter,
  connector, extension) to, or deprecating one from, the Red Hat build of
  OpenTelemetry. Takes a Jira ticket as input, gathers context, verifies
  upstream stability, and drives the cross-repo changes: manifest, docs, spec,
  PRs, and Jira updates. For deprecation the component stays shipped and a
  deprecation-warning alert is added.
argument-hint: "TRACING-1234"
---

# Onboard or Deprecate a Collector Component

Add or deprecate a receiver, processor, exporter, connector, or extension
in the Red Hat build of OpenTelemetry. This is a cross-repo procedure
spanning multiple repositories.

## Usage

```
/otel-onboard-component TRACING-1234
```

The Jira ticket key is required. The ticket is usually a Story created from a
spec (brainstorming → `/make-jira-from-spec` → implement with this skill), but
an Epic also works — the skill reads its child stories. If no argument is
provided, ask the user for the ticket key before proceeding.

## Workflow

### Step 1: Gather Context from Jira

Use the `/jira:jira` skill to fetch the ticket. If it is an Epic, also fetch
its child stories. Extract:
- Component name and type (receiver, processor, exporter, connector, extension)
- Whether this is an onboarding or a deprecation
- Target support level (TP or GA)
- Any existing PRs linked in the issues
- Blockers or decisions noted in issue comments

When the ticket is an Epic, review all child stories to identify which steps
have already been completed (e.g. manifest.yaml PR already merged) and which
are blocked.

### Step 2: Verify Upstream Stability

Check the component's `metadata.yaml` or `README.md` in
`opentelemetry-collector-contrib` (or core) for its stability level per
signal type.

- **For TP onboarding:** the component must be at **beta** stability or
  higher upstream. If it is alpha, stop and flag this to the user.
- **For GA promotion:** the component must have been shipped as TP for at
  least one release. Check `.ai/spec/what/collector.md` to confirm it is
  already listed as TP.

Identify the exact Go module path from the upstream repo (e.g.
`github.com/open-telemetry/opentelemetry-collector-contrib/receiver/webhookeventreceiver`).

### Step 3: Present the Plan

Before doing any work, present the user with a summary:
- Which repos will be touched
- What PRs will be opened (with Jira links)
- What is already done (from Step 1)
- What is blocked or needs a decision

Wait for user approval before proceeding.

### Step 4: Make Changes

Each target repo has its own `AGENTS.md` with build and test instructions.
Read the target repo's `AGENTS.md` before making changes there.

#### a) `redhat-opentelemetry-collector`

Add the component's gomod entry to `manifest.yaml` in the appropriate
section (`receivers`, `exporters`, `processors`, `connectors`, `extensions`).
The version must match the pinned collector version in `manifest.yaml` →
`dist.version`.

Build and verify per the repo's `AGENTS.md`. The generated `_build/` source
must be committed — it is required for Konflux hermetic builds.

Update the changelog noting the new component and its support level.

#### b) `openshift-docs` (branch `standalone-otel-docs-main`)

A component is not supported until it is documented. Two files are needed:

**Module file** in `modules/` — naming convention:
`otel-<type>-<name>-<type-singular>.adoc` (e.g. `otel-receivers-kafka-receiver.adoc`).

**Assembly include** in `otel-collector/otel-collector-<type>.adoc` — add
an `include::` directive for the new module in alphabetical order.

#### c) Workspace (this repo)

Add the component to the appropriate table in `.ai/spec/what/collector.md`
with the correct support level.

### Step 5: Open PRs

Open PRs in each repo. Every PR **must** link the Jira ticket (Epic and/or
Story) in the description.

PR title format: `TRACING-XXXX: <summary>`

### Step 6: Update Jira

After PRs are opened:
- Link each PR to its corresponding Jira Story
- Transition completed Stories
- Add a comment summarizing progress (on the Epic when there is one)

## Deprecating a Component

A deprecated component is **not removed** — it keeps shipping so existing
users' pipelines don't break. Deprecation announces the upcoming removal and
gives users time to migrate. Actual removal happens in a later, separate
release.

1. **`konflux-opentelemetry`** — add a `GoogleCloudExporterDeprecationWarning`-style
   deprecation alert. Edit
   `bundle-patch/manifests/opentelemetry-operator-alerts_monitoring.coreos.com_v1_prometheusrule.yaml`
   to add a `PrometheusRule` warning that names the component and the release
   in which it will be removed.
   Reference: https://github.com/os-observability/konflux-opentelemetry/pull/1002
2. **`openshift-docs`** — add a deprecation note to the component's existing
   doc module (do not delete it).
3. **Workspace (this repo)** — mark the component as deprecated in the
   `.ai/spec/what/collector.md` table (keep the row).
4. Note the deprecation in the release notes and update the changelog.
5. Update Jira accordingly.

Leave `manifest.yaml` and the `_build/` source untouched — the component is
still built and shipped.

## Checklist

### Onboarding
- [ ] Upstream stability verified (beta+ for TP, TP for one release for GA)
- [ ] `manifest.yaml` entry added with correct version
- [ ] `_build/` regenerated and committed
- [ ] Build passes (per repo's `AGENTS.md`)
- [ ] Changelog updated
- [ ] Component row added in `.ai/spec/what/collector.md`
- [ ] AsciiDoc module created in `openshift-docs/modules/`
- [ ] Assembly file updated with `include::` directive
- [ ] All PRs link Jira Epic and Story
- [ ] Jira Stories transitioned and updated

### Deprecation
- [ ] Deprecation-warning `PrometheusRule` alert added in `konflux-opentelemetry`
- [ ] Deprecation note added to the doc module (module kept)
- [ ] Component marked deprecated in `.ai/spec/what/collector.md` (row kept)
- [ ] Release notes and changelog updated
- [ ] `manifest.yaml` and `_build/` left unchanged (component still shipped)
- [ ] Jira updated
