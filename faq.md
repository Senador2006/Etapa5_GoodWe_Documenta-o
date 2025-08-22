# FAQ - Perguntas Frequentes

Esta seção responde às dúvidas mais comuns sobre as APIs GoodWe, desde questões básicas até problemas avançados de implementação.

## 📋 Índice de Perguntas

### [Questões Gerais](#questões-gerais)
- [O que são as APIs GoodWe?](#o-que-são-as-apis-goodwe)
- [Qual a diferença entre as duas APIs?](#qual-a-diferença-entre-as-duas-apis)
- [Posso usar apenas uma das APIs?](#posso-usar-apenas-uma-das-apis)

### [Instalação e Configuração](#instalação-e-configuração)
- [Quais são os pré-requisitos?](#quais-são-os-pré-requisitos)
- [Como instalar em Windows?](#como-instalar-em-windows)
- [Como configurar variáveis de ambiente?](#como-configurar-variáveis-de-ambiente)

### [API Excel](#api-excel)
- [Quais formatos de Excel são suportados?](#quais-formatos-de-excel-são-suportados)
- [Qual o tamanho máximo de arquivo?](#qual-o-tamanho-máximo-de-arquivo)
- [Como lidar com caracteres especiais?](#como-lidar-com-caracteres-especiais)

### [API Machine Learning](#api-machine-learning)
- [Como o modelo foi treinado?](#como-o-modelo-foi-treinado)
- [Posso retreinar o modelo?](#posso-retreinar-o-modelo)
- [Qual a precisão do modelo?](#qual-a-precisão-do-modelo)

### [Deployment e Produção](#deployment-e-produção)
- [Como fazer deploy em produção?](#como-fazer-deploy-em-produção)
- [Como configurar HTTPS?](#como-configurar-https)
- [Como fazer backup dos dados?](#como-fazer-backup-dos-dados)

### [Integração e Desenvolvimento](#integração-e-desenvolvimento)
- [Como integrar com outras aplicações?](#como-integrar-com-outras-aplicações)
- [Existe SDK ou biblioteca cliente?](#existe-sdk-ou-biblioteca-cliente)
- [Como implementar autenticação?](#como-implementar-autenticação)

### [Performance e Escalabilidade](#performance-e-escalabilidade)
- [Como otimizar performance?](#como-otimizar-performance)
- [Como escalar horizontalmente?](#como-escalar-horizontalmente)
- [Qual o throughput máximo?](#qual-o-throughput-máximo)

### [Troubleshooting](#troubleshooting)
- [API não responde](#api-não-responde)
- [Erro de memória](#erro-de-memória)
- [Problemas de conectividade](#problemas-de-conectividade)

---

## Questões Gerais

### O que são as APIs GoodWe?

As APIs GoodWe são um conjunto de serviços REST desenvolvidos para:

1. **API Excel**: Conversão de arquivos Excel (.xls/.xlsx) para formato JSON
2. **API Machine Learning**: Predição de quedas de energia baseada em condições climáticas

Foram desenvolvidas para facilitar a análise de dados energéticos e predição de riscos em sistemas elétricos.

### Qual a diferença entre as duas APIs?

| Característica | API Excel | API ML |
|----------------|-----------|--------|
| **Tecnologia** | Node.js + Express | Python + FastAPI |
| **Função Principal** | Conversão de dados | Predição de riscos |
| **Entrada** | Arquivos Excel | Dados climáticos JSON |
| **Saída** | Dados JSON estruturados | Probabilidades e níveis de risco |
| **Porta Padrão** | 3000 | 8000 |

### Posso usar apenas uma das APIs?

Sim! As APIs são independentes e podem ser usadas separadamente:

- **Apenas API Excel**: Para projetos focados em conversão de planilhas
- **Apenas API ML**: Para sistemas que já possuem dados estruturados
- **Ambas**: Para fluxo completo (Excel → JSON → Predições)

---

## Instalação e Configuração

### Quais são os pré-requisitos?

**Para API Excel (Node.js)**:
- Node.js >= 18.0.0
- NPM >= 8.0.0
- 512MB RAM mínimo
- 100MB espaço em disco

**Para API ML (Python)**:
- Python >= 3.11
- 2GB RAM mínimo (recomendado 4GB)
- 500MB espaço em disco
- Bibliotecas: scikit-learn, pandas, fastapi

**Para Docker**:
- Docker >= 20.0.0
- Docker Compose >= 2.0.0
- 4GB RAM mínimo
- 2GB espaço em disco

### Como instalar em Windows?

**Método 1 - Instalação Nativa**:
```cmd
# 1. Instalar Node.js
# Baixar de: https://nodejs.org/

# 2. Instalar Python
# Baixar de: https://www.python.org/

# 3. Clonar repositório
git clone <repo-url>

# 4. API Excel
cd API_GoodWe
npm install
npm start

# 5. API ML (em outro terminal)
cd Machine_learning_GoodWe
pip install -r requirements.txt
python start_api.py
```

**Método 2 - Docker (Recomendado)**:
```cmd
# 1. Instalar Docker Desktop
# Baixar de: https://www.docker.com/products/docker-desktop

# 2. Executar
docker-compose up -d
```

### Como configurar variáveis de ambiente?

**API Excel (.env)**:
```bash
NODE_ENV=production
PORT=3000
LOG_LEVEL=info

# Database (opcional)
DB_HOST=localhost
DB_PORT=5432
DB_NAME=goodwe
DB_USER=user
DB_PASS=password

# Security
API_KEY=your-secret-key
JWT_SECRET=your-jwt-secret
```

**API ML (.env)**:
```bash
HOST=0.0.0.0
PORT=8000
LOG_LEVEL=info
WORKERS=4

# Model settings
MODEL_PATH=./power_outage_model.pkl
FEATURES_PATH=./features.json
MAX_BATCH_SIZE=100
```

---

## API Excel

### Quais formatos de Excel são suportados?

✅ **Suportados**:
- `.xls` (Excel 97-2003)
- `.xlsx` (Excel 2007+)
- Planilhas com múltiplas abas
- Fórmulas (convertidas para valores)
- Formatação básica

❌ **Não suportados**:
- `.csv` (use conversão manual)
- Arquivos protegidos por senha
- Macros (VBA)
- Gráficos e imagens

### Qual o tamanho máximo de arquivo?

**Padrão**: 10MB

**Como aumentar**:
```javascript
// Multer configuration
const upload = multer({
  limits: {
    fileSize: 50 * 1024 * 1024 // 50MB
  }
});
```

```nginx
# Nginx configuration
client_max_body_size 50M;
```

**Recomendações**:
- Arquivos > 50MB: considere dividir em partes menores
- Para arquivos muito grandes: use processamento assíncrono

### Como lidar com caracteres especiais?

**Problema comum**: Acentos e caracteres especiais aparecem incorretamente.

**Solução**:
```javascript
// Especificar encoding correto
const workbook = XLSX.readFile(filePath, {
  codepage: 65001 // UTF-8
});

// Para arquivos antigos
const workbook = XLSX.readFile(filePath, {
  codepage: 1252 // Windows-1252
});
```

**Detecção automática**:
```javascript
function detectEncoding(buffer) {
  // Tentar UTF-8 primeiro
  try {
    const decoded = buffer.toString('utf8');
    if (!/�/.test(decoded)) return 'utf8';
  } catch (e) {}
  
  // Fallback para Latin1
  return 'latin1';
}
```

---

## API Machine Learning

### Como o modelo foi treinado?

**Dataset**: 50.000 registros sintéticos de condições climáticas

**Features utilizadas**:
1. Temperatura (°C): -10 a 50
2. Umidade (%): 0 a 100
3. Precipitação (mm/h): 0 a 100
4. Velocidade do vento (km/h): 0 a 150
5. Pressão atmosférica (hPa): 950 a 1050

**Algoritmo**: Random Forest Classifier
- 100 árvores
- Max depth: 10
- Min samples split: 5

**Métrica de avaliação**: Acurácia de 76.7%

### Posso retreinar o modelo?

**Sim!** Para retreinar:

```bash
cd Machine_learning_GoodWe

# 1. Preparar novos dados (CSV format)
# Colunas: temperatura_celsius, umidade_pct, precipitacao_mm_h, vento_kmh, pressao_hpa, queda_energia

# 2. Atualizar script de treinamento
python train_model.py --data new_data.csv

# 3. Reiniciar API
python start_api.py
```

**Customizar modelo**:
```python
# train_model.py
from sklearn.ensemble import RandomForestClassifier

# Ajustar hiperparâmetros
model = RandomForestClassifier(
    n_estimators=200,        # Mais árvores
    max_depth=15,           # Maior profundidade
    min_samples_split=3,    # Menos amostras para split
    random_state=42
)
```

### Qual a precisão do modelo?

**Métricas atuais**:
- **Acurácia geral**: 76.7%
- **Precisão classe "queda"**: 78.2%
- **Recall classe "queda"**: 74.5%
- **F1-score**: 76.3%

**Fatores que afetam precisão**:
- Qualidade dos dados de treinamento
- Representatividade das condições locais
- Balanceamento entre classes

**Como melhorar**:
1. Coletar mais dados reais
2. Adicionar features (estação do ano, histórico)
3. Usar ensemble de modelos
4. Calibrar probabilidades

---

## Deployment e Produção

### Como fazer deploy em produção?

**Opção 1 - Docker (Recomendado)**:
```bash
# 1. Preparar ambiente
git clone <repo>
cd projeto

# 2. Configurar secrets
echo "db_password" > secrets/db_password.txt
echo "api_key" > secrets/api_key.txt

# 3. Deploy
docker-compose -f docker-compose.yml up -d

# 4. Verificar
docker-compose ps
curl http://localhost:3000/health
curl http://localhost:8000/health
```

**Opção 2 - Servidor Linux**:
```bash
# 1. Instalar dependências
sudo apt update
sudo apt install nodejs npm python3 python3-pip nginx

# 2. Configurar serviços systemd
sudo systemctl enable api-goodwe
sudo systemctl enable ml-api

# 3. Configurar Nginx
sudo cp nginx/goodwe.conf /etc/nginx/sites-available/
sudo ln -s /etc/nginx/sites-available/goodwe.conf /etc/nginx/sites-enabled/

# 4. Iniciar
sudo systemctl start api-goodwe ml-api nginx
```

### Como configurar HTTPS?

**Com Let's Encrypt (Gratuito)**:
```bash
# 1. Instalar Certbot
sudo apt install certbot python3-certbot-nginx

# 2. Obter certificado
sudo certbot --nginx -d api.seudominio.com

# 3. Renovação automática
sudo crontab -e
# Adicionar: 0 12 * * * /usr/bin/certbot renew --quiet
```

**Configuração Nginx**:
```nginx
server {
    listen 443 ssl http2;
    server_name api.seudominio.com;
    
    ssl_certificate /etc/letsencrypt/live/api.seudominio.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.seudominio.com/privkey.pem;
    
    # SSL Settings
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    
    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Como fazer backup dos dados?

**Script de backup automático**:
```bash
#!/bin/bash
# backup.sh

DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backup/goodwe"

# Criar diretório
mkdir -p $BACKUP_DIR

# Backup PostgreSQL
docker exec postgres-goodwe pg_dump -U goodwe goodwe > $BACKUP_DIR/db_$DATE.sql

# Backup arquivos Excel
tar -czf $BACKUP_DIR/excel_data_$DATE.tar.gz ./API_GoodWe/scr/data/

# Backup modelo ML
tar -czf $BACKUP_DIR/ml_models_$DATE.tar.gz ./Machine_learning_GoodWe/*.pkl ./Machine_learning_GoodWe/*.json

# Upload para cloud (opcional)
aws s3 cp $BACKUP_DIR/ s3://meu-bucket/backup/ --recursive

# Limpeza (manter últimos 7 dias)
find $BACKUP_DIR -name "*.sql" -mtime +7 -delete
find $BACKUP_DIR -name "*.tar.gz" -mtime +7 -delete

echo "Backup concluído: $BACKUP_DIR"
```

**Agendamento no cron**:
```bash
# Backup diário às 2h
0 2 * * * /scripts/backup.sh

# Backup de logs semanalmente
0 3 * * 0 /scripts/backup_logs.sh
```

---

## Integração e Desenvolvimento

### Como integrar com outras aplicações?

**1. REST API Direto**:
```python
import requests

# Converter Excel
with open('dados.xlsx', 'rb') as f:
    response = requests.post('http://api.company.com/excel/upload', 
                           files={'excel': f})
    
excel_data = response.json()

# Fazer predição
weather = {
    "temperatura_celsius": 25.0,
    "umidade_pct": 70.0,
    "precipitacao_mm_h": 10.0,
    "vento_kmh": 40.0,
    "pressao_hpa": 1010.0
}

prediction = requests.post('http://api.company.com/ml/predict', 
                         json=weather).json()
```

**2. SDK Personalizado**:
```python
# goodwe_sdk.py
class GoodWeSDK:
    def __init__(self, base_url, api_key=None):
        self.base_url = base_url
        self.api_key = api_key
        self.session = requests.Session()
        
        if api_key:
            self.session.headers.update({
                'Authorization': f'Bearer {api_key}'
            })
    
    def convert_excel(self, file_path):
        with open(file_path, 'rb') as f:
            response = self.session.post(
                f'{self.base_url}/excel/upload',
                files={'excel': f}
            )
        return response.json()
    
    def predict_outage(self, weather_data):
        response = self.session.post(
            f'{self.base_url}/ml/predict',
            json=weather_data
        )
        return response.json()

# Uso
sdk = GoodWeSDK('http://api.company.com', 'your-api-key')
result = sdk.convert_excel('data.xlsx')
prediction = sdk.predict_outage(weather_data)
```

### Existe SDK ou biblioteca cliente?

Atualmente não há SDK oficial, mas você pode criar um facilmente:

**JavaScript/Node.js**:
```javascript
// goodwe-client.js
class GoodWeClient {
    constructor(baseUrl, apiKey) {
        this.baseUrl = baseUrl;
        this.apiKey = apiKey;
    }
    
    async uploadExcel(filePath) {
        const FormData = require('form-data');
        const fs = require('fs');
        
        const form = new FormData();
        form.append('excel', fs.createReadStream(filePath));
        
        const response = await fetch(`${this.baseUrl}/excel/upload`, {
            method: 'POST',
            body: form,
            headers: this.apiKey ? {
                'Authorization': `Bearer ${this.apiKey}`
            } : {}
        });
        
        return response.json();
    }
    
    async predictOutage(weatherData) {
        const response = await fetch(`${this.baseUrl}/ml/predict`, {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
                ...(this.apiKey && {'Authorization': `Bearer ${this.apiKey}`})
            },
            body: JSON.stringify(weatherData)
        });
        
        return response.json();
    }
}

module.exports = GoodWeClient;
```

### Como implementar autenticação?

**JWT Token Authentication**:

**1. Gerar token**:
```javascript
// auth.js
const jwt = require('jsonwebtoken');

function generateToken(user) {
    return jwt.sign(
        { 
            user_id: user.id, 
            email: user.email,
            role: user.role 
        },
        process.env.JWT_SECRET,
        { expiresIn: '24h' }
    );
}
```

**2. Middleware de autenticação**:
```javascript
// middleware/auth.js
function authenticateToken(req, res, next) {
    const authHeader = req.headers['authorization'];
    const token = authHeader && authHeader.split(' ')[1];
    
    if (!token) {
        return res.status(401).json({ error: 'Token necessário' });
    }
    
    jwt.verify(token, process.env.JWT_SECRET, (err, user) => {
        if (err) {
            return res.status(403).json({ error: 'Token inválido' });
        }
        req.user = user;
        next();
    });
}

// Aplicar em rotas protegidas
app.get('/excel/convert', authenticateToken, convertExcel);
```

**3. Rate Limiting**:
```javascript
const rateLimit = require('express-rate-limit');

const limiter = rateLimit({
    windowMs: 15 * 60 * 1000, // 15 minutos
    max: 100, // Máximo 100 requisições por IP
    message: 'Muitas requisições, tente novamente em 15 minutos'
});

app.use('/api/', limiter);
```

---

## Performance e Escalabilidade

### Como otimizar performance?

**1. Cache Redis**:
```javascript
const redis = require('redis');
const client = redis.createClient();

async function getCachedResult(key) {
    const cached = await client.get(key);
    return cached ? JSON.parse(cached) : null;
}

async function setCachedResult(key, data, ttl = 3600) {
    await client.setex(key, ttl, JSON.stringify(data));
}

// Usar no endpoint
app.post('/ml/predict', async (req, res) => {
    const cacheKey = `prediction:${JSON.stringify(req.body)}`;
    
    let result = await getCachedResult(cacheKey);
    if (!result) {
        result = await makePrediction(req.body);
        await setCachedResult(cacheKey, result);
    }
    
    res.json(result);
});
```

**2. Conexão Keep-Alive**:
```javascript
const http = require('http');

const agent = new http.Agent({
    keepAlive: true,
    maxSockets: 50
});

// Usar em requests HTTP
```

**3. Compressão**:
```javascript
const compression = require('compression');
app.use(compression());
```

### Como escalar horizontalmente?

**1. Load Balancer com Nginx**:
```nginx
upstream api_backend {
    server localhost:3001;
    server localhost:3002;
    server localhost:3003;
}

server {
    location / {
        proxy_pass http://api_backend;
    }
}
```

**2. Docker Swarm**:
```yaml
version: '3.8'
services:
  api-excel:
    image: goodwe/api-excel:1.0.0
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
```

**3. Kubernetes**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: goodwe-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: goodwe-api
  template:
    metadata:
      labels:
        app: goodwe-api
    spec:
      containers:
      - name: api
        image: goodwe/api-excel:1.0.0
        ports:
        - containerPort: 3000
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```

### Qual o throughput máximo?

**Testes de benchmark realizados**:

**API Excel**:
- Conversão de arquivo 1MB: ~500ms
- Upload simultâneo: 10 arquivos/segundo
- Throughput máximo: ~50 req/s

**API ML**:
- Predição individual: ~50ms
- Predição batch (100 itens): ~200ms
- Throughput máximo: ~200 req/s

**Fatores limitantes**:
- CPU para processamento ML
- I/O para leitura de arquivos Excel
- Memória para arquivos grandes

**Otimizações testadas**:
- Cache Redis: +40% performance
- Múltiplos workers: +60% throughput
- SSD storage: +20% I/O

---

## Troubleshooting

### API não responde

**Sintomas**: Timeout ou conexão recusada

**Diagnóstico**:
```bash
# Verificar se processo está rodando
ps aux | grep -E "(node|python)"

# Verificar portas
netstat -tulpn | grep -E "(3000|8000)"

# Verificar logs
tail -f /var/log/goodwe/api.log

# Testar conectividade
curl -v http://localhost:3000/health
curl -v http://localhost:8000/health
```

**Soluções**:
1. Reiniciar serviço
2. Verificar firewall
3. Aumentar timeout
4. Verificar recursos (CPU/Memória)

### Erro de memória

**Sintomas**: "Out of memory" ou "Heap limit exceeded"

**Soluções**:
```bash
# Node.js - Aumentar heap
node --max-old-space-size=4096 index.js

# Docker - Aumentar limite
docker run -m 2g goodwe/api-excel

# Python - Otimizar carregamento
# Usar chunking para arquivos grandes
```

### Problemas de conectividade

**Entre containers Docker**:
```bash
# Verificar rede
docker network ls
docker network inspect goodwe-network

# Testar conectividade
docker exec container1 ping container2
docker exec container1 curl http://container2:port/health
```

**Entre APIs**:
```bash
# Verificar DNS
nslookup api.domain.com

# Testar rota
traceroute api.domain.com

# Verificar certificado SSL
openssl s_client -connect api.domain.com:443
```

---

**💡 Dica**: Se você não encontrou sua dúvida aqui, consulte também:
- [Troubleshooting](./troubleshooting.md) - Guia completo de problemas
- [Examples](./examples.md) - Exemplos práticos
- [GitHub Issues](https://github.com/your-repo/issues) - Problemas conhecidos

**📧 Suporte**: Para questões não documentadas, entre em contato através dos canais oficiais.
