# Troubleshooting e FAQ - APIs GoodWe

Este guia fornece soluções para problemas comuns, dicas de debug e perguntas frequentes sobre as APIs GoodWe.

## 📋 Índice

1. [Problemas Comuns - API Excel](#problemas-comuns---api-excel)
2. [Problemas Comuns - API Machine Learning](#problemas-comuns---api-machine-learning)
3. [Problemas de Deployment](#problemas-de-deployment)
4. [Debug e Logging](#debug-e-logging)
5. [Performance e Otimização](#performance-e-otimização)
6. [FAQ - Perguntas Frequentes](#faq---perguntas-frequentes)

## 🔧 Problemas Comuns - API Excel

### 1. Erro "Arquivo não encontrado" ao converter arquivo específico

**Problema**: 
```json
{
  "error": "Arquivo não encontrado",
  "path": "/path/to/Potência Vegetal_20250820190939.xls"
}
```

**Soluções**:
```bash
# Verificar se o arquivo existe
ls -la API_GoodWe/scr/data/

# Verificar permissões
chmod 644 API_GoodWe/scr/data/*.xls

# Recriar estrutura de pastas
mkdir -p API_GoodWe/scr/data/

# Copiar arquivo se necessário
cp /source/path/file.xls API_GoodWe/scr/data/
```

### 2. Erro de upload "Apenas arquivos Excel são permitidos"

**Problema**: Arquivo rejeitado no upload

**Verificações**:
```javascript
// Verificar MIME type do arquivo
console.log(file.type); 
// Deve ser: 'application/vnd.ms-excel' ou 
// 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet'

// Verificar extensão
const validExtensions = ['.xls', '.xlsx'];
const fileExtension = file.name.toLowerCase().substr(file.name.lastIndexOf('.'));
console.log(validExtensions.includes(fileExtension));
```

**Soluções**:
```bash
# Converter arquivo para formato válido
libreoffice --headless --convert-to xlsx input.csv

# Verificar arquivo com file command
file your-file.xlsx
# Deve mostrar: Microsoft Excel 2007+
```

### 3. Erro "Request entity too large" (413)

**Problema**: Arquivo muito grande para upload

**Soluções**:

#### Nginx
```nginx
# /etc/nginx/nginx.conf
http {
    client_max_body_size 50M;
}
```

#### Node.js/Express
```javascript
// Aumentar limite no Express
app.use(express.json({ limit: '50mb' }));
app.use(express.urlencoded({ limit: '50mb', extended: true }));

// Configurar Multer
const upload = multer({
  storage: multer.memoryStorage(),
  limits: {
    fileSize: 50 * 1024 * 1024 // 50MB
  }
});
```

### 4. Erro de memória "FATAL ERROR: Reached heap limit"

**Problema**: Node.js fica sem memória processando arquivos grandes

**Soluções**:
```bash
# Aumentar heap do Node.js
node --max-old-space-size=4096 index.js

# Ou via variável de ambiente
export NODE_OPTIONS="--max-old-space-size=4096"
npm start
```

**Processamento em chunks**:
```javascript
// Processar arquivo em partes menores
const XLSX = require('xlsx');

function processLargeExcel(filePath) {
  const workbook = XLSX.readFile(filePath, {
    cellDates: true,
    cellNF: false,
    cellText: false,
    raw: false
  });
  
  // Processar uma planilha por vez
  const results = {};
  for (const sheetName of workbook.SheetNames) {
    const worksheet = workbook.Sheets[sheetName];
    const range = XLSX.utils.decode_range(worksheet['!ref']);
    
    // Processar em chunks de 1000 linhas
    const chunkSize = 1000;
    const data = [];
    
    for (let row = range.s.r; row <= range.e.r; row += chunkSize) {
      const endRow = Math.min(row + chunkSize - 1, range.e.r);
      const chunkRange = {
        s: { r: row, c: range.s.c },
        e: { r: endRow, c: range.e.c }
      };
      
      const chunkData = XLSX.utils.sheet_to_json(worksheet, {
        range: chunkRange,
        header: 1
      });
      
      data.push(...chunkData);
    }
    
    results[sheetName] = data;
  }
  
  return results;
}
```

### 5. Problemas de encoding/caracteres especiais

**Problema**: Caracteres especiais aparecem incorretamente

**Soluções**:
```javascript
// Especificar codepage ao ler Excel
const workbook = XLSX.readFile(filePath, { 
  codepage: 65001 // UTF-8
});

// Para arquivos antigos (.xls)
const workbook = XLSX.readFile(filePath, { 
  codepage: 1252 // Windows-1252
});
```

## 🧠 Problemas Comuns - API Machine Learning

### 1. Erro "Modelo não carregado" (503)

**Problema**: 
```json
{
  "detail": "Modelo não carregado"
}
```

**Soluções**:
```bash
# Verificar se arquivos existem
ls -la power_outage_model.pkl features.json

# Retreinar modelo se necessário
cd Machine_learning_GoodWe
python train_model.py

# Verificar permissões
chmod 644 *.pkl *.json

# Testar carregamento manual
python -c "import joblib; print(joblib.load('power_outage_model.pkl'))"
```

### 2. Erro de validação de entrada

**Problema**: 
```json
{
  "detail": [
    {
      "type": "greater_than_equal",
      "loc": ["temperatura_celsius"],
      "msg": "Input should be greater than or equal to -10"
    }
  ]
}
```

**Verificação de ranges válidos**:
```python
# Ranges aceitos pela API
valid_ranges = {
    "temperatura_celsius": (-10, 50),
    "umidade_pct": (0, 100),
    "precipitacao_mm_h": (0, 100),
    "vento_kmh": (0, 150),
    "pressao_hpa": (950, 1050)
}

def validate_weather_data(data):
    for field, (min_val, max_val) in valid_ranges.items():
        value = data.get(field)
        if value is None:
            raise ValueError(f"Campo obrigatório: {field}")
        if not (min_val <= value <= max_val):
            raise ValueError(f"{field} deve estar entre {min_val} e {max_val}")
    return True
```

### 3. Predições inconsistentes ou estranhas

**Problema**: Modelo retorna resultados inesperados

**Debug do modelo**:
```python
import joblib
import numpy as np

# Carregar modelo
model = joblib.load('power_outage_model.pkl')

# Verificar features
print("Features do modelo:", model.feature_names_in_)

# Verificar importância das features
if hasattr(model, 'feature_importances_'):
    importances = model.feature_importances_
    for feature, importance in zip(model.feature_names_in_, importances):
        print(f"{feature}: {importance:.3f}")

# Testar com dados conhecidos
test_data = np.array([[25, 60, 5, 20, 1015]])  # Condições normais
prediction = model.predict(test_data)
probability = model.predict_proba(test_data)
print(f"Predição: {prediction[0]}, Probabilidade: {probability[0]}")
```

### 4. Erro de dependências Python

**Problema**: ImportError ou módulos não encontrados

**Soluções**:
```bash
# Verificar versões
pip list | grep -E "(numpy|pandas|scikit-learn|fastapi)"

# Reinstalar dependências
pip uninstall -y numpy pandas scikit-learn
pip install --no-cache-dir -r requirements.txt

# Verificar compatibilidade
python -c "
import numpy as np
import pandas as pd
import sklearn
print(f'NumPy: {np.__version__}')
print(f'Pandas: {pd.__version__}')
print(f'Scikit-learn: {sklearn.__version__}')
"

# Usar ambiente virtual limpo
python -m venv fresh_env
source fresh_env/bin/activate  # Linux/Mac
# fresh_env\Scripts\activate  # Windows
pip install -r requirements.txt
```

### 5. Performance lenta da API

**Problema**: Respostas demoram muito

**Diagnóstico**:
```python
import time
import cProfile

def profile_prediction():
    # Profiling da predição
    profiler = cProfile.Profile()
    profiler.enable()
    
    start_time = time.time()
    prediction = model.predict([[25, 60, 10, 30, 1013]])
    end_time = time.time()
    
    profiler.disable()
    profiler.print_stats(sort='cumtime')
    
    print(f"Tempo de predição: {(end_time - start_time) * 1000:.2f}ms")

profile_prediction()
```

**Otimizações**:
```python
# Cache de modelo (usar singleton)
class ModelSingleton:
    _instance = None
    _model = None
    
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._model = joblib.load('power_outage_model.pkl')
        return cls._instance
    
    def predict(self, data):
        return self._model.predict(data)

# Usar modelo singleton
model_instance = ModelSingleton()

# Pré-aquecer modelo na inicialização
dummy_data = [[25, 60, 10, 30, 1013]]
model_instance.predict(dummy_data)
```

## 🐳 Problemas de Deployment

### 1. Container não inicia

**Problema**: Docker container falha ao iniciar

**Debug**:
```bash
# Verificar logs
docker logs container-name

# Entrar no container para debug
docker run -it --entrypoint /bin/bash image-name

# Verificar arquivos
docker run --rm -v $(pwd):/workspace image-name ls -la /app

# Testar comando manual
docker run --rm image-name python -c "import api; print('OK')"
```

### 2. Problemas de permissão em volumes

**Problema**: Arquivos não podem ser acessados

**Soluções**:
```bash
# Verificar ownership
ls -la /path/to/volume

# Corrigir permissões
sudo chown -R 1001:1001 /path/to/volume

# Docker compose com user
version: '3.8'
services:
  api:
    user: "1001:1001"
    volumes:
      - ./data:/app/data:rw
```

### 3. Problemas de rede entre containers

**Problema**: APIs não conseguem se comunicar

**Debug**:
```bash
# Verificar rede
docker network ls
docker network inspect network-name

# Testar conectividade entre containers
docker exec container1 ping container2
docker exec container1 curl http://container2:port/health

# Verificar DNS
docker exec container1 nslookup container2
```

### 4. SSL/HTTPS não funciona

**Problema**: Certificados SSL não carregam

**Verificações**:
```bash
# Testar certificado
openssl x509 -in certificate.crt -text -noout

# Verificar certificado no servidor
echo | openssl s_client -connect domain.com:443 -servername domain.com

# Nginx SSL config
server {
    listen 443 ssl;
    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/private.key;
    ssl_protocols TLSv1.2 TLSv1.3;
}
```

## 🔍 Debug e Logging

### 1. Habilitar logs detalhados

#### API Excel (Node.js)
```javascript
// Adicionar middleware de debug
app.use((req, res, next) => {
  console.log(`${new Date().toISOString()} - ${req.method} ${req.path}`);
  console.log('Headers:', req.headers);
  console.log('Body:', req.body);
  next();
});

// Logs de erro detalhados
app.use((err, req, res, next) => {
  console.error('Error Stack:', err.stack);
  console.error('Request:', {
    method: req.method,
    url: req.url,
    headers: req.headers,
    body: req.body
  });
  res.status(500).json({ error: err.message });
});
```

#### API ML (Python)
```python
import logging

# Configurar logging
logging.basicConfig(
    level=logging.DEBUG,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('api.log'),
        logging.StreamHandler()
    ]
)

logger = logging.getLogger(__name__)

# Middleware de logging no FastAPI
@app.middleware("http")
async def log_requests(request: Request, call_next):
    start_time = time.time()
    
    logger.info(f"Request: {request.method} {request.url}")
    
    response = await call_next(request)
    
    process_time = time.time() - start_time
    logger.info(f"Response: {response.status_code} - {process_time:.3f}s")
    
    return response
```

### 2. Monitoramento de recursos

```bash
#!/bin/bash
# monitor.sh

echo "=== Monitoramento APIs GoodWe ==="

# CPU e Memória
echo "CPU e Memória:"
ps aux | grep -E "(node|python)" | grep -v grep

# Espaço em disco
echo -e "\nEspaço em disco:"
df -h /app

# Conexões de rede
echo -e "\nConexões:"
netstat -tulpn | grep -E "(3000|8000)"

# Logs recentes
echo -e "\nErros recentes (últimos 10 minutos):"
journalctl --since "10 minutes ago" | grep -i error

# Status dos serviços
echo -e "\nStatus dos serviços:"
systemctl status api-goodwe ml-api
```

## ⚡ Performance e Otimização

### 1. Otimização da API Excel

```javascript
// Cache de arquivos convertidos
const NodeCache = require('node-cache');
const cache = new NodeCache({ stdTTL: 3600 }); // 1 hora

app.get('/excel/convert', (req, res) => {
  const cacheKey = 'converted_excel';
  const cached = cache.get(cacheKey);
  
  if (cached) {
    return res.json(cached);
  }
  
  // Processo de conversão...
  const result = convertExcel();
  cache.set(cacheKey, result);
  res.json(result);
});

// Streaming de arquivos grandes
const stream = require('stream');

app.post('/excel/upload-stream', (req, res) => {
  const chunks = [];
  
  req.on('data', chunk => chunks.push(chunk));
  req.on('end', () => {
    const buffer = Buffer.concat(chunks);
    // Processar buffer...
  });
});
```

### 2. Otimização da API ML

```python
# Cache de predições
from functools import lru_cache
import hashlib

@lru_cache(maxsize=1000)
def cached_predict(weather_hash):
    # Deserializar dados do hash
    weather_data = json.loads(weather_hash)
    input_data = [list(weather_data.values())]
    
    prediction = model.predict(input_data)[0]
    probability = model.predict_proba(input_data)[0][1]
    
    return prediction, probability

@app.post("/predict")
async def predict_power_outage(weather_data: WeatherData):
    # Criar hash dos dados para cache
    data_dict = weather_data.dict()
    weather_hash = hashlib.md5(
        json.dumps(data_dict, sort_keys=True).encode()
    ).hexdigest()
    
    # Usar cache
    prediction, probability = cached_predict(json.dumps(data_dict, sort_keys=True))
    
    # Resto da lógica...
```

### 3. Balanceamento de carga

```nginx
# nginx.conf
upstream excel_api {
    server localhost:3001;
    server localhost:3002;
    server localhost:3003;
}

upstream ml_api {
    server localhost:8001;
    server localhost:8002;
}

server {
    location /excel/ {
        proxy_pass http://excel_api;
    }
    
    location /ml/ {
        proxy_pass http://ml_api;
    }
}
```

## ❓ FAQ - Perguntas Frequentes

### Q1: Posso usar as APIs em produção?

**R**: Sim, mas certifique-se de:
- Configurar HTTPS
- Implementar autenticação
- Configurar rate limiting
- Fazer backup regular dos dados
- Monitorar performance e erros

### Q2: Qual o limite de tamanho de arquivo Excel?

**R**: Por padrão, 10MB. Pode ser aumentado configurando:
- Multer: `limits.fileSize`
- Nginx: `client_max_body_size`
- Express: `limit` no middleware

### Q3: O modelo ML pode ser retreinado?

**R**: Sim, execute:
```bash
cd Machine_learning_GoodWe
python train_model.py
# Reinicie a API para carregar o novo modelo
```

### Q4: Como adicionar autenticação?

**R**: Implemente middleware JWT:
```javascript
// API Excel
const jwt = require('jsonwebtoken');

function authenticateToken(req, res, next) {
  const token = req.header('Authorization')?.split(' ')[1];
  
  if (!token) {
    return res.status(401).json({ error: 'Token necessário' });
  }
  
  jwt.verify(token, process.env.JWT_SECRET, (err, user) => {
    if (err) return res.status(403).json({ error: 'Token inválido' });
    req.user = user;
    next();
  });
}

// Usar em rotas protegidas
app.get('/excel/convert', authenticateToken, convertExcel);
```

### Q5: Como configurar CORS adequadamente?

**R**: Configure CORS específico:
```javascript
// API Excel
const cors = require('cors');

app.use(cors({
  origin: ['https://your-frontend.com', 'https://dashboard.company.com'],
  credentials: true,
  methods: ['GET', 'POST'],
  allowedHeaders: ['Content-Type', 'Authorization']
}));
```

### Q6: A API ML funciona offline?

**R**: Sim, uma vez que o modelo esteja treinado e os arquivos `.pkl` e `.json` estejam disponíveis, a API funciona completamente offline.

### Q7: Como fazer backup dos dados?

**R**: Script de backup:
```bash
#!/bin/bash
# backup.sh
tar -czf backup-$(date +%Y%m%d).tar.gz \
  API_GoodWe/scr/data/ \
  Machine_learning_GoodWe/*.pkl \
  Machine_learning_GoodWe/*.json

# Upload para cloud (opcional)
aws s3 cp backup-$(date +%Y%m%d).tar.gz s3://backup-bucket/
```

### Q8: Como monitorar as APIs?

**R**: Use ferramentas como:
- **Prometheus** + **Grafana** para métricas
- **ELK Stack** para logs
- **Uptime Kuma** para status
- Scripts personalizados para health checks

### Q9: Posso usar outras linguagens para consumir as APIs?

**R**: Sim! As APIs são REST padrão e podem ser consumidas por qualquer linguagem que suporte HTTP (Java, C#, Go, Ruby, etc.).

### Q10: Como otimizar para muitas requisições simultâneas?

**R**: 
- Use PM2 para cluster Node.js
- Configure Gunicorn com múltiplos workers para Python
- Implemente cache (Redis)
- Use load balancer (Nginx)
- Configure auto-scaling em cloud

---

**Seção anterior**: [Exemplos de Uso](./examples.md)  
**Voltar ao início**: [README](./README.md)
