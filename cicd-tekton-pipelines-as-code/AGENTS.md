<!-- FOR AI AGENTS - Human readability is a side effect, not a goal -->
<!-- Managed by agent: opencode | Last updated: 2026-08-23 -->

# AGENTS.md (cicd-tekton-pipelines-as-code)

Git-triggered CI: Pipelines-as-Code (v0.50.0) + GitHub App (personal org). PaC runs the `.tekton/pipelinerun.yaml` from each repo: allow-listed repos in their `home-<team>` namespace, auto-configured repos (any repo where the app is installed + `.tekton/` exists) in `home-auto`.

## Overview
Overview of this component — see `README.md` for prose.

## Setup (deploy & teardown)

| Task | Command |
|------|---------|
| Expose | `helmfile -l stage=expose-operator apply` |
| Webhook relay (gosmee) | `helmfile -l stage=gosmee apply` — GitHub → `hook.pipelinesascode.com` → PaC controller (no domain needed) |
| Repositories CRs | `helmfile -l stage=pipelines apply` |
| GitHub App secret | `kubectl apply -f secrets/pipelines-as-code-github-secrets.yaml` (fill from example first) |

## Commands
Full command table lives in the Setup section; offline validation: `helm template <rel> ./charts/<name>` and `helm lint` via the alpine/helm container (see workspace root AGENTS.md).

## Key files

| File | Notes |
|------|-------|
| `kustomizations/upstream/pipelines-as-code-0.50.0.yaml` | upstream release manifest (refresh via documented curl in README) |
| `kustomizations/pipelines-as-code/patches/configmaps.yaml` | controller URL (`pipelines-as-code.192.168.1.200.sslip.io`) + dashboard URL + **PaC settings** (auto-configure → `home-auto`, `require-ok-to-test-sha`) |
| `charts/pipelines-as-code-repositories/values.yaml` | **3-way sync file** — `namespacePrefix: home`, namespaces `[infra, admin, service-a, auto]`, repo lists per `home-<team>` ns, `settings.policy` whitelist (`stijnklomp`) |
| `charts/expose-pipelines-as-code/` | ingress with `home-issuer` |
| `charts/gosmee-client/` | webhook relay client (ghcr.io/chmouel/gosmee:v0.32.0, pinned); forwards `hook.pipelinesascode.com` → PaC controller; reads `smee` from the secret |
| `secrets/pipelines-as-code-github-secrets.example.yaml` | GitHub App id/key/webhook + `smee` relay URL placeholders |

## Conventions & rules

- Repository CR namespaces must stay in sync with the 3-way sync (see workspace root `AGENTS.md`).
- Repo list = team-namespace pinning only (terraform plan→apply PVC handoff needs `home-infra`); new repos are auto-configured into `home-auto` — do NOT grow the list ad hoc.
- Auto-configured repos can't carry `settings.policy` (PaC creates the CR itself) — whitelist them via the `OWNERS` file on the repo's default branch.
- Upstream manifest must be re-downloaded on bump, never edited in place.
- gosmee is the no-domain webhook path; when a domain + Cloudflare Tunnel exist, remove the `gosmee` release and point the GitHub App webhook at the domain instead.
- GitHub App `repository` event must stay subscribed (auto-configure depends on it).
- Cross-repo rules: workspace root `AGENTS.md`.

## Security
- No real secrets, keys, or kubeconfigs may be committed; `*.example.yaml` + `.gitignore` are the only allowed pattern.
- `<ingress-ip>`, `<your-github-org>`, `<region>`, `<user-pool-id>` are TODOs, not values to invent.

## Checklist
- [ ] helmfile chart paths / terragrunt sources resolve (no dangling `./charts/...`)
- [ ] YAML parses (`helm template` or a yaml lint)
- [ ] 3-way sync files unchanged unless intentionally updated together
- [ ] No real secrets added (only `*.example.yaml`)
- [ ] No placeholder replaced with a fabricated value

## Examples
`base/home-ingress-nginx` is the reference helmfile pattern; the `helmfile-template` repo documents the skeleton.

## When stuck
- For pipeline/auth issues: `wiki/cicd/pipelines/troubleshoot/` and the restricted wiki runbooks.
