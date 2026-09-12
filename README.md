# Kubernetes CI/CD layer

CI/CD stack of the home cluster, deployed with helmfile on top of the base layer. One component per directory; each is a self-contained helmfile project.

## Components

| Directory | What it does |
|-----------|--------------|
| `cicd-tekton-operator` | Tekton Operator (0.81.1), TektonConfig, standalone write-mode Dashboard (v0.72.0) |
| `cicd-tekton-pipelines-as-code` | Git-triggered CI: Pipelines-as-Code (v0.50.0) + GitHub App, webhooks via gosmee relay |
| `cicd-verdaccio` | Cache-only NPM registry for TaskRuns (no auth, in-cluster only) |

## Prerequisites

- Base layer deployed (`home-cluster-kubernetes-base`): ingress + `home-issuer` (PaC URL) and Longhorn (PVCs, TaskRun workspaces).
- CI node roles labeled (see `talos-linux-setup/designate-node-roles.yaml`).

## Deploy

In this order (PaC needs Tekton Pipelines from the operator):

```sh
cd cicd-tekton-operator && helmfile -l stage=operator apply && helmfile -l stage=config apply
cd cicd-tekton-pipelines-as-code && helmfile -l stage=expose-operator apply && helmfile -l stage=gosmee apply && helmfile -l stage=pipelines apply
cd cicd-verdaccio && helmfile apply
```

For Pipelines-as-Code, also apply the GitHub App secret first (fill `secrets/pipelines-as-code-github-secrets.example.yaml` → `secrets/pipelines-as-code-github-secrets.yaml`).

## Cleanup

```sh
helmfile destroy
```

per directory, in reverse order.