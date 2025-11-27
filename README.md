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

### 1. Criar a instância EC2 para instalar o Rancher
1. #### Grupos de segurança
    * Crie um grupo de segurança, nome sugerido `launch-wizard`
    * Regras de entrada (Inbound rules):
        * SSH:
            * Type: SSH
            * Port: 22
            * Source: seu IP fixo ou 0.0.0.0/0 (para fins de laboratório; em produção use algo mais restrito).
        * HTTP:
            * Type: HTTP
            * Port: 80
            * Source: 0.0.0.0/0
        * HTTPS:
            * Type: HTTPS
            * Port: 443
            * Source: 0.0.0.0/0
        
2. #### Pré-requisitos
    * Conta AWS ativa.
    * Usuário IAM com permissão para:
       * `ec2:*`
       * `iam:*`
       * `eks:*`
       * `ecr:*`
       * `elasticloadbalancing:*`
3. #### Escolher a AMI e o tipo de instância
    1. Acesse EC2 > Instances > Launch instances.
    2. O nome da instancia
    3. Escolha o sistemea operacional (Reomendado `Ubuntu xx.xx lts`)
    4. Tipo de instancia: monimo 2vCPU, 8GiB Memória
    5. Par de chaves: crie ou selecione um par de chaves para acessar a instancia via ssh
    6. Grupo de segurança, selecione o sugerido `launch-wizard` ou crie um com as mesmas caracteristicas

4. #### Storage (disco)
    Na parte de Configure storage:
    * Defina pelo menos:
        * Root volume: 40–60 GB (GP3).
    * Rancher + Docker + logs + imagens podem encher disco rápido, então 30 GB é o mínimo do mínimo; 40 GB é mais confortável.

Apara acessar a isntacia criada via ssh:

``` bash
ssh -i caminho/para/sua-chave.pem ubuntu@SEU_IP_PUBLICO
```

### 2. Instalar Rancher

``` bash
sudo apt update
sudo apt install -y docker.io
sudo systemctl enable --now docker
```

``` bash
docker run -d --restart=unless-stopped   -p 80:80 -p 443:443   --privileged   rancher/rancher:latest
```

1. Verificar se o Rancher está rodando

``` bash
sudo docker ps
```

Você deve ver algo como:

``` bash
CONTAINER ID   IMAGE                STATUS         PORTS
xxxxxxx        rancher/rancher:v2.11.5   Up ...   0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp

```

Se não estiver rodando, reinicie:

``` bash
sudo docker restart <container-id>

```

2. Acessar o Rancher via navegador

   No seu navegador (local), abra:
``` bash
https://SEU_IP_PUBLICO
```

⚠️ Você verá um aviso de conexão insegura, isso é esperado, porque o Rancher está usando um certificado self-signed.
Clique em:
    * “Advanced”
    * “Proceed to …”
    * Ao acessar via navegar a mensagem `API Aggregation not ready` for apresentada, e so agardar a instalação, leva por vota de 3 mim.
    

3. Tela inicial: Definir a senha de administrador
    * No terminal `sudo docker ps` para exibir o id do container
    * Utilize o comando `docker logs  <CONTAINER_ID>  2>&1 | grep "Bootstrap Password:"` para recuperara a senha inicial
    * Na proxima tela defina uma nova senha. o usuário é por padrão é `admin`

4. Instalar o *kubectl*

    No terminal SSH da EC2, execute para baixar:
    
    ``` bash
    curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
    ```
    
    Instalar
    
    ``` bash
    sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
    ```
    
    Testar
    
    
    ``` bash
    kubectl version --client
    ```

### 3. Criçar e permissionar o cluster kubernates

1. Instalar AWS CLI, na EC2 que esta instalado o `Rancher` e o `Docker`

``` bash
sudo apt update
sudo apt install -y unzip
cd /tmp

curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version

```

* Caso não tenha um usuário com as permissões necessárias:
    * Va em `IAM - Users` e crie um usuário com as seguintes permissções
        * AmazonEC2ContainerRegistryPowerUser
        * AmazonEKSClusterPolicy
        * Crie uma nova politica em linha com o nome `EKSBasicCliAccess` e inclua o seguinte `json` 
          ``` json
          {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Effect": "Allow",
                    "Action": [
                        "eks:DescribeCluster",
                        "eks:ListClusters",
                        "eks:ListNodegroups",
                        "eks:DescribeNodegroup"
                    ],
                    "Resource": "*"
                }
            ]
          }
          ```
  Ao criar o usuário você vai obiter a `AWS Access Key ID` e o `AWS Secret Access Key`
  
  * Configurar o `aws`
      ``` bash
      aws configure
      ```
      
      E preencha:
    
      * AWS Access Key ID: `xxxxxxx`
      * AWS Secret Access Key: `xxxxxxx`
      * Region: `região padrao do cluste e do EC2 as duas devem ser a mesma`
      * Default output format: `json`
  
     Confira se está funcionado:

     ``` bash
     aws sts get-caller-identity
     ```
     O resultadado deve ser algo do tipo:

     ``` json
     {
          "UserId": "AIDA...XYZ",
          "Account": "880105240215",
          "Arn": "arn:aws:iam::880105240215:user/azuredevops-ecr-user"
     }
     ```

2. Criar o Cluster (EKS)
   Crie um cluster com configuração personalizadas e apenas em `Complementos` vamos selecionar:
   * Atendente de monitoramento de nós
   * DNS externo
   * kube-proxy
   * Agente de Identidade de Pods do Amazon EKS
   * Servidor de métricas
   * CNI da Amazon VPC
   * DNS externo
   
   em `Grupos de Segurança` selecione o gurpo que foi criado, que tenha acesso a HTTPS, HTTP e SSH
   
   Depois vamos editar outras informações, por agora basta criar para gerar o ID do Cluster
   Aguarde o ativação do cluster, entre 7-15 minutos

    Para facilitar o acesso via terminal pode criar mais um acesso no cluster vinculando o usuário do EC2
    * EKS → Clusters → app-cluster → Aba “Acesso” → botão “Criar”
    * Em ARN da entidade principal do IAM, cole exatamente o ARN do usuário que utiliza para acessar a isntancia do EC2, ou pesquise pelo nome do usuário.
    * Em Tipo, escolha:`Padrão (ou Standard), não precisa ser EC2.`
    * Na etapa Adicionar política de acesso: selecione `AmazonEKSClusterAdminPolicy`
   
4. Criar OIDC Provider
   Agora que seu cluster já foi criado (e está no estado Active), o EKS já gerou automaticamente um endpoint OIDC exclusivo para ele.

    Peque o URL OIDC em `Clusters → app-cluster → Visão Geral → URL do provedor OpenID Connect`
    Exemplo:
   
    ``` bash
    https://oidc.eks.<região>.amazonaws.com/id/<IDENTIFICADOR_UNICO>
    ```
    
    1. Acesse o console IAM: `AWS Console → IAM → Identity Providers`
    2. Clique em: `Adicionar provedor`
    3. Em Provider type, selecione: `OpenID Connect`
    4. Preencher os dados do OIDC
     ``` bash
    https://oidc.eks.<região>.amazonaws.com/id/<IDENTIFICADOR_UNICO>
    ```
    5. e no campo `Publico` preencher:
    ``` bash
    sts.amazonaws.com
    ```
    
5. Criar IAM Role (Web Identity) + policy completa (IAM Policy do LB Controller)
   Precisamos criar a role com um json com todas as premissões necessárias:
   Em `IAM → Políticas → criar Política `, selecione json, apague tudo e inclua o json a baixo:

   ``` json
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Action": [
            "iam:CreateServiceLinkedRole"
          ],
          "Resource": "*",
          "Condition": {
            "StringEquals": {
              "iam:AWSServiceName": "elasticloadbalancing.amazonaws.com"
            }
          }
        },
        {
          "Effect": "Allow",
          "Action": [
            "ec2:DescribeAccountAttributes",
            "ec2:DescribeAddresses",
            "ec2:DescribeAvailabilityZones",
            "ec2:DescribeInternetGateways",
            "ec2:DescribeVpcs",
            "ec2:DescribeVpcPeeringConnections",
            "ec2:DescribeSubnets",
            "ec2:DescribeRouteTables",
            "ec2:DescribeSecurityGroups",
            "ec2:DescribeInstances",
            "ec2:DescribeNetworkInterfaces",
            "ec2:DescribeTags",
            "ec2:GetCoipPoolUsage",
            "ec2:DescribeCoipPools",
            "elasticloadbalancing:DescribeLoadBalancers",
            "elasticloadbalancing:DescribeLoadBalancerAttributes",
            "elasticloadbalancing:DescribeListeners",
            "elasticloadbalancing:DescribeListenerCertificates",
            "elasticloadbalancing:DescribeSSLPolicies",
            "elasticloadbalancing:DescribeRules",
            "elasticloadbalancing:DescribeTargetGroups",
            "elasticloadbalancing:DescribeTargetGroupAttributes",
            "elasticloadbalancing:DescribeTargetHealth",
            "elasticloadbalancing:DescribeTags",
            "elasticloadbalancing:DescribeTrustStores",
            "elasticloadbalancing:DescribeListenerAttributes"
          ],
          "Resource": "*"
        },
        {
          "Effect": "Allow",
          "Action": [
            "cognito-idp:DescribeUserPoolClient",
            "acm:ListCertificates",
            "acm:DescribeCertificate",
            "iam:ListServerCertificates",
            "iam:GetServerCertificate",
            "waf-regional:GetWebACL",
            "waf-regional:GetWebACLForResource",
            "waf-regional:AssociateWebACL",
            "waf-regional:DisassociateWebACL",
            "wafv2:GetWebACL",
            "wafv2:GetWebACLForResource",
            "wafv2:AssociateWebACL",
            "wafv2:DisassociateWebACL",
            "shield:GetSubscriptionState",
            "shield:DescribeProtection",
            "shield:CreateProtection",
            "shield:DeleteProtection"
          ],
          "Resource": "*"
        },
        {
          "Effect": "Allow",
          "Action": [
            "ec2:AuthorizeSecurityGroupIngress",
            "ec2:RevokeSecurityGroupIngress"
          ],
          "Resource": "*"
        },
        {
          "Effect": "Allow",
          "Action": [
            "ec2:CreateSecurityGroup"
          ],
          "Resource": "*"
        },
        {
          "Effect": "Allow",
          "Action": [
            "ec2:CreateTags"
          ],
          "Resource": "arn:aws:ec2:*:*:security-group/*",
          "Condition": {
            "StringEquals": {
              "ec2:CreateAction": "CreateSecurityGroup"
            },
            "Null": {
              "aws:RequestTag/elbv2.k8s.aws/cluster": "false"
            }
          }
        },
        {
          "Effect": "Allow",
          "Action": [
            "ec2:CreateTags",
            "ec2:DeleteTags"
          ],
          "Resource": "arn:aws:ec2:*:*:security-group/*",
          "Condition": {
            "Null": {
              "aws:RequestTag/elbv2.k8s.aws/cluster": "true",
              "aws:ResourceTag/elbv2.k8s.aws/cluster": "false"
            }
          }
        },
        {
          "Effect": "Allow",
          "Action": [
            "ec2:AuthorizeSecurityGroupIngress",
            "ec2:RevokeSecurityGroupIngress",
            "ec2:DeleteSecurityGroup"
          ],
          "Resource": "*",
          "Condition": {
            "Null": {
              "aws:ResourceTag/elbv2.k8s.aws/cluster": "false"
            }
          }
        },
        {
          "Effect": "Allow",
          "Action": [
            "elasticloadbalancing:CreateLoadBalancer",
            "elasticloadbalancing:CreateTargetGroup"
          ],
          "Resource": "*",
          "Condition": {
            "Null": {
              "aws:RequestTag/elbv2.k8s.aws/cluster": "false"
            }
          }
        },
        {
          "Effect": "Allow",
          "Action": [
            "elasticloadbalancing:CreateListener",
            "elasticloadbalancing:DeleteListener",
            "elasticloadbalancing:CreateRule",
            "elasticloadbalancing:DeleteRule"
          ],
          "Resource": "*"
        },
        {
          "Effect": "Allow",
          "Action": [
            "elasticloadbalancing:AddTags",
            "elasticloadbalancing:RemoveTags"
          ],
          "Resource": [
            "arn:aws:elasticloadbalancing:*:*:targetgroup/*/*",
            "arn:aws:elasticloadbalancing:*:*:loadbalancer/net/*/*",
            "arn:aws:elasticloadbalancing:*:*:loadbalancer/app/*/*"
          ],
          "Condition": {
            "Null": {
              "aws:RequestTag/elbv2.k8s.aws/cluster": "true",
              "aws:ResourceTag/elbv2.k8s.aws/cluster": "false"
            }
          }
        },
        {
          "Effect": "Allow",
          "Action": [
            "elasticloadbalancing:AddTags",
            "elasticloadbalancing:RemoveTags"
          ],
          "Resource": [
            "arn:aws:elasticloadbalancing:*:*:listener/net/*/*/*",
            "arn:aws:elasticloadbalancing:*:*:listener/app/*/*/*",
            "arn:aws:elasticloadbalancing:*:*:listener-rule/net/*/*/*",
            "arn:aws:elasticloadbalancing:*:*:listener-rule/app/*/*/*"
          ]
        },
        {
          "Effect": "Allow",
          "Action": [
            "elasticloadbalancing:ModifyLoadBalancerAttributes",
            "elasticloadbalancing:SetIpAddressType",
            "elasticloadbalancing:SetSecurityGroups",
            "elasticloadbalancing:SetSubnets",
            "elasticloadbalancing:DeleteLoadBalancer",
            "elasticloadbalancing:ModifyTargetGroup",
            "elasticloadbalancing:ModifyTargetGroupAttributes",
            "elasticloadbalancing:DeleteTargetGroup"
          ],
          "Resource": "*",
          "Condition": {
            "Null": {
              "aws:ResourceTag/elbv2.k8s.aws/cluster": "false"
            }
          }
        },
        {
          "Effect": "Allow",
          "Action": [
            "elasticloadbalancing:AddTags"
          ],
          "Resource": [
            "arn:aws:elasticloadbalancing:*:*:targetgroup/*/*",
            "arn:aws:elasticloadbalancing:*:*:loadbalancer/net/*/*",
            "arn:aws:elasticloadbalancing:*:*:loadbalancer/app/*/*"
          ],
          "Condition": {
            "StringEquals": {
              "elasticloadbalancing:CreateAction": [
                "CreateTargetGroup",
                "CreateLoadBalancer"
              ]
            },
            "Null": {
              "aws:RequestTag/elbv2.k8s.aws/cluster": "false"
            }
          }
        },
        {
          "Effect": "Allow",
          "Action": [
            "elasticloadbalancing:RegisterTargets",
            "elasticloadbalancing:DeregisterTargets"
          ],
          "Resource": "arn:aws:elasticloadbalancing:*:*:targetgroup/*/*"
        },
        {
          "Effect": "Allow",
          "Action": [
            "elasticloadbalancing:SetWebAcl",
            "elasticloadbalancing:ModifyListener",
            "elasticloadbalancing:AddListenerCertificates",
            "elasticloadbalancing:RemoveListenerCertificates",
            "elasticloadbalancing:ModifyRule",
            "elasticloadbalancing:SetRulePriorities"
          ],
          "Resource": "*"
        }
      ]
    }

   ```
   De um nome para o plitica, por exemplo `AWSLBControllerIAMPolicy`:
   
   Agora vamos criar a `Funçao (role)` - IRSA do LB Controller - em `IAM → funções → criar perfil `:
   1. selecionar o tipo `Identidade Web`
   2. selecionar o provider criado
   3. selecionar o `Audience`: `sts.amazonaws.com`
   4. selecione a politica que foi criada: `AWSLBControllerIAMPolicy`
   5. de um nome para a `função`, por exemplo:`AmazonEKSLBControllerRole`

    Agora vamos Criar Role EC2 (Node Group)

   * Tipo de entidade confiável: `Serviço da AWS`
   * Serviço: `EC2`

    Clique em Avançar. Adicionar permissões essenciais da role EC2 do EKS

    * AmazonEKSWorkerNodePolicy
    * AmazonEKS_CNI_Policy
    * AmazonEC2ContainerRegistryReadOnly

    Nome da role `AmazonEKSNodeRole`    
   
7. Criar o Node Group `EKS → selecione o seu cluster → computação → grupo de nós → Adicionar grupo de nós`.
   1. nome `app-node-group`
   2. selecione a função que foi criada, no caso `AmazonEKSNodeRole`
   3. Configure as maquinas e nos de acordo com a necessidade
   4. Finalize a criação

   configure o `kubeclt` para acessar os nodes

    ``` bash
    aws eks update-kubeconfig \
      --region us-east-1 \
      --name app-cluster
    ```
9. Criar a ServiceAccount vinculada ao OIDC (kube-system/aws-load-balancer-controller)
    1. Instalar o Helm na EC2 (se ainda não tiver)
    ``` bash
    curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

    helm version
    ```
    Se o segundo comando mostrar a versão, Helm ok.
    2. Criar a ServiceAccount com a Role IRSA
    Vamos criar um YAML simples só pra ServiceAccount do controller, ligando ela à role `AmazonEKSLBControllerRole`.
    Ajuste apenas se o nome/ARN da role for diferente.
    ``` bash
    kubectl apply -f -  <<EOF | kubectl apply -f -
        apiVersion: v1
        kind: ServiceAccount
        metadata:
          name: aws-load-balancer-controller
          namespace: kube-system
          annotations:
            eks.amazonaws.com/role-arn: arn:aws:iam::880105240215:role/AmazonEKSLBControllerRole
        EOF
    ```
    Confirme:
    ``` bash
    kubectl get sa aws-load-balancer-controller -n kube-system -o yaml | grep -A2 annotations
    ```
    3. Instalar o AWS Load Balancer Controller via Helm
    * Instalar o AWS Load Balancer Controller via Helm
    ``` bash
    helm repo add eks https://aws.github.io/eks-charts
    helm repo update
    ```
    * Instalar/atualizar o controller: Coloque os nomes corretos para `clusterName`, `region` e `vpcId`(no painel da aws da pra pegar o vpcId no cluster)
    ``` bash
    helm upgrade -i aws-load-balancer-controller eks/aws-load-balancer-controller \
      -n kube-system \
      --set clusterName=app-cluster \
      --set region=us-east-1 \
      --set vpcId=vpc-017d0a7bf7e549b25 \
      --set serviceAccount.create=false \
      --set serviceAccount.name=aws-load-balancer-controller
    ```
    * Verificar se subiu:
    ``` bash
    kubectl get pods -n kube-system | grep aws-load-balancer
    ```
    Você deve ver algo como:
    ``` bash
    aws-load-balancer-controller-xxxxx   1/1   Running   ...
    ```
    
10. Deploy do kubesystem (manifesto do AWS Load Balancer Controller)

Precisa de um certificado para esse kube-infra funcionar, Pode criar um certificado em `ACM` na aws.
* crie o sertificado para o seu domino
* no dominio precisa colocar o dns do LoadBalance para que o certificado seja valido, da pra cirar um LB temporário pra isso
* quando o Certificado estiver criado, recupere o id e inclua aqui `<ID_CERTIFICADO>`
* remove o LB que foi criado de teste
```
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: <DOMINIO_EMPRESA>
  namespace: kube-system
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/group.name: <DOMINIO_EMPRESA>
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80,"HTTPS":443}]'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:<AWS_REGION>:<AWS_ACCOUNT_ID>:certificate/<ID_CERTIFICADO>
    alb.ingress.kubernetes.io/target-type: ip
spec:
  ingressClassName: alb
  rules:
    - host: dummy.project-zero.work
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: kube-dns
                port:
                  number: 53
EOF
```

apos a criação desse kube de infra. um LB será criado

11. Deploy do YAML dos seus apps (Ingress)
12. Configurar DNS *.dominio no Cloudflare → apontar para o ALB gerado
