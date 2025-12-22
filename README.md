# Flujo CI/CD + GitOps para dev y test en Minikube con ArgoCD y Helm

Estructura del proyecto:
El paso inicial es la creacion de los repositorios en Github unp para Infraestructura `infra-gitops`  como "fuente de la verdad" y ademas la creacion de dos repositorios que alojaran los microservicios (en esta demo solo serán dos simples `index.html`)  `service-a`  y `service-b` respectivamente. 

## Contenido de cada repositorio.

### infra-gitops
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

## Repositorios de código.
En el repositorio de código, viven los microservicios service-a y service-b. Cada uno tiene su lógica mínima, un Dockerfile que los envuelve en Nginx, y un pipeline de CI listo para trabajar.

### service-a
```
├─ index.html
├─ Dockerfile
├─ README.md
└─ .github/workflows/ci-pipeline.yaml
```
### service-b
 ```
├─ index.html
├─ Dockerfile
├─ README.md
└─ .github/workflows/ci-pipeline.yaml
```

Los `index.html` tiene la siguiente forma:

```
<!DOCTYPE html>
<html>
        <head><title>Service A</title></head>
        <body>
                  <h1>Hola, soy Service A!</h1>
                    <p>Demo  de CI/CD con GitOps.</p>
                    <p>version 0</p>
        </body>
</html>
```
El `Dockerfile`  que crea una imagen que sirve al index.html en el puerto 80. Es igual para ambos.

```
# service-a
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html

```
Finalmente el `ci-pipeline.yaml` que importante para el proceso CI:
Este sera el encargado de manejar los `secrets` del registry Docker Hub

```
name: CI Pipeline Service-A

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

      - name: Login to Docker
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}   #<---- Generar
          password: ${{ secrets.DOCKER_TOKEN }}      #<---- Generar

      - name: Build and push image
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: amolmedo/service-a:${{ env.VERSION }}   #<---- usuario Docker Hub

      - name: Checkout infra repo
        uses: actions/checkout@v4
        with:
          repository: AMOlmedo/infra-gitops    #<---- usuario Github 
          token: ${{ secrets.GH_PAT }}         #<---- Generar
          path: infra

      - name: Install yq
        run: sudo snap install yq

      - name: Update values
        run: |
          FILE="infra/environments/${ENV}/values-service-a.yaml"
          yq -i ".image.tag = \"${VERSION}\"" "$FILE"

      - name: Commit and push changes
        run: |
          cd infra
          git config user.name "ci-bot"
          git config user.email "ci-bot@adrian"
          git add .
          git commit -m "service-a: image.tag=${VERSION} for ${ENV}"
          git push origin main
```

Cuando el desarrollador o tester hace modificaciones en el `service-a` por ejemplo lo hace un *tag especial* `-dev` o `-test` acompañado de la version, ejemplo:

```
git tag 1.0.0-dev
```
 o 
```
git tag 1.0.0-test
```
Ese sufijo es quien abre el camino hacia el namespace correcto dentro de Minikube.
El pipeline de CI entra en acción. 
El commit con tag dispara el pipeline.
El CI construye la imagen Docker del servicio.
La etiqueta con el tag y el SHA del commit, asegurando trazabilidad.
Luego la sube al registry Docker Hub, en doode previamente se debe tener usuario creado.

Pero aquí viene lo importante: el CI nunca toca el cluster directamente. En lugar de eso, abre el repositorio de infraestructura y actualiza el `values.yaml`  correspondiente al ambiente y cambia el `image.tag` para que apunte a la nueva versión. Es en este punto donde ArgoCD "escucha" y entra en accion.

# Preparación del cluster Minikube y ArgoCD

## 1) Crear namespaces de ambientes
Para tal fin se dispone de una VPS en la cual previamente se ha instalado minikube para poder emular el cluster en k8s.
Luego se procede a la creacion de los nos namespaces solicitados, `dev` y `test`, como asi tambien el de `argocd` que luego lo necesitaremos.

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
Finalmente la última linea de comando abre un tunel y da un URL para acceder desde el navegador y expone la `IP` y el numero de puerto del servicio.

## 4) Obtener password inicial por default
Esta es la contraseña será requerida inicialmente para loguearnos en la consola ArgoCD con el usuario `admin`.
```
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

En el escenario actual en donde es necesario acceder al la consola de administracion desde  el host al servicio ArgoCD expuesto en la VPS, en necesario, en mi caso, crear un tunel. 

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

Y ahi finalmente  se acced a la consola web de administacion de ArgoCD.



# Estructura de Helm y manifiestos GitOps
## 1) Tree de infra-gitops de esta demo
Esta es el diseño que tendrá este proyecto.



## 2) Repo Dev  `values-service-a.yaml` correspondiente al primer microservicio.
En este caso se ha seleccionado "podinfo" para hacer las pruebas.


```
namespace: dev
replicaCount: 1
image:
  repository: ghcr.io/stefanprodan/podinfo
  tag: "6.3.6-dev"
service:
  type: ClusterIP
  port: 8080

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
    repoURL: https://github.com/AMOlmedo/infra-gitops.git
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

Igual para service-b y para test cambiando `paths`, `ValueFile` y `namespace`.

Aplicar apps:

```
git clone https://github.com/AMOlmedo/infra-gitops.git

```

```
kubectl apply -f infra-gitops/environments/dev/apps/ -n argocd
kubectl apply -f infra-gitops/environments/test/apps/ -n argocd
```


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


# CD

Infraestructura GitOps (infra-gitops)
Ya tenés el repo preparado con los values-service-a.yaml y values-service-b.yaml. El flujo ahora es:

Cada push de tag en service-a o service-b actualiza el values.yaml correspondiente en infra-gitops.

ArgoCD detecta el cambio y sincroniza automáticamente en el cluster.

🔹 3. ArgoCD Applications
En tu cluster Kubernetes, ArgoCD debe tener aplicaciones configuradas para cada servicio y entorno (dev, test). Ejemplo:

service-a-dev

service-a-test

service-b-dev

service-b-test

Cada una apunta al repo infra-gitops y al path correcto (charts/service-a o charts/service-b).

🔹 4. Validar despliegues en Kubernetes
Cuando hagas un nuevo tag:

El pipeline construye y sube la imagen.

Actualiza infra-gitops.

ArgoCD sincroniza.

En el cluster, revisás:

bash
kubectl get pods -n dev
kubectl get pods -n test
→ Deberías ver los pods de service-a y service-b corriendo.

Podés acceder con port-forward:

bash
kubectl port-forward svc/service-a 8080:80 -n dev
kubectl port-forward svc/service-b 8081:80 -n dev
Y abrir en el navegador:

http://localhost:8080 → Service A

http://localhost:8081 → Service B

🔹 5. Flujo completo ya armado
Desarrollás en service-a o service-b.

Taggeás una versión (1.0.7-dev).

GitHub Actions construye y sube la imagen.

Infra-gitops se actualiza automáticamente.

ArgoCD despliega en el cluster.

Kubernetes sirve tu index.html en el namespace correspondiente.

✅ En este punto ya tenés un circuito CI/CD + GitOps completo:

Código → Imagen → Infra → Cluster → Servicio accesible.

Todo automatizado y reproducible para ambos microservicios.




# instalar ArgoCD CLI en la  VPS

curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64

sudo mv argocd-linux-amd64 /usr/local/bin/

sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd


## luego hay que loguearse al ArgoCD

```
argocd login argocd-server --username admin --password <tu-pass>



argocd login localhost:8080 --username admin --password <argopass> --insecure

```
Para obtener el pass:
```
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

```
arcocd  app list
```


kubectl port-forward svc/argocd-server -n argocd 8080:443


en el WSL
```
ssh -L 8080:localhost:8080 adrian@<IP_DE_TU_VPS>
```











