# API Principal GoodWe - Documentação Técnica

A API Principal GoodWe é um serviço REST desenvolvido em Node.js/Express que fornece funcionalidades de conversão e processamento de dados Excel, especificamente para arquivos de dados de potência vegetal.

## 📋 Informações Gerais

- **Nome**: API GoodWe - Conversor de Dados Excel
- **Versão**: 1.0.0
- **Porta**: 3000
- **Base URL Local**: `http://localhost:3000`
- **Tecnologia**: Node.js + Express + Multer + XLSX
- **Arquivo Principal**: `index.js`
- **Estrutura**: MVC (Model-View-Controller)

## 🔧 Dependências Principais

```json
{
  "dependencies": {
    "cors": "^2.8.5",
    "dotenv": "^16.3.1",
    "express": "^4.18.2",
    "helmet": "^7.1.0",
    "morgan": "^1.10.0",
    "multer": "^2.0.2",
    "xlsx": "^0.18.5"
  },
  "devDependencies": {
    "nodemon": "^3.0.2"
  }
}
```

### Dependências de Sistema
- **Node.js**: 16+ (recomendado 18+)
- **NPM**: 8+
- **Arquivos Necessários**:
  - `scr/data/Potência Vegetal_20250820190939.xls` - Arquivo Excel de exemplo
  - `scr/data/potencia_vegetal.json` - Arquivo JSON convertido

## 🏗️ Arquitetura da API

### Estrutura de Diretórios
```
API_GoodWe/
├── controllers/
│   └── excelController.js      # Lógica de conversão Excel
├── routes/
│   └── excel.js               # Rotas do módulo Excel
├── scr/
│   ├── config/
│   │   └── database.js        # Configuração do banco
│   ├── data/                  # Arquivos de dados
│   └── routes/
│       └── index.js           # Rotas principais
├── database/
│   └── init/
│       └── 01-init.sql        # Scripts de inicialização
├── nginx/
│   ├── nginx.conf             # Configuração Nginx
│   └── ssl/                   # Certificados SSL
├── docker-scripts/
│   ├── start.bat              # Script Windows
│   └── start.sh               # Script Linux/Mac
├── index.js                   # Arquivo principal
├── package.json               # Dependências
├── Dockerfile                 # Configuração Docker
└── docker-compose.yml         # Orquestração de containers
```

### Middlewares Configurados
- **Helmet**: Segurança HTTP
- **CORS**: Cross-Origin Resource Sharing
- **Morgan**: Logging de requisições
- **Express.json()**: Parse de JSON
- **Express.urlencoded()**: Parse de URL-encoded

## 🚀 Endpoints Disponíveis

### 1. **GET /** - Página Principal
Retorna informações gerais da API e endpoints disponíveis.

**URL**: `GET /`

**Resposta de Sucesso**:
```json
{
  "message": "API GoodWe funcionando!",
  "version": "1.0.0",
  "timestamp": "2025-01-20T10:30:00.000Z",
  "availableEndpoints": {
    "GET /": "Esta página",
    "GET /health": "Health check",
    "GET /excel": "Informações do conversor Excel",
    "GET /excel/convert": "Converter arquivo Potência Vegetal",
    "POST /excel/upload": "Upload e conversão de Excel",
    "GET /excel/files": "Listar arquivos Excel"
  }
}
```

### 2. **GET /health** - Health Check
Verifica se a API está funcionando.

**URL**: `GET /health`

**Resposta de Sucesso**:
```json
{
  "status": "OK",
  "uptime": 3600.123,
  "timestamp": "2025-01-20T10:30:00.000Z"
}
```

### 3. **GET /excel** - Informações do Conversor
Retorna informações sobre o módulo de conversão Excel.

**URL**: `GET /excel`

**Resposta de Sucesso**:
```json
{
  "message": "Conversor Excel para JSON",
  "endpoints": {
    "GET /excel/convert": "Converte o arquivo Potência Vegetal para JSON",
    "POST /excel/upload": "Faz upload e converte arquivo Excel para JSON",
    "GET /excel/files": "Lista arquivos Excel disponíveis",
    "GET /excel": "Esta página de informações"
  },
  "supportedFormats": [".xls", ".xlsx"],
  "maxFileSize": "10MB"
}
```

### 4. **GET /excel/convert** - Conversão do Arquivo Específico
Converte o arquivo "Potência Vegetal_20250820190939.xls" para JSON.

**URL**: `GET /excel/convert`

**Resposta de Sucesso**:
```json
{
  "message": "Arquivo Excel convertido com sucesso!",
  "originalFile": "Potência Vegetal_20250820190939.xls",
  "convertedFile": "potencia_vegetal.json",
  "sheets": ["Sheet1", "Sheet2"],
  "data": {
    "Sheet1": [
      ["Header1", "Header2", "Header3"],
      ["Value1", "Value2", "Value3"]
    ],
    "Sheet2": [
      ["HeaderA", "HeaderB"],
      ["ValueA", "ValueB"]
    ]
  },
  "totalSheets": 2,
  "savedAt": "/app/scr/data/potencia_vegetal.json"
}
```

**Erro - Arquivo não encontrado**:
```json
{
  "error": "Arquivo não encontrado",
  "path": "/app/scr/data/Potência Vegetal_20250820190939.xls"
}
```
*Status Code: 404*

### 5. **POST /excel/upload** - Upload e Conversão
Faz upload de um arquivo Excel e converte para JSON.

**URL**: `POST /excel/upload`

**Content-Type**: `multipart/form-data`

**Parâmetros**:
- `excel` (file): Arquivo Excel (.xls ou .xlsx)

**Limitações**:
- Tamanho máximo: 10MB
- Formatos aceitos: .xls, .xlsx
- MIME types aceitos:
  - `application/vnd.ms-excel`
  - `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`

**Exemplo de Requisição**:
```bash
curl -X POST \
  http://localhost:3000/excel/upload \
  -F "excel=@arquivo.xlsx"
```

**Resposta de Sucesso**:
```json
{
  "message": "Arquivo Excel convertido com sucesso!",
  "originalFile": "arquivo.xlsx",
  "fileSize": 1024000,
  "sheets": ["Sheet1"],
  "data": {
    "Sheet1": [
      ["Header1", "Header2"],
      ["Value1", "Value2"]
    ]
  },
  "totalSheets": 1
}
```

**Erro - Nenhum arquivo enviado**:
```json
{
  "error": "Nenhum arquivo foi enviado",
  "message": "Envie um arquivo Excel (.xls ou .xlsx)"
}
```
*Status Code: 400*

**Erro - Formato inválido**:
```json
{
  "error": "Apenas arquivos Excel (.xls, .xlsx) são permitidos"
}
```
*Status Code: 400*

### 6. **GET /excel/files** - Listar Arquivos Excel
Lista todos os arquivos Excel disponíveis na pasta de dados.

**URL**: `GET /excel/files`

**Resposta de Sucesso**:
```json
{
  "message": "Arquivos Excel encontrados",
  "count": 2,
  "files": [
    {
      "name": "Potência Vegetal_20250820190939.xls",
      "size": 1024000,
      "created": "2025-01-20T08:00:00.000Z",
      "modified": "2025-01-20T09:30:00.000Z"
    },
    {
      "name": "outro_arquivo.xlsx",
      "size": 512000,
      "created": "2025-01-20T10:00:00.000Z",
      "modified": "2025-01-20T10:15:00.000Z"
    }
  ],
  "dataPath": "/app/scr/data"
}
```

**Erro - Pasta não encontrada**:
```json
{
  "error": "Pasta de dados não encontrada",
  "path": "/app/scr/data"
}
```
*Status Code: 404*

## 🔧 Configuração e Execução

### Arquivos Necessários
- `index.js` - Arquivo principal da API
- `package.json` - Dependências e scripts
- `scr/data/` - Pasta com arquivos de dados
- `.env` - Variáveis de ambiente (opcional)

### Comandos de Execução

#### 1. Preparação do Ambiente
```bash
# Navegar para o diretório
cd API_GoodWe

# Instalar dependências
npm install

# Configurar variáveis de ambiente (opcional)
cp .env.example .env
```

#### 2. Inicialização da API
```bash
# Desenvolvimento (com nodemon)
npm run dev

# Produção
npm start

# Ou diretamente
node index.js
```

#### 3. Verificação
```bash
# Testar API
curl http://localhost:3000/health

# Testar conversão
curl http://localhost:3000/excel/convert
```

### Scripts Disponíveis
```json
{
  "scripts": {
    "start": "node index.js",
    "dev": "nodemon index.js",
    "test": "echo \"Error: no test specified\" && exit 1"
  }
}
```

## 🐳 Docker

### Dockerfile (Configuração Real)
```dockerfile
FROM node:18-alpine

# Set working directory
WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy application code
COPY . .

# Create non-root user
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nodejs -u 1001
USER nodejs

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node healthcheck.js

# Start application
CMD ["npm", "start"]
```

### Docker Compose (Configuração Completa)
```yaml
version: '3.8'

services:
  # Aplicação API GoodWe
  api-goodwe:
    build: 
      context: .
      dockerfile: Dockerfile
    container_name: api-goodwe-container
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - PORT=3000
      - DB_HOST=postgres
      - DB_PORT=5432
      - DB_NAME=goodwe_db
      - DB_USER=goodwe_user
      - DB_PASS=goodwe_password
      - API_KEY=your_api_key_here
      - JWT_SECRET=your_super_secret_jwt_key_change_this
      - LOG_LEVEL=info
    volumes:
      - ./scr/data:/app/scr/data:rw
      - ./logs:/app/logs:rw
    depends_on:
      - postgres
      - redis
    networks:
      - goodwe-network
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.api-goodwe.rule=Host(`api.goodwe.local`)"
      - "traefik.http.services.api-goodwe.loadbalancer.server.port=3000"

  # Banco de dados PostgreSQL
  postgres:
    image: postgres:15-alpine
    container_name: postgres-goodwe
    restart: unless-stopped
    environment:
      - POSTGRES_DB=goodwe_db
      - POSTGRES_USER=goodwe_user
      - POSTGRES_PASSWORD=goodwe_password
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./database/init:/docker-entrypoint-initdb.d:ro
    ports:
      - "5432:5432"
    networks:
      - goodwe-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U goodwe_user -d goodwe_db"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Redis para cache e sessões
  redis:
    image: redis:7-alpine
    container_name: redis-goodwe
    restart: unless-stopped
    command: redis-server --appendonly yes --requirepass redis_password
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"
    networks:
      - goodwe-network
    healthcheck:
      test: ["CMD", "redis-cli", "--raw", "incr", "ping"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Nginx como proxy reverso
  nginx:
    image: nginx:alpine
    container_name: nginx-goodwe
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
    depends_on:
      - api-goodwe
    networks:
      - goodwe-network

# Volumes para persistência de dados
volumes:
  postgres_data:
    driver: local
  redis_data:
    driver: local

# Rede para comunicação entre containers
networks:
  goodwe-network:
    driver: bridge
```

## 🧪 Testes

### Teste Manual via cURL
```bash
# Health check
curl http://localhost:3000/health

# Informações da API
curl http://localhost:3000/

# Informações do conversor Excel
curl http://localhost:3000/excel

# Converter arquivo específico
curl http://localhost:3000/excel/convert

# Listar arquivos Excel
curl http://localhost:3000/excel/files

# Upload de arquivo Excel
curl -X POST \
  http://localhost:3000/excel/upload \
  -F "excel=@arquivo.xlsx"
```

### Teste com JavaScript/Node.js
```javascript
const axios = require('axios');
const FormData = require('form-data');
const fs = require('fs');

// Teste básico
async function testAPI() {
  try {
    // Health check
    const health = await axios.get('http://localhost:3000/health');
    console.log('Health:', health.data);

    // Converter arquivo específico
    const convert = await axios.get('http://localhost:3000/excel/convert');
    console.log('Convert:', convert.data);

    // Upload de arquivo
    const form = new FormData();
    form.append('excel', fs.createReadStream('arquivo.xlsx'));
    
    const upload = await axios.post('http://localhost:3000/excel/upload', form, {
      headers: form.getHeaders()
    });
    console.log('Upload:', upload.data);

  } catch (error) {
    console.error('Erro:', error.response?.data || error.message);
  }
}

testAPI();
```

## 🔒 Segurança e Validações

### Middlewares de Segurança
- **Helmet**: Headers de segurança HTTP
- **CORS**: Controle de acesso cross-origin
- **Multer**: Validação de upload de arquivos

### Validações de Upload
- **Tipo de arquivo**: Apenas .xls e .xlsx
- **Tamanho máximo**: 10MB
- **MIME type**: Validação de tipos MIME
- **Sanitização**: Limpeza de dados de entrada

### Tratamento de Erros
- **404**: Rota não encontrada
- **400**: Erro de validação
- **500**: Erro interno do servidor
- **Logs estruturados**: Morgan para logging

## 📊 Monitoramento

### Logs Disponíveis
- **Morgan**: Logs de requisições HTTP
- **Console**: Logs de aplicação
- **Arquivos**: Logs em `./logs/` (quando configurado)

### Métricas Disponíveis
- Status da API (uptime)
- Número de conversões realizadas
- Tamanho dos arquivos processados
- Tempo de resposta das requisições

## 🔮 Casos de Uso

### 1. Conversão Automática de Dados
```javascript
// Integração com sistema de monitoramento
const convertData = async () => {
  const response = await fetch('http://localhost:3000/excel/convert');
  const data = await response.json();
  
  // Processar dados convertidos
  processConvertedData(data.data);
};
```

### 2. Upload de Novos Arquivos
```javascript
// Upload de arquivos via interface web
const uploadFile = async (file) => {
  const formData = new FormData();
  formData.append('excel', file);
  
  const response = await fetch('http://localhost:3000/excel/upload', {
    method: 'POST',
    body: formData
  });
  
  return response.json();
};
```

### 3. Monitoramento de Arquivos
```javascript
// Verificar arquivos disponíveis
const checkFiles = async () => {
  const response = await fetch('http://localhost:3000/excel/files');
  const data = await response.json();
  
  // Notificar sobre novos arquivos
  if (data.count > 0) {
    notifyNewFiles(data.files);
  }
};
```

---

**Próxima seção**: [Guia de Integração](./integration-guide.md)
