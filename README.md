# Preparación del cluster Minikube y ArgoCD
## 1) Crear namespaces de ambientes
Para tal fin se dispone de una VPS el cual previamente se ha instalado minikube para poder emular el cluster en k8s.
Luego se procede a la creacion de nos namespaces solicitados, `dev` y `test`, como asi tambien el de `argocd` que luego lo necesitaremos.

```
kubectl create namespace dev
kubectl create namespace test
kubectl create namespace argocd
```
## 2) Instalar ArgoCD


```
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl rollout status deploy/argocd-server -n argocd      
```
La segunda linea de comando se utilizará para monitorear el progreso de un despliegue (deployment) específico en Kubernetes. 

## 3) Exponer ArgoCD en Minikube


```
kubectl -n argocd patch svc argocd-server -p '{"spec": {"type": "NodePort"}}'
kubectl -n argocd get svc argocd-server
minikube service argocd-server -n argocd --url
```

En la primera linea de comando se cambia el servicio argocd-server que inicialmente viene por defecto como ClusterIp por NodePort. Esto es porque al no tener un cluster real, no tenemmos un servicios de  LoadBalancer, por lo cual para poder acceder a la interfaz web de ArgoCD desde afuera del cluster, se asigna un puerto que redirige el servicio dentro del cluster.
Finalmete la última linea de comando abre un tunel y da un URL para acceder desde el navegador y expone la `IP` y el numero de puerto del servicio.

## 4) Obtener password inicial por default

```
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```
Esta es la contraseña será requerida inicialmente para loguearnos en la consola ArgoCD con el usuario `admin`.

En el escenario actual en donde necesito acceder al la consola de administracion desde  mi host al servicio ArgoCd expuesto en la VPS, en necesario en mi caso, crear un tunel. 

En la VPS:
```
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
Abre el servicio en el puerto 8080 de la VPS.

Luego  desde mi WSL  creo el túnel:
```
ssh -L 8080:localhost:8080 adrian@IP-VPS
```
y finalmente desde mi navegador mi host (Win10):
```
http://localhost:8080
```

Y ahi finalmente accedo a la consola de ArgoCD.

# Estructura de Helm y manifiestos GitOps
## 1) Árbol de infra-gitops (demo)
Esta es el diseño que tendrá este proyecto.

```
infra-gitops/
├─ charts/
│  ├─ service-a/
│  │  ├─ Chart.yaml
│  │  ├─ templates/deployment.yaml
│  │  ├─ templates/service.yaml
│  │  └─ values.yaml
│  └─ service-b/
│     ├─ Chart.yaml
│     ├─ templates/deployment.yaml
│     ├─ templates/service.yaml
│     └─ values.yaml
├─ environments/
│  ├─ dev/
│  │  ├─ values-service-a.yaml
│  │  ├─ values-service-b.yaml
│  │  └─ apps/
│  │     ├─ app-service-a.yaml
│  │     └─ app-service-b.yaml
│  └─ test/
│     ├─ values-service-a.yaml
│     ├─ values-service-b.yaml
│     └─ apps/
│        ├─ app-service-a.yaml
│        └─ app-service-b.yaml
└─ README.md
```

## 2) Ejemplo `values.yaml` base (charts/service-a/values.yaml)


```
image:
  repository: your-dockerhub-username/service-a
  tag: "latest"
  pullPolicy: IfNotPresent

replicaCount: 1

service:
  type: ClusterIP
  port: 8080

resources: {}
```

## 3) Overlays por ambiente
environments/dev/values-service-a.yaml

```
namespace: dev
replicaCount: 1
image:
  tag: "1.0.0-dev"
environments/test/values-service-a.yaml

yaml
namespace: test
replicaCount: 2
image:
  tag: "1.0.0-test"
```

environments/dev/values-service-b.yaml

```
image:
  repository: your-dockerhub-username/service-b
  tag: "latest"
  pullPolicy: IfNotPresent

replicaCount: 2

service:
  type: ClusterIP
  port: 8080

resources: {}
```

## 3) Overlays por ambiente
environments/dev/values-service-a.yaml

```
namespace: dev
replicaCount: 1
image:
  tag: "1.0.0-dev"
environments/test/values-service-a.yaml

yaml
namespace: test
replicaCount: 2
image:
  tag: "1.0.0-test"
```

## 4) Aplicaciones de ArgoCD (dev/test)
environments/dev/apps/app-service-a.yaml

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: service-a-dev
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/infra-gitops.git
    targetRevision: main
    path: charts/service-a
    helm:
      valueFiles:
        - ../../environments/dev/values-service-a.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Igual para service-b y para test cambiando paths, nombre y namespace.

Aplicar apps:

```
kubectl apply -f infra-gitops/environments/dev/apps/ -n argocd
kubectl apply -f infra-gitops/environments/test/apps/ -n argocd
```

# CI: build y actualización del repo de infraestructura
## 1) Supuestos

Registry: Docker Hub.

Branch principal: main.

Tag semántico: X.Y.Z-dev o X.Y.Z-test.

## 2) Pipeline de CI (GitHub Actions en service-a/.github/workflows/release.yml)

```
name: release
on:
  push:
    tags:
      - '*-dev'
      - '*-test'

jobs:
  build-and-update-infra:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set env from tag
        run: |
          TAG="${GITHUB_REF##*/}"
          if [[ "$TAG" == *-dev ]]; then echo "ENV=dev" >> $GITHUB_ENV; fi
          if [[ "$TAG" == *-test ]]; then echo "ENV=test" >> $GITHUB_ENV; fi
          echo "VERSION=$TAG" >> $GITHUB_ENV

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push image
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: your-dockerhub-username/service-a:${{ env.VERSION }}

      - name: Checkout infra repo
        uses: actions/checkout@v4
        with:
          repository: your-org/infra-gitops
          token: ${{ secrets.GH_PAT }}
          path: infra

      - name: Update values for env
        run: |
          FILE="infra/environments/${ENV}/values-service-a.yaml"
          yq -i ".image.tag = \"${VERSION}\"" "$FILE"

      - name: Commit and push changes
        run: |
          cd infra
          git config user.name "ci-bot"
          git config user.email "ci-bot@your-org"
          git add .
          git commit -m "service-a: set image.tag=${VERSION} for ${ENV}"
          git push origin main
```

Repetir para service-b cambiando nombres. Notas:

yq puede instalarse con pip install yq o paquete del runner.

El CI no toca el cluster: solo “declara” el estado deseado en infra-gitops.

# Verificación end-to-end
## 1) Disparar release dev
Crear tag en service-a:

```
git tag 1.0.0-dev
git push origin 1.0.0-dev
```

El pipeline publica la imagen y actualiza infra-gitops/environments/dev/values-service-a.yaml.

ArgoCD detecta el cambio, sincroniza y despliega en dev.

## 2) Comprobar en el cluster

```
kubectl get pods -n dev
kubectl describe deploy service-a -n dev
kubectl get svc -n dev
```

## 3) Repetir para test

```
git tag 1.0.0-test
git push origin 1.0.0-test
kubectl get pods -n test
```

# Manejo de secretos y seguridad básica
Secretos en CI/CD:

Guardar DOCKERHUB_TOKEN y GH_PAT en secretos del repo (no en el código).

Kubernetes secrets:

Evitar secretos en values.yaml. Usar External Secrets o Sealed Secrets:

Instalar Sealed Secrets (Bitnami) y versionar solo el secreto sellado.

RBAC y namespaces:

ArgoCD solo con permisos sobre dev y test.

Pull de imágenes privadas:

Crear imagePullSecrets por namespace:

```
kubectl create secret docker-registry regcred \
  --docker-server=index.docker.io \
  --docker-username=your-dockerhub-username \
  --docker-password=YOUR_TOKEN \
  --docker-email=you@example.com \
  -n dev
```

Referenciar en los charts:

```
imagePullSecrets:
  - name: regcred
```









