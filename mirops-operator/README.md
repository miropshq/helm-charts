# Mirops Operator Helm Chart

This Helm chart deploys the Mirops operator.

The operator keeps an always-on **ClusterMirror** of the cluster: it rebuilds the component dependency graph on an interval and publishes the current risk per namespace. Upgrade analysis (`UpgradeAnalysis`) is opt-in: set `upgrade.enabled=true` to have the operator also check whether the cluster is ready for a new Kubernetes version.

The chart installs the operator, its RBAC, the reports service and, optionally, a PVC for reports. It creates **no** ClusterMirror or UpgradeAnalysis — you create them after installing. The operator installs its own CRDs at startup, so choosing an `image.tag` also pins the matching CRDs.

## Requirements

- Kubernetes 1.31 – 1.34 (recent clusters; older may work but is untested)
- Helm 3.x

## Install

```sh
helm install mirops oci://ghcr.io/miropshq/charts/mirops \
  --namespace mirops --create-namespace --version 0.2.0
```

From a checkout of this repository:

```sh
helm install mirops ./mirops-operator --namespace mirops --create-namespace
```

> **Namespace — one knob.** Let Helm create the namespace with `--create-namespace` (Terraform: `create_namespace = true`). That's the only setting you need, and it works for every case, including `compatMatrix` (whose pre-install hook needs the namespace to exist first). The chart does **not** create the namespace by default (`namespace.create=false`) — set `namespace.create=true` only if you want the chart to own the Namespace resource, and then do **not** also pass `--create-namespace` (they'd collide).

Install with a custom image:

```sh
helm install mirops ./mirops-operator \
  --namespace mirops \
  --create-namespace \
  --set image.repository=<registry>/mirops/operator \
  --set image.tag=<tag>
```

## Verify

```sh
kubectl get pods -n mirops
kubectl logs -n mirops deployment/mirops-controller-manager
```

## Create Your ClusterMirror

Nothing is mirrored until you create a ClusterMirror. It is cluster-scoped, so it has no namespace. Name it `default` — the Headlamp plugin opens that one first:

```yaml
apiVersion: mirops.mirops.io/v1
kind: ClusterMirror
metadata:
  name: default
spec:
  scope:
    mode: all            # all | application (excludes system namespaces)
  refresh:
    interval: 5m         # how often the mirror is rebuilt
  # source:              # optional: where default.mirror is written (default: inside the operator pod)
  #   type: s3
  #   bucket: my-bucket
  #   region: us-east-1
  #   key: prod/default.mirror
```

```sh
kubectl apply -f mirror.yaml
kubectl get clustermirror
# NAME      COMPONENTS   EDGES   AT RISK   LAST SYNC   AGE
# default   184          297     3         30s         1h
kubectl get clustermirror default -o jsonpath='{.status.byNamespace}'
```

By default the report (`default.mirror`) lives inside the operator pod: it's lost on a restart and missing for a few seconds on a leader change, until the next rebuild. For a report a pipeline reads, write it to `s3`, `blob` or `pvc` (`reportPVC.enabled=true`) — the same `source` block an UpgradeAnalysis uses.

## Create An Analysis

Upgrade analysis is off by default; with it off, an UpgradeAnalysis is never processed. Enable it first:

```sh
helm upgrade mirops oci://ghcr.io/miropshq/charts/mirops --namespace mirops \
  --reuse-values --set upgrade.enabled=true
```

Then create the analysis (cluster-scoped, no namespace):

```yaml
apiVersion: mirops.mirops.io/v1
kind: UpgradeAnalysis
metadata:
  name: upgrade-check
spec:
  targetVersion: "1.35"   # must be higher than the cluster's version
  scope:
    mode: application
```

```sh
kubectl apply -f upgrade-analysis.yaml
kubectl get upgradeanalysis
kubectl describe upgradeanalysis upgrade-check
```

To turn it off again, `--set upgrade.enabled=false`. The ClusterMirror keeps running either way.

## Read The Reports

The `mirops-reports` service serves both reports, wherever they're stored — `file` from the pod, `s3`/`blob`/`pvc` read back with the operator's own credentials:

```sh
kubectl port-forward -n mirops svc/mirops-reports 8084:8084
curl http://localhost:8084/reports/default.mirror
curl http://localhost:8084/reports/upgrade-check.mirops
```

## Values

| Value | Default | Description |
| ----- | ------- | ----------- |
| `namespace.create` | `false` | Let the chart own the Namespace resource. Keep `false` and use Helm's `--create-namespace` instead (one knob). Don't combine `true` with `--create-namespace`. |
| `namespace.name` | `mirops` | Namespace name used by chart resources |
| `image.repository` | `ghcr.io/miropshq/mirops/operator` | Operator image repository |
| `image.tag` | `v0.2.0` | Operator image tag (also selects the CRDs the operator installs) |
| `image.pullPolicy` | `IfNotPresent` | Image pull policy |
| `upgrade.enabled` | `false` | Run the `UpgradeAnalysis` controller (upgrade-readiness analysis). The ClusterMirror controller always runs. |
| `imagePullSecrets` | `[]` | Existing image pull secrets (the images are public) |
| `replicaCount` | `1` | Number of operator replicas |
| `serviceAccount.create` | `true` | Create a service account |
| `serviceAccount.name` | `mirops-controller-manager` | Service account name |
| `serviceAccount.annotations` | `{}` | Service account annotations — IRSA (AWS) or Workload Identity (Azure) |
| `rbac.create` | `true` | Create RBAC resources |
| `podAnnotations` | `{}` | Pod annotations |
| `podLabels` | `{}` | Extra pod labels — e.g. `azure.workload.identity/use: "true"` on AKS |
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
| `reports.port` | `8084` | Port of the reports service (`mirops-reports`) |
| `reports.dir` | `/var/mirops/reports` | Directory in the pod for `file` reports |
| `reportPVC.enabled` | `false` | Mount a PVC for `source.type: pvc` reports |
| `reportPVC.existingClaim` | `""` | Use a PVC you already created; empty creates one |
| `reportPVC.mountPath` | `/mnt/mirops-reports` | Where the PVC is mounted (the operator's default `pvc` path) |
| `reportPVC.size` | `1Gi` | Size of the created PVC |
| `reportPVC.storageClass` | `""` | Storage class of the created PVC (empty = cluster default) |
| `reportPVC.accessModes` | `[ReadWriteOnce]` | Access modes of the created PVC |
| `leaderElection.enabled` | `true` | Enable leader election |
| `compatMatrix.enabled` | `false` | Pull a published compatibility-matrix version (off → use the operator's embedded matrix) |
| `compatMatrix.repo` | `ghcr.io/miropshq/mirops-compat` | OCI repo for the matrix artifact |
| `compatMatrix.version` | `v2026.09.03` | Matrix version: `latest`, or a date tag like `v2026.09.03` |
| `compatMatrix.pullSecret` | `""` | Pull secret, only if you mirror the artifact to a private registry |
| `compatMatrix.orasImage` | `ghcr.io/oras-project/oras:v1.2.0` | Image used by the pull Job |
| `compatMatrix.kubectlImage` | `alpine/k8s:1.31.0` | Image used to write the ConfigMap (must include a shell — distroless kubectl images won't work) |
| `remediation.enabled` | `true` | Deploy the remediation manager (executes approved RemediationPlans) |
| `remediation.image.repository` | `ghcr.io/miropshq/mirops/remediation` | Remediation manager image repository |
| `remediation.image.tag` | `v0.2.0` | Remediation manager image tag |

> **Changing values with `--reuse-values`.** `--set key={}` or `--set key=null` doesn't remove a nested key from the values Helm reuses. To drop one, write the current values to a file (`helm get values mirops -n mirops -o yaml > values.yaml`), edit it, and run `helm upgrade -f values.yaml` without `--reuse-values`.

## Compatibility matrix version

By default the operator uses its **embedded** matrix (baked into the image at build) — deterministic and offline-safe. To **decouple the matrix from the operator image** and update it without rebuilding, enable `compatMatrix`: a pre-install/pre-upgrade hook Job pulls the chosen version into the `mirops-compatibility-matrix` ConfigMap the operator reads.

> Because the pull runs as a **pre-install hook** (before the chart's normal resources), the namespace must already exist. Install with **`--create-namespace`** so Helm creates it first (the chart's default `namespace.create=false` already keeps it from creating a second one).

```sh
# always the newest matrix (non-prod / stay current)
helm upgrade --install mirops oci://ghcr.io/miropshq/charts/mirops \
  --namespace mirops --create-namespace \
  --set compatMatrix.enabled=true --set compatMatrix.version=latest

# pin a date tag (reproducible; recommended for production)
helm upgrade --install mirops oci://ghcr.io/miropshq/charts/mirops \
  --namespace mirops --create-namespace \
  --set compatMatrix.enabled=true --set compatMatrix.version=v2026.09.03
```

> Keep `latest` for non-production and **pin a date** in production, so the upgrade verdict stays reproducible. For air-gapped clusters, leave `compatMatrix.enabled=false` and rely on the embedded matrix.

### Inspecting what a matrix version covers

A matrix version is a frozen snapshot of every add-on rule (`addonRange` → `k8sRange`) at that date. To see what a tag covers, pull it and read `matrix.yaml`:

```sh
# a pinned date
oras pull ghcr.io/miropshq/mirops-compat:v2026.09.03 --output ./matrix
cat ./matrix/dist/matrix.yaml

# or the moving 'latest'
oras pull ghcr.io/miropshq/mirops-compat:latest --output ./matrix
cat ./matrix/dist/matrix.yaml
```

Once installed with `compatMatrix.enabled=true`, the same content lives in the ConfigMap the operator reads:

```sh
kubectl get configmap mirops-compatibility-matrix -n mirops -o jsonpath='{.data.matrix\.yaml}'
```

## Azure Workload Identity

On AKS, set the pod label and the identity client-id (and, when the cluster's default OIDC tenant isn't the identity's — e.g. cross-tenant — the tenant-id):

```sh
helm upgrade --install mirops oci://ghcr.io/miropshq/charts/mirops -n mirops \
  --set-string podLabels."azure\.workload\.identity/use"=true \
  --set serviceAccount.annotations."azure\.workload\.identity/client-id"=<client-id> \
  --set serviceAccount.annotations."azure\.workload\.identity/tenant-id"=<tenant-id>   # optional
```

For AWS EKS use IRSA instead: `--set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=<role-arn>`.

## Upgrade

```sh
helm upgrade mirops oci://ghcr.io/miropshq/charts/mirops --namespace mirops --version 0.2.0 --reuse-values
```

Upgrading from 0.1.0 needs nothing else: the new operator installs the `ClusterMirror` CRD at startup, and upgrade analysis is now off unless you set `upgrade.enabled=true` — set it if you already run UpgradeAnalyses.

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
