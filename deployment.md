# Guia de Deployment - APIs GoodWe

Este guia fornece instruções completas para fazer o deployment das APIs GoodWe em diferentes ambientes: desenvolvimento, produção, Docker e nuvem.

## 📋 Visão Geral do Deployment

O projeto GoodWe possui duas APIs principais que podem ser executadas independentemente ou em conjunto:

- **API Excel** (Node.js) - Porta 3000
- **API Machine Learning** (Python) - Porta 8000

## 🚀 Quick Start - Desenvolvimento Local

### Pré-requisitos
- **Node.js** >= 18.0.0
- **Python** >= 3.11
- **NPM** ou **Yarn**
- **Git**

### 1. API Excel (Node.js)

```bash
# Clone o repositório
git clone <repo-url>
cd Etapa5_GoodWe/API_GoodWe

# Instalar dependências
npm install

# Configurar variáveis de ambiente (opcional)
cp .env.example .env

# Iniciar em modo desenvolvimento
npm run dev

# Ou iniciar em produção
npm start
```

**Verificação**: http://localhost:3000

### 2. API Machine Learning (Python)

```bash
cd ../Machine_learning_GoodWe

# Criar ambiente virtual (recomendado)
python -m venv venv

# Ativar ambiente virtual
# Windows:
venv\Scripts\activate
# Linux/Mac:
source venv/bin/activate

# Instalar dependências
pip install -r requirements.txt

# Treinar modelo (se necessário)
python train_model.py

# Iniciar API
python start_api.py
```

**Verificação**: http://localhost:8000/docs

## 🐳 Deployment com Docker

### Docker Individual

#### API Excel
```bash
cd API_GoodWe

# Build da imagem
docker build -t api-goodwe .

# Executar container
docker run -d \
  --name api-goodwe-container \
  -p 3000:3000 \
  -v $(pwd)/scr/data:/app/scr/data \
  api-goodwe
```

#### API Machine Learning
```bash
cd Machine_learning_GoodWe

# Build da imagem
docker build -t ml-api .

# Executar container
docker run -d \
  --name ml-api-container \
  -p 8000:8000 \
  ml-api
```

### Docker Compose - Ambiente Completo

#### API Excel com PostgreSQL e Redis
```bash
cd API_GoodWe

# Iniciar todos os serviços
docker-compose up -d

# Verificar status
docker-compose ps

# Ver logs
docker-compose logs -f api-goodwe

# Parar serviços
docker-compose down
```

**Serviços incluídos**:
- API Excel (porta 3000)
- PostgreSQL (porta 5432)
- Redis (porta 6379)
- Nginx (portas 80/443)

#### API Machine Learning
```bash
cd Machine_learning_GoodWe

# Iniciar serviço
docker-compose up -d

# Verificar logs
docker-compose logs -f power-outage-api
```

## ⚙️ Configurações de Ambiente

### Variáveis de Ambiente - API Excel

```bash
# .env
NODE_ENV=production
PORT=3000

# Banco de dados
DB_HOST=localhost
DB_PORT=5432
DB_NAME=goodwe_db
DB_USER=goodwe_user
DB_PASS=goodwe_password

# Segurança
API_KEY=your_api_key_here
JWT_SECRET=your_super_secret_jwt_key

# Logs
LOG_LEVEL=info
```

### Variáveis de Ambiente - API ML

```bash
# .env
HOST=0.0.0.0
PORT=8000
RELOAD=false
LOG_LEVEL=info

# Modelo
MODEL_PATH=./power_outage_model.pkl
FEATURES_PATH=./features.json

# Performance
MAX_BATCH_SIZE=100
WORKERS=4
```

## 🌐 Deployment em Produção

### 1. Servidor Linux (Ubuntu/Debian)

#### Preparar Servidor
```bash
# Atualizar sistema
sudo apt update && sudo apt upgrade -y

# Instalar dependências
sudo apt install -y nginx postgresql redis-server curl wget

# Instalar Node.js (via NodeSource)
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt-get install -y nodejs

# Instalar Python 3.11
sudo apt install -y python3.11 python3.11-venv python3-pip
```

#### Deploy API Excel
```bash
# Criar usuário para aplicação
sudo useradd -m -s /bin/bash goodwe

# Clonar código
sudo -u goodwe git clone <repo> /home/goodwe/app
cd /home/goodwe/app/API_GoodWe

# Instalar dependências
sudo -u goodwe npm ci --production

# Configurar serviço systemd
sudo tee /etc/systemd/system/api-goodwe.service > /dev/null <<EOF
[Unit]
Description=API GoodWe Excel
After=network.target

[Service]
Type=simple
User=goodwe
WorkingDirectory=/home/goodwe/app/API_GoodWe
ExecStart=/usr/bin/node index.js
Restart=always
RestartSec=10
Environment=NODE_ENV=production
Environment=PORT=3000

[Install]
WantedBy=multi-user.target
EOF

# Iniciar serviço
sudo systemctl enable api-goodwe
sudo systemctl start api-goodwe
sudo systemctl status api-goodwe
```

#### Deploy API ML
```bash
# Preparar ambiente Python
sudo -u goodwe python3.11 -m venv /home/goodwe/ml-env
sudo -u goodwe /home/goodwe/ml-env/bin/pip install -r requirements.txt

# Treinar modelo
sudo -u goodwe /home/goodwe/ml-env/bin/python train_model.py

# Configurar serviço systemd
sudo tee /etc/systemd/system/ml-api.service > /dev/null <<EOF
[Unit]
Description=Machine Learning API
After=network.target

[Service]
Type=simple
User=goodwe
WorkingDirectory=/home/goodwe/app/Machine_learning_GoodWe
ExecStart=/home/goodwe/ml-env/bin/python api.py
Restart=always
RestartSec=10
Environment=HOST=0.0.0.0
Environment=PORT=8000

[Install]
WantedBy=multi-user.target
EOF

# Iniciar serviço
sudo systemctl enable ml-api
sudo systemctl start ml-api
sudo systemctl status ml-api
```

### 2. Configurar Nginx

```nginx
# /etc/nginx/sites-available/goodwe-apis
server {
    listen 80;
    server_name api.goodwe.com;

    # API Excel
    location /excel/ {
        proxy_pass http://localhost:3000/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }

    # API Machine Learning
    location /ml/ {
        proxy_pass http://localhost:8000/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }

    # Health checks
    location /health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }
}
```

```bash
# Ativar site
sudo ln -s /etc/nginx/sites-available/goodwe-apis /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

## ☁️ Deployment em Nuvem

### AWS (Amazon Web Services)

#### 1. EC2 Instance
```bash
# Criar instância EC2 Ubuntu 22.04
# Security Groups:
# - SSH (22) - Seu IP
# - HTTP (80) - 0.0.0.0/0
# - HTTPS (443) - 0.0.0.0/0
# - Custom (3000) - VPC
# - Custom (8000) - VPC

# Conectar via SSH
ssh -i your-key.pem ubuntu@ec2-xx-xx-xx-xx.compute-1.amazonaws.com

# Seguir passos de deployment em servidor Linux
```

#### 2. Docker com ECS
```yaml
# docker-compose.aws.yml
version: '3.8'
services:
  api-excel:
    image: your-account.dkr.ecr.region.amazonaws.com/api-goodwe:latest
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - DB_HOST=your-rds-endpoint
    
  ml-api:
    image: your-account.dkr.ecr.region.amazonaws.com/ml-api:latest
    ports:
      - "8000:8000"
    environment:
      - HOST=0.0.0.0
      - PORT=8000
```

### Google Cloud Platform

#### Cloud Run
```yaml
# cloudbuild.yaml
steps:
  # Build API Excel
  - name: 'gcr.io/cloud-builders/docker'
    args: ['build', '-t', 'gcr.io/$PROJECT_ID/api-goodwe', './API_GoodWe']
  
  # Build ML API  
  - name: 'gcr.io/cloud-builders/docker'
    args: ['build', '-t', 'gcr.io/$PROJECT_ID/ml-api', './Machine_learning_GoodWe']

  # Deploy to Cloud Run
  - name: 'gcr.io/cloud-builders/gcloud'
    args: ['run', 'deploy', 'api-goodwe', '--image', 'gcr.io/$PROJECT_ID/api-goodwe', '--region', 'us-central1']
```

### Azure Container Instances

```bash
# Criar grupo de recursos
az group create --name goodwe-rg --location eastus

# Deploy API Excel
az container create \
  --resource-group goodwe-rg \
  --name api-goodwe \
  --image api-goodwe:latest \
  --ports 3000 \
  --dns-name-label api-goodwe \
  --environment-variables NODE_ENV=production

# Deploy ML API
az container create \
  --resource-group goodwe-rg \
  --name ml-api \
  --image ml-api:latest \
  --ports 8000 \
  --dns-name-label ml-goodwe \
  --environment-variables HOST=0.0.0.0 PORT=8000
```

## 🔐 SSL/HTTPS com Let's Encrypt

```bash
# Instalar Certbot
sudo apt install certbot python3-certbot-nginx

# Obter certificado
sudo certbot --nginx -d api.goodwe.com

# Renovação automática
sudo crontab -e
# Adicionar linha:
0 12 * * * /usr/bin/certbot renew --quiet
```

## 📊 Monitoramento e Logs

### 1. Logs de Sistema
```bash
# API Excel
sudo journalctl -u api-goodwe -f

# API ML
sudo journalctl -u ml-api -f

# Nginx
sudo tail -f /var/log/nginx/access.log
sudo tail -f /var/log/nginx/error.log
```

### 2. Health Checks
```bash
#!/bin/bash
# health-check.sh

# Verificar API Excel
if curl -f http://localhost:3000/health > /dev/null 2>&1; then
    echo "API Excel: OK"
else
    echo "API Excel: FAIL"
    # Restart se necessário
    sudo systemctl restart api-goodwe
fi

# Verificar API ML
if curl -f http://localhost:8000/health > /dev/null 2>&1; then
    echo "API ML: OK"
else
    echo "API ML: FAIL"
    # Restart se necessário
    sudo systemctl restart ml-api
fi
```

### 3. Prometheus + Grafana (Opcional)
```yaml
# monitoring/docker-compose.yml
version: '3.8'
services:
  prometheus:
    image: prom/prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana
    ports:
      - "3001:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
```

## 🚨 Backup e Recuperação

### Backup de Dados
```bash
#!/bin/bash
# backup.sh

# Backup arquivos Excel
tar -czf backup-$(date +%Y%m%d).tar.gz \
  /home/goodwe/app/API_GoodWe/scr/data/

# Backup banco PostgreSQL
pg_dump -h localhost -U goodwe_user goodwe_db > backup-db-$(date +%Y%m%d).sql

# Upload para S3 (opcional)
aws s3 cp backup-$(date +%Y%m%d).tar.gz s3://your-backup-bucket/
```

### Recuperação
```bash
# Restaurar arquivos
tar -xzf backup-20250120.tar.gz -C /home/goodwe/app/API_GoodWe/scr/data/

# Restaurar banco
psql -h localhost -U goodwe_user goodwe_db < backup-db-20250120.sql
```

## 📝 Checklist de Deployment

### Pré-deployment
- [ ] Testes locais passando
- [ ] Variáveis de ambiente configuradas
- [ ] Dependências atualizadas
- [ ] Documentação atualizada
- [ ] Backup de dados atual

### Durante deployment
- [ ] Serviços parados graciosamente
- [ ] Código atualizado
- [ ] Dependências instaladas
- [ ] Configurações aplicadas
- [ ] Serviços reiniciados

### Pós-deployment
- [ ] Health checks passando
- [ ] APIs respondendo corretamente
- [ ] Logs sem erros críticos
- [ ] Performance dentro do esperado
- [ ] Monitoramento funcionando

---

**Próxima seção**: [Exemplos de Uso](./examples.md)
