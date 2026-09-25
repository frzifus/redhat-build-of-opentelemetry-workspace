# Deploy Periodic Agent

Codebase navigation for the `deploy-periodic-agent` workspace tooling. It spans
two repos: this workspace (`rhobs/redhat-build-of-opentelemetry-workspace`),
which holds the runner image and the skills, and `openshift/release`, which
holds the CI step and periodic-job configuration.

Of the new tooling, only `ci/Dockerfile` exists today (existing workspace skills
predate it). Everything else is
**[PLANNED: TRACING-6824]**. Behavioral rules are authoritative in
`what/deploy-periodic-agent.md`; the full rationale is in
`docs/superpowers/specs/2026-09-18-deploy-periodic-agent-design.md`.

## Module Map

### This repo (`rhobs/redhat-build-of-opentelemetry-workspace`)

| File/Directory | Status | Responsibility |
|---|---|---|
| `ci/Dockerfile` | exists (PR #33) | Builds the runner image (Claude Code CLI + this repo's content baked at `/tmp/redhat-build-of-opentelemetry-workspace`); the image name `rhosdt-skill-agent-runner` comes from `to:` in the ci-operator config, not the Dockerfile |
| `.claude/skills/deploy-periodic-agent/SKILL.md` | [PLANNED] | The meta-skill: generator + guide that onboards a skill as a periodic job |
| `.claude/skills/rhosdt-release-notes-audit/SKILL.md` | [PLANNED] | Placeholder target skill (stub) — the first onboarding subject |
| `.claude/skills/<name>/SKILL.md` | existing | Any workspace skill; read by the agent step at runtime from the baked image |

### `openshift/release`

| File/Directory | Status | Responsibility |
|---|---|---|
| `ci-operator/step-registry/openshift-observability/skill-agent/openshift-observability-skill-agent-ref.yaml` | [PLANNED] | Default (Vertex-only) shared agent step: `from: rhosdt-skill-agent-runner`, mounts `ci-claude-code` + Vertex env, declares `AGENT_SKILL` |
| `ci-operator/step-registry/openshift-observability/skill-agent/openshift-observability-skill-agent-jira-ref.yaml` | [PLANNED] | Vertex + Jira variant: identical to the default plus the `distributed-tracing` credential; used only by skills that need Jira |
| `ci-operator/step-registry/openshift-observability/skill-agent/…-commands.sh` | [PLANNED] | De-gated variant of the qe-agent script (shared by both refs): validates `AGENT_SKILL`, reads `SKILL.md` from the baked image, runs `claude --print` |
| `ci-operator/step-registry/openshift-observability/skill-agent/OWNERS`, `README.md`, `.metadata.json` | [PLANNED] | Step metadata; OWNERS reuses the existing `openshift-observability` approvers |
| `ci-operator/config/rhobs/redhat-build-of-opentelemetry-workspace/…__periodics.yaml` | [PLANNED] | Base repo config: `zz_generated_metadata`, `images.items` building the runner from `ci/Dockerfile` (`to: rhosdt-skill-agent-runner`), one `test:` entry per onboarded skill referencing its tier's ref |
| `ci-operator/jobs/rhobs/redhat-build-of-opentelemetry-workspace/…-periodics.yaml` | [PLANNED] | Generated Prow jobs — produced by `make update`, never hand-edited |

## Data Flow

```
cron fires (per-skill periodic)
        │
        ▼
ci-operator images: builds rhosdt-skill-agent-runner from this repo @ main
        (ci/Dockerfile, context_dir: ., to: rhosdt-skill-agent-runner)
        │
        ▼
skill-agent step (de-gated, always runs) — default (Vertex-only) ref,
        or the -jira ref when the skill needs Jira
        │
        ▼
reads /tmp/.../.claude/skills/$AGENT_SKILL/SKILL.md
        │
        ▼
claude --print --dangerously-skip-permissions
        --system-prompt "$SKILL_CONTENT"   (run from a cwd without CLAUDE.md/AGENTS.md)
        │
        ▼
cost / audit / metrics artifacts (Prow, TestGrid)
```

## Key Abstractions

**Baked image as single source of truth:** the runner image copies this repo's
content at build time, so the step reads `SKILL.md` from disk with no runtime
network fetch. `ci-operator` rebuilds the image from `main` on every job run, so
skill content is always current and is never vendored into `openshift/release`.

**De-gated shared steps, split by credential tier:** two refs serve every skill,
parameterized by the `AGENT_SKILL` environment variable —
`openshift-observability-skill-agent` (Vertex only, the default) and
`openshift-observability-skill-agent-jira` (Vertex + Jira). Unlike the qe-agent
they clone, they have **no `has_test_failures` gate** — they always run. The
split exists because `ci-operator` binds `credentials:` to a ref and a `test:`
entry can only reference a ref, so credentials cannot be dropped from a single
shared step per skill; two tier refs keep Jira secrets off the default path
without a per-skill ref (the rejected Approach B).

**Two phases:** a **one-time setup** (onboard this repo + create the shared
steps, done once ever) versus **recurring per-skill onboarding** (one periodic per
skill). The meta-skill runs the setup only when it detects it is missing, then
never again.

**Generation, not hand-editing:** after editing `ci-operator/config/**`, Prow
jobs are regenerated with `make update` and validated with `make checkconfig`.
`ci-operator/jobs/**` is generated output.

## Integration Points

| Consumer | Provider | Mechanism |
|---|---|---|
| both skill-agent refs | `rhosdt-skill-agent-runner` image | `from:` in the step ref; image built from `ci/Dockerfile` |
| both skill-agent refs | Vertex AI | `ci-claude-code` credential collection → SA token at `/var/run/claude-code-service-account` |
| `…-skill-agent-jira` ref only | Jira | `distributed-tracing` credential collection → `/var/run/dt-secrets` (not mounted by the default ref) |
| periodic job | skill-agent step | `steps.test: [{ref: openshift-observability-skill-agent}]` (or `…-skill-agent-jira`) + `env.AGENT_SKILL` |
| operator/reviewer | pre-merge test | `/pj-rehearse <job-name>` PR comment |
| `openshift/release` | contributor | fork-based PR, human/OWNERS review + merge |

## Implementation Notes

- The entire capability is **[PLANNED: TRACING-6824]**; only `ci/Dockerfile` is
  on `main`. This is internal CI tooling, so GA/TP support levels do not apply.
- The skill name is validated against `^[A-Za-z0-9_-]+$`, which also prevents
  path traversal into the baked image.
- A skill must be merged to this repo's `main` **before** it is onboarded,
  because the runner image is built from `main`. `/pj-rehearse` validates the job
  wiring and the presence of the skill on `main`, not unmerged skill content.
- Per-skill periodics carry no cluster profile, workflow, or Slack reporter by
  default — only the agent step and default Prow/TestGrid reporting.
- **Credential/network gaps (follow-ups):** the placeholder skill is the only
  current consumer of the `…-skill-agent-jira` tier, and even that is not enough
  to make it functional — it also needs a GitLab token (not in the qe-agent
  credential set) and VPN / Red Hat internal-network access to reach services
  such as `gitlab.cee.redhat.com`. Both are unresolved and block the placeholder
  skill from functioning.
