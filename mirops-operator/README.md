# Mirops Helm Chart

Este es un Helm chart para desplegar el operador **mirops** en Kubernetes.

## Requisitos

- Kubernetes 1.20+
- Helm 3.x

## Instalación

### 1. Build de la imagen Docker (si no tienes una disponible)

```bash
# Desde la raíz del proyecto
make docker-build IMG=mirops:0.1.0
# Opcional: empujar a registry
make docker-push IMG=mirops:0.1.0
```

### 2. Instalar con Helm

```bash
# Con valores por defecto (crea namespace mirops-system automáticamente)
helm install mirops ./helm/mirops

# Con el namespace ya existente
helm install mirops ./helm/mirops --namespace mirops-system --create-namespace

# Con imagen personalizada
helm install mirops ./helm/mirops \
  --set image.repository=myregistry/mirops \
  --set image.tag=0.1.0
```

### 3. Verificar la instalación

```bash
# Ver el namespace
kubectl get ns mirops-system

# Ver el deployment en el namespace mirops-system
kubectl get deployment -n mirops-system

# Ver logs del controller
kubectl logs -f -n mirops-system deployment/mirops-controller-manager-controller-manager
```

## Valores configurables (values.yaml)

| Pnamespace.create` | `true` | Crear el namespace automáticamente |
| `namespace.name` | `mirops-system` | Nombre del namespace |
| `arámetro | Defecto | Descripción |
|-----------|---------|-------------|
| `image.repository` | `mirops` | Repository de la imagen |
| `image.tag` | `latest` | Tag de la imagen |
| `image.pullPolicy` | `IfNotPresent` | Pull policy |
| `replicaCount` | `1` | Número de réplicas |
| `resources.requests.cpu` | `100m` | CPU request |
| `resources.requests.memory` | `64Mi` | Memory request |
| `resources.limits.cpu` | `500m` | CPU limit |
| `resources.limits.memory` | `128Mi` | Memory limit |
| `leaderElection.enabled` | `true` | Habilitar leader election |
| `metrics.enabled` | `true` | Habilitar métricas |

## Ejemplo de uso en cualquier namespace:

```bash
# Crear en el namespace default
cat <<EOF | kubectl apply -f -
apiVersion: mirops.mirops.io/v1
kind: UpgradeAnalysis
metadata:
  name: my-upgrade-check
  namespace: default
spec:
  foo: "kubernetes-1.28"
EOF
```

Ver el status:

```bash
kubectl describe upgradeanalysis my-upgrade-check -n default
```

El controller en `mirops-system` procesará automáticamente el recurso.

## Desinstalar

```bash
helm uninstall mirops
# El namespace mirops-system será eliminado si no contiene otros recursos
```bash
helm uninstall mirops -n mirops-system
```

## Ficheros incluidos

- `Chart.yaml` - Metadata del chart
- `values.yaml` - Valores por defecto
- `templates/deployment.yaml` - Deployment del controller
- `templates/service-account.yaml` - ServiceAccount
- `templates/cluster-role.yaml` - ClusterRole
- `templates/cluster-role-binding.yaml` - ClusterRoleBinding
- `templates/service.yaml` - Service para métricas
- `templates/_helpers.tpl` - Helper functions
- `templates/NOTES.txt` - Instrucciones post-install
