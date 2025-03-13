# Proyecto integrador DevOps Grupo 10

## Integrantes

- Marquinho Moya
- Carlos Navarro
- Gabriel Barrera
- Federico Pascarella
- Julio Sejas

---

## Requisitos

- [**aws cli**](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
- [**kubectl**](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html)
- [**eksctl**](https://eksctl.io/installation/)
- [**helm**](https://helm.sh/docs/intro/install/)

## Planificación del Cluster EKS

- Numero de nodos: 3
- Tipo de instancia: t3.small
- Region: us-east-1
- Zonas:
    - us-east-1a
    - us-east-1b
    - us-east-1c

---

## Configuración del Entorno Local para Desplegar un Cluster de EKS en AWS

> OS: Linux, Ubuntu 24.04.1 LTS
> 

### Instalar los paquete necesarios en requisitos

- aws cli
    
    Se ejecuto lo siguiente
    
    ```bash
    sudo apt install unzip -y 
    curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
    unzip awscliv2.zip
    sudo ./aws/install
    ```
    
    
    Y para verificar ejecutamos
    
    ```bash
    aws --version
    ```
    
    
- Kubectl
    
    Se ejecuto lo siguiente
    
    ```bash
    curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.32.0/2024-12-20/bin/linux/amd64/kubectl
    chmod +x ./kubectl
    mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$HOME/bin:$PATH
    echo 'export PATH=$HOME/bin:$PATH' >> ~/.bashrc
    ```
    
    Ahora ejecutamos lo siguiente para verificar que esta todo correcto
    
    ```bash
    kubectl version --client
    ```
    
    
- eksctl
    
    Se ejecuto lo siguiente
    
    ```bash
    # for ARM systems, set ARCH to: `arm64`, `armv6` or `armv7`
    ARCH=amd64
    PLATFORM=$(uname -s)_$ARCH
    
    curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"
    
    # (Optional) Verify checksum
    curl -sL "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_checksums.txt" | grep $PLATFORM | sha256sum --check
    
    tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz
    
    sudo mv /tmp/eksctl /usr/local/bin
    ```
    
    Ahora ejecutamos lo siguiente para verificar que esta todo correcto
    
    ```bash
    eksctl info
    ```
    
    
- Helm
    
    Se ejecuto lo siguiente
    
    ```bash
    curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
    sudo apt-get install apt-transport-https --yes
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
    sudo apt-get update
    sudo apt-get install helm -y
    ```
    
    Ahora ejecutamos lo siguiente para verificar que esta todo correcto
    
    ```bash
    helm version
    ```
    
    

### Configurar AWS

Ahora vamos a configurar el AWS cli con las credenciales. Para mas información consulte

[Setting up the AWS CLI - AWS Command Line Interface](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-quickstart.html)

Una vez que tenemos el KEY ID y el ACCESS KEY, lo configuramos de la siguiente manera

Por seguridad pintamos las keys

Una vez que tenemos configurado el AWS cli podemos verificar con el siguiente comando

```bash
aws sts get-caller-identity
```

Con esto tenemos todo configurado para continuar

---

## Crear el Cluster EKS

### Creando Key Pair

Antes de crear en cluster necesitamos crear un ssh-key y subirlo a AWS.
Creamos una carpeta en $HOME y generamos la key
Se ejecuto lo siguiente

```bash
mkdir $HOME/eks && cd $HOME/eks && ssh-keygen -t rsa -b 4096 -C "eks-ssh" -f ./eks-ssh -N ""
```

Con **`ls`** verificamos que este creada la key


### Iniciar Cluster

Para la creación del cluster eks usamos este comando

```bash
eksctl create cluster \
--name eks-tp-m \
--region us-east-1 \
--node-type t3.small \
--nodes 3 \
--with-oidc \
--ssh-access \
--ssh-public-key $HOME/eks/eks-ssh.pub \
--managed \
--full-ecr-access \
--zones us-east-1a,us-east-1b,us-east-1c
```

Esto podría tardar entre 15m-40m


Con `kubectl get nodes` podemos ver los nodos disponibles

---

## Levantar Servicios

Ahora vamos a levantar varios servicios

### Nginx

Para levantar nginx creamos un archivo `nginx.yaml` con el siguiente contenido

```yaml
---
apiVersion: apps/v1 
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  selector:
    matchLabels:
      app: nginx  
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: default
  labels:
    app: nginx
spec:
  externalTrafficPolicy: Local
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx
  type: LoadBalancer
```

Ahora levantamos con

```bash
kubectl apply -f nginx.yaml
```

Una vez levantado podemos acceder desde el dns de elb. Para traer los dns de los elb podemos ejecutar el siguiente comando

```bash
kubectl get svc -n default
```

Vamos a tener dos entramos al EXTERNAL-IP de nginx

accedemos y listo ya tenemos nginx


### Prometheus

Para levantar Prometheus primero vamos a configurar el storage EBS

Primero instalamos el driver con el siguiente comando

```bash
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-1.30"
```

Ahora verificamos con el siguiente comando

```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-ebs-csi-driver
```

Una vez que tenemos los driver ahora configuramos los permisos con el siguiente comando

```bash
eksctl create iamserviceaccount \
--name ebs-csi-controller-sa \
--namespace kube-system \
--cluster eks-tp-m \
--attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
--approve \
--role-only \
--role-name AmazonEKS_EBS_CSI_DriverRole
```

Para habilitar el uso del EBS, seguimos las recomendaciones incluidas al final del PIN indicado

```bash
eksctl create addon \
--name aws-ebs-csi-driver \
--cluster eks-tp-m \
--service-account-role-arn arn:aws:iam::$(**aws sts get-caller-identity --query Account --output text**):role/AmazonEKS_EBS_CSI_DriverRole --force
```

Ahora que tenemos el EBS configurado vamos a levantar el Prometheus

primero creamos el namespace con el siguiente comando

```bash
kubectl create namespace prometheus
```

Agregamos los repositorios con el siguiente comando

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```

Ahora implementamos Prometheus con el siguiente comando

```bash
helm upgrade -i prometheus prometheus-community/prometheus \
    --namespace prometheus \
    --set alertmanager.persistence.storageClass="gp2" \
    --set server.persistentVolume.storageClass="gp2"
```

Ahora verificamos que todos los pods este `Running`con el siguiente comando

```bash
kubectl get pods -n prometheus
```

Ahora para acceder usamos `kubectl` para el enrutamiento del puerto con el siguiente comando

```bash
kubectl --namespace=prometheus port-forward deploy/prometheus-server 9090
```

Ahora accedemos desde [http://localhost:9090](http://localhost:9090/query)

Listo tenemos Prometheus

### Grafana

Primero vamos a crear un namespace para Grafana con el comando:

```bash
kubectl create namespace grafana
```

Ahora dentro de `$HOME/eks` creamos una carpeta para las confs de Grafana

```bash
mkdir ${HOME}/eks/grafana
```

Ahora dentro de `$HOME/eks/grafana` creamos el archivo `grafana.yaml` con los siguientes datos

```yaml
datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
    - name: Prometheus
      type: prometheus
      url: http://prometheus-server.prometheus.svc.cluster.local
      access: proxy
      isDefault: true

```

Ahora agregamos el repo de Grafana 

```bash
helm repo add grafana https://grafana.github.io/helm-charts
```

Ahora desplegamos `grafana` con helm con el siguiente comando

```bash
helm install grafana grafana/grafana \
    --namespace grafana \
    --set persistence.storageClassName="gp2" \
    --set persistence.enabled=true \
    --set adminPassword='PASSWD' \
    --values ${HOME}/eks/grafana/grafana.yaml \
    --set service.type=LoadBalancer
```

ahora con el siguiente comando verificamos que este correctamente levantado

```bash
kubectl get pods -n grafana
```

Buscamos el DNS del ELB para acceder con el siguiente comando

```yaml
kubectl get svc -n grafana
```


Ahora accedemos a Grafana 
![usamos PASSWD como password declarado]

usamos PASSWD como password declarado

Ahora vamos a `dashboard`

Ahora desde `New` importamos el dashboard
Usamos el ID `3119`
Hacemos click en `Load` 
Ahora en la última opción ponemos Prometheus y hacemos click en `Import`
Vamos a ver el Dashboard

De igual forma importamos el dashboard con ID  `6417` y le damos en `Load`
Luego seleccionamos Prometheus:
Y vamos a ver el segundo dashboard


## Clean Up

Después de tener todo funcionando correctamente, eliminaremos los recursos creados.

### **Eliminar Addons y Roles IAM Relacionados al CSI Driver de EBS**

Primero, se debe eliminar el addon del controlador CSI de EBS y el role IAM que se crearon:

```bash
eksctl delete addon --name aws-ebs-csi-driver --cluster eks-tp-m
eksctl delete iamserviceaccount --name ebs-csi-controller-sa --namespace kube-system --cluster eks-tp-m –wait
```


### **Eliminar las instalaciones de Helm**

Se debe desinstalar las instalaciones de Helm antes de eliminar el cluster de EKS. Esto asegurará que todos los recursos creados por Helm se eliminen correctamente.

```bash
helm uninstall grafana -n grafana
helm uninstall prometheus -n prometheus
```


### **Eliminar Namespaces**

Se tienen que eliminar los namespaces de Prometheus y Grafana mediante los siguientes comandos:

```bash
kubectl delete namespace grafana
kubectl delete namespace prometheus
```

### **Eliminar la Aplicación Nginx**

Elimina la aplicación Nginx que desplegaste:

```bash
kubectl delete -f nginx.yml
```

### **Eliminar el Cluster de EKS**

Ahora, por último, se debe eliminar el cluster de EKS. Este proceso puede tomar algún tiempo:

```bash
eksctl delete cluster --name eks-tp-m --region us-east-1
```


