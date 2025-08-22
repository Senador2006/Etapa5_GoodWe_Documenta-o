# Configuração Docker - APIs GoodWe

Este documento fornece informações detalhadas sobre a configuração Docker para as APIs GoodWe, incluindo otimizações, segurança e deployment em diferentes ambientes.

## 📋 Visão Geral

O projeto GoodWe utiliza Docker para facilitar o deployment e garantir consistência entre ambientes. Cada API possui sua própria configuração Docker, mas também podem ser executadas em conjunto usando docker-compose.

## 🐳 Estrutura Docker

```
Projeto GoodWe/
├── API_GoodWe/
│   ├── Dockerfile
│   ├── docker-compose.yml
│   ├── docker-compose.dev.yml
│   └── .dockerignore
├── Machine_learning_GoodWe/
│   ├── Dockerfile
│   ├── docker-compose.yml
│   └── .dockerignore
└── docker-compose.global.yml
```

## 🔧 API Excel - Configuração Docker

### Dockerfile Detalhado

```dockerfile
# Use a imagem oficial do Node.js como base
FROM node:18-alpine

# Definir metadados da imagem
LABEL maintainer="GoodWe Team <team@goodwe.com>"
LABEL version="1.0.0"
LABEL description="API GoodWe para conversão de arquivos Excel"

# Definir variáveis de ambiente
ENV NODE_ENV=production
ENV PORT=3000
ENV TZ=America/Sao_Paulo

# Instalar dependências do sistema necessárias
RUN apk add --no-cache \
    dumb-init \
    curl \
    tini

# Criar usuário não-root para segurança
RUN addgroup -g 1001 -S nodejs && \
    adduser -S goodwe -u 1001 -G nodejs

# Definir diretório de trabalho
WORKDIR /app

# Copiar arquivos de dependências primeiro (melhor cache)
COPY package*.json ./

# Instalar dependências de produção
RUN npm ci --only=production && \
    npm cache clean --force && \
    rm -rf /tmp/*

# Copiar código da aplicação
COPY . .

# Criar pastas necessárias
RUN mkdir -p scr/data logs && \
    chown -R goodwe:nodejs /app

# Mudar para usuário não-root
USER goodwe

# Expor a porta da aplicação
EXPOSE 3000

# Comando de health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:3000/health || exit 1

# Usar tini como init system
ENTRYPOINT ["/sbin/tini", "--"]

# Comando para iniciar a aplicação
CMD ["node", "index.js"]
```

### .dockerignore

```dockerignore
# Dependências
node_modules
npm-debug.log*
yarn-debug.log*
yarn-error.log*

# Logs
logs
*.log

# Arquivos de ambiente
.env
.env.local
.env.*.local

# Arquivos de sistema
.DS_Store
Thumbs.db

# IDE
.vscode
.idea
*.swp
*.swo

# Git
.git
.gitignore

# Docker
Dockerfile*
docker-compose*
.dockerignore

# Documentação
README.md
docs/

# Testes
test/
coverage/
.nyc_output

# Arquivos temporários
tmp/
temp/
```

### docker-compose.yml (Produção)

```yaml
version: '3.8'

services:
  # Aplicação API GoodWe
  api-goodwe:
    build: 
      context: .
      dockerfile: Dockerfile
      args:
        NODE_ENV: production
    image: goodwe/api-excel:1.0.0
    container_name: api-goodwe-container
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - PORT=3000
      - TZ=America/Sao_Paulo
      # Database
      - DB_HOST=postgres
      - DB_PORT=5432
      - DB_NAME=goodwe_db
      - DB_USER=goodwe_user
      - DB_PASS_FILE=/run/secrets/db_password
      # Security
      - API_KEY_FILE=/run/secrets/api_key
      - JWT_SECRET_FILE=/run/secrets/jwt_secret
      # Logging
      - LOG_LEVEL=info
      - LOG_FORMAT=json
    volumes:
      - ./scr/data:/app/scr/data:rw
      - ./logs:/app/logs:rw
      - /etc/localtime:/etc/localtime:ro
    networks:
      - goodwe-network
      - database-network
    depends_on:
      - postgres
      - redis
    secrets:
      - db_password
      - api_key
      - jwt_secret
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: '0.5'
        reservations:
          memory: 256M
          cpus: '0.25'
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  # Banco de dados PostgreSQL
  postgres:
    image: postgres:15-alpine
    container_name: postgres-goodwe
    restart: unless-stopped
    environment:
      - POSTGRES_DB=goodwe_db
      - POSTGRES_USER=goodwe_user
      - POSTGRES_PASSWORD_FILE=/run/secrets/db_password
      - POSTGRES_INITDB_ARGS=--encoding=UTF-8 --lc-collate=C --lc-ctype=C
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./database/init:/docker-entrypoint-initdb.d:ro
      - ./database/backup:/backup:rw
    networks:
      - database-network
    secrets:
      - db_password
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U goodwe_user -d goodwe_db"]
      interval: 30s
      timeout: 10s
      retries: 5
    deploy:
      resources:
        limits:
          memory: 1G
          cpus: '1.0'

  # Redis para cache
  redis:
    image: redis:7-alpine
    container_name: redis-goodwe
    restart: unless-stopped
    command: >
      redis-server 
      --appendonly yes 
      --requirepass $$(cat /run/secrets/redis_password)
      --maxmemory 256mb
      --maxmemory-policy allkeys-lru
    volumes:
      - redis_data:/data
      - ./redis/redis.conf:/usr/local/etc/redis/redis.conf:ro
    networks:
      - goodwe-network
    secrets:
      - redis_password
    healthcheck:
      test: ["CMD", "redis-cli", "--raw", "incr", "ping"]
      interval: 30s
      timeout: 10s
      retries: 3
    deploy:
      resources:
        limits:
          memory: 256M
          cpus: '0.25'

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
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
      - ./nginx/logs:/var/log/nginx:rw
    depends_on:
      - api-goodwe
    networks:
      - goodwe-network
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    deploy:
      resources:
        limits:
          memory: 128M
          cpus: '0.25'

# Secrets para informações sensíveis
secrets:
  db_password:
    file: ./secrets/db_password.txt
  api_key:
    file: ./secrets/api_key.txt
  jwt_secret:
    file: ./secrets/jwt_secret.txt
  redis_password:
    file: ./secrets/redis_password.txt

# Volumes para persistência de dados
volumes:
  postgres_data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /opt/goodwe/data/postgres
  redis_data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /opt/goodwe/data/redis

# Redes para comunicação entre containers
networks:
  goodwe-network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
  database-network:
    driver: bridge
    internal: true
```

### docker-compose.dev.yml (Desenvolvimento)

```yaml
version: '3.8'

services:
  api-goodwe-dev:
    build:
      context: .
      dockerfile: Dockerfile.dev
      args:
        NODE_ENV: development
    container_name: api-goodwe-dev
    ports:
      - "3000:3000"
      - "9229:9229"  # Debug port
    environment:
      - NODE_ENV=development
      - PORT=3000
      - DEBUG=goodwe:*
      - LOG_LEVEL=debug
    volumes:
      - .:/app:rw
      - /app/node_modules
      - ./logs:/app/logs:rw
    networks:
      - goodwe-dev-network
    command: ["npm", "run", "dev"]
    stdin_open: true
    tty: true

  postgres-dev:
    image: postgres:15-alpine
    container_name: postgres-dev
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_DB=goodwe_dev
      - POSTGRES_USER=dev_user
      - POSTGRES_PASSWORD=dev_password
    volumes:
      - postgres_dev_data:/var/lib/postgresql/data
    networks:
      - goodwe-dev-network

volumes:
  postgres_dev_data:

networks:
  goodwe-dev-network:
    driver: bridge
```

## 🧠 API Machine Learning - Configuração Docker

### Dockerfile Otimizado

```dockerfile
# Multi-stage build para reduzir tamanho da imagem final
FROM python:3.11-slim as builder

# Instalar dependências de compilação
RUN apt-get update && apt-get install -y \
    gcc \
    g++ \
    make \
    && rm -rf /var/lib/apt/lists/*

# Criar ambiente virtual
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# Copiar requirements
COPY requirements.txt .

# Instalar dependências Python
RUN pip install --no-cache-dir --upgrade pip && \
    pip install --no-cache-dir -r requirements.txt

# Estágio final - imagem menor
FROM python:3.11-slim

# Definir metadados
LABEL maintainer="GoodWe Team <team@goodwe.com>"
LABEL version="1.0.0"
LABEL description="API GoodWe Machine Learning para predição de quedas de energia"

# Definir variáveis de ambiente
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1
ENV PATH="/opt/venv/bin:$PATH"
ENV TZ=America/Sao_Paulo

# Instalar dependências de runtime
RUN apt-get update && apt-get install -y \
    curl \
    tini \
    && rm -rf /var/lib/apt/lists/* \
    && apt-get autoremove -y \
    && apt-get clean

# Copiar ambiente virtual do builder
COPY --from=builder /opt/venv /opt/venv

# Criar usuário não-root
RUN useradd --create-home --shell /bin/bash --uid 1001 appuser

# Definir diretório de trabalho
WORKDIR /app

# Copiar aplicação
COPY --chown=appuser:appuser . .

# Mudar para usuário não-root
USER appuser

# Expor porta
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=30s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# Usar tini como init
ENTRYPOINT ["/usr/bin/tini", "--"]

# Comando para executar a aplicação
CMD ["python", "api.py"]
```

### docker-compose.yml (ML API)

```yaml
version: '3.8'

services:
  ml-api:
    build:
      context: .
      dockerfile: Dockerfile
      args:
        PYTHON_VERSION: 3.11
    image: goodwe/ml-api:1.0.0
    container_name: ml-prediction-api
    restart: unless-stopped
    ports:
      - "8000:8000"
    environment:
      - PYTHONPATH=/app
      - TZ=America/Sao_Paulo
      - HOST=0.0.0.0
      - PORT=8000
      - WORKERS=4
      - LOG_LEVEL=info
      - MODEL_CACHE_SIZE=100
      - PREDICTION_TIMEOUT=30
    volumes:
      - ./models:/app/models:ro
      - ./logs:/app/logs:rw
      - /etc/localtime:/etc/localtime:ro
    networks:
      - ml-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    deploy:
      resources:
        limits:
          memory: 2G
          cpus: '2.0'
        reservations:
          memory: 1G
          cpus: '1.0'
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  # Redis para cache de predições
  redis-ml:
    image: redis:7-alpine
    container_name: redis-ml-cache
    restart: unless-stopped
    command: redis-server --appendonly yes --maxmemory 512mb --maxmemory-policy allkeys-lru
    volumes:
      - redis_ml_data:/data
    networks:
      - ml-network
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: '0.5'

  # Monitoring com Prometheus
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus-ml
    restart: unless-stopped
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus
    networks:
      - ml-network
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'

volumes:
  redis_ml_data:
  prometheus_data:

networks:
  ml-network:
    driver: bridge
```

## 🌐 Docker Compose Global

### docker-compose.global.yml

```yaml
version: '3.8'

services:
  # API Excel
  api-excel:
    build: ./API_GoodWe
    image: goodwe/api-excel:1.0.0
    container_name: goodwe-api-excel
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - ML_API_URL=http://api-ml:8000
    volumes:
      - excel_data:/app/scr/data
      - excel_logs:/app/logs
    networks:
      - goodwe-global
    depends_on:
      - postgres
      - redis

  # API Machine Learning
  api-ml:
    build: ./Machine_learning_GoodWe
    image: goodwe/api-ml:1.0.0
    container_name: goodwe-api-ml
    restart: unless-stopped
    ports:
      - "8000:8000"
    environment:
      - PYTHONPATH=/app
      - EXCEL_API_URL=http://api-excel:3000
    volumes:
      - ml_models:/app/models
      - ml_logs:/app/logs
    networks:
      - goodwe-global

  # Banco de dados compartilhado
  postgres:
    image: postgres:15-alpine
    container_name: goodwe-postgres
    restart: unless-stopped
    environment:
      - POSTGRES_DB=goodwe
      - POSTGRES_USER=goodwe
      - POSTGRES_PASSWORD_FILE=/run/secrets/db_password
    volumes:
      - postgres_global_data:/var/lib/postgresql/data
      - ./database/init-global.sql:/docker-entrypoint-initdb.d/init.sql:ro
    networks:
      - goodwe-global
    secrets:
      - db_password

  # Redis compartilhado
  redis:
    image: redis:7-alpine
    container_name: goodwe-redis
    restart: unless-stopped
    command: redis-server --appendonly yes --requirepass $$(cat /run/secrets/redis_password)
    volumes:
      - redis_global_data:/data
    networks:
      - goodwe-global
    secrets:
      - redis_password

  # Gateway API
  api-gateway:
    image: nginx:alpine
    container_name: goodwe-gateway
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/gateway.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
    depends_on:
      - api-excel
      - api-ml
    networks:
      - goodwe-global

  # Monitoring Stack
  grafana:
    image: grafana/grafana:latest
    container_name: goodwe-grafana
    restart: unless-stopped
    ports:
      - "3001:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD_FILE=/run/secrets/grafana_password
    volumes:
      - grafana_data:/var/lib/grafana
      - ./monitoring/grafana-datasources.yml:/etc/grafana/provisioning/datasources/datasources.yml:ro
    networks:
      - goodwe-global
    secrets:
      - grafana_password

  prometheus:
    image: prom/prometheus:latest
    container_name: goodwe-prometheus
    restart: unless-stopped
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus-global.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_global_data:/prometheus
    networks:
      - goodwe-global

# Secrets
secrets:
  db_password:
    external: true
  redis_password:
    external: true
  grafana_password:
    external: true

# Volumes
volumes:
  excel_data:
    driver: local
  excel_logs:
    driver: local
  ml_models:
    driver: local
  ml_logs:
    driver: local
  postgres_global_data:
    driver: local
  redis_global_data:
    driver: local
  grafana_data:
    driver: local
  prometheus_global_data:
    driver: local

# Networks
networks:
  goodwe-global:
    driver: bridge
    ipam:
      config:
        - subnet: 172.25.0.0/16
```

## 🔐 Configurações de Segurança

### Secrets Management

```bash
# Criar secrets
echo "super_secure_db_password" | docker secret create db_password -
echo "super_secure_redis_password" | docker secret create redis_password -
echo "super_secure_grafana_password" | docker secret create grafana_password -

# Ou usando arquivos
docker secret create db_password ./secrets/db_password.txt
docker secret create api_key ./secrets/api_key.txt
```

### Configuração de Rede Segura

```yaml
# Rede isolada para database
networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true  # Sem acesso à internet
  database:
    driver: bridge
    internal: true  # Completamente isolada
```

### Hardening de Containers

```dockerfile
# Exemplo de Dockerfile com segurança reforçada
FROM node:18-alpine

# Instalar apenas o necessário
RUN apk add --no-cache dumb-init

# Criar usuário com UID específico
RUN adduser -D -s /bin/sh -u 1001 appuser

# Usar usuário não-root
USER appuser

# Definir pontos de montagem seguros
VOLUME ["/app/data"]

# Remover capacidades desnecessárias
# (usar no docker-compose)
# cap_drop:
#   - ALL
# cap_add:
#   - SETGID
#   - SETUID
```

## 📊 Monitoramento e Logging

### Configuração do Prometheus

```yaml
# monitoring/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "alert_rules.yml"

scrape_configs:
  - job_name: 'goodwe-apis'
    static_configs:
      - targets: ['api-excel:3000', 'api-ml:8000']
    metrics_path: /metrics
    scrape_interval: 10s

  - job_name: 'postgres'
    static_configs:
      - targets: ['postgres:5432']

  - job_name: 'redis'
    static_configs:
      - targets: ['redis:6379']

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093
```

### Logging Centralizado

```yaml
# Configuração de logging
services:
  app:
    logging:
      driver: "fluentd"
      options:
        fluentd-address: localhost:24224
        tag: goodwe.api
        
  fluentd:
    image: fluent/fluentd:latest
    volumes:
      - ./fluentd/conf:/fluentd/etc
      - /var/log:/var/log
    ports:
      - "24224:24224"
```

## 🚀 Scripts de Deployment

### deploy.sh

```bash
#!/bin/bash
set -e

echo "🚀 Iniciando deployment das APIs GoodWe..."

# Variáveis
ENVIRONMENT=${1:-production}
COMPOSE_FILE="docker-compose.yml"

if [ "$ENVIRONMENT" = "development" ]; then
    COMPOSE_FILE="docker-compose.dev.yml"
fi

echo "📋 Ambiente: $ENVIRONMENT"
echo "📁 Arquivo compose: $COMPOSE_FILE"

# Parar containers antigos
echo "🛑 Parando containers existentes..."
docker-compose -f $COMPOSE_FILE down

# Fazer backup se necessário
if [ "$ENVIRONMENT" = "production" ]; then
    echo "💾 Fazendo backup dos dados..."
    ./scripts/backup.sh
fi

# Build das imagens
echo "🔨 Building imagens..."
docker-compose -f $COMPOSE_FILE build --no-cache

# Executar testes de healthcheck
echo "🧪 Testando imagens..."
docker-compose -f $COMPOSE_FILE up -d
sleep 30

# Verificar health checks
echo "❤️ Verificando health checks..."
for service in api-goodwe postgres redis; do
    if docker-compose -f $COMPOSE_FILE ps | grep -q "$service.*healthy"; then
        echo "✅ $service: OK"
    else
        echo "❌ $service: FAIL"
        docker-compose -f $COMPOSE_FILE logs $service
        exit 1
    fi
done

echo "🎉 Deployment concluído com sucesso!"
```

### backup.sh

```bash
#!/bin/bash
set -e

BACKUP_DIR="/backup/goodwe"
DATE=$(date +%Y%m%d_%H%M%S)

echo "💾 Iniciando backup - $DATE"

# Criar diretório de backup
mkdir -p $BACKUP_DIR

# Backup PostgreSQL
echo "📊 Backup do banco de dados..."
docker exec goodwe-postgres pg_dump -U goodwe goodwe > $BACKUP_DIR/db_$DATE.sql

# Backup de dados da aplicação
echo "📁 Backup dos dados da aplicação..."
tar -czf $BACKUP_DIR/app_data_$DATE.tar.gz /opt/goodwe/data/

# Backup de configurações
echo "⚙️ Backup das configurações..."
tar -czf $BACKUP_DIR/configs_$DATE.tar.gz nginx/ monitoring/ secrets/

# Limpeza de backups antigos (manter apenas últimos 7 dias)
echo "🧹 Limpando backups antigos..."
find $BACKUP_DIR -name "*.sql" -mtime +7 -delete
find $BACKUP_DIR -name "*.tar.gz" -mtime +7 -delete

echo "✅ Backup concluído: $BACKUP_DIR"
```

---

**Seção anterior**: [Deployment](./deployment.md)  
**Próxima seção**: [Troubleshooting](./troubleshooting.md)  
**Voltar ao início**: [README](./README.md)
