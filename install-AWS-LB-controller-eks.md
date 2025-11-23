# install-AWS-LB-controller-eks.md

Guia para instalação do **AWS Load Balancer Controller** em um cluster **Amazon EKS** usando **IRSA (IAM Roles for Service Accounts)**.

> Ajuste sempre os placeholders:
> - `<AWS_ACCOUNT_ID>`
> - `<AWS_REGION>`
> - `<CLUSTER_NAME>`
> - `<VPC_ID>`
> - `<LB_ROLE_NAME>`

---

## 0. Pré-requisitos

- Cluster EKS já criado.
- `kubectl` configurado para o cluster:
  ```bash
  aws eks update-kubeconfig --name <CLUSTER_NAME> --region <AWS_REGION>
  ```
- `awscli` instalado e autenticado.
- `helm` instalado:
  ```bash
  curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
  ```

---

## 1. Criar a IAM Policy `AWSLoadBalancerControllerIAMPolicy`

  1. Acesse IAM → Policies → Create policy.
  2. Aba JSON: cole o JSON oficial da policy do AWS Load Balancer Controller (disponível na documentação oficial da AWS).
  
    https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.7.1/docs/install/iam_policy.json
    
  3. Clique em Next, dê o nome `AWSLoadBalancerControllerIAMPolicy`, Conclua em Create policy.
  4. Essa policy contém as permissões para o controller criar/atualizar/excluir:
      - Load Balancers (ALB/NLB)
      - Target Groups
      - Listeners
      - Regras de roteamento
      - Descrições de recursos EC2/VPC/IAM necessários

---

## 2. Criar o OIDC Provider do EKS

  Se o cluster ainda não tiver OIDC habilitado:
  1. Acesse EKS → Clusters → <CLUSTER_NAME>.
  2. Vá em Configuration → Authentication.
  3. Na seção OIDC provider, clique em Associate identity provider (ou similar) ou, em algumas UIs, o atalho leva para o IAM:
     - Em IAM → Identity providers → Add provider:
       - Provider type: OpenID Connect
       - Provider URL: URL OIDC do cluster EKS (campo mostrado na tela do cluster).
       - Audience: `sts.amazonaws.com`
  4. Confirme a criação.

  Esse provedor permite que o EKS use tokens OIDC para assumir uma IAM Role via `sts.amazonaws.com` (IRSA).

---

## 3. Criar a IAM Role do Controller

  1. Vá em IAM → Roles → Create role.
  2. Em Trusted entity type, escolha Web identity.
  3. Selecione:
    - Identity provider: o provider OIDC criado no passo anterior.
    - Audience: sts.amazonaws.com.
  4. Avance para permissões e anexe a policy criada no passo 1:
    - AWSLoadBalancerControllerIAMPolicy
  5. (Opcional, mas recomendado) Em Add permissions boundary / Edit trust policy, ajuste a trust policy para restringir ao ServiceAccount específico:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<AWS_ACCOUNT_ID>:oidc-provider/<OIDC_PROVIDER_HOSTPATH>"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "<OIDC_PROVIDER_HOSTPATH>:sub": "system:serviceaccount:kube-system:aws-load-balancer-controller",
          "<OIDC_PROVIDER_HOSTPATH>:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
```
    
  Substitua <OIDC_PROVIDER_HOSTPATH> pelo valor exibido no console (ex.: oidc.eks.<AWS_REGION>.amazonaws.com/id/XXXXXXXXXXXX).
  
  6. Dê um nome para a role, por exemplo:
    - Name: <LB_ROLE_NAME> (ex.: EKS-AWSLoadBalancerController-Role).
  7. Crie a role.
    
---

## 4. Criar o ServiceAccount `aws-load-balancer-controller` (kube-system)

```bach
kubectl apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: aws-load-balancer-controller
  namespace: kube-system
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<AWS_ACCOUNT_ID>:role/<LB_ROLE_NAME>
EOF

```
  - Troque <AWS_ACCOUNT_ID> pelo ID da conta.
  - Troque <LB_ROLE_NAME> pelo nome da role criada no passo 3.

Para conferir:

```bach
kubectl get sa aws-load-balancer-controller -n kube-system -o yaml
```
---

## 5. Instalar o `aws-load-balancer-controller` via Helm

  1. Adicionar o repositório Helm do EKS.

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update
```

  2. Descobrir o VPC:

```bash
VPC_ID=$(aws eks describe-cluster \
  --name <CLUSTER_NAME> \
  --region <AWS_REGION> \
  --query "cluster.resourcesVpcConfig.vpcId" \
  --output text)

echo "VPC ID = $VPC_ID"

```

  Se quiser fixar manualmente, use diretamente o valor retornado, por exemplo:
  
```bash
VPC_ID=<VPC_ID>
```

  3. Instalar o chart do `aws-load-balancer-controller`
  
```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=<CLUSTER_NAME> \
  --set region=<AWS_REGION> \
  --set vpcId=$VPC_ID \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller

```

  Explicando os parâmetros principais:
  
  - `-n kube-system` → instala no namespace kube-system.
  - `clusterName` → nome do cluster EKS.
  - `region` → região do cluster.
  - `vpcId` → VPC onde o cluster está rodando.
  - `serviceAccount.create=false` → não cria outra SA.
  - `serviceAccount.name=aws-load-balancer-controller` → usa a SA criada no passo 4.

---

## 6. Verificação

  1. kubectl get pods -n kube-system | grep aws-load-balancer-controller
  
``` bash
kubectl get pods -n kube-system | grep aws-load-balancer-controller
```

  2. Verificar o release do Helm
  
```
helm list -n kube-system | grep aws-load-balancer-controller
```

  Saída esperada:
``` bash
aws-load-balancer-controller   eks/aws-load-balancer-controller   1   deployed   ...
```

---
## 7. Próximos passos (uso prático)

Com o controller instalado:
  - Crie Ingress ou Services com o tipo esperado (ALB/NLB) e annotations da AWS.
      - O `aws-load-balancer-controller` irá:
          - Criar automaticamente o Load Balancer.
          - Criar Target Groups e Listeners.
          - Registrar os pods como targets.

Use os logs do controller para debug, se necessário:

``` bash
kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
```
Fim do guia de instalação do AWS Load Balancer Controller no EKS.
