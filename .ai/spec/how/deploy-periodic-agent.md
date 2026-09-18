# Deploy Periodic Agent

Codebase navigation for the `deploy-periodic-agent` workspace tooling. It spans
two repos: this workspace (`rhobs/redhat-build-of-opentelemetry-workspace`),
which holds the runner image and the skills, and `openshift/release`, which
holds the CI step and periodic-job configuration.

Only `ci/Dockerfile` exists today. Everything else is
**[PLANNED: TRACING-6824]**. Behavioral rules are authoritative in
`what/deploy-periodic-agent.md`; the full rationale is in
`docs/superpowers/specs/2026-09-18-deploy-periodic-agent-design.md`.

## Module Map

### This repo (`rhobs/redhat-build-of-opentelemetry-workspace`)

| File/Directory | Status | Responsibility |
|---|---|---|
| `ci/Dockerfile` | exists (PR #33) | Builds the `rhosdt-skill-agent-runner` image: Claude Code CLI + this repo's content baked at `/tmp/redhat-build-of-opentelemetry-workspace` |
| `.claude/skills/deploy-periodic-agent/SKILL.md` | [PLANNED] | The meta-skill: generator + guide that onboards a skill as a periodic job |
| `.claude/skills/rhosdt-release-notes-audit/SKILL.md` | [PLANNED] | Placeholder target skill (stub) — the first onboarding subject |
| `.claude/skills/<name>/SKILL.md` | existing | Any workspace skill; read by the agent step at runtime from the baked image |

### `openshift/release`

| File/Directory | Status | Responsibility |
|---|---|---|
| `ci-operator/step-registry/openshift-observability/skill-agent/…-ref.yaml` | [PLANNED] | The shared agent step: `from: rhosdt-skill-agent-runner`, reused qe-agent credentials + Vertex env, declares `AGENT_SKILL` |
| `ci-operator/step-registry/openshift-observability/skill-agent/…-commands.sh` | [PLANNED] | De-gated variant of the qe-agent script: validates `AGENT_SKILL`, reads `SKILL.md` from the baked image, runs `claude --print` |
| `ci-operator/step-registry/openshift-observability/skill-agent/OWNERS`, `README.md`, `.metadata.json` | [PLANNED] | Step metadata; OWNERS reuses the existing `openshift-observability` approvers |
| `ci-operator/config/rhobs/redhat-build-of-opentelemetry-workspace/…__periodics.yaml` | [PLANNED] | Base repo config: `zz_generated_metadata`, `images.items` building the runner from `ci/Dockerfile`, one `test:` entry per onboarded skill |
| `ci-operator/jobs/rhobs/redhat-build-of-opentelemetry-workspace/…-periodics.yaml` | [PLANNED] | Generated Prow jobs — produced by `make update`, never hand-edited |

## Data Flow

```
this repo @ main ──(ci-operator images:)──> rhosdt-skill-agent-runner image
                                             (ci/Dockerfile, context_dir: .)
                                                        │
cron fires (per-skill periodic) ───────────────────────┤
                                                        ▼
                                   skill-agent step (de-gated, always runs)
                                                        │
                          reads /tmp/.../.claude/skills/$AGENT_SKILL/SKILL.md
                                                        │
                                                        ▼
                     claude --print --dangerously-skip-permissions
                            --system-prompt "$SKILL_CONTENT"
                                                        │
                                                        ▼
                          cost / audit / metrics artifacts (Prow, TestGrid)
```

## Key Abstractions

**Baked image as single source of truth:** the runner image copies this repo's
content at build time, so the step reads `SKILL.md` from disk with no runtime
network fetch. `ci-operator` rebuilds the image from `main` on every job run, so
skill content is always current and is never vendored into `openshift/release`.

**De-gated shared step:** one step, `openshift-observability-skill-agent`, serves
every skill, parameterized by the `AGENT_SKILL` environment variable. Unlike the
qe-agent it clones, it has **no `has_test_failures` gate** — it always runs.

**Two phases:** a **one-time setup** (onboard this repo + create the shared step,
done once ever) versus **recurring per-skill onboarding** (one periodic per
skill). The meta-skill runs the setup only when it detects it is missing, then
never again.

**Generation, not hand-editing:** after editing `ci-operator/config/**`, Prow
jobs are regenerated with `make update` and validated with `make checkconfig`.
`ci-operator/jobs/**` is generated output.

## Integration Points

| Consumer | Provider | Mechanism |
|---|---|---|
| skill-agent step | `rhosdt-skill-agent-runner` image | `from:` in the step ref; image built from `ci/Dockerfile` |
| skill-agent step | Vertex AI | `ci-claude-code` credential collection → SA token at `/var/run/claude-code-service-account` |
| skill-agent step | Jira | `distributed-tracing` credential collection → `/var/run/dt-secrets` |
| periodic job | skill-agent step | `steps.test: [{ref: openshift-observability-skill-agent}]` + `env.AGENT_SKILL` |
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
- **Credential/network gaps (follow-ups):** the placeholder skill needs a GitLab
  token (not in the qe-agent credential set) and VPN / Red Hat internal-network
  access to reach services such as `gitlab.cee.redhat.com`. Both are unresolved
  and block the placeholder skill from functioning.
