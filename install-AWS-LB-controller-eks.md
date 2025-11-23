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

Crie a policy no IAM com o JSON oficial do controller.

---

## 2. Criar o OIDC Provider do EKS

No IAM → Identity Providers → Add Provider  
Provider type: **OpenID Connect**  
Audience: `sts.amazonaws.com`

---

## 3. Criar a IAM Role do Controller

- Tipo: **Web identity**
- Identity provider: OIDC do EKS
- Audience: `sts.amazonaws.com`
- Anexar a policy `AWSLoadBalancerControllerIAMPolicy`
- Ajustar a trust policy para o SA:
  ```
  "sub": "system:serviceaccount:kube-system:aws-load-balancer-controller"
  ```

---

## 4. Criar o ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: aws-load-balancer-controller
  namespace: kube-system
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<AWS_ACCOUNT_ID>:role/<LB_ROLE_NAME>
```

Aplicar:

```bash
kubectl apply -f aws-load-balancer-controller-sa.yaml
```

---

## 5. Instalar o aws-load-balancer-controller via Helm

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update
```

Descobrir o VPC:

```bash
VPC_ID=$(aws eks describe-cluster   --name <CLUSTER_NAME>   --region <AWS_REGION>   --query "cluster.resourcesVpcConfig.vpcId"   --output text)
```

Instalar:

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller   -n kube-system   --set clusterName=<CLUSTER_NAME>   --set region=<AWS_REGION>   --set vpcId=$VPC_ID   --set serviceAccount.create=false   --set serviceAccount.name=aws-load-balancer-controller
```

---

## 6. Verificação

```bash
kubectl get pods -n kube-system | grep aws-load-balancer-controller
```

Deve exibir 2 pods em execução.

---

Fim do guia.
