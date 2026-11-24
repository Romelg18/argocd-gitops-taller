# Despliegue GitOps con Argo CD y Kubernetes

Taller de Procesos de Software. Una aplicación web (nginx) se despliega en Kubernetes
usando Argo CD, con este repositorio como fuente de verdad.

## Estructura

```
k8s/
  deployment.yml   # 2 réplicas, labels/selectors, requests/limits, puerto 80
  service.yml      # NodePort 30080 -> targetPort 80
  hpa.yml          # min 2, max 5, CPU objetivo 50%
argocd/
  application.yml  # (opcional) definición declarativa de la app en Argo CD
```

## Requisitos previos (Minikube en Windows + Docker Desktop)

```powershell
minikube start --driver=docker
minikube addons enable metrics-server
kubectl get pods -n kube-system | findstr metrics
```

## Instalar Argo CD

```powershell
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl get pods -n argocd -w
```

Acceso a la interfaz (dejar corriendo en una terminal aparte):

```powershell
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Abrir https://localhost:8080 (usuario `admin`). Contraseña inicial:

```powershell
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | % { [Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($_)) }
```

## Crear la aplicación en Argo CD (UI)

En **+ NEW APP**:

| Campo | Valor |
|---|---|
| Application Name | webapp |
| Project | default |
| Sync Policy | Automatic (activar Prune y Self Heal) |
| Repository URL | https://github.com/Romelg18/argocd-gitops-taller.git |
| Revision | main |
| Path | k8s |
| Cluster URL | https://kubernetes.default.svc |
| Namespace | default |

Alternativa por línea de comandos: `kubectl apply -f argocd/application.yml`
(el repoURL ya apunta a tu repo).

## Verificación

```powershell
kubectl get deploy,svc,hpa,pods
minikube service webapp-service --url     # abre/entrega la URL de acceso
kubectl get hpa webapp-hpa                # TARGETS debe mostrar un % y no <unknown>
```

### Probar el autoscaling

```powershell
kubectl run carga --image=busybox:1.36 --restart=Never -- /bin/sh -c "while true; do wget -q -O- http://webapp-service; done"
kubectl get hpa webapp-hpa -w             # las réplicas suben hasta 5
kubectl delete pod carga                   # al bajar la carga, vuelve a 2 (tarda unos minutos)
```

### Probar GitOps (detección de cambios)

1. Editar `k8s/deployment.yml` (por ejemplo cambiar `replicas` o la imagen) y hacer `git push`.
2. Argo CD detecta el cambio (hasta ~3 min, o botón **Refresh**) y sincroniza solo.
3. Comprobar con `kubectl get pods` y en la interfaz de Argo CD.

> Nota: con el HPA activo, este controla las réplicas; el valor `replicas: 2` es solo el inicial.
