# Mirops Helm Charts

This directory contains Helm charts for installing Mirops components on Kubernetes.

## Charts

| Chart | Description |
| ----- | ----------- |
| `mirops-operator` | Deploys the Mirops Kubernetes upgrade-analysis operator |

## Requirements

- Kubernetes 1.20 or newer
- Helm 3.x
- A published Mirops operator image, or access to build and push one

## Install The Operator

From this directory:

```sh
helm install mirops ./mirops-operator --namespace mirops --create-namespace
```

Use a custom image:

```sh
helm install mirops ./mirops-operator \
  --namespace mirops \
  --create-namespace \
  --set image.repository=<registry>/mirops \
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
