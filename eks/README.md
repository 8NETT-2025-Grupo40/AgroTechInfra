# AgroTechInfra - Setup do Cluster EKS

> Repositório de infraestrutura compartilhada para gerenciar o cluster EKS e add-ons comuns

## Visão Geral

Este repositório gerencia a infraestrutura EKS compartilhada para todas as APIs AgroTech. É responsável por:

- Criar e configurar o cluster EKS
- Instalar add-ons compartilhados (AWS Load Balancer Controller, External Secrets Operator)
- Gerenciar políticas IAM e configurações IRSA
- Criar namespaces compartilhados

**IMPORTANTE:** Este repositório NÃO faz deploy de aplicações. Cada repositório de API (AgroTechUserApi, AgroTechPaymentApi, etc.) gerencia seu próprio deployment.

## Pré-requisitos

Antes de executar o setup, certifique-se de ter:

- **AWS Account ID:** 478511033947
- **Políticas IAM** criadas na AWS (podem ser criadas pelo workflow):
  - `AWSLoadBalancerControllerIAMPolicy`
  - `AgroTechExternalSecretsPolicy`


---

## Setup via GitHub Actions (Recomendado)

1. Acesse GitHub Actions: `https://github.com/8NETT-2025-Grupo40/AgroTechInfra/actions`
2. Selecione o workflow: **"Setup EKS Cluster & Add-ons"**
3. Clique em **"Run workflow"**
4. Configure as opções:
   - `skip_cluster`: Marque se o cluster já existe
   - `skip_addons`: Marque se os add-ons já estão instalados
5. Clique em **"Run workflow"**

**Tempo estimado:** 20-25 minutos


---

## O Que é Criado

### Componentes de Infraestrutura

1. **Cluster EKS**
   - Nome: `agro-tech`
   - Região: `us-east-1`
   - Versão Kubernetes: 1.34
   - VPC: `vpc-0e6d1df089da1ec39` (existente)
   - OIDC provider habilitado

2. **Node Group**
   - Nome: `low-cost`
   - Tipo de instância: `t3a.small`
   - Capacidade desejada: 2 nodes
   - Mín: 2, Máx: 3
   - Zonas de disponibilidade: us-east-1a, us-east-1b

3. **Namespaces**
   - `agro-tech` (para deployments de aplicações)
   - `external-secrets` (para External Secrets Operator)

4. **Políticas IAM** (se não existirem)
   - `AWSLoadBalancerControllerIAMPolicy`
   - `AgroTechExternalSecretsPolicy`

5. **IRSA (IAM Roles for Service Accounts)**
   - Service account do AWS Load Balancer Controller
   - Vinculado à AWSLoadBalancerControllerIAMPolicy

6. **Add-ons Helm**
   - AWS Load Balancer Controller (namespace kube-system)
   - External Secrets Operator (namespace external-secrets)

---

## Estrutura do Repositório

```
AgroTechInfra/
├── .github/
│   └── workflows/
│       ├── cluster-setup.yml     # Workflow GitHub Actions para criação do cluster
│       └── cluster-destroy.yml   # Workflow GitHub Actions para deleção do cluster
├── eks/
│   ├── cluster-config.yaml       # Configuração do cluster eksctl
│   └── README.md                 # Este arquivo
├── iam/
│   ├── alb-controller-policy.json        # Política IAM para ALB Controller
│   └── external-secrets-policy.json      # Política IAM para External Secrets
└── README.md                     # Documentação principal do repositório
```

---

---


## Verificação Após Setup

Após o setup bem-sucedido, verifique a instalação:

```powershell
# Verificar nodes do cluster
kubectl get nodes

# Verificar AWS Load Balancer Controller
kubectl get deployment -n kube-system aws-load-balancer-controller

# Verificar External Secrets Operator
kubectl get deployment -n external-secrets external-secrets

# Verificar namespaces
kubectl get namespace agro-tech
kubectl get namespace external-secrets

# Verificar ausência de recursos de aplicação (esperado)
kubectl get all -n agro-tech
# No resources found in agro-tech namespace.
```

**Nota:** Neste ponto, NÃO deve haver recursos Ingress ou ALB. Estes serão criados pelos deployments individuais das APIs.

---

## Deleção do Cluster

1. Acesse: `https://github.com/8NETT-2025-Grupo40/AgroTechInfra/actions`
2. Selecione o workflow: **"Destroy EKS Cluster"**
3. Clique em **"Run workflow"**
4. Digite **"DESTROY"** no campo de confirmação
5. Clique em **"Run workflow"**


### O Que é Deletado

- Cluster EKS e node groups
- IAM Service Accounts (roles IRSA)
- OIDC Provider
- Helm releases (Load Balancer Controller, External Secrets Operator)
- Application Load Balancers (se existirem)
- Target Groups

### O Que Permanece

Os seguintes recursos NÃO são deletados e devem ser gerenciados manualmente:

- Políticas IAM (`AWSLoadBalancerControllerIAMPolicy`, `AgroTechExternalSecretsPolicy`)
- VPC e networking (`vpc-0e6d1df089da1ec39`)
- Secrets do AWS Secrets Manager
- Repositórios ECR

---

## Troubleshooting

### Load Balancer Controller Não Está Executando

**Verificar status do deployment:**
```powershell
kubectl get deployment -n kube-system aws-load-balancer-controller
kubectl logs -n kube-system deployment/aws-load-balancer-controller
```

**Causas comuns:**
- IRSA não configurado corretamente
- Política IAM ausente ou incorreta
- VPC ID incompatível

### External Secrets Operator Não Está Executando

**Verificar status do deployment:**
```powershell
kubectl get deployment -n external-secrets external-secrets
kubectl logs -n external-secrets deployment/external-secrets
```

**Causas comuns:**
- CRDs não instalados
- Namespace não criado
- Timeout na instalação do Helm

### Falha na Criação do Cluster

**Verificar logs do eksctl:**
```powershell
eksctl utils describe-stacks --region us-east-1 --cluster agro-tech
```

**Causas comuns:**
- IDs de VPC ou subnet incorretos
- Permissões IAM insuficientes
- Limites de recursos na conta AWS
- Versão do Kubernetes não suportada pelo EKS

### Falha na Criação do IRSA

**Verificar se o OIDC provider existe:**
```powershell
aws eks describe-cluster --name agro-tech --region us-east-1 --query 'cluster.identity.oidc.issuer'
```

**Solução:** Re-executar com flag `--override-existing-serviceaccounts` (já incluída nos scripts)

---

## Configuração do Cluster

### Detalhes do Cluster
- **Nome:** `agro-tech`
- **Região:** `us-east-1`
- **Versão Kubernetes:** `1.34`
- **VPC ID:** `vpc-0e6d1df089da1ec39` (existente)
- **Modo de Autenticação:** `API_AND_CONFIG_MAP`

### Node Group
- **Nome:** `low-cost`
- **Tipo de Instância:** `t3a.small`
- **Capacidade Desejada:** 2 nodes
- **Tamanho Mínimo:** 2
- **Tamanho Máximo:** 3
- **Zonas de Disponibilidade:** us-east-1a, us-east-1b

### Add-ons
- **AWS Load Balancer Controller:** Instalado via Helm no namespace `kube-system`
- **External Secrets Operator:** Instalado via Helm no namespace `external-secrets`

---

## Integração com Repositórios de APIs

Cada repositório de API (AgroTechUserApi, AgroTechPaymentApi, etc.) deve:

1. **Criar seu próprio IRSA** para External Secrets:
   ```bash
   eksctl create iamserviceaccount \
     --cluster=agro-tech \
     --namespace=agro-tech \
     --name=<api-name>-sa \
     --attach-policy-arn=arn:aws:iam::478511033947:policy/AgroTechExternalSecretsPolicy \
     --approve \
     --region=us-east-1
   ```

2. **Usar ALB compartilhado** via annotation no Ingress:
   ```yaml
   metadata:
     annotations:
       alb.ingress.kubernetes.io/group.name: agro-tech
   ```

3. **Fazer deploy no namespace `agro-tech`:**
   ```bash
   helm upgrade --install <release-name> ./charts -n agro-tech
   ```

---

## Melhores Práticas

1. **Controle de Versão:** Sempre commit mudanças no `cluster-config.yaml` no Git
2. **Políticas IAM:** Mantenha as políticas IAM - elas são reutilizáveis em recriações do cluster
3. **Execução em Fases:** Use flags de skip (`skip_cluster`, `skip_addons`) para atualizações parciais
4. **Monitorar Recursos:** Verifique regularmente custos e utilização de recursos do cluster
5. **Documentação:** Atualize este README ao fazer mudanças na infraestrutura

---

## Referências

- [Documentação eksctl](https://eksctl.io/)
- [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)
- [External Secrets Operator](https://external-secrets.io/)
- [Guia de Melhores Práticas EKS](https://aws.github.io/aws-eks-best-practices/)

---

**Última Atualização:** 20 de Novembro de 2025  
**AWS Account ID:** 478511033947  
**Repositório:** AgroTechInfra
