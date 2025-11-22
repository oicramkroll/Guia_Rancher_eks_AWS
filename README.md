# Rancher + EKS: Guia Definitivo para Importação Perfeita

**Versão otimizada --- funciona de primeira**

## 🧩 Requisitos

-   AWS conta com permissões para EC2, IAM e EKS\
-   Ubuntu EC2 para o Rancher\
-   Docker instalado\
-   Usuário IAM com permissões EKS\
-   EKS com NodeGroup ativo

## 🏗 Arquitetura Ideal

    [Internet]
         |
         v
    [EC2 Rancher]  <-- Security Group com portas 22/80/443 liberadas
         |
         v
    [EKS Control Plane] <-- SG aceita tráfego do SG da EC2 do Rancher
         |
         v
    [Node Group]

## 🚀 Passo a Passo Completo

### 1. Criar EC2 para o Rancher

-   t3.small ou t3.medium\
-   Disco: 20--30 GB\
-   SG:
    -   22: 0.0.0.0/0
    -   80: 0.0.0.0/0
    -   443: 0.0.0.0/0

### 2. Instalar Rancher

``` bash
sudo apt update
sudo apt install -y docker.io
sudo systemctl enable --now docker
```

``` bash
docker run -d --restart=unless-stopped   -p 80:80 -p 443:443   --privileged   rancher/rancher:latest
```

### 3. Criar Cluster EKS

Criar o cluster e o NodeGroup.

### 4. Ajustar Security Group do EKS

Adicionar no SG do EKS:

  Porta   Origem
  ------- -------------------------------
  443     SG da EC2 onde está o Rancher

### 5. Criar usuário IAM com acesso ao EKS

Adicionar política:

-   AmazonEKSClusterAdminPolicy

### 6. Configurar kubectl na EC2 do Rancher

``` bash
aws eks update-kubeconfig --name <CLUSTER_NAME> --region <REGION>
kubectl get nodes
```

### 7. Importar o cluster no Rancher

``` bash
kubectl apply -f https://<rancher-ip>/v3/import/<token>.yaml
```

Se estiver usando certificado self-signed:

``` bash
curl --insecure -sfL https://<rancher-ip>/v3/import/<token>.yaml | kubectl apply -f -
```

### 8. Verificar até ficar ACTIVE

``` bash
State: Active
Connected: True
Ready: True
```
  Se aparecer Provisioning → Active, está OK.

------------------------------------------------------------------------

## 🛠 Resolução de Problemas

### "Unable to connect to server"

→ SG do EKS bloqueando tráfego

### "the server has asked for the client to provide credentials"

→ IAM sem permissão cluster-admin

### "cluster not found"

→ Token antigo

### x509 certificate unknown authority

→ Rancher self-signed → usar `--insecure`

------------------------------------------------------------------------

## 📋 Checklist Final

-   EC2 criada com SG correto\
-   Docker e Rancher instalados\
-   EKS criado\
-   SG do EKS configurado\
-   IAM com acesso cluster-admin\
-   kubectl funcionando\
-   Importação concluída
