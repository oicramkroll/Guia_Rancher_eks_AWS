# install-AWS-LB-controller-eks.md

Guia para instalação do **AWS Load Balancer Controller** em um cluster **Amazon EKS** usando **IRSA (IAM Roles for Service Accounts)**.

> Ajuste sempre os placeholders:
> - `<AWS_ACCOUNT_ID>`
> - `<AWS_REGION>`
> - `<CLUSTER_NAME>`
> - `<VPC_ID>`
> - `<LB_ROLE_NAME>`
> - `<ID_CERTIFICADO>`
> - `<DOMINIO_EMPRESA>`

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

  Mas tive que incluir algumas politicas a mais, então segue o json completo:

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

## 8. Para definir o certificado e utilizar *.seudominio.com.br

1. Crei um certificado na aws (ACM)
2. Crie um ingress infra no seu EC2

``` bash
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: <DOMINIO_EMPRESA>
  namespace: app-dev
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/group.name: <DOMINIO_EMPRESA>
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80,"HTTPS":443}]'
    alb.ingress.kubernetes.io/ssl-redirect: '443'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:<AWS_REGION>:<AWS_ACCOUNT_ID>:certificate/<ID_CERTIFICADO>
spec:
  ingressClassName: alb
  rules:
    - host: SEU_APP.<DOMINIO_EMPRESA>
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
