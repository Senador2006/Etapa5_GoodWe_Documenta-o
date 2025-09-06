# Guia de Desenvolvimento - Alexa Skills GoodWe

## 📋 Visão Geral

Este guia fornece instruções detalhadas para desenvolver a Alexa Skill de integração com as APIs GoodWe, incluindo configuração do ambiente, desenvolvimento da skill e testes.

## 🛠️ Configuração do Ambiente

### Pré-requisitos

#### Software Necessário
- **Node.js**: 18+ (recomendado LTS)
- **Python**: 3.11+ (para API ML)
- **Git**: Controle de versão
- **AWS CLI**: Configuração AWS
- **ASK CLI**: Amazon Skills Kit CLI

#### Contas Necessárias
- **Amazon Developer Account**: Para criar skills
- **AWS Account**: Para Lambda functions
- **GitHub Account**: Para versionamento (opcional)

### Instalação das Ferramentas

#### 1. Node.js e NPM
```bash
# Verificar versão
node --version
npm --version

# Instalar dependências globais
npm install -g ask-cli
npm install -g serverless
```

#### 2. Python e Dependências
```bash
# Verificar versão
python --version
pip --version

# Instalar dependências
pip install fastapi uvicorn scikit-learn pandas numpy
```

#### 3. AWS CLI
```bash
# Instalar AWS CLI
pip install awscli

# Configurar credenciais
aws configure
```

#### 4. ASK CLI
```bash
# Instalar ASK CLI
npm install -g ask-cli

# Configurar perfil
ask configure
```

## 🏗️ Estrutura do Projeto

### Organização de Diretórios

```
GoodWe-Alexa-Skill/
├── lambda/                          # Código Lambda
│   ├── index.js                     # Handler principal
│   ├── package.json                 # Dependências Lambda
│   ├── local-debugger.js            # Debug local
│   └── util.js                      # Utilitários
├── skill-package/                   # Configuração da skill
│   ├── skill.json                   # Manifest da skill
│   └── interactionModels/           # Modelos de interação
│       └── custom/                  # Modelo customizado
│           └── pt-BR.json          # Modelo em português
├── .ask/                           # Configurações ASK CLI
│   └── config                      # Configurações
├── .gitignore                      # Arquivos ignorados
├── README.md                       # Documentação
└── deployment/                     # Scripts de deploy
    ├── deploy.sh                   # Deploy automático
    └── rollback.sh                 # Rollback
```

### Configuração Inicial

#### 1. Inicializar Projeto ASK
```bash
# Criar novo projeto
ask new --skill-name "GoodWe Solar Assistant" --template "hello-world"

# Navegar para o diretório
cd GoodWe-Solar-Assistant
```

#### 2. Configurar package.json
```json
{
  "name": "goodwe-alexa-skill",
  "version": "1.0.0",
  "description": "Alexa Skill para integração com APIs GoodWe",
  "main": "lambda/index.js",
  "scripts": {
    "deploy": "ask deploy",
    "test": "ask dialog --locale pt-BR",
    "simulate": "ask simulate --locale pt-BR",
    "build": "npm run build:lambda",
    "build:lambda": "cd lambda && npm install"
  },
  "dependencies": {
    "ask-sdk-core": "^2.14.0",
    "ask-sdk-model": "^1.66.0",
    "axios": "^1.6.0"
  },
  "devDependencies": {
    "ask-cli": "^2.0.0"
  }
}
```

## 🎯 Desenvolvimento da Skill

### 1. Configuração do Manifest (skill.json)

```json
{
  "manifest": {
    "apis": {
      "custom": {
        "endpoint": {
          "uri": "arn:aws:lambda:us-east-1:123456789012:function:GoodWeSolarAssistant"
        },
        "interfaces": []
      },
      "smartHome": {
        "endpoint": {
          "uri": "arn:aws:lambda:us-east-1:123456789012:function:GoodWeSmartHome"
        }
      }
    },
    "manifestVersion": "1.0",
    "publishingInformation": {
      "locales": {
        "pt-BR": {
          "name": "Assistente Solar GoodWe",
          "summary": "Controle seu sistema solar GoodWe por voz",
          "description": "Gerencie e monitore seu sistema de energia solar GoodWe através de comandos de voz com Alexa. Obtenha status do sistema, dados de geração, nível da bateria e muito mais.",
          "keywords": [
            "solar",
            "energia",
            "goodwe",
            "bateria",
            "sustentável"
          ],
          "smallIconUri": "https://example.com/small-icon.png",
          "largeIconUri": "https://example.com/large-icon.png"
        }
      },
      "isAvailableWorldwide": false,
      "distributionCountries": ["BR"],
      "testingInstructions": "Teste com comandos como 'qual o status do sistema' e 'quanta energia estou gerando'",
      "category": "SMART_HOME",
      "distributionMode": "PUBLISHED"
    },
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

### 2. Modelo de Interação (interactionModels/custom/pt-BR.json)

```json
{
  "interactionModel": {
    "languageModel": {
      "invocationName": "assistente solar",
      "intents": [
        {
          "name": "AMAZON.CancelIntent",
          "samples": []
        },
        {
          "name": "AMAZON.HelpIntent",
          "samples": []
        },
        {
          "name": "AMAZON.StopIntent",
          "samples": []
        },
        {
          "name": "GetSystemStatus",
          "slots": [],
          "samples": [
            "qual o status do sistema",
            "como está o sistema solar",
            "status do sistema",
            "sistema solar",
            "status geral"
          ]
        },
        {
          "name": "GetEnergyGeneration",
          "slots": [
            {
              "name": "TimePeriod",
              "type": "AMAZON.Duration"
            }
          ],
          "samples": [
            "quanta energia estou gerando",
            "geração de energia agora",
            "quanto estou produzindo",
            "energia gerada hoje",
            "produção de energia"
          ]
        },
        {
          "name": "GetBatteryLevel",
          "slots": [],
          "samples": [
            "qual o nível da bateria",
            "como está a bateria",
            "status da bateria",
            "carga da bateria",
            "bateria"
          ]
        },
        {
          "name": "GetPowerConsumption",
          "slots": [],
          "samples": [
            "quanto estou consumindo",
            "consumo de energia",
            "energia consumida",
            "consumo atual"
          ]
        },
        {
          "name": "GetEfficiencyAnalysis",
          "slots": [
            {
              "name": "AnalysisType",
              "type": "AnalysisType"
            }
          ],
          "samples": [
            "análise de eficiência",
            "como está a eficiência",
            "relatório de eficiência",
            "eficiência {AnalysisType}",
            "análise {AnalysisType}"
          ]
        },
        {
          "name": "GetWeatherPrediction",
          "slots": [],
          "samples": [
            "predição do tempo",
            "previsão climática",
            "risco de queda de energia",
            "alerta climático",
            "condições do tempo"
          ]
        },
        {
          "name": "SetBatteryMode",
          "slots": [
            {
              "name": "BatteryMode",
              "type": "BatteryMode"
            }
          ],
          "samples": [
            "configurar bateria para {BatteryMode}",
            "modo da bateria {BatteryMode}",
            "definir bateria {BatteryMode}",
            "bateria {BatteryMode}"
          ]
        },
        {
          "name": "ActivateEmergencyMode",
          "slots": [],
          "samples": [
            "ativar modo de emergência",
            "modo de emergência",
            "emergência",
            "ativar emergência"
          ]
        }
      ],
      "types": [
        {
          "name": "AnalysisType",
          "values": [
            {
              "id": "daily",
              "name": {
                "value": "diária"
              }
            },
            {
              "id": "weekly",
              "name": {
                "value": "semanal"
              }
            },
            {
              "id": "monthly",
              "name": {
                "value": "mensal"
              }
            }
          ]
        },
        {
          "name": "BatteryMode",
          "values": [
            {
              "id": "economy",
              "name": {
                "value": "econômico"
              }
            },
            {
              "id": "balanced",
              "name": {
                "value": "balanceado"
              }
            },
            {
              "id": "performance",
              "name": {
                "value": "performance"
              }
            }
          ]
        }
      ]
    }
  }
}
```

### 3. Código Lambda (lambda/index.js)

```javascript
const Alexa = require('ask-sdk-core');
const axios = require('axios');

// Configurações das APIs
const API_CONFIG = {
  goodwe: {
    baseUrl: process.env.GOODWE_API_URL || 'http://localhost:3000',
    timeout: 5000
  },
  ml: {
    baseUrl: process.env.ML_API_URL || 'http://localhost:8000',
    timeout: 10000
  }
};

// Handlers de Intents
const GetSystemStatusHandler = {
  canHandle(handlerInput) {
    return Alexa.getRequestType(handlerInput.requestEnvelope) === 'IntentRequest'
      && Alexa.getIntentName(handlerInput.requestEnvelope) === 'GetSystemStatus';
  },
  async handle(handlerInput) {
    try {
      const systemData = await getSystemData();
      const speechText = formatSystemStatus(systemData);
      
      return handlerInput.responseBuilder
        .speak(speechText)
        .withSimpleCard('Status do Sistema', speechText)
        .getResponse();
    } catch (error) {
      console.error('Erro ao obter status:', error);
      return handlerInput.responseBuilder
        .speak('Desculpe, não consegui obter o status do sistema. Tente novamente.')
        .getResponse();
    }
  }
};

const GetEnergyGenerationHandler = {
  canHandle(handlerInput) {
    return Alexa.getRequestType(handlerInput.requestEnvelope) === 'IntentRequest'
      && Alexa.getIntentName(handlerInput.requestEnvelope) === 'GetEnergyGeneration';
  },
  async handle(handlerInput) {
    try {
      const timePeriod = Alexa.getSlotValue(handlerInput.requestEnvelope, 'TimePeriod');
      const energyData = await getEnergyData(timePeriod);
      const speechText = formatEnergyGeneration(energyData, timePeriod);
      
      return handlerInput.responseBuilder
        .speak(speechText)
        .withSimpleCard('Geração de Energia', speechText)
        .getResponse();
    } catch (error) {
      console.error('Erro ao obter geração:', error);
      return handlerInput.responseBuilder
        .speak('Desculpe, não consegui obter os dados de geração. Tente novamente.')
        .getResponse();
    }
  }
};

const GetBatteryLevelHandler = {
  canHandle(handlerInput) {
    return Alexa.getRequestType(handlerInput.requestEnvelope) === 'IntentRequest'
      && Alexa.getIntentName(handlerInput.requestEnvelope) === 'GetBatteryLevel';
  },
  async handle(handlerInput) {
    try {
      const batteryData = await getBatteryData();
      const speechText = formatBatteryLevel(batteryData);
      
      return handlerInput.responseBuilder
        .speak(speechText)
        .withSimpleCard('Nível da Bateria', speechText)
        .getResponse();
    } catch (error) {
      console.error('Erro ao obter bateria:', error);
      return handlerInput.responseBuilder
        .speak('Desculpe, não consegui obter o status da bateria. Tente novamente.')
        .getResponse();
    }
  }
};

const GetWeatherPredictionHandler = {
  canHandle(handlerInput) {
    return Alexa.getRequestType(handlerInput.requestEnvelope) === 'IntentRequest'
      && Alexa.getIntentName(handlerInput.requestEnvelope) === 'GetWeatherPrediction';
  },
  async handle(handlerInput) {
    try {
      const weatherData = await getCurrentWeatherData();
      const prediction = await getWeatherPrediction(weatherData);
      const speechText = formatWeatherPrediction(prediction);
      
      return handlerInput.responseBuilder
        .speak(speechText)
        .withSimpleCard('Predição Climática', speechText)
        .getResponse();
    } catch (error) {
      console.error('Erro ao obter predição:', error);
      return handlerInput.responseBuilder
        .speak('Desculpe, não consegui obter a predição climática. Tente novamente.')
        .getResponse();
    }
  }
};

// Handlers padrão do Alexa
const LaunchRequestHandler = {
  canHandle(handlerInput) {
    return Alexa.getRequestType(handlerInput.requestEnvelope) === 'LaunchRequest';
  },
  handle(handlerInput) {
    const speechText = 'Bem-vindo ao Assistente Solar GoodWe! Como posso ajudar com seu sistema de energia solar?';
    
    return handlerInput.responseBuilder
      .speak(speechText)
      .reprompt('Você pode perguntar sobre o status do sistema, geração de energia ou nível da bateria.')
      .getResponse();
  }
};

const HelpIntentHandler = {
  canHandle(handlerInput) {
    return Alexa.getRequestType(handlerInput.requestEnvelope) === 'IntentRequest'
      && Alexa.getIntentName(handlerInput.requestEnvelope) === 'AMAZON.HelpIntent';
  },
  handle(handlerInput) {
    const speechText = 'Posso ajudar você a monitorar seu sistema solar GoodWe. Você pode perguntar sobre o status do sistema, geração de energia, nível da bateria, consumo de energia, análise de eficiência ou predição climática. O que gostaria de saber?';
    
    return handlerInput.responseBuilder
      .speak(speechText)
      .reprompt('O que gostaria de saber sobre seu sistema solar?')
      .getResponse();
  }
};

const CancelAndStopIntentHandler = {
  canHandle(handlerInput) {
    return Alexa.getRequestType(handlerInput.requestEnvelope) === 'IntentRequest'
      && (Alexa.getIntentName(handlerInput.requestEnvelope) === 'AMAZON.CancelIntent'
        || Alexa.getIntentName(handlerInput.requestEnvelope) === 'AMAZON.StopIntent');
  },
  handle(handlerInput) {
    const speechText = 'Até logo! Seu sistema solar está monitorado.';
    
    return handlerInput.responseBuilder
      .speak(speechText)
      .getResponse();
  }
};

const SessionEndedRequestHandler = {
  canHandle(handlerInput) {
    return Alexa.getRequestType(handlerInput.requestEnvelope) === 'SessionEndedRequest';
  },
  handle(handlerInput) {
    console.log(`Sessão encerrada: ${JSON.stringify(handlerInput.requestEnvelope)}`);
    return handlerInput.responseBuilder.getResponse();
  }
};

const ErrorHandler = {
  canHandle() {
    return true;
  },
  handle(handlerInput, error) {
    console.log(`Erro: ${JSON.stringify(error)}`);
    
    return handlerInput.responseBuilder
      .speak('Desculpe, ocorreu um erro. Tente novamente.')
      .reprompt('Como posso ajudar?')
      .getResponse();
  }
};

// Funções auxiliares para integração com APIs
async function getSystemData() {
  try {
    const response = await axios.get(`${API_CONFIG.goodwe.baseUrl}/data/paginated?limit=1`, {
      timeout: API_CONFIG.goodwe.timeout
    });
    return response.data.data[0];
  } catch (error) {
    console.error('Erro na API GoodWe:', error);
    throw error;
  }
}

async function getEnergyData(timePeriod = 'now') {
  try {
    const endpoint = timePeriod === 'now' 
      ? `${API_CONFIG.goodwe.baseUrl}/data/paginated?limit=1`
      : `${API_CONFIG.goodwe.baseUrl}/analytics/hourly`;
    
    const response = await axios.get(endpoint, {
      timeout: API_CONFIG.goodwe.timeout
    });
    return response.data;
  } catch (error) {
    console.error('Erro na API GoodWe:', error);
    throw error;
  }
}

async function getBatteryData() {
  try {
    const response = await axios.get(`${API_CONFIG.goodwe.baseUrl}/data/paginated?limit=1`, {
      timeout: API_CONFIG.goodwe.timeout
    });
    return response.data.data[0];
  } catch (error) {
    console.error('Erro na API GoodWe:', error);
    throw error;
  }
}

async function getCurrentWeatherData() {
  // Simulação de dados climáticos - em produção, integrar com API de clima
  return {
    temperatura_celsius: 25.0,
    umidade_pct: 65.0,
    precipitacao_mm_h: 10.0,
    vento_kmh: 30.0,
    pressao_hpa: 1013.0
  };
}

async function getWeatherPrediction(weatherData) {
  try {
    const response = await axios.post(`${API_CONFIG.ml.baseUrl}/predict`, weatherData, {
      timeout: API_CONFIG.ml.timeout,
      headers: {
        'Content-Type': 'application/json'
      }
    });
    return response.data;
  } catch (error) {
    console.error('Erro na API ML:', error);
    throw error;
  }
}

// Funções de formatação de resposta
function formatSystemStatus(data) {
  const fvPower = data.fv_power || 0;
  const soc = data.soc_percentage || 0;
  const batteryPower = data.battery_power || 0;
  const gridPower = data.grid_power || 0;
  const loadPower = data.load_power || 0;
  
  return `Seu sistema solar está funcionando. A geração atual é de ${fvPower} watts, a bateria está com ${soc}% de carga, e o consumo atual é de ${loadPower} watts.`;
}

function formatEnergyGeneration(data, timePeriod) {
  if (timePeriod === 'now' || !timePeriod) {
    const fvPower = data.data[0].fv_power || 0;
    return `A geração atual de energia é de ${fvPower} watts.`;
  } else {
    const totalGeneration = data.total_generation || 0;
    return `A geração total no período foi de ${totalGeneration} watts.`;
  }
}

function formatBatteryLevel(data) {
  const soc = data.soc_percentage || 0;
  const batteryPower = data.battery_power || 0;
  const status = soc > 80 ? 'excelente' : soc > 50 ? 'boa' : soc > 20 ? 'baixa' : 'crítica';
  
  return `A bateria está com ${soc}% de carga, status ${status}. ${batteryPower > 0 ? 'Está carregando.' : 'Está descarregando.'}`;
}

function formatWeatherPrediction(prediction) {
  const risk = prediction.nivel_risco;
  const probability = prediction.probabilidade_pct;
  
  let riskMessage = '';
  switch (risk) {
    case 'Baixo':
      riskMessage = 'O risco de queda de energia é baixo.';
      break;
    case 'Médio':
      riskMessage = 'Há um risco médio de queda de energia.';
      break;
    case 'Alto':
      riskMessage = 'Atenção! Há um alto risco de queda de energia.';
      break;
    case 'Crítico':
      riskMessage = 'Alerta! Risco crítico de queda de energia!';
      break;
  }
  
  return `${riskMessage} A probabilidade é de ${probability}.`;
}

// Configuração do SDK
const skillBuilder = Alexa.SkillBuilders.custom();

exports.handler = skillBuilder
  .addRequestHandlers(
    LaunchRequestHandler,
    GetSystemStatusHandler,
    GetEnergyGenerationHandler,
    GetBatteryLevelHandler,
    GetWeatherPredictionHandler,
    HelpIntentHandler,
    CancelAndStopIntentHandler,
    SessionEndedRequestHandler
  )
  .addErrorHandlers(ErrorHandler)
  .lambda();
```

### 4. Package.json da Lambda (lambda/package.json)

```json
{
  "name": "goodwe-alexa-lambda",
  "version": "1.0.0",
  "description": "Lambda function para Alexa Skill GoodWe",
  "main": "index.js",
  "dependencies": {
    "ask-sdk-core": "^2.14.0",
    "ask-sdk-model": "^1.66.0",
    "axios": "^1.6.0"
  },
  "scripts": {
    "test": "node test.js"
  }
}
```

## 🧪 Testes e Validação

### 1. Testes Locais

#### Teste de Simulação
```bash
# Simular interação
ask simulate --text "qual o status do sistema" --locale pt-BR

# Teste de diálogo
ask dialog --locale pt-BR
```

#### Teste de Código
```javascript
// test.js
const { handler } = require('./index');

async function testHandler() {
  const mockRequest = {
    request: {
      type: 'IntentRequest',
      intent: {
        name: 'GetSystemStatus'
      }
    },
    response: {
      speak: (text) => console.log('Speech:', text),
      withSimpleCard: (title, content) => console.log('Card:', title, content),
      getResponse: () => ({})
    }
  };
  
  try {
    await handler(mockRequest);
    console.log('✅ Teste passou');
  } catch (error) {
    console.error('❌ Teste falhou:', error);
  }
}

testHandler();
```

### 2. Testes de Integração

#### Teste de APIs
```javascript
// test-api-integration.js
const axios = require('axios');

async function testAPIs() {
  console.log('🧪 Testando APIs...');
  
  // Teste API GoodWe
  try {
    const goodweResponse = await axios.get('http://localhost:3000/health');
    console.log('✅ API GoodWe:', goodweResponse.data.status);
  } catch (error) {
    console.error('❌ API GoodWe:', error.message);
  }
  
  // Teste API ML
  try {
    const mlResponse = await axios.get('http://localhost:8000/health');
    console.log('✅ API ML:', mlResponse.data.status);
  } catch (error) {
    console.error('❌ API ML:', error.message);
  }
}

testAPIs();
```

### 3. Testes de Usuário

#### Cenários de Teste
```javascript
const testScenarios = [
  {
    name: 'Status do Sistema',
    input: 'qual o status do sistema',
    expectedIntent: 'GetSystemStatus'
  },
  {
    name: 'Geração de Energia',
    input: 'quanta energia estou gerando',
    expectedIntent: 'GetEnergyGeneration'
  },
  {
    name: 'Nível da Bateria',
    input: 'qual o nível da bateria',
    expectedIntent: 'GetBatteryLevel'
  },
  {
    name: 'Predição Climática',
    input: 'risco de queda de energia',
    expectedIntent: 'GetWeatherPrediction'
  }
];
```

## 🚀 Deploy e Configuração

### 1. Deploy da Lambda

```bash
# Deploy da skill
ask deploy

# Deploy apenas da Lambda
ask deploy --target lambda
```

### 2. Configuração de Variáveis de Ambiente

```bash
# Configurar variáveis na Lambda
aws lambda update-function-configuration \
  --function-name GoodWeSolarAssistant \
  --environment Variables='{
    "GOODWE_API_URL":"https://api.goodwe.com",
    "ML_API_URL":"https://ml-api.goodwe.com"
  }'
```

### 3. Configuração de Permissões

```json
{
  "Version": "2012-10-17",
  "Statement": [
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

## 📊 Monitoramento e Logs

### 1. CloudWatch Logs

```javascript
// Configuração de logging
const logger = {
  info: (message, data = {}) => {
    console.log(JSON.stringify({
      timestamp: new Date().toISOString(),
      level: 'INFO',
      message,
      data
    }));
  },
  
  error: (message, error = {}) => {
    console.error(JSON.stringify({
      timestamp: new Date().toISOString(),
      level: 'ERROR',
      message,
      error: {
        name: error.name,
        message: error.message
      }
    }));
  }
};
```

### 2. Métricas Personalizadas

```javascript
// Envio de métricas para CloudWatch
const AWS = require('aws-sdk');
const cloudwatch = new AWS.CloudWatch();

async function sendMetric(metricName, value, unit = 'Count') {
  try {
    await cloudwatch.putMetricData({
      Namespace: 'GoodWe/AlexaSkill',
      MetricData: [{
        MetricName: metricName,
        Value: value,
        Unit: unit,
        Timestamp: new Date()
      }]
    }).promise();
  } catch (error) {
    console.error('Erro ao enviar métrica:', error);
  }
}
```

## 🔧 Debugging e Troubleshooting

### 1. Logs de Debug

```javascript
// Ativar logs detalhados
const DEBUG = process.env.DEBUG === 'true';

function debugLog(message, data = {}) {
  if (DEBUG) {
    console.log(`[DEBUG] ${message}`, JSON.stringify(data, null, 2));
  }
}
```

### 2. Tratamento de Erros

```javascript
// Tratamento robusto de erros
function handleAPIError(error, context) {
  logger.error(`Erro na API ${context}`, error);
  
  if (error.code === 'ECONNREFUSED') {
    return 'Serviço temporariamente indisponível. Tente novamente em alguns minutos.';
  } else if (error.code === 'ETIMEDOUT') {
    return 'A requisição demorou muito para responder. Tente novamente.';
  } else {
    return 'Ocorreu um erro inesperado. Tente novamente.';
  }
}
```

---

**Próximo**: [Exemplos Práticos](./03-exemplos-praticos.md)
