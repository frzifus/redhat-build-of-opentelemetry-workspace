# Design: `deploy-periodic-agent` — deploy a workspace skill as an agentic periodic job

**Date:** 2026-09-18
**Status:** Design — pending review
**Repo:** `rhobs/redhat-build-of-opentelemetry-workspace` (this workspace)

## Summary

Add a meta-skill, `deploy-periodic-agent`, that takes an existing
`.claude/skills/<name>` skill from this repo and walks a user through deploying
it as an **agentic periodic (Prow) job** in `openshift/release`. The periodic
runs the **Claude Code CLI** with the chosen skill loaded as its system prompt,
on a schedule the user picks (e.g. daily), and does nothing else.

The design reuses the existing Distributed-Tracing QE agent machinery
(`openshift-observability-qe-agent`) — its runner image, Google Vertex AI
backend, and credentials — but with a **new, de-gated agent step** that runs
unconditionally (the existing qe-agent only fires after a test failure) and
reads the skill from a purpose-built runner image that bakes in this repo's
content (built from `ci/Dockerfile`), so there is no runtime network fetch.

**Two distinct phases, with very different frequencies:**

- **One-time setup (done once, ever):** onboard this repo into
  `openshift/release` and create the shared, generalized agent step. After this
  lands, it is never repeated.
- **Per-skill onboarding (the recurring job — what the skill is *for*):** for
  each `.claude/skills/<name>` you want to run on a schedule, generate a
  periodic job that invokes the shared agent step with that skill. This is the
  common path and the reason the skill exists.

The `deploy-periodic-agent` skill is **primarily about the recurring per-skill
onboarding.** It only performs the one-time setup when it detects the setup is
missing, then never again.

## Goals

- Make **onboarding a new skill to run on a schedule** the fast, repeatable
  common case — one command per skill, no CI hand-authoring.
- One command in this repo to deploy any `.claude/skills/<name>` as a scheduled
  agentic job in `openshift/release`.
- Reuse the proven qe-agent Vertex AI config and credentials.
- Keep a single source of truth for skills: the runner image is built from this
  repo's `ci/Dockerfile` (checked out at `main`), so `SKILL.md` is baked into the
  image and skills are not vendored into `openshift/release`.
- Stay within existing DT-QE / openshift-observability conventions and OWNERS.

## Non-goals

- Making the placeholder skill functional. It is a stub (see below).
- Building a per-skill step-registry ref (Approach B, rejected) or reusing the
  failure-gated qe-agent step unchanged (Approach C, rejected).
- Solving GitLab and VPN access now — these are documented follow-ups.

## Chosen approach (A): one generalized agent step + a thin meta-skill

Add **one** new step-registry ref to `openshift/release` that runs Claude
unconditionally against a skill named by `AGENT_SKILL`, read from the baked
runner image built from this repo.
The meta-skill in this repo is a generator + guide: for any chosen skill it
produces a periodic-job config that runs only that step, regenerates the Prow
jobs, and walks the user through pre-merge testing and the PR.

Rejected alternatives:
- **B — a dedicated step per skill:** more boilerplate and OWNERS churn in
  `openshift/release` for every skill.
- **C — reuse `openshift-observability-qe-agent` unchanged:** it is gated on a
  preceding test's failure and fetches skills from `release`'s own
  `resources/skills`, so it cannot run a workspace skill standalone on a
  schedule.

## Artifact map

### In `openshift/release`

1. **New agent step** — under the existing observability namespace:
   `ci-operator/step-registry/openshift-observability/skill-agent/`
   → ref name `openshift-observability-skill-agent` (proposed; see Open questions).
   - `openshift-observability-skill-agent-ref.yaml`: `from: rhosdt-skill-agent-runner`
     (the image built from this repo's `ci/Dockerfile`); `credentials:` copied
     from the qe-agent (`ci-claude-code` → Vertex SA at
     `/var/run/claude-code-service-account`, `distributed-tracing` → Jira secrets
     at `/var/run/dt-secrets`); Vertex env (`CLAUDE_CODE_USE_VERTEX=1`,
     `CLOUD_ML_REGION`, `ANTHROPIC_VERTEX_PROJECT_ID`, `CLAUDE_MODEL`,
     `STEP_TIMEOUT_MINUTES`); and `env:` declaring `AGENT_SKILL`.
   - `openshift-observability-skill-agent-commands.sh`: a **de-gated** variant
     of the qe-agent script. Validates `AGENT_SKILL` against `^[A-Za-z0-9_-]+$`
     (also prevents path traversal), reads the skill from the baked image at
     `/tmp/redhat-build-of-opentelemetry-workspace/.claude/skills/$AGENT_SKILL/SKILL.md`
     (no `curl`, no size-cap/`--max-redirs` handling — the content is local),
     then runs `claude --print --dangerously-skip-permissions
     --system-prompt "$SKILL_CONTENT" ...` and emits the same cost/audit/metrics
     artifacts. **No `has_test_failures` gate** — it always runs.
   - `OWNERS`: inherits / reuses the existing `openshift-observability` step
     OWNERS (no new approver to invent).
   - `README.md`, `.metadata.json`.

2. **Repo onboarding config** — base config so ci-operator knows this repo:
   `ci-operator/config/rhobs/redhat-build-of-opentelemetry-workspace/…__periodics.yaml`
   with `zz_generated_metadata` (org `rhobs`, repo
   `redhat-build-of-opentelemetry-workspace`, branch `main`), a minimal
   `resources:` skeleton, and an `images.items` entry that builds the runner from
   this repo's `ci/Dockerfile`:
   `images.items: [{context_dir: ., dockerfile_path: ci/Dockerfile, to: rhosdt-skill-agent-runner}]`.

3. **One periodic per deployed skill** — a `test:` entry in the config above:
   - `as: <skill>-agent`
   - `cron: <user value>` (schedule-agnostic skill; the user picks the cadence)
   - `steps.test: [{ref: openshift-observability-skill-agent}]` — **only** the
     agent step, nothing else
   - `steps.env.AGENT_SKILL: <name>`
   - **no** cluster profile / workflow by default (added only if a future skill
     needs a cluster)
   - **no** Slack reporter (default Prow/TestGrid reporting only)
   - regenerated into `ci-operator/jobs/rhobs/…/*-periodics.yaml` via
     `make update`.

### In this workspace repo

4. **The meta-skill** — `.claude/skills/deploy-periodic-agent/SKILL.md`
   (see "The meta-skill" below).

5. **Placeholder target skill** — `.claude/skills/rhosdt-release-notes-audit/SKILL.md`
   (proposed name; see "Placeholder skill" below).

6. **Runner image Dockerfile** — `ci/Dockerfile` (already on `main` via PR #33):
   bakes the Claude Code CLI plus this repo's content (including
   `.claude/skills/`) into the `rhosdt-skill-agent-runner` image that the agent
   step runs on.

## One-time setup (done once, ever)

This is **not** the skill's recurring job. It establishes the shared
infrastructure the per-skill onboarding depends on, and after it merges it is
never repeated. The skill runs it only when it detects the setup is missing;
otherwise it skips straight to per-skill onboarding.

1. **Onboard the repo** — create the base
   `ci-operator/config/rhobs/redhat-build-of-opentelemetry-workspace/…` config.
2. **Create the shared agent step** under
   `ci-operator/step-registry/openshift-observability/skill-agent/`, cloning the
   qe-agent's `ref.yaml` (credentials, Vertex env, model, timeout) with the
   de-gated `commands.sh`. This one step serves every skill, parameterized by
   `AGENT_SKILL`.
3. **Confirm prerequisites** — the `ci-claude-code` credential collection and
   `distributed-tracing` rover group are org-agnostic collections and should be
   usable from the new `rhobs/...` config; the skill flags it if
   `make checkconfig` complains.
4. **Regenerate & validate** — `make update` then `make checkconfig`; surface
   errors verbatim, stop on failure.

Detection is idempotent: once the config and the step exist, every later run
goes directly to per-skill onboarding.

## Per-skill onboarding (the recurring job — what the skill is for)

Inputs: target `.claude/skills/<name>` and a cron schedule.

1. **Validate the source skill** — confirm `.claude/skills/<name>/SKILL.md`
   exists on `main` and `<name>` matches `^[A-Za-z0-9_-]+$` (safe as
   `AGENT_SKILL` and as a path segment into the baked image).
2. **Generate the periodic** in the onboarded repo's config (fields above).
3. **Regenerate** — `make update` then `make checkconfig`; stop on failure.
4. **Hand off to testing** — print the generated Prow job name and the exact
   `/pj-rehearse <job-name>` comment.

## The meta-skill

- **Path:** `.claude/skills/deploy-periodic-agent/SKILL.md`
- **Frontmatter:** `name: deploy-periodic-agent`; `description`;
  `argument-hint: 'skill: name of a .claude/skills/<name> to deploy, cron: schedule (e.g. "0 6 * * *")'`
- **Body:** the recurring per-skill flow is the focus — validate source skill →
  generate periodic → `make update`/`make checkconfig` → test & PR. A short
  guard at the top runs the one-time setup only if it is missing, then never
  again.
- **Dependencies:** `release` repo cloned in the workspace (already in
  `make clone-repos` and `.gitignore`), `gh`, `make`; the
  `openshift-observability` step OWNERS as review contact.

## Credentials & services

Reused from the qe-agent (sufficient for the step to run and to reach Jira):

- **Vertex AI** — `ci-claude-code` collection → SA token at
  `/var/run/claude-code-service-account/google-token`.
- **Jira** — `distributed-tracing` collection → `/var/run/dt-secrets`.

**Known gaps — required by the placeholder skill, not yet solved (follow-ups):**

1. **GitLab token** — reading/writing a GitLab release object needs a GitLab
   PAT that is not in the qe-agent credential set. A new credential collection /
   mount must be added when the skill is built.
2. **VPN / Red Hat internal-network access** — internal services such as
   `gitlab.cee.redhat.com` are not reachable from a default Prow job. The agent
   step will need internal-network access (mechanism TBD — dedicated network
   config or an internal-capable cluster profile). This must be resolved before
   the placeholder skill can function.

Both gaps are out of scope for wiring the placeholder; they are documented so
they are not silent.

## Placeholder skill

`.claude/skills/rhosdt-release-notes-audit/SKILL.md` (proposed name) — a **stub**,
clearly marked `Status: placeholder — not yet implemented` in its body and
frontmatter. Intended future behavior:

> For a given RHOSDT release: check the Jira features/bugs associated with that
> release; verify release notes exist on the corresponding GitLab release
> object; if they do not, move those tickets back to *In Progress*.

It exists so `deploy-periodic-agent` has a concrete first target to wire up. It
will be developed in a future session and needs the GitLab + VPN follow-ups
above.

## Testing & PR workflow

The meta-skill merges nothing. After generating files and
`make update`/`make checkconfig`, it:

1. Commits on a branch and pushes to the user's **fork** of `openshift/release`
   (fork-based workflow).
2. Opens a PR against `openshift/release`.
3. Prints the exact `/pj-rehearse periodic-ci-…` comment to run the job
   pre-merge.
4. Tells the user a human/OWNERS review + merge is required. Because the runner
   image is built from this repo's `main`, the skill must already be merged to
   `main` before onboarding: a `/pj-rehearse` run validates the job wiring and
   the presence of the skill on `main`, not unmerged skill content.

Errors stop the flow and are surfaced verbatim.

## Open questions / follow-ups

- Exact new step ref name (`openshift-observability-skill-agent` proposed).
- GitLab credential collection + mount path.
- VPN / internal-network mechanism for the agent step.
