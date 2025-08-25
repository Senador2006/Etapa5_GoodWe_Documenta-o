# 6. Troubleshooting e FAQ

## 🐛 Problemas Comuns e Soluções

### 1. Problemas de Desenvolvimento

#### ❌ Skill não responde ou não inicia
**Sintomas:**
- "Não sei como ajudar com isso"
- Skill não abre quando invocada
- Timeout de resposta

**Soluções:**
```javascript
// 1. Verificar logs do CloudWatch
console.log('LaunchRequest received:', JSON.stringify(handlerInput.requestEnvelope, null, 2));

// 2. Validar handler registration
exports.handler = Alexa.SkillBuilders.custom()
    .addRequestHandlers(
        LaunchRequestHandler, // Certifique-se que está registrado
        // ... outros handlers
    )
    .addErrorHandlers(ErrorHandler)
    .lambda();

// 3. Verificar canHandle method
const LaunchRequestHandler = {
    canHandle(handlerInput) {
        return Alexa.getRequestType(handlerInput.requestEnvelope) === 'LaunchRequest';
    },
    handle(handlerInput) {
        // Implementação
    }
};
```

#### ❌ Intent não é reconhecido
**Sintomas:**
- IntentReflectorHandler sempre executado
- "Não entendi esse comando"

**Soluções:**
```json
// 1. Verificar interaction model - utterances suficientes
{
  "name": "GetWeatherIntent",
  "samples": [
    "qual o tempo",
    "como está o clima",
    "previsão do tempo",
    "vai chover",
    "como está o tempo hoje",
    "me diga o clima"
  ]
}

// 2. Build do modelo
ask api update-interaction-model --skill-id YOUR_SKILL_ID --stage development --locale pt-BR --interaction-model file:models/pt-BR.json

// 3. Aguardar build completion
ask api get-interaction-model-build-status --skill-id YOUR_SKILL_ID --stage development --locale pt-BR
```

#### ❌ Slots não são preenchidos corretamente
**Sintomas:**
- Slot value sempre null/undefined
- Slot não captura variações

**Soluções:**
```javascript
// 1. Validação robusta de slots
const slotValue = Alexa.getSlotValue(handlerInput.requestEnvelope, 'slotName');
const resolvedValue = Alexa.getSlotValue(handlerInput.requestEnvelope, 'slotName', 'SYNONYM');

if (!slotValue) {
    return handlerInput.responseBuilder
        .speak('Não entendi o valor. Pode repetir?')
        .addElicitSlotDirective('slotName')
        .getResponse();
}

// 2. Slot types mais específicos
{
  "types": [
    {
      "name": "CityType",
      "values": [
        {
          "name": {
            "value": "São Paulo",
            "synonyms": ["SP", "sampa", "são paulo", "capital paulista"]
          }
        }
      ]
    }
  ]
}

// 3. Usar slot validation
{
  "name": "GetWeatherIntent",
  "slots": [
    {
      "name": "city",
      "type": "CityType",
      "samples": [
        "em {city}",
        "na cidade de {city}",
        "para {city}"
      ]
    }
  ]
}
```

### 2. Problemas de Deploy

#### ❌ Lambda function timeout
**Sintomas:**
- Skill para de responder após 8 segundos
- Timeout errors nos logs

**Soluções:**
```javascript
// 1. Otimizar performance
const https = require('https');
const keepAliveAgent = new https.Agent({ keepAlive: true });

// Reutilizar conexões
const fetchWithTimeout = async (url, options = {}) => {
    const controller = new AbortController();
    const timeoutId = setTimeout(() => controller.abort(), 5000); // 5s timeout
    
    try {
        const response = await fetch(url, {
            ...options,
            agent: keepAliveAgent,
            signal: controller.signal
        });
        clearTimeout(timeoutId);
        return response;
    } catch (error) {
        clearTimeout(timeoutId);
        throw error;
    }
};

// 2. Implementar cache
const cache = new Map();
const CACHE_TTL = 5 * 60 * 1000; // 5 minutos

async function getCachedData(key, fetchFunction) {
    const cached = cache.get(key);
    
    if (cached && Date.now() - cached.timestamp < CACHE_TTL) {
        return cached.data;
    }
    
    const data = await fetchFunction();
    cache.set(key, { data, timestamp: Date.now() });
    return data;
}

// 3. Configurar timeout no AWS Lambda
// Memory: 512MB (mínimo recomendado)
// Timeout: 30 segundos (máximo para Alexa)
```

#### ❌ Problemas de permissão IAM
**Sintomas:**
- "Access Denied" nos logs
- Função não consegue acessar DynamoDB/outros serviços

**Soluções:**
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "dynamodb:GetItem",
                "dynamodb:PutItem",
                "dynamodb:UpdateItem",
                "dynamodb:DeleteItem",
                "dynamodb:Query",
                "dynamodb:Scan"
            ],
            "Resource": "arn:aws:dynamodb:us-east-1:123456789:table/YourTableName"
        },
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogGroup",
                "logs:CreateLogStream", 
                "logs:PutLogEvents"
            ],
            "Resource": "*"
        }
    ]
}
```

### 3. Problemas de Certificação

#### ❌ Skill rejeitada na certificação
**Razões comuns:**

1. **Invocation name inadequado**
```json
// ❌ Muito genérico
"invocationName": "app"

// ✅ Específico e único
"invocationName": "assistente pessoal empresa"
```

2. **Help intent inadequado**
```javascript
// ❌ Ajuda genérica
const HelpIntentHandler = {
    handle(handlerInput) {
        return handlerInput.responseBuilder
            .speak('Como posso ajudar?')
            .getResponse();
    }
};

// ✅ Ajuda específica e útil
const HelpIntentHandler = {
    handle(handlerInput) {
        const speakOutput = `Eu posso ajudar você com:
            - Previsão do tempo: diga "qual o tempo hoje"
            - Notícias: diga "quais as notícias"
            - Lembretes: diga "criar lembrete"
            O que gostaria de fazer?`;
            
        return handlerInput.responseBuilder
            .speak(speakOutput)
            .reprompt('Como posso ajudar você?')
            .getResponse();
    }
};
```

3. **Tratamento inadequado de SessionEndedRequest**
```javascript
// ✅ Sempre incluir
const SessionEndedRequestHandler = {
    canHandle(handlerInput) {
        return Alexa.getRequestType(handlerInput.requestEnvelope) === 'SessionEndedRequest';
    },
    handle(handlerInput) {
        console.log(`Session ended: ${JSON.stringify(handlerInput.requestEnvelope)}`);
        return handlerInput.responseBuilder.getResponse();
    }
};
```

### 4. Problemas de Performance

#### ❌ Resposta lenta da skill
**Otimizações:**

```javascript
// 1. Paralelizar calls de API
const [weatherData, newsData] = await Promise.all([
    getWeatherData(city),
    getNewsData(category)
]);

// 2. Implementar connection pooling
const AWS = require('aws-sdk');
const dynamodb = new AWS.DynamoDB.DocumentClient({
    maxRetries: 3,
    retryDelayOptions: {
        customBackoff: function(retryCount) {
            return Math.pow(2, retryCount) * 100;
        }
    }
});

// 3. Otimizar queries DynamoDB
const params = {
    TableName: 'UserData',
    KeyConditionExpression: 'userId = :userId',
    ExpressionAttributeValues: {
        ':userId': userId
    },
    Limit: 10 // Limitar results
};

// 4. Usar lazy loading
let cachedUserData = null;

async function getUserData(userId) {
    if (!cachedUserData) {
        cachedUserData = await dynamodb.get({
            TableName: 'UserData',
            Key: { userId }
        }).promise();
    }
    
    return cachedUserData.Item;
}
```

## 🔍 Debugging Avançado

### 1. Logs Estruturados

```javascript
const winston = require('winston');

const logger = winston.createLogger({
    level: process.env.LOG_LEVEL || 'info',
    format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.errors({ stack: true }),
        winston.format.json()
    ),
    transports: [
        new winston.transports.Console()
    ]
});

// Usar em handlers
const MyIntentHandler = {
    handle(handlerInput) {
        const userId = handlerInput.requestEnvelope.session.user.userId;
        const sessionId = handlerInput.requestEnvelope.session.sessionId;
        
        logger.info('Intent started', {
            intent: 'MyIntent',
            userId: userId.substring(0, 10) + '...', // Não logar userId completo
            sessionId,
            timestamp: new Date().toISOString()
        });
        
        try {
            // Lógica do handler
            const result = processIntent();
            
            logger.info('Intent completed successfully', {
                intent: 'MyIntent',
                sessionId,
                duration: Date.now() - startTime
            });
            
            return result;
            
        } catch (error) {
            logger.error('Intent failed', {
                intent: 'MyIntent',
                sessionId,
                error: error.message,
                stack: error.stack
            });
            
            throw error;
        }
    }
};
```

### 2. Testing Framework

```javascript
// test/handlers.test.js
const Alexa = require('ask-sdk-core');
const { handler } = require('../index');

describe('Alexa Skill Tests', () => {
    test('LaunchRequest should return welcome message', async () => {
        const event = {
            version: '1.0',
            session: {
                new: true,
                sessionId: 'test-session',
                user: { userId: 'test-user' }
            },
            request: {
                type: 'LaunchRequest',
                requestId: 'test-request',
                timestamp: new Date().toISOString(),
                locale: 'pt-BR'
            }
        };
        
        const context = {};
        const result = await handler(event, context);
        
        expect(result.response.outputSpeech.ssml).toContain('Bem-vindo');
        expect(result.response.shouldEndSession).toBe(false);
    });
    
    test('GetWeatherIntent should return weather info', async () => {
        const event = {
            // ... setup event for GetWeatherIntent
            request: {
                type: 'IntentRequest',
                intent: {
                    name: 'GetWeatherIntent',
                    slots: {
                        city: {
                            name: 'city',
                            value: 'São Paulo'
                        }
                    }
                }
            }
        };
        
        const result = await handler(event, {});
        
        expect(result.response.outputSpeech.ssml).toContain('temperatura');
    });
});

// Executar testes
npm test
```

### 3. Monitoramento em Tempo Real

```javascript
// CloudWatch Custom Metrics
const AWS = require('aws-sdk');
const cloudwatch = new AWS.CloudWatch();

class MetricsCollector {
    static async putMetric(metricName, value, unit = 'Count', dimensions = []) {
        const params = {
            Namespace: 'AlexaSkill/MySkill',
            MetricData: [{
                MetricName: metricName,
                Value: value,
                Unit: unit,
                Dimensions: dimensions,
                Timestamp: new Date()
            }]
        };
        
        try {
            await cloudwatch.putMetricData(params).promise();
        } catch (error) {
            console.error('Failed to put metric:', error);
        }
    }
    
    static async putTimer(metricName, startTime, dimensions = []) {
        const duration = Date.now() - startTime;
        await this.putMetric(metricName, duration, 'Milliseconds', dimensions);
    }
}

// Usar nos handlers
const TimedHandler = {
    async handle(handlerInput) {
        const startTime = Date.now();
        
        try {
            const result = await processRequest(handlerInput);
            
            await MetricsCollector.putMetric('RequestSuccess', 1);
            await MetricsCollector.putTimer('RequestDuration', startTime);
            
            return result;
            
        } catch (error) {
            await MetricsCollector.putMetric('RequestError', 1);
            throw error;
        }
    }
};
```

## ❓ FAQ - Perguntas Frequentes

### **Q: Como fazer minha skill entender diferentes sotaques regionais?**
**A:** Use utterances variadas e sinônimos:
```json
{
  "samples": [
    "tá chovendo",
    "está chovendo", 
    "tem chuva",
    "vai chover",
    "vai pingar",
    "tá pingando"
  ]
}
```

### **Q: Posso usar minha skill em outros países?**
**A:** Sim, configure locales adicionais:
```json
{
  "publishingInformation": {
    "locales": {
      "pt-BR": { "name": "Minha Skill Brasil" },
      "en-US": { "name": "My Skill US" },
      "es-ES": { "name": "Mi Skill España" }
    },
    "distributionCountries": ["BR", "US", "ES"]
  }
}
```

### **Q: Como proteger dados sensíveis na skill?**
**A:** 
1. Use AWS KMS para encryption
2. Não logue dados pessoais
3. Implemente account linking
4. Use HTTPS para todas as APIs

```javascript
const AWS = require('aws-sdk');
const kms = new AWS.KMS();

async function encryptSensitiveData(data) {
    const params = {
        KeyId: process.env.KMS_KEY_ID,
        Plaintext: Buffer.from(data)
    };
    
    const result = await kms.encrypt(params).promise();
    return result.CiphertextBlob.toString('base64');
}
```

### **Q: Como implementar rate limiting?**
**A:** Use DynamoDB para controle:
```javascript
async function checkRateLimit(userId) {
    const now = Date.now();
    const windowStart = now - (60 * 1000); // 1 minuto
    
    const params = {
        TableName: 'RateLimits',
        Key: { userId },
        UpdateExpression: 'SET requestCount = if_not_exists(requestCount, :zero) + :incr, lastRequest = :now',
        ConditionExpression: 'attribute_not_exists(lastRequest) OR lastRequest < :windowStart',
        ExpressionAttributeValues: {
            ':zero': 0,
            ':incr': 1,
            ':now': now,
            ':windowStart': windowStart
        }
    };
    
    try {
        await dynamodb.update(params).promise();
        return true; // Permitido
    } catch (error) {
        if (error.code === 'ConditionalCheckFailedException') {
            return false; // Rate limit excedido
        }
        throw error;
    }
}
```

### **Q: Como implementar multi-tenancy?**
**A:** Use organization ID em todos os dados:
```javascript
async function getUserData(userId, organizationId) {
    const params = {
        TableName: 'UserData',
        Key: {
            pk: `ORG#${organizationId}`,
            sk: `USER#${userId}`
        }
    };
    
    return await dynamodb.get(params).promise();
}
```

## 🛠️ Ferramentas de Debug

### 1. ASK CLI Debug Commands
```bash
# Verificar skill status
ask api get-skill-status --skill-id YOUR_SKILL_ID

# Simular request
ask dialog --locale pt-BR --replay-file replay.json

# Ver logs em tempo real
ask logs --skill-id YOUR_SKILL_ID --stage development
```

### 2. Testing Tools
```bash
# Bespoken Tools para testing
npm install -g bespoken-tools

# Criar test suite
bst test test/test-suite.yml
```

### 3. APL Testing
```bash
# APL Author tool para designs visuais
# https://developer.amazon.com/alexa/console/ask/displays
```

## 📞 Suporte e Recursos

### Canais Oficiais:
- **Developer Forums**: [forums.developer.amazon.com](https://forums.developer.amazon.com)
- **Stack Overflow**: Tag `alexa-skills-kit`
- **GitHub**: [alexa/alexa-skills-kit-sdk-for-nodejs](https://github.com/alexa/alexa-skills-kit-sdk-for-nodejs)

### Documentação:
- **Official Docs**: [developer.amazon.com/docs/alexa](https://developer.amazon.com/docs/alexa)
- **SDK Reference**: [ask-sdk-for-nodejs.readthedocs.io](https://ask-sdk-for-nodejs.readthedocs.io)

### Comunidade Brasileira:
- **Slack**: #alexa-brasil
- **Telegram**: @alexabrasil
- **Meetups**: Pesquise por "Alexa" na sua cidade

---

**Parabéns!** 🎉 Você completou o guia completo de Alexa Skills. Com esse conhecimento, você está pronto para criar skills profissionais e robustas para a Amazon Alexa.

---
**Anterior:** [← Casos Avançados](05-casos-avancados.md) | **Início:** [README](README.md)
