# Tekton Operator

Installs the Tekton Operator (0.81.1), the cluster's TektonConfig, and the standalone Tekton Dashboard (v0.72.0).

## Prerequisites

- Cluster up (Talos home-cluster-1), connected via Pinniped kubeconfig.
- Worker labeled for TaskRun pods: `kubectl label node <worker> node-role.kubernetes.io/worker=""`

## Download upstreams

Already vendored in `charts/upstream/` + `kustomizations/upstream/`. To refresh:

```sh
curl -L -o ./charts/upstream/tekton-operator-0.81.1.tgz https://github.com/tektoncd/operator/releases/download/tekton-operator-0.81.1/tekton-operator-0.81.1.tgz

curl -L -o ./kustomizations/upstream/tekton-dashboard-0.72.0.yaml https://infra.tekton.dev/tekton-releases/dashboard/previous/v0.72.0/release-full.yaml
```

## Deploy

```sh
helmfile -l stage=operator apply

# Wait for the operator to be Ready before applying config
kubectl wait --for=condition=Ready --timeout=300s -n tekton-operator deploy/tekton-operator

helmfile -l stage=config apply

# Wait for all pods in tekton-pipelines to be ready

helmfile -l stage=dashboard apply   # standalone Tekton Dashboard
```

Resulting features (from `kustomizations/tekton-operator-config`): Pipelines, Triggers; cluster-resolver restricted to `tekton-pipelines,home-infra`; hourly pruner (24h retention); TaskRuns scheduled on the worker.

Tekton Dashboard (from `kustomizations/tekton-dashboard`): **write-mode** v0.72.0, standalone (not operator-managed — the operator would restore the stripped cluster-wide SA bindings). All SA RBAC is stripped; every request acts as the logged-in user via oauth2-proxy impersonation (`home-sso-auth-rbac/charts/expose-tekton-dashboard`), so visibility/writes follow `cluster-rbac` per-user RBAC.

Log retention (Results/Vector/MinIO) is TODO for a later date.

## Cleanup

```sh
helmfile -l stage=dashboard destroy
helmfile -l stage=config destroy
helmfile -l stage=operator destroy
```