# Mirops Helm Charts

Helm charts for **Mirops** — the live Kubernetes Cluster Mirror: a dependency graph of your cluster, the blast radius of whatever is down, and the risk per namespace, rebuilt continuously. Upgrade analysis is opt-in.

## Charts

| Chart | Description |
| ----- | ----------- |
| `mirops-operator` | Deploys the Mirops operator: the always-on ClusterMirror, plus opt-in upgrade analysis · [Artifact Hub](https://artifacthub.io/packages/helm/mirops-operator/mirops) |

## Requirements

- Kubernetes 1.31 – 1.34 (recent clusters; older may work but is untested)
- Helm 3.x

## Install The Operator

From this directory:

```sh
helm install mirops ./mirops-operator --namespace mirops --create-namespace
```

From the published OCI repository — also listed on [Artifact Hub](https://artifacthub.io/packages/helm/mirops-operator/mirops):

```sh
helm install mirops oci://ghcr.io/miropshq/charts/mirops \
  --namespace mirops \
  --create-namespace \
  --version 0.2.0
```

The chart publishes from `main`:

| Branch | Chart repo | Default image tag |
| ------ | ---------- | ----------------- |
| `main` | `charts` | `appVersion` from `Chart.yaml`, currently `0.2.0` |

Use a custom image:

```sh
helm install mirops ./mirops-operator \
  --namespace mirops \
  --create-namespace \
  --set image.repository=<registry>/mirops/operator \
  --set image.tag=<tag>
```

## Upgrade

```sh
helm upgrade mirops ./mirops-operator --namespace mirops
```

## Uninstall

```sh
helm uninstall mirops --namespace mirops
```

## Development

Render the chart locally:

```sh
helm template mirops ./mirops-operator --namespace mirops
```

Lint the chart:

```sh
helm lint ./mirops-operator
```

See `mirops-operator/README.md` for chart-specific values and examples.
