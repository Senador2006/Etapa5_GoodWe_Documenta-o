# API Principal GoodWe - Documentação Técnica

API REST robusta e eficiente para distribuição e análise de dados de monitoramento de sistemas de energia solar da GoodWe. A API oferece endpoints especializados para acesso aos dados, análises estatísticas avançadas e busca inteligente.

## 📋 Informações Gerais

- **Nome**: API GoodWe - Sistema de Monitoramento de Energia Solar
- **Versão**: 1.0.0
- **Porta**: 3000
- **Base URL Local**: `http://localhost:3000`
- **Tecnologia**: Node.js + Express + CORS + Helmet + Morgan
- **Arquivo Principal**: `index.js`
- **Estrutura**: MVC (Model-View-Controller) com módulos especializados

## 🔧 Dependências Principais

```json
{
  "dependencies": {
    "cors": "^2.8.5",
    "dotenv": "^16.3.1",
    "express": "^4.18.2",
    "helmet": "^7.1.0",
    "morgan": "^1.10.0"
  },
  "devDependencies": {
    "nodemon": "^3.0.2"
  }
}
```

### Dependências de Sistema
- **Node.js**: 18+ (recomendado)
- **NPM**: 8+
- **Arquivos Necessários**:
  - `scr/data/potencia_vegetal.json` - Arquivo JSON com dados de monitoramento

## 🏗️ Arquitetura da API

### Estrutura de Diretórios
```
API_GoodWe/
├── controllers/
│   ├── dataController.js      # Lógica de acesso aos dados
│   ├── analyticsController.js # Lógica de análises estatísticas
│   └── searchController.js    # Lógica de busca avançada
├── routes/
│   ├── data.js               # Rotas do módulo de dados
│   ├── analytics.js          # Rotas do módulo de análises
│   └── search.js             # Rotas do módulo de busca
├── scr/
│   ├── config/
│   │   └── database.js        # Configuração do banco
│   └── data/                  # Arquivos de dados JSON
├── database/
│   └── init/
│       └── 01-init.sql        # Scripts de inicialização
├── nginx/
│   ├── nginx.conf             # Configuração Nginx
│   └── ssl/                   # Certificados SSL
├── docker-scripts/
│   ├── start.bat              # Script Windows
│   └── start.sh               # Script Linux/Mac
├── logs/                      # Logs da aplicação
├── index.js                   # Arquivo principal
├── package.json               # Dependências
├── Dockerfile                 # Configuração Docker
├── docker-compose.yml         # Orquestração de containers
├── docker-compose.dev.yml     # Docker Compose para desenvolvimento
├── API_DOCUMENTATION.md       # Documentação completa
└── SEARCH_GUIDE.md           # Guia de busca
```

### Middlewares Configurados
- **Helmet**: Segurança HTTP
- **CORS**: Cross-Origin Resource Sharing
- **Morgan**: Logging de requisições
- **Express.json()**: Parse de JSON
- **Express.urlencoded()**: Parse de URL-encoded

### Estrutura dos Dados
A API trabalha com dados de monitoramento contendo as seguintes métricas:

| Campo | Descrição |
|-------|-----------|
| `fv_power` | Potência fotovoltaica (W) |
| `soc_percentage` | Estado de carga da bateria (%) |
| `battery_power` | Potência da bateria (W) |
| `grid_power` | Potência da rede elétrica (W) |
| `load_power` | Potência da carga (W) |

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
    "GET /data": "Acesso aos dados da GoodWe",
    "GET /data/info": "Informações sobre endpoints de dados",
    "GET /data/hour/:{hour}":"Acessar dados pelas horas",
    "GET /analytics": "Análises e estatísticas",
    "GET /analytics/info": "Informações sobre endpoints de análise",
    "GET /analytics/hourly":"Obter ",
    "GET /search": "Busca avançada e filtros",
    "GET /search/info": "Informações sobre endpoints de busca"
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

## 📊 Módulo de Dados (`/data`)

### 3. **GET /data** - Obter Todos os Dados
Retorna todos os dados de monitoramento disponíveis.

**URL**: `GET /data`

### 4. **GET /data/paginated** - Dados com Paginação
Retorna dados com paginação para melhor performance.

**URL**: `GET /data/paginated?page=1&limit=100`

**Parâmetros**:
- `page` (opcional): Número da página (padrão: 1)
- `limit` (opcional): Itens por página (padrão: 100)

### 5. **GET /data/period** - Dados por Período
Retorna dados filtrados por período específico.

**URL**: `GET /data/period?start_date=2025-08-19&end_date=2025-08-20`

**Parâmetros**:
- `start_date`: Data inicial (YYYY-MM-DD)
- `end_date`: Data final (YYYY-MM-DD)

### 6. **GET /data/hour/:hour** - Dados por Hora
Retorna dados de uma hora específica do dia.

**URL**: `GET /data/hour/12`

**Parâmetros**:
- `hour`: Hora do dia (0-23)

### 7. **GET /data/info** - Informações do Módulo
Retorna informações sobre os endpoints de dados.

**URL**: `GET /data/info`

## 📈 Módulo de Análises (`/analytics`)

### 8. **GET /analytics/stats** - Estatísticas Gerais
Retorna estatísticas gerais do sistema (mínimo, máximo, média, total).

**URL**: `GET /analytics/stats`

### 9. **GET /analytics/hourly** - Análise por Hora
Retorna análise estatística agrupada por hora do dia.

**URL**: `GET /analytics/hourly`

### 10. **GET /analytics/efficiency** - Eficiência Energética
Retorna análise de eficiência energética e autossuficiência.

**URL**: `GET /analytics/efficiency`

### 11. **GET /analytics/trends** - Análise de Tendências
Retorna análise de tendências por períodos do dia.

**URL**: `GET /analytics/trends`

### 12. **GET /analytics/info** - Informações do Módulo
Retorna informações sobre os endpoints de análise.

**URL**: `GET /analytics/info`

## 🔍 Módulo de Busca (`/search`)

### 13. **GET /search/advanced** - Busca Avançada
Busca com múltiplos filtros, paginação e ordenação.

**URL**: `GET /search/advanced?start_date=2025-08-19&min_fv_power=1000&sort_by=fv_power&sort_order=desc&limit=50`

**Parâmetros**:
- Filtros temporais: `start_date`, `end_date`
- Filtros de valores: `min_fv_power`, `max_fv_power`, `min_soc`, `max_soc`, etc.
- Ordenação: `sort_by`, `sort_order` (asc/desc)
- Limite: `limit`

### 14. **GET /search/value** - Busca por Valor
Busca com operadores de comparação.

**URL**: `GET /search/value?field=fv_power&operator=gt&value=2000`

**Parâmetros**:
- `field`: Campo a ser pesquisado
- `operator`: Operador (eq, gt, gte, lt, lte, ne)
- `value`: Valor a ser comparado

### 15. **GET /search/peaks** - Busca Picos de Energia
Encontra valores acima de um threshold.

**URL**: `GET /search/peaks?field=fv_power&threshold=3000&limit=20`

### 16. **GET /search/anomalies** - Busca Anomalias
Encontra valores que se desviam significativamente da média.

**URL**: `GET /search/anomalies?field=soc_percentage&deviation=2.5&limit=30`

### 17. **GET /search/patterns** - Padrões Temporais
Análise de padrões por horário e dia da semana.

**URL**: `GET /search/patterns?start_hour=6&end_hour=18&days_of_week=1,2,3,4,5&pattern_type=daily`

### 18. **GET /search/info** - Informações do Módulo
Retorna informações sobre os endpoints de busca.

**URL**: `GET /search/info`

## 🔧 Configuração e Execução

### Arquivos Necessários
- `index.js` - Arquivo principal da API
- `package.json` - Dependências e scripts
- `scr/data/potencia_vegetal.json` - Arquivo com dados de monitoramento
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

# Testar dados
curl http://localhost:3000/data/info

# Testar análises
curl http://localhost:3000/analytics/info

# Testar busca
curl http://localhost:3000/search/info
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

# Informações dos módulos
curl http://localhost:3000/data/info
curl http://localhost:3000/analytics/info
curl http://localhost:3000/search/info

# Testar dados
curl http://localhost:3000/data/paginated?page=1&limit=10
curl http://localhost:3000/data/period?start_date=2025-08-19&end_date=2025-08-20
curl http://localhost:3000/data/hour/12

# Testar análises
curl http://localhost:3000/analytics/stats
curl http://localhost:3000/analytics/hourly
curl http://localhost:3000/analytics/efficiency
curl http://localhost:3000/analytics/trends

# Testar busca
curl "http://localhost:3000/search/advanced?min_fv_power=1000&sort_by=fv_power&sort_order=desc&limit=10"
curl "http://localhost:3000/search/value?field=fv_power&operator=gt&value=2000"
curl "http://localhost:3000/search/peaks?field=fv_power&threshold=3000&limit=20"
curl "http://localhost:3000/search/anomalies?field=soc_percentage&deviation=2.5&limit=30"
```

### Teste com JavaScript/Node.js
```javascript
const axios = require('axios');

// Teste básico
async function testAPI() {
  try {
    // Health check
    const health = await axios.get('http://localhost:3000/health');
    console.log('Health:', health.data);

    // Testar dados
    const data = await axios.get('http://localhost:3000/data/paginated?page=1&limit=5');
    console.log('Data:', data.data);

    // Testar análises
    const stats = await axios.get('http://localhost:3000/analytics/stats');
    console.log('Stats:', stats.data);

    // Testar busca avançada
    const search = await axios.get('http://localhost:3000/search/advanced?min_fv_power=1000&limit=5');
    console.log('Search:', search.data);

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
- **Morgan**: Logging estruturado de requisições

### Validações de Dados
- **Parâmetros de query**: Validação de tipos e formatos
- **Filtros temporais**: Validação de datas
- **Operadores de busca**: Validação de operadores permitidos
- **Sanitização**: Limpeza de dados de entrada

### Tratamento de Erros
- **404**: Rota não encontrada
- **400**: Erro de validação de parâmetros
- **500**: Erro interno do servidor
- **Logs estruturados**: Morgan para logging de requisições

## 📊 Monitoramento

### Logs Disponíveis
- **Morgan**: Logs de requisições HTTP
- **Console**: Logs de aplicação
- **Arquivos**: Logs em `./logs/` (quando configurado)

### Métricas Disponíveis
- Status da API (uptime)
- Número de requisições processadas
- Tempo de resposta das requisições
- Estatísticas de uso por endpoint

## 🔮 Casos de Uso

### 1. Monitoramento em Tempo Real
```javascript
// Obter dados de monitoramento
const getMonitoringData = async () => {
  const response = await fetch('http://localhost:3000/data/paginated?page=1&limit=100');
  const data = await response.json();
  
  // Processar dados de monitoramento
  processMonitoringData(data.data);
};
```

### 2. Análise de Eficiência Energética
```javascript
// Analisar eficiência do sistema
const analyzeEfficiency = async () => {
  const response = await fetch('http://localhost:3000/analytics/efficiency');
  const data = await response.json();
  
  // Implementar otimizações baseadas na análise
  optimizeSystem(data.efficiency_metrics);
};
```

### 3. Detecção de Anomalias
```javascript
// Detectar anomalias no sistema
const detectAnomalies = async () => {
  const response = await fetch('http://localhost:3000/search/anomalies?field=fv_power&deviation=2.5&limit=10');
  const data = await response.json();
  
  // Alertar sobre anomalias detectadas
  if (data.anomalies.length > 0) {
    alertAnomalies(data.anomalies);
  }
};
```

### 4. Busca de Picos de Energia
```javascript
// Identificar picos de geração solar
const findEnergyPeaks = async () => {
  const response = await fetch('http://localhost:3000/search/peaks?field=fv_power&threshold=3000&limit=20');
  const data = await response.json();
  
  // Analisar padrões de picos
  analyzePeakPatterns(data.peaks);
};
```

---

**Próxima seção**: [Guia de Integração](./integration-guide.md)
