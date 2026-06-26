# Guía completa: OCI VM Linux, OCI CLI, Docker, kubectl, OCIR, Container Instances y OKE

> **Objetivo:** dejar un flujo completo desde una VM Linux en OCI hasta publicar una app en **OCI Container Registry (OCIR)**, desplegarla en **OCI Container Instances** y también en **Oracle Kubernetes Engine (OKE)** con **managed nodes**.

> **Región usada en los ejemplos:** **US West (San Jose)**, cuyo region name es `us-sanjose-1` y region key es `SJC`. En OCIR, el endpoint regional para esa región es `sjc.ocir.io`. citeturn343587search1turn343587search9

---

## 1) Creación de la VM Linux en OCI

OCI Compute permite crear instancias Linux o Windows para ejecutar aplicaciones. Al crear la instancia, esta se lanza con una VNIC primaria y puedes elegir si tendrá IP pública o no. Si es tu primera vez, Oracle recomienda crear antes una VCN y subnet con el wizard de red. citeturn976435search6turn976435search1turn976435search5

### Pasos en la consola

1. En OCI Console, ve a **Compute > Instances > Create instance**.
2. Elige el **Compartment** donde vivirá la VM.
3. Selecciona la imagen **Oracle Linux 9**.
4. Elige la shape que necesites.
5. En red:
   - usa una **VCN** existente o crea una nueva;
   - usa una **subnet** pública si vas a conectarte por SSH desde Internet.
6. Agrega tu **SSH public key**.
7. Crea la instancia.

### Validación inicial

Conéctate por SSH:

```bash
ssh -i <tu_clave_privada> opc@<public-ip>
```

---

## 2) Configuración de OCI CLI en la Linux

Oracle documenta que la CLI se configura con un archivo local y una **API signing key**. En esta guía la llave se genera primero desde la consola de OCI y posteriormente se descarga para configurar la CLI. citeturn699212search5turn699212search9

### Instalación / verificación

En Oracle Linux 9, Oracle indica que puedes usar `dnf` para instalar la CLI. Si ya la tienes instalada, pasa directo a la configuración. citeturn976435search14

### Generar la API Key desde OCI

1. Inicia sesión en OCI.
2. Ve a **Profile → My Profile → API Keys**.
3. Selecciona **Add API Key**.
4. Elige **Generate API Key Pair**.
5. Descarga:
   - La **Private Key** (`.pem`).
   - El archivo de configuración sugerido por OCI.
6. Guarda la llave privada en la VM Linux, por ejemplo:
   `~/.oci/oci_api_key.pem`
7. Copia los valores del archivo de configuración al archivo:
   `~/.oci/config`
8. Protege la llave:

```bash
chmod 600 ~/.oci/oci_api_key.pem
```

### Validación

```bash
oci iam region-subscription list
oci os ns get
```

### Pruebas útiles

```bash
oci iam region list
oci iam region-subscription list
oci os ns get
```

Si usas varios perfiles, puedes elegir uno con:

```bash
oci os ns get --profile <perfil>
export OCI_CLI_PROFILE=<perfil>
```

---

## 3) Configuración de Docker en la Linux

Para Linux tipo RHEL/Oracle Linux 9, Docker recomienda agregar su repositorio y luego instalar `docker-ce`, `docker-ce-cli`, `containerd.io`, `docker-buildx-plugin` y `docker-compose-plugin`. Después debes habilitar el servicio Docker. citeturn485640search6turn485640search12

### Instalación

```bash
sudo dnf -y install dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### Habilitar y arrancar

```bash
sudo systemctl enable --now docker
sudo systemctl status docker
```

### Ejecutar Docker sin `sudo`

```bash
sudo usermod -aG docker $USER
newgrp docker
```

### Validación

```bash
docker --version
docker run hello-world
```

---

## 4) Configuración de kubectl en la Linux

Kubernetes recomienda instalar `kubectl` con el binario oficial para Linux y mantener una versión compatible con el cluster. Oracle OKE usa `oci ce cluster create-kubeconfig` para generar el kubeconfig local. citeturn526581search0turn561960search5turn561960search0

### Instalar kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
kubectl version --client
```

### Conectar kubectl a OKE

Primero obtén el OCID del cluster en OCI. Luego genera el kubeconfig:

```bash
oci ce cluster create-kubeconfig \
  --cluster-id <CLUSTER_OCID> \
  --file $HOME/.kube/config \
  --region us-sanjose-1 \
  --token-version 2.0.0 \
  --kube-endpoint PUBLIC_ENDPOINT
```

Oracle indica que el kubeconfig se guarda por defecto en `~/.kube/config`, y que `kubectl` puede usar una versión compatible con el cluster; en su documentación de kubeconfig para OKE, Oracle usa `--token-version 2.0.0` y `--kube-endpoint PUBLIC_ENDPOINT` para acceso público. citeturn561960search1turn561960search5

### Validación

```bash
kubectl config get-contexts
kubectl config current-context
kubectl get nodes
```

---

## 5) Creación previa de repositorios en OCI Container Registry

Container Registry es el registry administrado por OCI para almacenar imágenes contenedorizadas. Oracle recomienda crear el repositorio antes de hacer push, y los nombres de repositorio son únicos en toda la tenancy; además, normalmente se agrupan versiones distintas de la misma imagen en un mismo repositorio. citeturn446084search5turn446084search3turn343587search4

### Crear el repositorio

```bash
oci artifacts container repository create \
  --compartment-id <COMPARTMENT_OCID> \
  --display-name myokeapp
```

### Notas importantes

- Si el repositorio va en **root**, usa el OCID de la tenancy como `--compartment-id`.
- Si tu repositorio vive en un compartment específico, usa el OCID de ese compartment.
- En San Jose, el endpoint regional es `sjc.ocir.io`. citeturn343587search1turn343587search9

### Validación

```bash
oci artifacts container repository list \
  --compartment-id <COMPARTMENT_OCID> \
  --all
```

---

## 6) Creación de imágenes y subida a OCIR
### Generar un Auth Token

El **Auth Token** es el mecanismo recomendado para autenticar Docker contra OCIR.

1. Inicia sesión en OCI.
2. Ve a **Profile → My Profile → Auth Tokens**.
3. Haz clic en **Generate Token**.
4. Asigna una descripción (por ejemplo `docker-ol9`).
5. **Copia el token**, ya que solo se muestra una vez.

El login al registry utiliza:

- **Servidor:** `sjc.ocir.io`
- **Usuario:** `<namespace>/<usuario>` (o `<namespace>/<identity-domain>/<usuario>` si utilizas IAM Identity Domains).
- **Contraseña:** el **Auth Token**.


OCIR permite empujar y extraer imágenes Docker con el Docker CLI. Para hacerlo necesitas un **Auth Token** y el nombre de usuario OCI correcto; Oracle documenta que el token se genera desde el perfil del usuario y se usa como contraseña para el login del registry. citeturn446084search0turn559812search5turn559812search14

### Login a OCIR

```bash
docker login sjc.ocir.io
```

Usa:

- **Username:** `<namespace>/<usuario>`
- **Password:** `<Auth Token>`

Oracle documenta que el nombre de usuario se forma con el **tenancy namespace** y el usuario, y el token se usa como contraseña. citeturn343587search7turn559812search2

### Build

Desde la carpeta donde está el `Dockerfile`:

```bash
docker build -t myokeapp:1.0 .
```

### Tag para OCIR

```bash
docker tag myokeapp:1.0 sjc.ocir.io/<namespace>/myokeapp:1.0
```

### Push

```bash
docker push sjc.ocir.io/<namespace>/myokeapp:1.0
```

### Validación

```bash
docker images
```

Después del push, la imagen debe quedar visible en el repositorio de OCIR en el compartment correcto. Oracle también indica que las imágenes se identifican por la combinación de repositorio y tag/version. citeturn343587search4turn446084search3

---

## 7) Creación del OCI Container Instance y publicación de la app

OCI Container Instances es un servicio serverless para ejecutar contenedores sin administrar servidores. Puedes crear una instancia con uno o más contenedores, definir shape, red, variables de entorno y políticas de restart. citeturn772318search3turn136907search11

### Paso a paso en la consola

1. Ve a **Developer Services > Container Instances**.
2. Haz clic en **Create container instance**. Oracle documenta ese flujo en la consola como el modo estándar de creación. citeturn772318search0
3. Define:
   - **Name**
   - **Compartment**
   - **Availability Domain**
   - **Shape** / recursos
4. En **Containers**, agrega tu contenedor:
   - **Image**: `sjc.ocir.io/<namespace>/myokeapp:1.0`
   - **Ports**: por ejemplo `8080`
   - **Environment variables**: las que tu app requiera
5. En red, usa una subnet que pueda alcanzar el registry:
   - si la imagen está en **OCIR**, Oracle indica que la subnet debe poder llegar al registry y recomienda **Service Gateway** para ese caso. citeturn136907search13
6. Crea la instancia.

### Publicar la app

Si tu contenedor escucha en `8080`, asegúrate de exponer ese puerto en la app y, si necesitas acceso desde Internet, coloca la instancia en una subnet pública o delante de un balanceador según tu diseño de red.

### Validación

En la consola, revisa el estado de la instancia y el contenedor. Desde la app, comprueba que responde en el puerto configurado.

---

## 8) Volver a hacer push de la app y refrescar el contenedor

Si reconstruyes una nueva versión de la imagen, repites el flujo de **build -> tag -> push**. En Container Registry, cada `repo:tag` apunta a una versión concreta de la imagen, y Oracle recomienda usar tags diferentes para distintas versiones. citeturn343587search4turn446084search3

### Ejemplo de nueva versión

```bash
docker build -t myokeapp:2.0 .
docker tag myokeapp:2.0 sjc.ocir.io/<namespace>/myokeapp:2.0
docker push sjc.ocir.io/<namespace>/myokeapp:2.0
```

### Para que el container tome la nueva imagen

- **En OCI Container Instances:** usa **Update** o **Restart** de la instancia para que los contenedores se recrean. Oracle documenta que al reiniciar una Container Instance, los contenedores se recrean con almacenamiento efímero nuevo. citeturn136907search3turn136907search2
- **En OKE:** cambia el tag en el `Deployment` y aplica de nuevo el YAML, o usa:
  ```bash
  kubectl set image deployment/myokeapp myokeapp=sjc.ocir.io/<namespace>/myokeapp:2.0
  kubectl rollout status deployment/myokeapp
  ```
  Si reutilizas el mismo tag, ejecuta:
  ```bash
  kubectl rollout restart deployment/myokeapp
  ```
  para forzar un redeploy.

---

## 9) Creación del cluster de OKE en OCI con Managed Nodes

OKE es el servicio administrado de Kubernetes en OCI. Oracle indica que puedes usar **managed nodes**, **virtual nodes** o **self-managed nodes**, y que los managed nodes son instancias de Compute dentro de tu tenancy que tú controlas. citeturn949683search4turn949683search12turn949683search0

### En la consola

1. Ve a **Developer Services > Kubernetes Engine (OKE)**.
2. Elige **Create cluster**.
3. Selecciona **Quick Create** o **Custom Create**.
4. Define:
   - **Compartment**
   - **Cluster name**
   - **Kubernetes version**
   - **VCN**
   - **Control plane endpoint**
5. Cuando configures los nodos, elige **Managed nodes**.
6. Selecciona:
   - shape
   - subnet de workers
   - número de nodos
   - OCPU y memoria si aplica
7. Crea el cluster y espera a que los nodos estén listos.

### Validación del cluster

```bash
oci ce cluster list --compartment-id <COMPARTMENT_OCID>
kubectl get nodes
kubectl get pods -A
```

### Acceso al cluster

Después de crear el cluster, genera o actualiza el kubeconfig con la OCI CLI. Oracle documenta ese paso como la forma estándar de acceder al cluster con `kubectl`. citeturn561960search5turn561960search1

---

## 10) Archivos YAML de la app y del service, despliegue y validación

Para desplegar una app desde OCIR a OKE, Oracle indica dos piezas clave:

1. Un **Docker registry secret** para autenticarse contra OCIR.
2. Un manifiesto que incluya `imagePullSecrets` y la ruta completa de la imagen. citeturn949683search2turn343587search6turn446084search7

### 10.1 Crear el secret de OCIR

```bash
kubectl create secret docker-registry ocir-secret \
  --docker-server=sjc.ocir.io \
  --docker-username='<namespace>/<usuario>' \
  --docker-password='<AUTH_TOKEN>' \
  --docker-email='<tu_correo>'
```

### 10.2 deployment.yaml

```yaml
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

### 10.3 svc.yaml

Para exponer la app por un balanceador de OCI, usa `type: LoadBalancer`. Oracle documenta que OKE puede provisionar un OCI Load Balancer para un Service de ese tipo. citeturn949683search1turn949683search5

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myokeapp
spec:
  type: LoadBalancer
  selector:
    app: myokeapp
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
```

### 10.4 Aplicar los manifiestos

```bash
kubectl apply -f deployment.yaml
kubectl apply -f svc.yaml
```

### 10.5 Validar el despliegue

```bash
kubectl get pods
kubectl describe pod <pod_name>
kubectl rollout status deployment/myokeapp
kubectl get svc myokeapp
kubectl get endpoints myokeapp
```

### 10.6 Confirmar que la app quedó publicada

- `kubectl get pods` debe mostrar el pod en `Running`.
- `kubectl describe pod <pod_name>` debe mostrar la imagen de OCIR y el evento `Pulled`.
- `kubectl get svc myokeapp` debe mostrar una `EXTERNAL-IP` cuando el balanceador esté listo.
- Abre la IP del servicio en el puerto `8080`.

### 10.7 Actualizar la app en OKE con una nueva imagen

Cuando hagas un nuevo `push` a OCIR:

```bash
docker build -t myokeapp:2.0 .
docker tag myokeapp:2.0 sjc.ocir.io/<namespace>/myokeapp:2.0
docker push sjc.ocir.io/<namespace>/myokeapp:2.0
kubectl set image deployment/myokeapp myokeapp=sjc.ocir.io/<namespace>/myokeapp:2.0
kubectl rollout status deployment/myokeapp
```

Si conservas el mismo tag, fuerza el redeploy:

```bash
kubectl rollout restart deployment/myokeapp
kubectl rollout status deployment/myokeapp
```

---

## Comandos de verificación rápida

```bash
# OCI CLI
oci iam region-subscription list
oci os ns get

# Docker
docker images
docker ps

# kubectl
kubectl config get-contexts
kubectl get nodes
kubectl get pods
kubectl get svc
```

---

## Flujo resumido

1. Crear VM Linux en OCI.
2. Configurar OCI CLI con API signing key.
3. Instalar Docker.
4. Instalar kubectl.
5. Crear el repo en OCIR antes del push.
6. Hacer `docker build`, `docker tag` y `docker push`.
7. Crear OCI Container Instance y publicarla.
8. Hacer nuevos pushes y reiniciar/recrear el runtime para que tome la nueva imagen.
9. Crear OKE con **managed nodes**.
10. Crear `deployment.yaml` y `svc.yaml`, aplicar y validar con `kubectl`.

---

## Referencias oficiales

- OCI Compute / Instances. citeturn976435search6turn976435search0
- OCI CLI config y setup. citeturn699212search5turn699212search9
- Docker Engine on RHEL-compatible Linux. citeturn485640search6turn485640search12
- kubectl install on Linux. citeturn526581search0
- OCIR / Container Registry. citeturn446084search5turn446084search3turn343587search4
- OCIR login / auth token. citeturn559812search5turn343587search7
- OCI Container Instances. citeturn772318search3turn772318search0turn136907search13
- OKE / managed nodes / kubeconfig / LoadBalancer. citeturn949683search12turn561960search5turn949683search1turn949683search5
