# CI/CD Tekton Pipelines as Code

Git-triggered CI via Pipelines-as-Code (v0.50.0) + a GitHub App in the personal org. GitHub webhooks reach the cluster via the gosmee relay (`hook.pipelinesascode.com`) — no domain or public IP needed.

## Prerequisites

- Cluster up, connected via Pinniped kubeconfig.
- Tekton Operator installed first (`cicd-tekton-operator`).
- LAN ingress URL reachable (PaC calls `pipelines-as-code.192.168.1.200.sslip.io`).

## Download upstreams

Already vendored in `kustomizations/upstream/`. To refresh:

```sh
curl -L -o ./kustomizations/upstream/pipelines-as-code-0.50.0.yaml https://github.com/openshift-pipelines/pipelines-as-code/releases/download/v0.50.0/release.k8s.yaml
```

## Webhook relay (gosmee)

GitHub → `hook.pipelinesascode.com/<channel>` (public relay) → `gosmee-client` (in-cluster, outbound SSE) → PaC controller. The relay channel is random and unguessable; payloads are still HMAC-validated by the webhook secret.

1. Get a channel: open `https://hook.pipelinesascode.com/new` and copy the **Webhook Proxy URL**.
2. Put it in the secret as `smee` (see below).
3. The `gosmee-client` Deployment (stage `gosmee`) reads it from the secret.

## GitHub App

1. Create/verify the GitHub App in the personal org (`https://github.com/apps/pac-talos-linux-home-setup`):
   - Webhook URL = the relay URL from step 1 above (`https://hook.pipelinesascode.com/<channel>`)
   - Repository permissions: Checks `Read & Write`, Contents `Read & Write`, Issues `Read & Write`, Metadata `Read-only`, Pull requests `Read & Write`; Organization: Members `Read-only`
   - Events: `check_run`, `check_suite`, `commit_comment`, `issue_comment`, `pull_request`, `push`, **`repository`** (required for auto-configure)
2. Copy `secrets/pipelines-as-code-github-secrets.example.yaml` → `secrets/pipelines-as-code-github-secrets.yaml` and fill in app id, webhook secret, the app's private key, and `smee` (the relay URL).
3. Install the App on the org (and any other account/org that should run CI).

## Auto-configure (`home-auto`)

`auto-configure-new-github-repo: "true"` (set in `kustomizations/pipelines-as-code/patches/configmaps.yaml`): any repo where the app is installed and that contains a `.tekton/` dir gets a Repository CR in `home-auto` automatically — no registration needed. The namespace is pre-provisioned by the cluster-rbac chart (`auto` in the 3-way sync files) with the `tekton-bot` SA. Whitelist for auto-configured repos = the repo's `OWNERS` file (approvers/reviewers); allow-listed repos additionally carry `settings.policy` (`pull_request` + `ok_to_test`) from the repositories chart values.

## Deploy

```sh
kubectl apply -f secrets/pipelines-as-code-github-secrets.yaml

helmfile -l stage=expose-operator apply
helmfile -l stage=gosmee apply
helmfile -l stage=pipelines apply
```

The repositories chart (`charts/pipelines-as-code-repositories`) renders a `Repository` CR per GitHub repo in the matching `home-*` namespace — the list only pins repos to team namespaces (terraform plan→apply handoff needs `home-infra`); new repos are auto-configured into `home-auto` and need no list entry.

## Cleanup

```sh
helmfile -l stage=pipelines destroy
helmfile -l stage=gosmee destroy
helmfile -l stage=expose-operator destroy
helmfile -l stage=operator destroy

kubectl delete -f secrets/pipelines-as-code-github-secrets.yaml

kubectl delete namespace pipelines-as-code
```

## Moving to a domain (later)

When a domain + Cloudflare Tunnel exist: change the GitHub App's webhook URL to `https://pipelines-as-code.<domain>`, remove the `gosmee` release from `helmfile.yaml`, and switch the PaC ingress to `cloudflare-issuer` (see `home-cloudflare-tunnel` enable checklist).