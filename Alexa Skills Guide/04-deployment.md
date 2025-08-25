# 4. Deployment e Publicação

## 🚀 Preparação para Deploy

### 1. Checklist Pré-Deploy

#### ✅ Funcionalidade
- [ ] Todos os intents funcionam corretamente
- [ ] Tratamento de erros implementado
- [ ] Fallbacks para cenários não mapeados
- [ ] Testes em diferentes dispositivos Alexa
- [ ] Performance otimizada (< 8 segundos resposta)

#### ✅ Qualidade de Código
- [ ] Logs adequados para debugging
- [ ] Código documentado e organizado
- [ ] Validação de entrada de dados
- [ ] Segurança implementada (não exposição de dados sensíveis)

#### ✅ Experiência do Usuário
- [ ] Respostas naturais e conversacionais
- [ ] Ajuda contextual disponível
- [ ] Comandos de cancelar/parar funcionando
- [ ] Cards visuais (quando aplicável)

## 🧪 Ambiente de Testes

### 1. Teste Local com ASK CLI

```bash
# Testar skill localmente
ask dialog --locale pt-BR

# Exemplo de sessão de teste
User > abra assistente pessoal
Alexa > Olá! Bem-vindo ao seu assistente pessoal. Como posso ajudar?
User > qual o tempo hoje
Alexa > A temperatura hoje é 25 graus e o tempo está ensolarado.
```

### 2. Simulador Web
- Acesse Developer Console > Test
- Configure locale para pt-BR
- Teste scenarios completos
- Verifique JSON requests/responses

### 3. Dispositivos Beta
```bash
# Habilitar skill para teste beta
ask api enable-skill-for-development --skill-id amzn1.ask.skill.12345
```

## 🔧 Configuração de Produção

### 1. AWS Lambda Configuration

#### Environment Variables
```javascript
// No console AWS Lambda
const config = {
    NEWS_API_KEY: process.env.NEWS_API_KEY,
    DATABASE_URL: process.env.DATABASE_URL,
    LOG_LEVEL: process.env.LOG_LEVEL || 'info'
};
```

#### Timeout Settings
```javascript
// Configuração recomendada
{
    "timeout": 30, // segundos
    "memorySize": 512, // MB
    "runtime": "nodejs18.x"
}
```

#### IAM Permissions
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
                "dynamodb:Scan",
                "dynamodb:Query"
            ],
            "Resource": "arn:aws:dynamodb:*:*:table/AlexaSkillTable"
        },
        {
            "Effect": "Allow", 
            "Action": [
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": "arn:aws:logs:*:*:*"
        }
    ]
}
```

### 2. Skill Manifest Completo

```json
{
    "manifest": {
        "publishingInformation": {
            "locales": {
                "pt-BR": {
                    "summary": "Assistente pessoal para tarefas do dia a dia",
                    "examplePhrases": [
                        "Alexa, abra assistente pessoal",
                        "Alexa, peça para assistente pessoal o tempo",
                        "Alexa, diga para assistente pessoal olá"
                    ],
                    "keywords": [
                        "assistente",
                        "pessoal", 
                        "produtividade",
                        "tarefas"
                    ],
                    "name": "Assistente Pessoal",
                    "description": "Um assistente pessoal completo que ajuda com informações do tempo, notícias, lembretes e muito mais.",
                    "smallIconUri": "https://s3.amazonaws.com/my-bucket/small-icon.png",
                    "largeIconUri": "https://s3.amazonaws.com/my-bucket/large-icon.png"
                }
            },
            "isAvailableWorldwide": false,
            "testingInstructions": "Para testar, diga 'Alexa, abra assistente pessoal' e siga as instruções de voz.",
            "category": "PRODUCTIVITY",
            "distributionCountries": ["BR"]
        },
        "apis": {
            "custom": {
                "endpoint": {
                    "uri": "arn:aws:lambda:us-east-1:123456789:function:assistente-pessoal"
                },
                "interfaces": [
                    {
                        "type": "AUDIO_PLAYER"
                    },
                    {
                        "type": "ALEXA_PRESENTATION_APL"
                    }
                ]
            }
        },
        "manifestVersion": "1.0",
        "permissions": [
            {
                "name": "alexa::profile:name:read"
            },
            {
                "name": "alexa::profile:email:read"
            }
        ],
        "privacyAndCompliance": {
            "allowsPurchases": false,
            "usesPersonalInfo": true,
            "isChildDirected": false,
            "isExportCompliant": true,
            "containsAds": false,
            "locales": {
                "pt-BR": {
                    "privacyPolicyUrl": "https://example.com/privacy-policy",
                    "termsOfUseUrl": "https://example.com/terms-of-use"
                }
            }
        }
    }
}
```

## 📦 Deploy Automatizado

### 1. CI/CD com GitHub Actions

```yaml
# .github/workflows/deploy-alexa-skill.yml
name: Deploy Alexa Skill

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    
    - name: Setup Node.js
      uses: actions/setup-node@v2
      with:
        node-version: '18'
        
    - name: Install dependencies
      run: |
        cd lambda
        npm install
        
    - name: Run tests
      run: |
        cd lambda
        npm test
        
  deploy:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
    - uses: actions/checkout@v2
    
    - name: Setup Node.js
      uses: actions/setup-node@v2
      with:
        node-version: '18'
        
    - name: Install ASK CLI
      run: npm install -g ask-cli
      
    - name: Configure ASK CLI
      run: |
        ask configure --no-browser << EOF
        ${{ secrets.ASK_ACCESS_TOKEN }}
        ${{ secrets.ASK_REFRESH_TOKEN }}
        ${{ secrets.ASK_VENDOR_ID }}
        EOF
        
    - name: Deploy skill
      run: |
        ask deploy --target skill
        ask deploy --target lambda
```

### 2. Script de Deploy Local

```bash
#!/bin/bash
# deploy.sh

echo "🚀 Iniciando deploy da Alexa Skill..."

# Validar skill
echo "📋 Validando configuração..."
ask api validate-skill --skill-id $SKILL_ID

# Deploy do modelo
echo "🎯 Deploy do interaction model..."
ask api update-interaction-model --skill-id $SKILL_ID --stage development --locale pt-BR --interaction-model file:models/pt-BR.json

# Deploy da função Lambda
echo "⚡ Deploy do código Lambda..."
cd lambda
zip -r ../lambda-function.zip .
aws lambda update-function-code --function-name assistente-pessoal --zip-file fileb://../lambda-function.zip
cd ..

# Aguardar build do modelo
echo "⏳ Aguardando build do modelo..."
ask api get-interaction-model-build-status --skill-id $SKILL_ID --stage development --locale pt-BR

echo "✅ Deploy concluído!"
```

## 🎯 Processo de Certificação

### 1. Requisitos de Certificação

#### Funcionalidade
- Skill deve funcionar conforme descrito
- Todos os intents documentados devem funcionar
- Comandos de help, stop, cancel obrigatórios
- Tratamento adequado de erros

#### Conteúdo
- Conteúdo apropriado para todas as idades
- Não infringir direitos autorais
- Informações precisas e atualizadas
- Política de privacidade quando necessário

#### Experiência do Usuário
- Respostas claras e concisas
- Fluxo de conversação natural
- Ajuda contextual disponível
- Tempo de resposta adequado (< 8s)

### 2. Submissão para Certificação

```bash
# Submeter para certificação
ask api submit-skill-for-certification --skill-id $SKILL_ID
```

### 3. Checklist de Certificação

#### ✅ Testes Obrigatórios
- [ ] Launch request funciona
- [ ] Help intent fornece ajuda útil
- [ ] Stop/Cancel encerram adequadamente
- [ ] Intents customizados funcionam
- [ ] Tratamento de erro gracioso
- [ ] SessionEndedRequest tratado

#### ✅ Metadados
- [ ] Nome da skill único e descritivo
- [ ] Descrição clara e completa
- [ ] Ícones de alta qualidade (108x108, 512x512)
- [ ] Frases de exemplo representativas
- [ ] Keywords relevantes
- [ ] Categoria apropriada

#### ✅ Compliance
- [ ] Política de privacidade (se aplicável)
- [ ] Termos de uso (se aplicável)
- [ ] Não coleta dados de menores
- [ ] Declaração de exportação

## 📊 Monitoramento e Analytics

### 1. CloudWatch Logs

```javascript
// Configurar logs estruturados
const winston = require('winston');

const logger = winston.createLogger({
    level: 'info',
    format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.json()
    ),
    transports: [
        new winston.transports.Console()
    ]
});

// Uso nos handlers
const LaunchRequestHandler = {
    handle(handlerInput) {
        logger.info('LaunchRequest received', {
            userId: handlerInput.requestEnvelope.session.user.userId,
            sessionId: handlerInput.requestEnvelope.session.sessionId
        });
        
        // ... resto do código
    }
};
```

### 2. Métricas Personalizadas

```javascript
const AWS = require('aws-sdk');
const cloudwatch = new AWS.CloudWatch();

async function enviarMetrica(nomeMetrica, valor, unidade = 'Count') {
    const params = {
        Namespace: 'AlexaSkill/AssistentePessoal',
        MetricData: [{
            MetricName: nomeMetrica,
            Value: valor,
            Unit: unidade,
            Timestamp: new Date()
        }]
    };
    
    try {
        await cloudwatch.putMetricData(params).promise();
    } catch (error) {
        console.error('Erro ao enviar métrica:', error);
    }
}

// Uso
await enviarMetrica('IntentInvocations', 1);
await enviarMetrica('ResponseTime', responseTime, 'Milliseconds');
```

### 3. Dashboard de Analytics

```javascript
// Integração com Analytics da Amazon
const analytics = {
    trackIntent: (intentName, userId) => {
        logger.info('Intent tracked', {
            intent: intentName,
            userId: userId,
            timestamp: new Date().toISOString()
        });
    },
    
    trackError: (error, context) => {
        logger.error('Error tracked', {
            error: error.message,
            stack: error.stack,
            context: context,
            timestamp: new Date().toISOString()
        });
    }
};
```

## 🔄 Atualizações e Manutenção

### 1. Versionamento

```bash
# Criar nova versão
ask api create-skill-version --skill-id $SKILL_ID

# Listar versões
ask api list-skill-versions --skill-id $SKILL_ID
```

### 2. Rollback de Emergência

```bash
# Reverter para versão anterior
ask api rollback-skill --skill-id $SKILL_ID --target-version "1.0"
```

### 3. Monitoramento Contínuo

```javascript
// Health check endpoint
app.get('/health', (req, res) => {
    const health = {
        status: 'healthy',
        timestamp: new Date().toISOString(),
        version: process.env.APP_VERSION || '1.0.0',
        dependencies: {
            database: checkDatabaseHealth(),
            externalApi: checkExternalApiHealth()
        }
    };
    
    res.json(health);
});
```

## 📈 Otimização Pós-Lançamento

### 1. Análise de Uso
- Revisar logs de conversação
- Identificar padrões de falha
- Otimizar intents mais utilizados
- Adicionar utterances baseadas no uso real

### 2. A/B Testing
```javascript
// Feature flags para testes
const isFeatureEnabled = (featureName, userId) => {
    const hash = hashFunction(userId + featureName);
    return hash % 100 < 50; // 50% dos usuários
};

if (isFeatureEnabled('newResponse', userId)) {
    speakOutput = 'Nova versão da resposta';
} else {
    speakOutput = 'Versão original da resposta';
}
```

### 3. Feedback dos Usuários
- Implementar sistema de rating
- Coletar feedback via cards
- Monitorar reviews na store
- Responder a comentários

## ⚡ Próximos Passos

1. **Execute** o checklist de pré-deploy
2. **Configure** ambiente de produção
3. **Submeta** para certificação
4. **Monitore** performance pós-lançamento
5. **Explore** [Casos Avançados](05-casos-avancados.md)

---
**Anterior:** [← Exemplos Práticos](03-exemplos-praticos.md) | **Próximo:** [Casos Avançados →](05-casos-avancados.md)
