# AgroTech Infrastructure - Cluster EKS e Recursos Compartilhados

Este repositório gerencia a **infraestrutura compartilhada** para a plataforma AgroTech, incluindo o cluster EKS, add-ons e recursos comuns utilizados por todos os serviços de API.

## Estrutura do Repositório

```
agro-tech-infra/
├── .github/workflows/
│   ├── cluster-setup.yml      # Create/update EKS cluster
│   ├── cluster-destroy.yml    # Destroy EKS cluster
│   └── vpc-setup.yml          # Create VPC and public subnets
├── eks/
│   ├── cluster-config.template.yaml # Template eksctl cluster configuration
│   └── README.md                    # EKS documentation
├── iam/
│   ├── alb-controller-policy.json                  # IAM policy for ALB Controller
│   ├── external-secrets-policy.json                # IAM policy for External Secrets
│   └── cloudwatch-application-signals-policy.json  # IAM policy for CloudWatch
└── README.md                  # This file
```

## Responsabilidades

Este repositório é responsável por:

**Gerenciamento do Cluster EKS**
- Criar/deletar o cluster informado no input `CLUSTER_NAME`
- Gerenciar node groups e políticas de scaling
- Configurar OIDC provider para IRSA (IAM Roles for Service Accounts)

**Add-ons Compartilhados**
- AWS Load Balancer Controller (gerencia ALB para todos os Ingresses)
- External Secrets Operator (sincroniza secrets do AWS Secrets Manager)
- Futuro: Cluster Autoscaler, Metrics Server, etc.

**Políticas IAM**
- `AWSLoadBalancerControllerIAMPolicy`: Para gerenciamento de ALB
- `AgroTechExternalSecretsPolicy`: Para leitura de secrets do Secrets Manager

**Namespaces**
- `K8S_NAMESPACE`: Namespace principal para todos os serviços de API
- `external-secrets`: Para External Secrets Operator


**NÃO é Responsável Por**
- Código das aplicações (reside nos repositórios individuais de cada API)
- Helm charts específicos de aplicações (cada API gerencia o seu próprio)
- Valores de secrets específicos de aplicações (gerenciados por cada time de API)
- Repositórios ECR (cada API cria/gerencia o seu próprio)

## Uso

### Pré-requisitos

- GitHub Secrets configurados:
  - `AWS_ACCESS_KEY_ID`
  - `AWS_SECRET_ACCESS_KEY`

### GitHub Actions (Recomendado)

#### Criar VPC Pública

1. Acesse **Actions** → **Create VPC (Public)**
2. Clique em **Run workflow**
3. Preencha os inputs obrigatórios:
   - `AWS_REGION`: ex. `us-east-1`
   - `VPC_NAME`: ex. `agro-tech-vpc`
   - `CLUSTER_NAME`: ex. `agro-tech`
   - `VPC_CIDR`: ex. `10.0.0.0/16`
   - `PUBLIC_SUBNET_CIDRS`: lista separada por vírgula
   - `PUBLIC_SUBNET_AZS`: lista separada por vírgula
4. Digite **`CREATE`** no campo de confirmação
5. Copie o `VPC_ID` e `PUBLIC_SUBNET_IDS` do resumo do workflow

#### Criar/Atualizar Cluster

1. Acesse **Actions** → **Setup EKS Cluster & Add-ons**
2. Clique em **Run workflow**
3. Preencha os inputs obrigatórios:
   - `AWS_REGION`: ex. `us-east-1`
   - `CLUSTER_NAME`: ex. `agro-tech`
   - `VPC_ID`: ex. `vpc-0123456789abcdef0`
   - `PUBLIC_SUBNET_IDS`: lista separada por vírgula
   - `K8S_NAMESPACE`: ex. `agro-tech`
   - `SECRETS_PREFIX`: ex. `agro-tech`
   - `NODE_INSTANCE_TYPE`: ex. `t3a.medium`
   - `NODE_MIN`: ex. `2`
   - `NODE_DESIRED`: ex. `3`
   - `NODE_MAX`: ex. `10`
4. Configure as opções:
   - `skip_cluster`: Marque se o cluster já existe
   - `skip_addons`: Marque se os add-ons já estão instalados
5. Clique em **Run workflow** e monitore o progresso (~15-20 minutos)

#### Destruir Cluster

1. Acesse **Actions** → **Destroy EKS Cluster**
2. Clique em **Run workflow**
3. Preencha `AWS_REGION` e `CLUSTER_NAME`
4. Digite **`DESTROY`** no campo de confirmação
5. Clique em **Run workflow** e monitore o progresso (~10-15 minutos)

**AVISO**: Isso deleta o cluster inteiro e todos os deployments!



## Configuração do Cluster

### Detalhes do Cluster

- **Nome**: input `CLUSTER_NAME`
- **Região**: input `AWS_REGION`
- **Versão Kubernetes**: `1.34`
- **Account ID**: derivado via STS no workflow
- **VPC**: input `VPC_ID`
- **Modo de Autenticação**: `API_AND_CONFIG_MAP`


### Node Group

- **Nome**: `low-cost`
- **Tipo de Instância**: input `NODE_INSTANCE_TYPE`
- **Capacidade Desejada**: input `NODE_DESIRED`
- **Tamanho Mínimo**: input `NODE_MIN`
- **Tamanho Máximo**: input `NODE_MAX`
- **Subnets**: input `PUBLIC_SUBNET_IDS`


### Add-ons Instalados

1. **AWS Load Balancer Controller** (namespace `kube-system`)
   - Gerencia ALB para Ingresses do Kubernetes
   - ServiceAccount: `aws-load-balancer-controller`
   - IRSA vinculado à `AWSLoadBalancerControllerIAMPolicy`

2. **External Secrets Operator** (namespace `external-secrets`)
   - Sincroniza secrets do AWS Secrets Manager para Kubernetes
   - Cada API cria seu próprio SecretStore com IRSA

3. **CloudWatch Application Signals** (namespace `amazon-cloudwatch`)
   - Observabilidade automática com métricas, traces e logs
   - Usa ADOT (AWS Distro for OpenTelemetry) para coleta
   - ServiceAccount: `cloudwatch-agent`
   - IRSA vinculado à `CloudWatchApplicationSignalsPolicy`
   - Retenção de logs: **7 dias**

## CloudWatch Application Signals

O cluster possui observabilidade automática via CloudWatch Application Signals, que fornece:
- **Service Map**: Visualização de dependências entre serviços
- **Traces Distribuídos**: Rastreamento de requests através dos serviços
- **Métricas Automáticas**: Latência, erros, throughput
- **SLO Monitoring**: Monitoramento de objetivos de nível de serviço

### Estratégia de Rollout (Recomendada)

1. **Começar com 1 serviço não-crítico** - Adicione a annotation de instrumentação
2. **Monitorar por 24-48h** - Verifique uso de memória com `kubectl top nodes`
3. **Validar telemetria** - Acesse CloudWatch Console → Application Signals
4. **Expandir gradualmente** - Instrumente os demais serviços

### Verificar Status do Application Signals

```bash
# Status do add-on
aws eks describe-addon \
  --cluster-name <CLUSTER_NAME> \
  --addon-name amazon-cloudwatch-observability \
  --region <AWS_REGION>

# Pods do CloudWatch
kubectl get pods -n amazon-cloudwatch
kubectl get daemonsets -n amazon-cloudwatch

# Verificar uso de memória (importante para t3a.medium)
kubectl top nodes
kubectl top pods -n <K8S_NAMESPACE>

# Nota: o namespace principal depende do input K8S_NAMESPACE

```

### Atualizar Add-ons

```bash
# Atualizar repositórios Helm
helm repo update

# Atualizar ALB Controller
helm upgrade aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --reuse-values

# Atualizar External Secrets Operator
helm upgrade external-secrets external-secrets/external-secrets \
  -n external-secrets \
  --reuse-values
```

## Documentação

- [Guia de Setup EKS](eks/README.md)
- [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)
- [External Secrets Operator](https://external-secrets.io/)
- [CloudWatch Application Signals](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Application-Monitoring-Sections.html)
- [Documentação eksctl](https://eksctl.io/)
