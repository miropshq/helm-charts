# Mirops Operator Helm Chart

This Helm chart deploys the Mirops Kubernetes upgrade-analysis operator.

The operator watches `UpgradeAnalysis` resources, collects cluster readiness signals, computes an upgrade-risk score, and writes a report that can be consumed by `mirops-cli`.

## Requirements

- Kubernetes 1.20 or newer
- Helm 3.x
- A reachable Mirops operator image

## Install

Install with default values:

```sh
helm install mirops ./mirops-operator --namespace mirops --create-namespace
```

Install with a custom image:

```sh
helm install mirops ./mirops-operator \
  --namespace mirops \
  --create-namespace \
  --set image.repository=<registry>/mirops \
  --set image.tag=<tag>
```

Install with private registry credentials:

```sh
helm install mirops ./mirops-operator \
  --namespace mirops \
  --create-namespace \
  --set registryCredentials.enabled=true \
  --set registryCredentials.registry=ghcr.io \
  --set registryCredentials.username=<username> \
  --set registryCredentials.password=<token> \
  --set registryCredentials.email=<email>
```

## Verify

```sh
kubectl get pods -n mirops
kubectl get deployment -n mirops
kubectl logs -n mirops deployment/mirops-controller-manager
```

## Create An Analysis

```yaml
apiVersion: mirops.mirops.io/v1
kind: UpgradeAnalysis
metadata:
  name: upgrade-check
  namespace: mirops
spec:
  targetVersion: "1.29"
  scope:
    mode: application
  source:
    type: file
    path: /tmp/mirops-report.json
```

Apply it:

```sh
kubectl apply -f upgrade-analysis.yaml
kubectl get upgradeanalysis -n mirops
kubectl describe upgradeanalysis upgrade-check -n mirops
```

## Values

| Value | Default | Description |
| ----- | ------- | ----------- |
| `namespace.create` | `true` | Create the namespace from the chart |
| `namespace.name` | `mirops` | Namespace name used by chart resources |
| `image.repository` | `ghcr.io/miropshq/mirops` | Operator image repository |
| `image.tag` | `latest` | Operator image tag |
| `image.pullPolicy` | `IfNotPresent` | Image pull policy |
| `imagePullSecrets` | `[]` | Existing image pull secrets |
| `registryCredentials.enabled` | `false` | Create an image pull secret from provided credentials |
| `registryCredentials.registry` | `ghcr.io` | Private registry host |
| `registryCredentials.username` | `""` | Registry username |
| `registryCredentials.password` | `""` | Registry password or token |
| `registryCredentials.email` | `""` | Registry email |
| `replicaCount` | `1` | Number of operator replicas |
| `serviceAccount.create` | `true` | Create a service account |
| `serviceAccount.name` | `mirops-controller-manager` | Service account name |
| `serviceAccount.annotations` | `{}` | Service account annotations |
| `rbac.create` | `true` | Create RBAC resources |
| `podAnnotations` | `{}` | Pod annotations |
| `podSecurityContext.runAsNonRoot` | `true` | Run pod as non-root |
| `podSecurityContext.runAsUser` | `65532` | User ID for the pod |
| `securityContext.allowPrivilegeEscalation` | `false` | Disable privilege escalation |
| `securityContext.readOnlyRootFilesystem` | `true` | Use a read-only root filesystem |
| `resources.requests.cpu` | `100m` | CPU request |
| `resources.requests.memory` | `64Mi` | Memory request |
| `resources.limits.cpu` | `500m` | CPU limit |
| `resources.limits.memory` | `128Mi` | Memory limit |
| `nodeSelector` | `{}` | Node selector |
| `tolerations` | `[]` | Pod tolerations |
| `affinity` | `{}` | Pod affinity |
| `metrics.enabled` | `true` | Enable metrics service |
| `metrics.port` | `8080` | Metrics port |
| `healthProbe.port` | `8081` | Health probe port |
| `report.path` | `/tmp/mirops-report.json` | Default local report path |
| `leaderElection.enabled` | `true` | Enable leader election |
| `logLevel` | `info` | Operator log level |

## Upgrade

```sh
helm upgrade mirops ./mirops-operator --namespace mirops
```

## Uninstall

```sh
helm uninstall mirops --namespace mirops
```

If the chart created the namespace and no other resources depend on it, delete it manually when needed:

```sh
kubectl delete namespace mirops
```

## Render Or Lint

```sh
helm template mirops ./mirops-operator --namespace mirops
helm lint ./mirops-operator
```
