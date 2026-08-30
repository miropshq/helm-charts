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
| `compatMatrix.enabled` | `false` | Pull a published compatibility-matrix version (off → use the operator's embedded matrix) |
| `compatMatrix.repo` | `ghcr.io/miropshq/mirops-compat` | OCI repo for the matrix artifact |
| `compatMatrix.version` | `latest` | Matrix version: `latest`, or a date tag like `v2026.06.15` |
| `compatMatrix.orasImage` | `ghcr.io/oras-project/oras:v1.2.0` | Image used by the pull Job |
| `compatMatrix.kubectlImage` | `bitnami/kubectl:1.31` | Image used to write the ConfigMap |

## Compatibility matrix version

By default the operator uses its **embedded** matrix (baked into the image at build) — deterministic and offline-safe. To **decouple the matrix from the operator image** and update it without rebuilding, enable `compatMatrix`: a pre-install/pre-upgrade hook Job pulls the chosen version into the `mirops-compatibility-matrix` ConfigMap the operator reads.

```sh
# always the newest matrix (non-prod / stay current)
helm upgrade --install mirops oci://ghcr.io/miropshq/charts-prod/mirops -n mirops \
  --set compatMatrix.enabled=true --set compatMatrix.version=latest

# pin a date tag (reproducible; recommended for production)
helm upgrade --install mirops oci://ghcr.io/miropshq/charts-prod/mirops -n mirops \
  --set compatMatrix.enabled=true --set compatMatrix.version=v2026.06.15
```

> Keep `latest` for non-production and **pin a date** in production, so the upgrade verdict stays reproducible. For air-gapped clusters, leave `compatMatrix.enabled=false` and rely on the embedded matrix.

**Private matrix artifact.** While the OCI artifact is private, the pull Job needs registry auth. It reuses your registry credentials automatically:

```sh
# reuse the same PAT that pulls the operator image
helm upgrade --install mirops oci://ghcr.io/miropshq/charts-prod/mirops -n mirops \
  --set compatMatrix.enabled=true \
  --set registryCredentials.enabled=true \
  --set registryCredentials.username=<user> --set registryCredentials.password=<PAT read:packages>
```

Alternatively, point `compatMatrix.pullSecret` at an existing `kubernetes.io/dockerconfigjson` secret. When the artifact is **public**, set neither — the Job pulls anonymously.

## Azure Workload Identity

On AKS, set the pod label and the identity client-id (and, when the cluster's default OIDC tenant isn't the identity's — e.g. cross-tenant — the tenant-id):

```sh
helm upgrade --install mirops oci://ghcr.io/miropshq/charts-prod/mirops -n mirops \
  --set-string podLabels."azure\.workload\.identity/use"=true \
  --set serviceAccount.annotations."azure\.workload\.identity/client-id"=<client-id> \
  --set serviceAccount.annotations."azure\.workload\.identity/tenant-id"=<tenant-id>   # optional
```

For AWS EKS use IRSA instead: `--set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=<role-arn>`.

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
