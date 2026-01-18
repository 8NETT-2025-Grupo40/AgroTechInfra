# AgroTech Infrastructure - Cluster EKS e Recursos Compartilhados

Este repositório gerencia a **infraestrutura compartilhada** para a plataforma AgroTech, incluindo o cluster EKS, add-ons e recursos comuns utilizados por todos os serviços de API.

## Estrutura do Repositório

```
agro-tech-infra/
├── .github/workflows/
│   ├── cluster-setup.yml      # Create/update EKS cluster
│   └── cluster-destroy.yml    # Destroy EKS cluster
├── eks/
│   ├── cluster-config.yaml    # eksctl cluster configuration
│   └── README.md              # EKS documentation
├── iam/
│   ├── alb-controller-policy.json                  # IAM policy for ALB Controller
│   ├── external-secrets-policy.json                # IAM policy for External Secrets
│   └── cloudwatch-application-signals-policy.json  # IAM policy for CloudWatch
└── README.md                  # This file
```

## Responsabilidades

Este repositório é responsável por:

**Gerenciamento do Cluster EKS**
- Criar/deletar o cluster `agro-tech` em `us-east-1`
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
- `agro-tech`: Namespace principal para todos os serviços de API
- `external-secrets`: Para External Secrets Operator

**NÃO é Responsável Por**
- Código das aplicações (reside nos repositórios individuais de cada API)
- Helm charts específicos de aplicações (cada API gerencia o seu próprio)
- Valores de secrets específicos de aplicações (gerenciados por cada time de API)
- Repositórios ECR (cada API cria/gerencia o seu próprio)

## Uso

### Pré-requisitos

- AWS Account: `478511033947`
- AWS Region: `us-east-1`
- VPC Existente: `vpc-0e6d1df089da1ec39`
- GitHub Secrets configurados:
  - `AWS_ACCESS_KEY_ID`
  - `AWS_SECRET_ACCESS_KEY`

### GitHub Actions (Recomendado)

#### Criar/Atualizar Cluster

1. Acesse **Actions** → **Setup EKS Cluster & Add-ons**
2. Clique em **Run workflow**
3. Configure as opções:
   - `skip_cluster`: Marque se o cluster já existe
   - `skip_addons`: Marque se os add-ons já estão instalados
4. Clique em **Run workflow** e monitore o progresso (~15-20 minutos)

#### Destruir Cluster

1. Acesse **Actions** → **Destroy EKS Cluster**
2. Clique em **Run workflow**
3. Digite **`DESTROY`** no campo de confirmação
4. Clique em **Run workflow** e monitore o progresso (~10-15 minutos)

**AVISO**: Isso deleta o cluster inteiro e todos os deployments!


## Configuração do Cluster

### Detalhes do Cluster

- **Nome**: `agro-tech`
- **Região**: `us-east-1`
- **Versão Kubernetes**: `1.34`
- **Account ID**: `478511033947`
- **VPC**: `vpc-0e6d1df089da1ec39`
- **Modo de Autenticação**: `API_AND_CONFIG_MAP`

### Node Group

- **Nome**: `low-cost`
- **Tipo de Instância**: `t3a.small`
- **Capacidade Desejada**: 2 nodes
- **Tamanho Mínimo**: 2 nodes
- **Tamanho Máximo**: 3 nodes
- **Zonas de Disponibilidade**: `us-east-1a`, `us-east-1b`

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
  --cluster-name agro-tech \
  --addon-name amazon-cloudwatch-observability \
  --region us-east-1

# Pods do CloudWatch
kubectl get pods -n amazon-cloudwatch
kubectl get daemonsets -n amazon-cloudwatch

# Verificar uso de memória (importante para t3a.small)
kubectl top nodes
```

### Configurar Retenção de Logs (Novos Log Groups)

Quando você instrumenta novas aplicações, novos log groups podem ser criados. 

**Dica:** Re-execute o workflow `Setup EKS Cluster & Add-ons` com `skip_cluster=true` para aplicar retenção automaticamente.

## Integração com Serviços de API

### O Que Cada Repositório de API Deve Fazer

Cada serviço de API (ex: `AgroTechUserApi`, `AgroTechPaymentApi`) é responsável por:

1. **Build e Push da Imagem Docker para ECR**
   ```bash
   docker build -t <account>.dkr.ecr.us-east-1.amazonaws.com/<api-name>:latest .
   docker push <account>.dkr.ecr.us-east-1.amazonaws.com/<api-name>:latest
   ```

2. **Criar IRSA (IAM Role for Service Account)**
   ```bash
   eksctl create iamserviceaccount \
     --cluster=agro-tech \
     --namespace=agro-tech \
     --name=<api-name>-sa \
     --attach-policy-arn=arn:aws:iam::478511033947:policy/AgroTechExternalSecretsPolicy \
     --approve \
     --override-existing-serviceaccounts \
     --region=us-east-1
   ```

3. **Deploy via Helm no namespace `agro-tech`**
   ```bash
   helm upgrade --install <api-name> ./k8s \
     --namespace agro-tech \
     --set image.tag=<version> \
     --wait
   ```

4. **Configurar Ingress com ALB Compartilhado**
   ```yaml
   annotations:
     alb.ingress.kubernetes.io/group.name: agro-tech  # Compartilha ALB com outras APIs
     alb.ingress.kubernetes.io/target-type: ip
   ```

### Pré-requisitos para Deploy de API

Antes de fazer deploy de qualquer API, certifique-se:

- Cluster EKS `agro-tech` existe e está rodando
- Namespace `agro-tech` existe
- AWS Load Balancer Controller está instalado
- External Secrets Operator está instalado
- Repositório ECR para a API existe
- Secrets do AWS Secrets Manager estão criados (ex: `agro-tech-api-<name>-connection-string`)

### Exemplo: Deploy da User API

Veja o [repositório AgroTechUserApi](https://github.com/8NETT-2025-Grupo40/AgroTechUserApi) para um exemplo completo de:
- Estrutura do Helm chart (diretório `k8s/`)
- Workflow GitHub Actions para deployment no EKS
- Configuração IRSA para External Secrets

## Políticas IAM

### AWSLoadBalancerControllerIAMPolicy

**ARN**: `AWSLoadBalancerControllerIAMPolicy`

**Propósito**: Permitir que o ALB Controller gerencie Application Load Balancers

**Vinculado a**: ServiceAccount `aws-load-balancer-controller` no `kube-system`

### AgroTechExternalSecretsPolicy

**ARN**: `AgroTechExternalSecretsPolicy`

**Propósito**: Permitir leitura de secrets do AWS Secrets Manager

**Permissões**:
- `secretsmanager:GetSecretValue`
- `secretsmanager:DescribeSecret`

**Recursos**:
- `agro-tech-api-user-connection-string*`
- `agro-tech-jwt-config*`
- (Adicionar mais conforme necessário para outras APIs)

**Vinculado a**: ServiceAccount de cada API (criado durante o deployment da API)

### CloudWatchApplicationSignalsPolicy

**ARN**: `CloudWatchApplicationSignalsPolicy`

**Propósito**: Permitir envio de telemetria para CloudWatch e X-Ray

**Permissões**:
- `logs:PutLogEvents`, `logs:CreateLogGroup`, `logs:PutRetentionPolicy`
- `xray:PutTraceSegments`, `xray:PutTelemetryRecords`
- `cloudwatch:PutMetricData`
- `ec2:Describe*` (para metadata)
- `eks:DescribeCluster`

**Vinculado a**: ServiceAccount `cloudwatch-agent` no `amazon-cloudwatch`

## Troubleshooting

### Cluster Não Está Criando

```bash
# Verificar versão do eksctl
eksctl version

# Validar configuração do cluster
eksctl create cluster -f eks/cluster-config.yaml --dry-run

# Verificar credenciais AWS
aws sts get-caller-identity
```

### ALB Não Está Provisionando

```bash
# Verificar logs do ALB Controller
kubectl logs -n kube-system deployment/aws-load-balancer-controller

# Verificar status do Ingress
kubectl describe ingress -n agro-tech <ingress-name>

# Verificar annotation IRSA do ServiceAccount
kubectl get sa -n kube-system aws-load-balancer-controller -o yaml
```

### External Secrets Não Está Sincronizando

```bash
# Verificar logs do External Secrets Operator
kubectl logs -n external-secrets deployment/external-secrets

# Verificar status do SecretStore
kubectl get secretstore -n agro-tech
kubectl describe secretstore -n agro-tech <secretstore-name>

# Verificar status do ExternalSecret
kubectl get externalsecret -n agro-tech
kubectl describe externalsecret -n agro-tech <externalsecret-name>
```

### Nodes Não Estão Ready

```bash
# Verificar status dos nodes
kubectl get nodes

# Descrever node
kubectl describe node <node-name>

# Verificar node group no eksctl
eksctl get nodegroup --cluster=agro-tech --region=us-east-1
```

## Monitoramento e Manutenção

### Verificar Saúde do Cluster

```bash
# Obter informações do cluster
aws eks describe-cluster --name agro-tech --region us-east-1

# Verificar nodes
kubectl get nodes

# Verificar pods do sistema
kubectl get pods -n kube-system
kubectl get pods -n external-secrets
kubectl get pods -n amazon-cloudwatch

# Verificar todos os deployments no namespace agro-tech
kubectl get deployments -n agro-tech

# Verificar uso de recursos (importante para t3a.small)
kubectl top nodes
kubectl top pods -n agro-tech
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