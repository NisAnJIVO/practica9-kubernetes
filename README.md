# Práctica 9 — Despliegue en Kubernetes con Helm y GitOps

Este repositorio contiene los manifiestos declarativos de Kubernetes, el empaquetado mediante Helm Charts para múltiples entornos y la configuración GitOps con Argo CD para el microservicio de tareas (`tareas-api`).

## Estructura del Proyecto

```text
practica9-kubernetes/
├── k8s/
│   ├── deployment.yaml      # Manifiesto declarativo del Deployment con sondas y variables
│   ├── service.yaml         # Service ClusterIP en puerto 80 hacia el puerto 3000
│   ├── ingress.yaml         # Regla de Ingress para enrutamiento por host tareas.localhost
│   └── configmap.yaml       # Configuración no sensible desacoplada de la imagen
├── chart/
│   ├── Chart.yaml           # Metadatos del chart Helm v2
│   ├── values.yaml          # Valores predeterminados (Laboratorio/Desarrollo)
│   ├── values-prod.yaml     # Valores para el entorno de Producción
│   └── templates/
│       ├── deployment.yaml  # Plantilla de Deployment parametrizada
│       ├── service.yaml     # Plantilla de Service parametrizada
│       └── ingress.yaml     # Plantilla de Ingress parametrizada
├── argocd/
│   └── application.yaml     # Manifiesto GitOps Application con auto-sync y selfHeal
└── README.md
```

## Reproducción del Despliegue desde Cero

### 1. Requisitos Previos
- Docker Desktop iniciado y funcionando
- `k3d` v5.6+
- `kubectl` v1.29+
- `helm` v3.14+

### 2. Creación del Clúster Local
```powershell
k3d cluster create com600 --agents 2 -p "8080:80@loadbalancer"
```

> **Nota para Windows**: Si `kubectl` reporta problemas de conexión con `host.docker.internal`, edite `~/.kube/config` para reemplazar `host.docker.internal` por `127.0.0.1`.

### 3. Carga de Imágenes en el Clúster
```powershell
k3d image import ghcr.io/nisanjivo/tareas-api:1.0.0 ghcr.io/nisanjivo/tareas-api:2.0.0 -c com600
```

### 4. Despliegue Declarativo (Laboratorios 2 al 5)
```powershell
kubectl create secret generic tareas-secret --from-literal=APP_CLAVE_API=clave-de-laboratorio-2026
kubectl apply -f k8s/
```
Comprobación de acceso:
- Abrir en navegador: `http://tareas.localhost:8080/tareas` o `http://tareas.localhost:8080/salud`

### 5. Despliegue con Helm (Laboratorio 6)
```powershell
# Limpieza de manifiestos previos
kubectl delete -f k8s/

# Despliegue entorno desarrollo / laboratorio
helm install tareas ./chart

# Despliegue entorno producción aislado
helm install tareas-prod ./chart -f chart/values-prod.yaml

# Comprobar ambos entornos
helm list
```

### 6. Flujo GitOps con Argo CD (Laboratorio 7)
```powershell
# 1. Instalar Argo CD en el clúster
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 2. Obtener contraseña del admin
[Text.Encoding]::UTF8.GetString([Convert]::FromBase64String((kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}")))

# 3. Acceder al dashboard de Argo CD
kubectl port-forward svc/argocd-server -n argocd 8090:443
# Abrir https://localhost:8090 en el navegador (usuario: admin)

# 4. Registrar la Application
kubectl apply -f argocd/application.yaml
```
Cualquier cambio realizado con `git commit` y `git push` en `k8s/` será automáticamente reconciliado en el clúster por Argo CD (`selfHeal` y `prune` activos).
