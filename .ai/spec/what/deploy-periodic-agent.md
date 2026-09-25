# Deploy Periodic Agent

`deploy-periodic-agent` is workspace tooling — a skill in this repo
(`.claude/skills/deploy-periodic-agent/`) that onboards any workspace skill
(`.claude/skills/<name>`) to run **on a schedule as an agentic Prow periodic
job** in `openshift/release`. The periodic runs the Claude Code CLI with the
chosen skill loaded as its system prompt, and nothing else.

It reuses the Distributed-Tracing QE agent's Google Vertex AI backend and
credentials via **de-gated** shared agent steps that run unconditionally (the
qe-agent only fires after a test failure). Because `ci-operator` binds
`credentials:` to a step ref and a `test:` entry can only reference a ref, the
credential tiers are split across **two** refs: a **Vertex-only default** and a
**Vertex + Jira** variant used only by skills that need Jira. The skill is
**baked into a purpose-built runner image** (`ci/Dockerfile`) rather than fetched
over the network: `ci-operator` checks out this repo at `main` and rebuilds the
image on every job run, so the skill content is a single source of truth and
always current, with no runtime network dependency.

The entire capability is **[PLANNED: TRACING-6824]**. This is internal
workspace/CI tooling, not a customer-facing product feature, so the GA/TP
support levels do not apply.

## Scope

**In scope:**
- One-time onboarding of `rhobs/redhat-build-of-opentelemetry-workspace` into
  `openshift/release`, plus one shared, generalized agent step.
- Recurring per-skill onboarding: generate a periodic job per skill to run on a
  user-chosen cadence.
- A placeholder target skill, `rhosdt-release-notes-audit`, as the first
  onboarding subject.

**Out of scope:**
- Making the placeholder skill functional (it is a stub).
- A per-skill step-registry ref, or reusing the failure-gated qe-agent step
  unchanged.
- Solving GitLab and VPN/internal-network access (documented as follow-ups).

## Behavioral Rules

### Phases

1. **[PLANNED: TRACING-6824]**: The tooling has two phases with different
   frequencies: a **one-time setup** (done once, ever) and a **recurring
   per-skill onboarding**. The skill is primarily about the recurring
   onboarding; it performs the one-time setup only when it detects the setup is
   missing, then never again.

2. **[PLANNED: TRACING-6824]**: One-time setup onboards this repo into
   `openshift/release` (base `ci-operator/config/rhobs/redhat-build-of-opentelemetry-workspace/…`)
   and creates **two** shared agent steps split by credential tier, under
   `ci-operator/step-registry/openshift-observability/skill-agent/`:
   `openshift-observability-skill-agent` (Vertex only, the default) and
   `openshift-observability-skill-agent-jira` (Vertex + Jira).

### Shared agent steps

3. **[PLANNED: TRACING-6824]**: The shared steps are **de-gated** — they run
   Claude unconditionally, with no `has_test_failures` check. Both refs are
   otherwise identical and are parameterized by the `AGENT_SKILL` environment
   variable; they differ only in the credentials they mount (rule 5). A skill is
   wired to exactly one of them at onboarding, defaulting to the Vertex-only
   `openshift-observability-skill-agent`.

4. **[PLANNED: TRACING-6824]**: The step **reads `SKILL.md` from the baked
   image** at
   `/tmp/redhat-build-of-opentelemetry-workspace/.claude/skills/$AGENT_SKILL/SKILL.md`.
   The image is built by `ci-operator`'s `images:` phase from this repo's
   `ci/Dockerfile` with `context_dir: .`, checked out at `main` and rebuilt every
   run. Skills are the single source of truth in this repo and are **not
   vendored** into `openshift/release`; there is no runtime network fetch. To
   keep the chosen skill as the system prompt **and nothing else**, the step runs
   `claude` from a working directory that does **not** contain this repo's
   `CLAUDE.md`/`AGENTS.md` (the baked image sets `WORKDIR` to the repo root, which
   `claude --print` would otherwise auto-load), so no repo instructions are added
   to the skill's context.

5. **[PLANNED: TRACING-6824]**: Both steps run on the purpose-built runner image
   (`ci/Dockerfile`, built as `rhosdt-skill-agent-runner`) and reuse the
   qe-agent's Vertex AI configuration. Because `ci-operator` binds `credentials:`
   to the ref and a `test:` entry can only reference a ref (it cannot drop
   credentials from a shared ref), the two tiers are separate refs rather than
   one shared step:
   - `openshift-observability-skill-agent` (**default**): mounts only
     `ci-claude-code` (Vertex service account). No Jira secrets.
   - `openshift-observability-skill-agent-jira`: mounts `ci-claude-code` **and**
     `distributed-tracing` (Jira secrets), for the skills that need Jira.

   Jira secrets are therefore **not** granted to every onboarded skill by
   default. This is not the rejected per-skill ref (Approach B): there are two
   fixed refs by tier, not one per skill.

### Per-skill onboarding

6. **[PLANNED: TRACING-6824]**: Onboarding a skill generates a periodic `test:`
   entry that runs **only** one shared agent step (no cluster profile by
   default, no test suite), with `AGENT_SKILL` set to the skill name and a
   user-chosen `cron`. The step defaults to the Vertex-only
   `openshift-observability-skill-agent`; a skill that needs Jira is wired to
   `openshift-observability-skill-agent-jira` instead. The deployed skill is
   schedule-agnostic; the cadence is chosen at onboarding time.

7. **[PLANNED: TRACING-6824]**: The skill name must match `^[A-Za-z0-9_-]+$`
   (which also prevents path traversal into the baked image) and correspond to
   an existing `.claude/skills/<name>/SKILL.md` in this repo.

8. **[PLANNED: TRACING-6824]**: After editing `ci-operator/config/**`, the
   generated Prow jobs are produced with `make update` and validated with
   `make checkconfig`; `ci-operator/jobs/**` is never hand-edited. Errors stop
   the flow and are surfaced verbatim.

### Delivery

9. **[PLANNED: TRACING-6824]**: Changes are delivered as a fork-based PR to
   `openshift/release`, testable pre-merge via a `/pj-rehearse <job-name>` PR
   comment, and require human/OWNERS review and merge.

10. **[PLANNED: TRACING-6824]**: A skill must be merged to this repo's `main`
    **before** it is onboarded, because the runner image is built from `main`.
    A `/pj-rehearse` run therefore validates only the job wiring and the presence
    of the skill on `main` — it does not validate unmerged skill content.

## Constraints

- **Credential gaps (follow-ups, required by the placeholder skill):** reading
  or writing a GitLab release object needs a **GitLab token** not in the
  qe-agent credential set; reaching internal services such as
  `gitlab.cee.redhat.com` needs **VPN / Red Hat internal-network access** not
  available to a default Prow job. Both must be resolved before the placeholder
  skill can function.
- **Placeholder skill:** `rhosdt-release-notes-audit` is a stub, clearly marked
  as not yet implemented. Its intended future behavior: for a given RHOSDT
  release, check the associated Jira features/bugs, verify release notes exist on
  the corresponding GitLab release object, and move tickets back to *In
  Progress* if they do not.

## Cross-Reference

- Codebase navigation (module map, data flow, integration points):
  `how/deploy-periodic-agent.md`.
- Detailed design and rationale:
  `docs/superpowers/specs/2026-09-18-deploy-periodic-agent-design.md`.
- Tracking: **TRACING-6824** (sub-task of **TRACING-6383** — "How to automate
  release notes?").
