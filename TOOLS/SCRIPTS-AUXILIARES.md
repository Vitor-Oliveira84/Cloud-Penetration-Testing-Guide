# Scripts Auxiliares - Cloud Pentesting Toolkit

Ferramentas prontas para usar em engagements.

---

## 1. Cloud Reconnaissance Script (Python)

```python
#!/usr/bin/env python3
"""
cloud-recon.py - Enumeração multi-cloud
Uso: python3 cloud-recon.py --provider aws --project acme
"""

import argparse
import subprocess
import json
import sys

class CloudRecon:
    def __init__(self, provider, project):
        self.provider = provider
        self.project = project
        
    def aws_enum(self):
        """Enumerar recursos AWS"""
        print(f"[*] Enumerando AWS Projeto: {self.project}")
        
        # S3 Buckets
        result = subprocess.run(['aws', 's3', 'ls'], capture_output=True, text=True)
        print("[+] S3 Buckets:")
        for line in result.stdout.split('\n')[:5]:
            if line:
                print(f"    {line}")
        
        # EC2 Instances
        cmd = ['aws', 'ec2', 'describe-instances', 
               '--query', 'Reservations[*].Instances[*].[InstanceId,PublicIpAddress,PrivateIpAddress]',
               '--output', 'table']
        result = subprocess.run(cmd, capture_output=True, text=True)
        print("[+] EC2 Instances:")
        print(result.stdout)
        
        # RDS Databases
        cmd = ['aws', 'rds', 'describe-db-instances',
               '--query', 'DBInstances[*].[DBInstanceIdentifier,Endpoint.Address]',
               '--output', 'table']
        result = subprocess.run(cmd, capture_output=True, text=True)
        print("[+] RDS Databases:")
        print(result.stdout)
        
        # Lambda Functions
        result = subprocess.run(['aws', 'lambda', 'list-functions'],
                              capture_output=True, text=True)
        funcs = json.loads(result.stdout)
        print(f"[+] Lambda Functions: {len(funcs['Functions'])}")
        
        # Secrets Manager
        result = subprocess.run(['aws', 'secretsmanager', 'list-secrets'],
                              capture_output=True, text=True)
        secrets = json.loads(result.stdout)
        print(f"[+] Secrets Manager: {len(secrets['SecretList'])} secrets")
        
    def azure_enum(self):
        """Enumerar recursos Azure"""
        print(f"[*] Enumerando Azure Subscription: {self.project}")
        
        result = subprocess.run(['az', 'resource', 'list', '--query', '[*].[name, type]',
                               '--output', 'table'],
                              capture_output=True, text=True)
        print("[+] Recursos Azure:")
        print(result.stdout[:500])
        
    def gcp_enum(self):
        """Enumerar recursos GCP"""
        print(f"[*] Enumerando GCP Projeto: {self.project}")
        
        # Cloud Storage
        result = subprocess.run(['gsutil', 'ls', f'-p', self.project],
                              capture_output=True, text=True)
        print("[+] Cloud Storage Buckets:")
        for line in result.stdout.split('\n')[:5]:
            if line:
                print(f"    {line}")
        
        # Compute Instances
        cmd = ['gcloud', 'compute', 'instances', 'list',
               f'--project={self.project}',
               '--format=table(name, status, INTERNAL_IP)']
        result = subprocess.run(cmd, capture_output=True, text=True)
        print("[+] Compute Engine Instances:")
        print(result.stdout)

if __name__ == '__main__':
    parser = argparse.ArgumentParser()
    parser.add_argument('--provider', choices=['aws', 'azure', 'gcp'], required=True)
    parser.add_argument('--project', required=True)
    args = parser.parse_args()
    
    recon = CloudRecon(args.provider, args.project)
    
    if args.provider == 'aws':
        recon.aws_enum()
    elif args.provider == 'azure':
        recon.azure_enum()
    elif args.provider == 'gcp':
        recon.gcp_enum()
```

**Uso:**
```bash
python3 cloud-recon.py --provider aws --project my-account
python3 cloud-recon.py --provider azure --project my-subscription
python3 cloud-recon.py --provider gcp --project my-project
```

---

## 2. S3 Public Bucket Finder (Bash)

```bash
#!/bin/bash
# s3-public-finder.sh - Localizar S3 buckets públicos

TARGET_DOMAIN="acme.com"
WORDLIST=("backup" "test" "dev" "prod" "staging" "data" "logs" "archive" "backup2024")

echo "[*] Procurando S3 buckets públicos para: $TARGET_DOMAIN"
echo "[*] Wordlist size: ${#WORDLIST[@]}"

FOUND=0
NOT_FOUND=0

for word in "${WORDLIST[@]}"; do
  # Variação 1: name-word-year
  for year in 2023 2024 2025; do
    BUCKET="${TARGET_DOMAIN//./-}-${word}-${year}"
    
    # Teste rápido
    if timeout 3 curl -s -I "https://${BUCKET}.s3.amazonaws.com/" | grep -q "200\|403"; then
      if curl -s "https://${BUCKET}.s3.amazonaws.com/" 2>/dev/null | grep -q "<"; then
        echo "[+] PÚBLICO: s3://$BUCKET"
        ((FOUND++))
      else
        echo "[?] PRIVADO: s3://$BUCKET"
      fi
    else
      ((NOT_FOUND++))
    fi
  done
done

echo "[*] Resumo:"
echo "    Buckets públicos encontrados: $FOUND"
echo "    Buckets privados/inexistentes: $NOT_FOUND"
```

---

## 3. IAM Permission Scanner (Bash)

```bash
#!/bin/bash
# iam-scanner.sh - Verificar permissões IAM perigosas

echo "[*] Verificando permissões IAM"

# Obter usuário atual
IDENTITY=$(aws sts get-caller-identity)
ACCOUNT=$(echo "$IDENTITY" | jq -r '.Account')
ARN=$(echo "$IDENTITY" | jq -r '.Arn')

echo "[*] Identidade: $ARN"
echo "[*] Conta: $ACCOUNT"

# Testar permissões críticas
DANGEROUS_ACTIONS=(
  "iam:AttachUserPolicy"
  "iam:CreateAccessKey"
  "iam:CreateUser"
  "iam:DeleteUser"
  "iam:PutUserPolicy"
  "sts:AssumeRole"
  "s3:*"
  "ec2:*"
  "rds:*"
)

echo "[?] Testando permissões perigosas..."

for action in "${DANGEROUS_ACTIONS[@]}"; do
  # Simular ação (não executa realmente)
  SERVICE=$(echo $action | cut -d: -f1)
  ACTION=$(echo $action | cut -d: -f2)
  
  case $SERVICE in
    "iam")
      aws iam list-users &>/dev/null && echo "[+] $action ✓" || echo "[-] $action ✗"
      ;;
    "s3")
      aws s3 ls &>/dev/null && echo "[+] $action ✓" || echo "[-] $action ✗"
      ;;
    "ec2")
      aws ec2 describe-instances &>/dev/null && echo "[+] $action ✓" || echo "[-] $action ✗"
      ;;
  esac
done
```

---

## 4. Data Exfiltration Monitor (Python)

```python
#!/usr/bin/env python3
"""
exfil-monitor.py - Detectar padrões de exfiltração
Uso: python3 exfil-monitor.py --account acme --analyze
"""

import argparse
import subprocess
import json
from datetime import datetime, timedelta

class ExfilMonitor:
    def __init__(self, account):
        self.account = account
        
    def check_s3_download_patterns(self, hours=24):
        """Verificar downloads anormais de S3"""
        print(f"[*] Analisando S3 downloads últimas {hours} horas")
        
        # CloudTrail query
        start_time = datetime.utcnow() - timedelta(hours=hours)
        
        cmd = [
            'aws', 'cloudtrail', 'lookup-events',
            '--lookup-attributes', 'AttributeKey=EventName,AttributeValue=GetObject',
            '--start-time', start_time.isoformat(),
            '--max-results', '50'
        ]
        
        result = subprocess.run(cmd, capture_output=True, text=True)
        events = json.loads(result.stdout)
        
        print(f"[+] {len(events['Events'])} GetObject eventos encontrados")
        
        # Agrupar por IP
        ip_counts = {}
        for event in events['Events']:
            ip = event.get('CloudTrailEvent', {}).get('sourceIPAddress', 'Unknown')
            ip_counts[ip] = ip_counts.get(ip, 0) + 1
        
        print("[!] Top IPs com GetObject:")
        for ip, count in sorted(ip_counts.items(), key=lambda x: x[1], reverse=True)[:5]:
            print(f"    {ip}: {count} requests")
            
    def check_bandwidth_anomaly(self):
        """Verificar anomalias de bandwidth"""
        print("[*] Verificando anomalias de bandwidth...")
        
        # CloudWatch metrics
        cmd = [
            'aws', 'cloudwatch', 'get-metric-statistics',
            '--namespace', 'AWS/S3',
            '--metric-name', 'BytesDownloaded',
            '--start-time', datetime.utcnow() - timedelta(days=7),
            '--end-time', datetime.utcnow(),
            '--period', '3600',
            '--statistics', 'Sum'
        ]
        
        result = subprocess.run(cmd, capture_output=True, text=True)
        data = json.loads(result.stdout)
        
        datapoints = data.get('Datapoints', [])
        if datapoints:
            avg_bytes = sum(d['Sum'] for d in datapoints) / len(datapoints)
            max_bytes = max(d['Sum'] for d in datapoints)
            
            print(f"[+] Média: {avg_bytes/1024/1024:.2f} MB/hora")
            print(f"[+] Máximo: {max_bytes/1024/1024:.2f} MB/hora")
            
            if max_bytes > avg_bytes * 5:
                print("[!] ALERTA: Spike de download detectado!")

if __name__ == '__main__':
    parser = argparse.ArgumentParser()
    parser.add_argument('--account', required=True)
    parser.add_argument('--analyze', action='store_true')
    args = parser.parse_args()
    
    monitor = ExfilMonitor(args.account)
    
    if args.analyze:
        monitor.check_s3_download_patterns()
        monitor.check_bandwidth_anomaly()
```

---

## 5. Cloud Privilege Escalation Tester (Bash)

```bash
#!/bin/bash
# priv-esc-tester.sh - Testar possíveis escalações

echo "[*] Testando possíveis escalações de privilégio"

# Testar AttachUserPolicy
echo "[?] Testando iam:AttachUserPolicy..."
if aws iam attach-user-policy --user-name test-user \
    --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess \
    2>/dev/null; then
  echo "[+] ✓ Você CAN attach policies!"
  # Limpar
  aws iam detach-user-policy --user-name test-user \
    --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess
else
  echo "[-] ✗ Sem permissão attach policies"
fi

# Testar CreateAccessKey
echo "[?] Testando iam:CreateAccessKey..."
TEMP_USER="test-user-$$"
if aws iam create-user --user-name "$TEMP_USER" 2>/dev/null; then
  echo "[+] ✓ Você CAN criar usuários!"
  
  if aws iam create-access-key --user-name "$TEMP_USER" 2>/dev/null; then
    echo "[+] ✓ Você CAN criar access keys!"
  fi
  
  # Limpar
  aws iam delete-user --user-name "$TEMP_USER" 2>/dev/null
else
  echo "[-] ✗ Sem permissão criar usuários"
fi

# Testar AssumeRole
echo "[?] Testando sts:AssumeRole..."
if aws iam list-roles --query 'Roles[0].Arn' --output text | xargs -I {} \
    aws sts assume-role --role-arn {} --role-session-name test 2>/dev/null; then
  echo "[+] ✓ Você CAN assumir roles!"
else
  echo "[-] ✗ Sem permissão assumir roles (esperado)"
fi
```

---

## 📦 RESUMO DE TOOLS

| Script | Função | Cloud | Critério |
|---|---|---|---|
| cloud-recon.py | Enumerar todos recursos | AWS/Azure/GCP | Reconhecimento |
| s3-public-finder.sh | Localizar S3 públicos | AWS | S3 exposure |
| iam-scanner.sh | Testar permissões | AWS | IAM enum |
| exfil-monitor.py | Detectar exfiltração | AWS | Detecção |
| priv-esc-tester.sh | Testar escalação | AWS | Privilege Esc |

---

**Usar com responsabilidade!** ⚠️

