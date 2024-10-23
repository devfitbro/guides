# Instalación Kubernetes en un solo nodo con k3s

- Actualizar el sistema

   ```shell
   sudo apt update && sudo apt upgrade -y
   ```

- Instalar Docker

   ```shell
    # Add Docker's official GPG key:
    sudo apt-get update
    sudo apt-get install ca-certificates curl
    sudo install -m 0755 -d /etc/apt/keyrings
    sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
    sudo chmod a+r /etc/apt/keyrings/docker.asc

    # Add the repository to Apt sources:
    echo \
      "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
      $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
      sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    sudo apt-get update

    # Install
    sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y

- Crear grupo de docker

    ```shell
    sudo groupadd docker
    ```

- Crear grupo de docker

    ```shell
    sudo usermod -aG docker $USER
    ```

- Cerrar sesión y volver a iniciar

    ```shell
    newgrp docker
    ```

- Verificar que docker esté instalado

    ```shell
    docker ps
    ```

- Arrancar al inicio del sistema

    ```shell
    sudo systemctl enable docker.service
    sudo systemctl enable containerd.service
    ```

- Instalar k3s

   ```shell
   curl -sfL https://get.k3s.io | K3S_KUBECONFIG_MODE="644" sh -s -
   ```

- Probar kubernetes

   ```shell
   kubectl get nodes
   kubectl cluster-info
   ```

- Instalar Helm

   ```shell
    curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
    chmod 700 get_helm.sh
    ./get_helm.sh
   ```

- Agregar un dashboard

  Primero debemos agregar una variable de entorno, para que helm sepa que nuestra comunicación es con k3s

    ```shell
    export KUBECONFIG=/etc/rancher/k3s/k3s.yaml  
    ```

  Ahora instalamos kubernetes-dashboard

   ```shell
    helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
    helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard --create-namespace --namespace kubernetes-dashboard
    kubectl patch svc kubernetes-dashboard-kong-proxy --type='json' -p '[{"op":"replace","path":"/spec/type","value":"NodePort"}]' -n kubernetes-dashboard
   ```

- Ver estado de dashboard

   ```shell
    kubectl get pods -n kubernetes-dashboard
    kubectl get svc -n kubernetes-dashboard
   ```

- Crear el archivo `kubernetes-dashboard.yml`

   ```yaml
  apiVersion: v1
  kind: List
  items:
    # ServiceAccount Definition
    - apiVersion: v1
      kind: ServiceAccount
      metadata:
        name: admin-user  # Name of the service account
        namespace: kube-system  # Namespace in which the service account exists
    # ServiceAccount Definition
    - apiVersion: v1
      kind: ServiceAccount
      metadata:
        name: admin-user  # Name of the service account
        namespace: kube-system  # Namespace in which the service account exists
      secrets:
        - name: admin-user-token
    
    # ClusterRoleBinding Definition
    - apiVersion: rbac.authorization.k8s.io/v1
      kind: ClusterRoleBinding
      metadata:
        name: admin-user  # Name of the cluster role binding
      roleRef:
        apiGroup: rbac.authorization.k8s.io
        kind: ClusterRole
        name: cluster-admin  # Name of the cluster role
      subjects:
        - kind: ServiceAccount
          name: admin-user  # Name of the service account
          namespace: kube-system  # Namespace in which the service account exists
   ```

- Aplicar los cambios

   ```yaml
    kubectl create -f kubernetes-dashboard.yml
   ```

- Crear un token de larga duración

  ```shell
  kubectl -n kube-system create token admin-user --duration=31536000s
  ```

- Crear namespace para cert-manager

   ```yaml
    kubectl create namespace cert-manager
   ```

- Descargar k3s cert-manager y let's encrypt

   ```yaml
    kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.16.1/cert-manager.yaml
   ```

- Validar que estén corriendo los pods

   ```yaml
    kubectl get pods --namespace cert-manager
   ```

- Crear **certs.yml** para prepara let's encrypt y traefik usando ingress

  ```shell
  nano certs.yml
  ```

  > Nota: Cambiar <example@gmail.com> por un correo y example.letsencrypt.key.tls por un nombre de secreto

  ```yaml
  apiVersion: cert-manager.io/v1
  kind: ClusterIssuer
  metadata:
    name: letsencrypt-certificate
    # namespace must be cert-manager as such
    namespace: cert-manager
  spec:
    acme:
      server: https://acme-v02.api.letsencrypt.org/directory  # Let's Encrypt ACME server URL
      email: example@gmail.com  # Email address for Let's Encrypt notifications
      privateKeySecretRef:
        name: devys.letsencrypt.key.tls  # Name of the Kubernetes secret storing Let's Encrypt private key
      solvers:
        - selector: {}  # Use all eligible ingresses to solve ACME challenges
          http01:
            ingress:
              class: traefik  # Use Traefik ingress controller to handle HTTP01 challenges
  ```

- Aplicar para desplegar

  ```shell
  kubectl apply -f certs.yml
  ```

- Revisar que cert-manager este corriendo

  ```shell
  kubectl get cert-manager
  ```

- Instalar metrics server (para autoescalamiento)

  ```shell
  kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
  ```

- Creemos una aplicación de ejemplo

  ```shell
  nano example-deployment.yml
  ```

  Ahora agregamos el siguiente contenido, cambiando your_domain_name por un dominio válido:

  ```yaml
  apiVersion: v1
  kind: List
  items:
    # NGINX Deployment
    - apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: nginx-deployment
      spec:
        replicas: 3
        selector:
          matchLabels:
            app: nginx
        template:
          metadata:
            labels:
              app: nginx
          spec:
            containers:
              - name: nginx
                image: nginx:latest
                ports:
                  # Container port where NGINX listens
                  - containerPort: 80
    # NGINX Service
    - apiVersion: v1
      kind: Service
      metadata:
        labels:
          app: nginx
        name: nginx-service
      spec:
        ports:
          - name: http
            port: 80
            protocol: TCP
            # Expose port 80 on the service
            targetPort: 80
        selector:
          # Link this service to pods with the label app=nginx
          app: nginx
    # NGINX Ingress
    - apiVersion: networking.k8s.io/v1
      kind: Ingress
      metadata:
        name: nginx-ingress
        annotations:
          cert-manager.io/cluster-issuer: letsencrypt-certificate  # Use Let's Encrypt TLS certificates ClusterIssuer
      spec:
        tls:
          - secretName: example.letsencrypt.key.tls  # Secret containing the TLS key and certificate for your_domain_name
            hosts:
              - app.devsys.app
        rules:
          - host: app.devsys.app  # Ingress rule Host
            http:
              paths:
                - path: /  # Path configuration
                  pathType: Prefix
                  backend:
                    service:
                      name: nginx-service  # Name of your nginx service
                      port:
                        number: 80  # Port number exposing nginx service

  ```

- Aplicar el deployment

  ```shell
  kubectl apply -f example-deployment.yml
  ```

- Listar los pods (Debemos ver 3 pods)
  
  ```shell
  kubectl get pods
  ```

- Eliminar aplicación de ejemplo (Opcional)

  ```shell
  kubectl delete -f example-deployment.yml
  ```
