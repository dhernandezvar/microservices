# Oracle Cloud Infrastructure - Guía Completa

## Docker, OCI Container Registry, OCI Container Instances y Oracle Kubernetes Engine

## Arquitectura

``` text
Oracle Linux VM
      │
      ▼
 Docker Build
      │
      ▼
OCI Container Registry (OCIR)
      ├──────────────► OCI Container Instance
      │
      └──────────────► Oracle Kubernetes Engine (OKE)
                           │
                           └── LoadBalancer Service
```

# 1. Crear una VM Oracle Linux 9

1.  OCI Console → Compute → Instances.
2.  Create Instance.
3.  Imagen: Oracle Linux 9.
4.  Seleccionar Shape.
5.  Crear o seleccionar VCN/Subnet.
6.  Asignar IP pública.
7.  Agregar la llave SSH pública.
8.  Crear la instancia.

Conectarse:

``` bash
ssh -i llave.pem opc@PUBLIC_IP
```

# 2. Instalar OCI CLI

``` bash
sudo dnf update -y
sudo dnf install -y python3 python3-pip

bash -c "$(curl -L https://raw.githubusercontent.com/oracle/oci-cli/master/scripts/install/install.sh)"

oci --version
```

# 3. Generar una API Key

En OCI Console:

-   Profile
-   My Profile
-   API Keys
-   Add API Key
-   Generate API Key Pair

Descargar:

-   Private Key
-   Config File

En la VM:

``` bash
mkdir -p ~/.oci
cp oci_api_key.pem ~/.oci/
chmod 600 ~/.oci/oci_api_key.pem
chmod 600 ~/.oci/config
```

Validar:

``` bash
oci iam region-subscription list
oci os ns get
```

# 4. Obtener el Namespace

``` bash
oci os ns get
```

# 5. Generar un Auth Token

OCI Console

Profile → My Profile → Auth Tokens → Generate Token

Se utilizará para el login de Docker contra OCIR.

# 6. Instalar Docker

``` bash
sudo dnf install -y dnf-plugins-core

sudo dnf config-manager \
--add-repo https://download.docker.com/linux/rhel/docker-ce.repo

sudo dnf install -y \
docker-ce docker-ce-cli \
containerd.io \
docker-buildx-plugin \
docker-compose-plugin

sudo systemctl enable --now docker

sudo usermod -aG docker $USER
newgrp docker
```

Validar:

``` bash
docker version
docker run hello-world
```

# 7. Crear previamente el repositorio en OCIR

OCI Console

Developer Services → Container Registry

Create Repository

o mediante CLI

``` bash
oci artifacts container repository create \
--compartment-id <COMPARTMENT_OCID> \
--display-name myokeapp
```

# 8. Construir la imagen

``` bash
docker build -t myokeapp:1.0 .
```

Verificar:

``` bash
docker images
```

# 9. Login a OCIR

``` bash
docker login sjc.ocir.io
```

Usuario:

    <namespace>/<identity-domain>/<usuario>

Contraseña:

    <Auth Token> (este debes obtenerlo de tu perfil de usuario en OCI Console)

# 10. Tag y Push

``` bash
docker tag myokeapp:1.0 \
sjc.ocir.io/<namespace>/myokeapp:1.0

docker push \
sjc.ocir.io/<namespace>/myokeapp:1.0
```

# 11. Crear una OCI Container Instance

Developer Services → Container Instances

Create Container Instance

Configurar:

-   Nombre
-   Compartment
-   Shape
-   VCN
-   Subnet
-   Imagen: `sjc.ocir.io/<namespace>/myokeapp:1.0`
-   Puerto 8080

Crear.

# 12. Actualizar la Container Instance

Nueva versión:

``` bash
docker build -t myokeapp:2.0 .

docker tag myokeapp:2.0 \
sjc.ocir.io/<namespace>/myokeapp:2.0

docker push \
sjc.ocir.io/<namespace>/myokeapp:2.0
```

Editar la Container Instance para utilizar la nueva imagen o reiniciarla
si reutilizas el mismo tag.

# 13. Instalar kubectl

``` bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

chmod +x kubectl

sudo mv kubectl /usr/local/bin/
```

# 14. Crear un Cluster OKE

Developer Services

Kubernetes Clusters

Create Cluster

Seleccionar:

-   Managed Nodes
-   Kubernetes Version
-   Node Pool
-   Shape
-   Número de nodos

Esperar estado Active.

# 15. Configurar kubeconfig

``` bash
oci ce cluster create-kubeconfig \
--cluster-id <CLUSTER_OCID> \
--file ~/.kube/config \
--region us-sanjose-1 \
--token-version 2.0.0 \
--kube-endpoint PUBLIC_ENDPOINT
```
(este comando lo puedes tomar de la consola de OCI en OKE en la sección QuickStart - Access cluster)
Validar:

``` bash
kubectl get nodes
```

# 16. Crear Secret para OCIR

``` bash
kubectl create secret docker-registry ocir-secret \
--docker-server=sjc.ocir.io \
--docker-username='<namespace>/<identity-domain>/<usuario>' \
--docker-password='<AUTH_TOKEN>'
```
(acá debes utilizar el Auth Token generado en tu perfil en la consola de OCI)

# 17. deployment.yaml

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myokeapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myokeapp
  template:
    metadata:
      labels:
        app: myokeapp
    spec:
      imagePullSecrets:
      - name: ocir-secret
      containers:
      - name: myokeapp
        image: sjc.ocir.io/<namespace>/myokeapp:1.0
        ports:
        - containerPort: 8080
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
```

Aplicar:

``` bash
kubectl apply -f deployment.yaml
```

# 18. svc.yaml

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: myokeapp
spec:
  type: LoadBalancer
  selector:
    app: myokeapp
  ports:
  - port: 8080
    targetPort: 8080
```

``` bash
kubectl apply -f svc.yaml
```

# 19. Validaciones

``` bash
kubectl get pods - muestra los pods activos
kubectl get svc  - muestra los servicios activos (de aquí tomas la ip pública para el acceso a la aplicación)
kubectl describe pod <pod> - muestra el estado actual del pod de tu aplicación
kubectl rollout status deployment/myokeapp - muestra si la aplicación ya está activa
```

# 20. Publicación

Esperar una IP externa:

``` bash
kubectl get svc
```

Acceder:

    http://EXTERNAL-IP:8080

# 21. Nueva versión

``` bash
docker build -t myokeapp:2.0 .
docker tag myokeapp:2.0 sjc.ocir.io/<namespace>/myokeapp:2.0
docker push sjc.ocir.io/<namespace>/myokeapp:2.0

kubectl set image deployment/myokeapp \
myokeapp=sjc.ocir.io/<namespace>/myokeapp:2.0

kubectl rollout status deployment/myokeapp
```

Si reutilizas el mismo tag:

``` bash
kubectl rollout restart deployment/myokeapp
```

# 22. Troubleshooting

-   Invalid username format → usar
    `<namespace>/<identity-domain>/<usuario>`
-   NXDOMAIN → usar `sjc.ocir.io`
-   ImagePullBackOff → revisar `ocir-secret` e `imagePullSecrets`
-   Repositorio invisible → verificar el compartment
-   Push correcto → termina con `digest: sha256:...`

# Buenas prácticas

-   Crear previamente los repositorios OCIR.
-   Usar Auth Token.
-   Versionar imágenes (`1.0`, `1.1`, `2.0`).
-   Evitar `latest`.
-   Verificar siempre `kubectl rollout status`.
