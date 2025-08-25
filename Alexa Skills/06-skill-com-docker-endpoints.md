# Alexa Skill com Docker e Endpoints Customizados

## Visão Geral da Arquitetura

Em vez de usar AWS Lambda, podemos criar uma Alexa Skill que se conecta a um endpoint HTTPS customizado rodando em Docker. Esta abordagem oferece mais flexibilidade e controle sobre a infraestrutura.

## Arquitetura Completa

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────────┐
│   Alexa Device  │───▶│  Amazon Alexa    │───▶│   Seu Servidor      │
│   (Echo, etc.)  │    │     Service      │    │   Docker + HTTPS   │
└─────────────────┘    └──────────────────┘    └─────────────────────┘
                                                          │
                                                          ▼
                                               ┌─────────────────────┐
                                               │   API GoodWe        │
                                               │   (Energia Solar)   │
                                               └─────────────────────┘
```

## 1. Configuração da Skill no Alexa Developer Console

### Endpoint Configuration
No Alexa Developer Console, em vez de usar AWS Lambda, você configurará:

```json
{
  "manifest": {
    "apis": {
      "custom": {
        "endpoint": {
          "uri": "https://seu-dominio.com/alexa-webhook",
          "sslCertificateType": "Trusted"
        }
      }
    }
  }
}
```

### Requisitos do Endpoint
- **HTTPS obrigatório**: Certificado SSL válido
- **Porta 443**: Padrão HTTPS
- **Verificação de Request**: Validar que requisições vêm da Amazon
- **Timeout**: Responder em até 8 segundos

## 2. Estrutura do Projeto Docker

```
alexa-skill-docker/
├── docker-compose.yml
├── Dockerfile
├── nginx/
│   ├── nginx.conf
│   └── ssl/
│       ├── cert.pem
│       └── key.pem
├── app/
│   ├── package.json
│   ├── server.js
│   ├── routes/
│   │   └── alexa.js
│   ├── middleware/
│   │   ├── alexaVerification.js
│   │   └── requestLogger.js
│   ├── handlers/
│   │   ├── energyHandlers.js
│   │   ├── deviceHandlers.js
│   │   └── builtInHandlers.js
│   ├── services/
│   │   ├── goodweApi.js
│   │   └── alexaResponse.js
│   └── utils/
│       ├── dataFormatter.js
│       └── constants.js
└── .env
```

## 3. Configuração Docker

### docker-compose.yml
```yaml
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./nginx/ssl:/etc/nginx/ssl
    depends_on:
      - alexa-app
    restart: unless-stopped

  alexa-app:
    build: .
    environment:
      - NODE_ENV=production
      - PORT=3000
      - GOODWE_API_KEY=${GOODWE_API_KEY}
      - GOODWE_API_URL=${GOODWE_API_URL}
      - ALEXA_SKILL_ID=${ALEXA_SKILL_ID}
    volumes:
      - ./app:/usr/src/app
    expose:
      - "3000"
    restart: unless-stopped
    depends_on:
      - redis

  redis:
    image: redis:alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    restart: unless-stopped

volumes:
  redis_data:
```

### Dockerfile
```dockerfile
FROM node:18-alpine

# Instalar dependências do sistema
RUN apk add --no-cache \
    ca-certificates \
    openssl

# Criar diretório da aplicação
WORKDIR /usr/src/app

# Copiar package.json e package-lock.json
COPY app/package*.json ./

# Instalar dependências
RUN npm ci --only=production && npm cache clean --force

# Copiar código da aplicação
COPY app/ .

# Criar usuário não-root
RUN addgroup -g 1001 -S nodejs && \
    adduser -S alexa -u 1001 -G nodejs

# Mudar propriedade dos arquivos
RUN chown -R alexa:nodejs /usr/src/app
USER alexa

# Expor porta
EXPOSE 3000

# Healthcheck
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1

# Comando para iniciar a aplicação
CMD ["npm", "start"]
```

### nginx/nginx.conf
```nginx
events {
    worker_connections 1024;
}

http {
    upstream alexa_app {
        server alexa-app:3000;
    }

    # Rate limiting
    limit_req_zone $binary_remote_addr zone=alexa:10m rate=10r/s;

    server {
        listen 80;
        server_name seu-dominio.com;
        
        # Redirect HTTP to HTTPS
        return 301 https://$server_name$request_uri;
    }

    server {
        listen 443 ssl http2;
        server_name seu-dominio.com;

        # SSL Configuration
        ssl_certificate /etc/nginx/ssl/cert.pem;
        ssl_certificate_key /etc/nginx/ssl/key.pem;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384;
        ssl_prefer_server_ciphers off;

        # Security headers
        add_header X-Frame-Options DENY;
        add_header X-Content-Type-Options nosniff;
        add_header X-XSS-Protection "1; mode=block";

        location /alexa-webhook {
            # Rate limiting
            limit_req zone=alexa burst=20 nodelay;
            
            # Proxy to Node.js app
            proxy_pass http://alexa_app/alexa;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_cache_bypass $http_upgrade;
            
            # Timeout settings
            proxy_connect_timeout 30s;
            proxy_send_timeout 30s;
            proxy_read_timeout 30s;
        }

        location /health {
            proxy_pass http://alexa_app/health;
        }
    }
}
```

## 4. Implementação Node.js

### app/package.json
```json
{
  "name": "alexa-skill-goodwe",
  "version": "1.0.0",
  "description": "Alexa Skill para energia solar GoodWe com Docker",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js",
    "test": "jest",
    "lint": "eslint ."
  },
  "dependencies": {
    "express": "^4.18.2",
    "ask-sdk-express-adapter": "^2.13.0",
    "ask-sdk-core": "^2.13.0",
    "ask-sdk-model": "^1.49.0",
    "axios": "^1.6.0",
    "redis": "^4.6.0",
    "helmet": "^7.0.0",
    "cors": "^2.8.5",
    "compression": "^1.7.4",
    "morgan": "^1.10.0",
    "dotenv": "^16.3.0",
    "crypto": "^1.0.1"
  },
  "devDependencies": {
    "nodemon": "^3.0.0",
    "jest": "^29.0.0",
    "eslint": "^8.0.0"
  }
}
```

### app/server.js
```javascript
const express = require('express');
const helmet = require('helmet');
const compression = require('compression');
const morgan = require('morgan');
const cors = require('cors');
require('dotenv').config();

const alexaRoute = require('./routes/alexa');
const requestLogger = require('./middleware/requestLogger');

const app = express();
const PORT = process.env.PORT || 3000;

// Security middleware
app.use(helmet({
    contentSecurityPolicy: {
        directives: {
            defaultSrc: ["'self'"],
            scriptSrc: ["'self'"],
            styleSrc: ["'self'", "'unsafe-inline'"],
            imgSrc: ["'self'", "data:", "https:"],
        },
    },
}));

// Compression and CORS
app.use(compression());
app.use(cors({
    origin: ['https://alexa.amazon.com', 'https://alexa.amazon.co.uk'],
    credentials: true
}));

// Logging
app.use(morgan('combined'));
app.use(requestLogger);

// Body parsing
app.use(express.json({ limit: '10mb' }));
app.use(express.urlencoded({ extended: true, limit: '10mb' }));

// Routes
app.use('/alexa', alexaRoute);

// Health check
app.get('/health', (req, res) => {
    res.status(200).json({
        status: 'healthy',
        timestamp: new Date().toISOString(),
        uptime: process.uptime(),
        version: process.env.npm_package_version || '1.0.0'
    });
});

// Error handling
app.use((err, req, res, next) => {
    console.error('Global error handler:', err);
    
    res.status(err.status || 500).json({
        error: process.env.NODE_ENV === 'production' 
            ? 'Internal Server Error' 
            : err.message
    });
});

// 404 handler
app.use('*', (req, res) => {
    res.status(404).json({ error: 'Route not found' });
});

// Graceful shutdown
process.on('SIGTERM', () => {
    console.log('SIGTERM received. Shutting down gracefully...');
    process.exit(0);
});

process.on('SIGINT', () => {
    console.log('SIGINT received. Shutting down gracefully...');
    process.exit(0);
});

app.listen(PORT, '0.0.0.0', () => {
    console.log(`🚀 Alexa Skill server running on port ${PORT}`);
    console.log(`📊 Environment: ${process.env.NODE_ENV || 'development'}`);
});

module.exports = app;
```

### app/middleware/alexaVerification.js
```javascript
const crypto = require('crypto');

class AlexaVerification {
    static verifySignature(req, res, next) {
        try {
            const signature = req.headers['signature'];
            const signatureCertChainUrl = req.headers['signaturecertchainurl'];
            const body = JSON.stringify(req.body);

            if (!signature || !signatureCertChainUrl) {
                return res.status(401).json({ error: 'Missing required headers' });
            }

            // Verificar URL do certificado
            if (!AlexaVerification.isValidCertUrl(signatureCertChainUrl)) {
                return res.status(401).json({ error: 'Invalid certificate URL' });
            }

            // Verificar timestamp da requisição (não pode ser mais que 150s antiga)
            const requestTimestamp = new Date(req.body.request.timestamp);
            const now = new Date();
            const timeDiff = Math.abs(now - requestTimestamp);
            
            if (timeDiff > 150000) { // 150 segundos
                return res.status(401).json({ error: 'Request too old' });
            }

            // Verificar se a requisição é para nossa skill
            const skillId = req.body.session?.application?.applicationId || 
                           req.body.context?.System?.application?.applicationId;
            
            if (skillId !== process.env.ALEXA_SKILL_ID) {
                return res.status(401).json({ error: 'Invalid skill ID' });
            }

            next();
        } catch (error) {
            console.error('Alexa verification error:', error);
            res.status(401).json({ error: 'Verification failed' });
        }
    }

    static isValidCertUrl(url) {
        try {
            const urlObj = new URL(url);
            return urlObj.protocol === 'https:' &&
                   urlObj.hostname === 's3.amazonaws.com' &&
                   urlObj.pathname.startsWith('/echo.api/') &&
                   urlObj.pathname.endsWith('.pem');
        } catch {
            return false;
        }
    }
}

module.exports = AlexaVerification;
```

### app/routes/alexa.js
```javascript
const express = require('express');
const { ExpressAdapter } = require('ask-sdk-express-adapter');
const { SkillBuilders } = require('ask-sdk-core');

const AlexaVerification = require('../middleware/alexaVerification');
const energyHandlers = require('../handlers/energyHandlers');
const deviceHandlers = require('../handlers/deviceHandlers');
const builtInHandlers = require('../handlers/builtInHandlers');

const router = express.Router();

// Construir skill
const skill = SkillBuilders.custom()
    .addRequestHandlers(
        builtInHandlers.LaunchRequestHandler,
        energyHandlers.GetEnergyConsumptionHandler,
        energyHandlers.GetEnergyProductionHandler,
        energyHandlers.GetSavingsHandler,
        deviceHandlers.GetDeviceStatusHandler,
        deviceHandlers.GetWeatherImpactHandler,
        builtInHandlers.HelpIntentHandler,
        builtInHandlers.CancelAndStopIntentHandler,
        builtInHandlers.SessionEndedRequestHandler,
        builtInHandlers.FallbackIntentHandler
    )
    .addErrorHandlers(builtInHandlers.ErrorHandler)
    .addRequestInterceptors({
        process(handlerInput) {
            console.log('📥 Request:', JSON.stringify(handlerInput.requestEnvelope.request, null, 2));
        }
    })
    .addResponseInterceptors({
        process(handlerInput, response) {
            console.log('📤 Response:', JSON.stringify(response, null, 2));
        }
    })
    .create();

// Criar adapter
const adapter = new ExpressAdapter(skill, false, false);

// Aplicar verificação apenas em produção
if (process.env.NODE_ENV === 'production') {
    router.use(AlexaVerification.verifySignature);
}

// Rota principal para requisições da Alexa
router.post('/', adapter.getRequestHandlers());

// Rota de teste para desenvolvimento
if (process.env.NODE_ENV !== 'production') {
    router.get('/test', (req, res) => {
        res.json({
            message: 'Alexa Skill endpoint is working',
            timestamp: new Date().toISOString(),
            environment: process.env.NODE_ENV
        });
    });
}

module.exports = router;
```

## 5. Configuração de Deploy

### .env (template)
```bash
# Ambiente
NODE_ENV=production
PORT=3000

# Alexa
ALEXA_SKILL_ID=amzn1.ask.skill.xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx

# GoodWe API
GOODWE_API_KEY=sua_chave_api_aqui
GOODWE_API_URL=https://api.goodwe.com/v1

# Redis
REDIS_URL=redis://redis:6379

# SSL (para desenvolvimento local)
SSL_CERT_PATH=/path/to/cert.pem
SSL_KEY_PATH=/path/to/key.pem
```

### Scripts de Deploy

#### deploy.sh
```bash
#!/bin/bash

echo "🚀 Deploying Alexa Skill..."

# Verificar se .env existe
if [ ! -f .env ]; then
    echo "❌ .env file not found!"
    exit 1
fi

# Build e start containers
docker-compose down
docker-compose build --no-cache
docker-compose up -d

# Aguardar serviços ficarem prontos
echo "⏳ Waiting for services to be ready..."
sleep 30

# Verificar health
echo "🔍 Checking health..."
curl -f https://seu-dominio.com/health || {
    echo "❌ Health check failed!"
    docker-compose logs
    exit 1
}

echo "✅ Deploy completed successfully!"
```

## 6. Configuração no Alexa Developer Console

### 1. Criar Nova Skill
- Acesse [Alexa Developer Console](https://developer.amazon.com/alexa/console/ask)
- Create Skill → Custom → Provision your own

### 2. Configurar Endpoint
```
Skill Invocation → Endpoint
- HTTPS
- Default Region: https://seu-dominio.com/alexa-webhook
- SSL Certificate Type: My development endpoint has a certificate from a trusted certificate authority
```

### 3. Configurar Interaction Model
Use o mesmo JSON do arquivo `02-criando-intents.md`

### 4. Testar
- Use o simulador integrado
- Teste com dispositivos físicos

## 7. Monitoramento e Logs

### Docker Logs
```bash
# Ver logs em tempo real
docker-compose logs -f alexa-app

# Ver logs do nginx
docker-compose logs -f nginx

# Ver métricas
docker stats
```

### Métricas Customizadas
```javascript
// app/middleware/metrics.js
const redis = require('redis');
const client = redis.createClient(process.env.REDIS_URL);

const metrics = {
    async incrementCounter(metric) {
        try {
            await client.incr(`metrics:${metric}`);
            await client.expire(`metrics:${metric}`, 86400); // 24h TTL
        } catch (error) {
            console.error('Metrics error:', error);
        }
    },

    async getMetrics() {
        try {
            const keys = await client.keys('metrics:*');
            const metrics = {};
            
            for (const key of keys) {
                const value = await client.get(key);
                metrics[key.replace('metrics:', '')] = parseInt(value) || 0;
            }
            
            return metrics;
        } catch (error) {
            console.error('Get metrics error:', error);
            return {};
        }
    }
};

module.exports = metrics;
```

## 8. Vantagens desta Abordagem

### ✅ Benefícios
- **Controle Total**: Infraestrutura própria
- **Debugging**: Logs completos e acesso direto
- **Escalabilidade**: Docker Swarm ou Kubernetes
- **Integração**: Fácil conexão com outros serviços
- **Custo**: Potencialmente menor que AWS Lambda

### ⚠️ Considerações
- **SSL**: Certificado válido obrigatório
- **Uptime**: Você é responsável pela disponibilidade
- **Segurança**: Implementar verificações adequadas
- **Performance**: Responder em até 8 segundos

## 9. Próximos Passos

1. Configure certificado SSL válido
2. Ajuste o domínio nos arquivos de configuração
3. Configure as variáveis de ambiente
4. Execute o deploy com Docker
5. Configure a skill no Alexa Developer Console
6. Teste a integração

Esta arquitetura oferece máxima flexibilidade mantendo compatibilidade total com o ecosistema Alexa!

---

*Este documento mostra como implementar uma Alexa Skill usando Docker e endpoints customizados em vez de AWS Lambda.*
