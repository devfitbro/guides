# Instalación Vault HashiCorp en Kubernetes

- Crear espacio dedicado para vault:

  ```shell
  kubectl create namespace vault
  ```

- Indicar con que archivo de configuración HELM debe usar para conectarse (Nosotros estamos usando k3s):

  ```shell
  export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
  ```

- Añadir repositorio de vault hashicorp a Helm:

  ```shell
  helm repo add hashicorp https://helm.releases.hashicorp.com
  ```

- Validar que se tiene acceso al repo:

  ```shell
  helm search repo hashicorp/vault
  ```

- Crear archivo de configuración `helm-vault-raft-values.yaml`:

  ```shell
  server:
    affinity: ""
    ha:
      enabled: true
      raft:
        enabled: true
        setNodeId: true
        config: |
          cluster_name = "vault-integrated-storage"
          storage "raft" {
              path    = "/vault/data/"
          }

          listener "tcp" {
              address = "[::]:8200"
              cluster_address = "[::]:8201"
              tls_disable = "true"
          }
          service_registration "kubernetes" {}
  ```

- Instalar:

  ```shell
  helm install vault hashicorp/vault \
    --set='server.dev.enabled=true' \
    --set='ui.enabled=true' \
    --set='ui.serviceType=LoadBalancer' \
    --namespace vault \
    --values helm-vault-raft-values.yml \
    --namespace vault
  ```

- Conectarse al pod:

  ```shell
  kubectl exec -it vault-0 -n vault -- /bin/sh
  ```

- Dentro del pod crear y aplicar una política de lectura:

  ```shell
  cat <<EOF > /home/vault/read-policy.hcl
  path "secret*" {
    capabilities = ["read"]
  }
  EOF
  ```

  ```shell
  vault policy write read-policy /home/vault/read-policy.hcl
  ```

- Habilitar autenticación de kubernetes

  ```shell
  vault auth enable kubernetes
  ```

- Configurar autenticación de kubernetes

  ```shell
  vault write auth/kubernetes/config \
   token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
   kubernetes_host=https://${KUBERNETES_PORT_443_TCP_ADDR}:443 \
   kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
  ```

- Crear un rol:

  ```shell
  vault write auth/kubernetes/role/vault-role \
   bound_service_account_names=vault-serviceaccount \
   bound_service_account_namespaces=vault \
   policies=read-policy \
   ttl=1h
  ```

- Ingresar y crear secretos:

  A través de la consola o del UI (con puerto 8200) se puede ingresar para crear secretos.
