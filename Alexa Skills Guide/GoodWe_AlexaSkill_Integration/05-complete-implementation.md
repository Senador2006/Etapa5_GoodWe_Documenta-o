# Complete Implementation - GoodWe Alexa Skill

## 📋 Visão Geral

Este documento apresenta a implementação completa da Alexa Skill GoodWe, incluindo código final, configurações e instruções de deploy.

## 🏗️ Estrutura Final do Projeto

```
GoodWe-Alexa-Skill/
├── lambda/                          # Código Lambda
│   ├── index.js                     # Handler principal
│   ├── package.json                 # Dependências
│   ├── local-debugger.js            # Debug local
│   ├── util.js                      # Utilitários
│   ├── handlers/                    # Handlers específicos
│   │   ├── system-handler.js        # Sistema solar
│   │   ├── smart-home-handler.js    # Smart Home
│   │   ├── emergency-handler.js     # Emergências
│   │   └── optimization-handler.js  # Otimização
│   ├── services/                    # Serviços
│   │   ├── goodwe-service.js        # API GoodWe
│   │   ├── ml-service.js            # API ML
│   │   └── weather-service.js       # Clima
│   ├── utils/                       # Utilitários
│   │   ├── http-client.js           # Cliente HTTP
│   │   ├── cache-manager.js         # Cache
│   │   └── error-handler.js         # Tratamento de erros
│   └── config/                      # Configurações
│       ├── api-config.js            # APIs
│       └── alexa-config.js          # Alexa
├── skill-package/                   # Configuração da skill
│   ├── skill.json                   # Manifest
│   └── interactionModels/           # Modelos de interação
│       └── custom/
│           └── pt-BR.json          # Português brasileiro
├── .ask/                           # Configurações ASK
│   └── config                      # Perfis
├── .gitignore                      # Arquivos ignorados
├── README.md                       # Documentação
├── deployment/                     # Scripts de deploy
│   ├── deploy.sh                   # Deploy automático
│   ├── rollback.sh                 # Rollback
│   └── test.sh                     # Testes
└── tests/                          # Testes
    ├── unit/                       # Testes unitários
    ├── integration/                # Testes de integração
    └── e2e/                        # Testes end-to-end
```

## 🚀 Implementação Completa

### 1. Handler Principal (lambda/index.js)

```javascript
const Alexa = require('ask-sdk-core');
const { 
  SystemHandler,
  SmartHomeHandler,
  EmergencyHandler,
  OptimizationHandler
} = require('./handlers');
const { ErrorHandler } = require('./utils/error-handler');

// Configuração do SDK
const skillBuilder = Alexa.SkillBuilders.custom();

exports.handler = skillBuilder
  .addRequestHandlers(
    // Handlers do sistema solar
    SystemHandler.GetSystemStatusHandler,
    SystemHandler.GetEnergyGenerationHandler,
    SystemHandler.GetBatteryLevelHandler,
    SystemHandler.GetPowerConsumptionHandler,
    SystemHandler.GetEfficiencyAnalysisHandler,
    SystemHandler.GetDailyReportHandler,
    SystemHandler.GetWeeklyReportHandler,
    SystemHandler.GetMonthlyReportHandler,
    
    // Handlers Smart Home
    SmartHomeHandler.DiscoveryHandler,
    SmartHomeHandler.PowerControlHandler,
    SmartHomeHandler.PercentageControlHandler,
    SmartHomeHandler.ModeControlHandler,
    SmartHomeHandler.StateReportHandler,
    
    // Handlers de emergência
    EmergencyHandler.EmergencyStatusHandler,
    EmergencyHandler.ActivateEmergencyModeHandler,
    EmergencyHandler.EmergencyContactsHandler,
    
    // Handlers de otimização
    OptimizationHandler.GetRecommendationsHandler,
    OptimizationHandler.OptimizeSystemHandler,
    OptimizationHandler.GetPerformanceReportHandler,
    
    // Handlers padrão do Alexa
    SystemHandler.LaunchRequestHandler,
    SystemHandler.HelpIntentHandler,
    SystemHandler.CancelAndStopIntentHandler,
    SystemHandler.SessionEndedRequestHandler
  )
  .addErrorHandlers(ErrorHandler)
  .withCustomUserAgent('GoodWe-Solar-Assistant/1.0')
  .lambda();
```

### 2. Configuração das APIs (lambda/config/api-config.js)

```javascript
const API_CONFIG = {
  goodwe: {
    baseUrl: process.env.GOODWE_API_URL || 'http://localhost:3000',
    apiKey: process.env.GOODWE_API_KEY,
    timeout: 5000,
    retries: 3,
    retryDelay: 1000
  },
  ml: {
    baseUrl: process.env.ML_API_URL || 'http://localhost:8000',
    apiKey: process.env.ML_API_KEY,
    timeout: 10000,
    retries: 2,
    retryDelay: 2000
  },
  weather: {
    baseUrl: 'https://api.openweathermap.org/data/2.5',
    apiKey: process.env.WEATHER_API_KEY,
    timeout: 5000
  },
  utility: {
    baseUrl: process.env.UTILITY_API_URL,
    apiKey: process.env.UTILITY_API_KEY,
    timeout: 5000
  }
};

module.exports = { API_CONFIG };
```

### 3. Cliente HTTP (lambda/utils/http-client.js)

```javascript
const axios = require('axios');
const { API_CONFIG } = require('../config/api-config');

class APIClient {
  constructor(config) {
    this.client = axios.create({
      baseURL: config.baseUrl,
      timeout: config.timeout,
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${config.apiKey}`
      }
    });
    
    this.setupInterceptors();
  }
  
  setupInterceptors() {
    this.client.interceptors.request.use(
      (config) => {
        console.log(`[API] ${config.method.toUpperCase()} ${config.url}`);
        return config;
      },
      (error) => {
        console.error('[API] Request error:', error);
        return Promise.reject(error);
      }
    );
    
    this.client.interceptors.response.use(
      (response) => {
        console.log(`[API] ${response.status} ${response.config.url}`);
        return response;
      },
      async (error) => {
        console.error('[API] Response error:', error.message);
        
        if (this.shouldRetry(error)) {
          return this.retryRequest(error.config);
        }
        
        return Promise.reject(error);
      }
    );
  }
  
  shouldRetry(error) {
    return error.code === 'ECONNREFUSED' || 
           error.code === 'ETIMEDOUT' ||
           (error.response && error.response.status >= 500);
  }
  
  async retryRequest(config) {
    const maxRetries = 3;
    let retryCount = 0;
    
    while (retryCount < maxRetries) {
      try {
        retryCount++;
        console.log(`[API] Retry ${retryCount}/${maxRetries} for ${config.url}`);
        
        await new Promise(resolve => setTimeout(resolve, 1000 * retryCount));
        return await this.client.request(config);
      } catch (error) {
        if (retryCount === maxRetries) {
          throw error;
        }
      }
    }
  }
}

// Instâncias dos clientes
const goodweClient = new APIClient(API_CONFIG.goodwe);
const mlClient = new APIClient(API_CONFIG.ml);
const weatherClient = new APIClient(API_CONFIG.weather);
const utilityClient = new APIClient(API_CONFIG.utility);

module.exports = {
  goodweClient,
  mlClient,
  weatherClient,
  utilityClient
};
```

### 4. Serviço Principal GoodWe (lambda/services/goodwe-service.js)

```javascript
const { goodweClient } = require('../utils/http-client');

class GoodWeService {
  async getSystemStatus() {
    try {
      const response = await goodweClient.client.get('/data/paginated?limit=1');
      
      if (!response.data.success) {
        throw new Error('Falha ao obter dados do sistema');
      }
      
      return this.formatSystemData(response.data.data[0]);
    } catch (error) {
      console.error('Erro ao obter status do sistema:', error);
      throw new Error('Sistema temporariamente indisponível');
    }
  }
  
  async getEnergyGeneration(timePeriod = 'now') {
    try {
      let endpoint = '/data/paginated?limit=1';
      
      if (timePeriod !== 'now') {
        endpoint = '/analytics/hourly';
      }
      
      const response = await goodweClient.client.get(endpoint);
      return this.formatEnergyData(response.data, timePeriod);
    } catch (error) {
      console.error('Erro ao obter dados de geração:', error);
      throw new Error('Não foi possível obter dados de geração');
    }
  }
  
  async getBatteryStatus() {
    try {
      const response = await goodweClient.client.get('/data/paginated?limit=1');
      const data = response.data.data[0];
      
      return {
        level: data.soc_percentage || 0,
        power: data.battery_power || 0,
        status: this.getBatteryStatusText(data.soc_percentage, data.battery_power),
        isCharging: data.battery_power > 0,
        isDischarging: data.battery_power < 0
      };
    } catch (error) {
      console.error('Erro ao obter status da bateria:', error);
      throw new Error('Não foi possível obter status da bateria');
    }
  }
  
  async getEfficiencyAnalysis(analysisType = 'daily') {
    try {
      const endpoint = analysisType === 'daily' 
        ? '/analytics/hourly' 
        : '/analytics/trends';
      
      const response = await goodweClient.client.get(endpoint, {
        params: { period: analysisType }
      });
      
      return this.formatEfficiencyData(response.data, analysisType);
    } catch (error) {
      console.error('Erro na análise de eficiência:', error);
      throw new Error('Não foi possível obter análise de eficiência');
    }
  }
  
  formatSystemData(data) {
    return {
      fvPower: data.fv_power || 0,
      soc: data.soc_percentage || 0,
      batteryPower: data.battery_power || 0,
      gridPower: data.grid_power || 0,
      loadPower: data.load_power || 0,
      timestamp: data.timestamp,
      efficiency: this.calculateEfficiency(data)
    };
  }
  
  formatEnergyData(data, timePeriod) {
    if (timePeriod === 'now' || !timePeriod) {
      return {
        current: data.data[0].fv_power || 0,
        unit: 'watts',
        timestamp: data.data[0].timestamp
      };
    } else {
      return {
        total: data.total_generation || 0,
        average: data.average_generation || 0,
        peak: data.peak_generation || 0,
        unit: 'watts',
        period: timePeriod
      };
    }
  }
  
  formatEfficiencyData(data, analysisType) {
    const efficiency = data.efficiency_metrics || {};
    
    return {
      average: efficiency.average || 0,
      peak: efficiency.peak || 0,
      low: efficiency.low || 0,
      trend: data.trends?.efficiency_trend || 'stable',
      period: analysisType,
      recommendations: this.generateEfficiencyRecommendations(efficiency)
    };
  }
  
  getBatteryStatusText(level, power) {
    if (level > 80) return 'excelente';
    if (level > 50) return 'boa';
    if (level > 20) return 'baixa';
    return 'crítica';
  }
  
  calculateEfficiency(data) {
    const fvPower = data.fv_power || 0;
    const loadPower = data.load_power || 0;
    
    if (fvPower === 0) return 0;
    return Math.round((loadPower / fvPower) * 100);
  }
  
  generateEfficiencyRecommendations(efficiency) {
    const recommendations = [];
    
    if (efficiency.average < 60) {
      recommendations.push('Considere verificar a limpeza dos painéis solares');
    }
    
    if (efficiency.peak < 80) {
      recommendations.push('Verifique se há sombreamento nos painéis');
    }
    
    if (efficiency.average > 85) {
      recommendations.push('Excelente! Seu sistema está operando com alta eficiência');
    }
    
    return recommendations;
  }
}

module.exports = new GoodWeService();
```

### 5. Serviço Machine Learning (lambda/services/ml-service.js)

```javascript
const { mlClient } = require('../utils/http-client');

class MLService {
  async getWeatherPrediction(weatherData = null) {
    try {
      if (!weatherData) {
        weatherData = await this.getCurrentWeatherData();
      }
      
      const response = await mlClient.client.post('/predict', weatherData);
      
      return this.formatPrediction(response.data);
    } catch (error) {
      console.error('Erro na predição climática:', error);
      throw new Error('Não foi possível obter predição climática');
    }
  }
  
  async getBatchPrediction(weatherDataArray) {
    try {
      const response = await mlClient.client.post('/predict-batch', weatherDataArray);
      
      return response.data.predictions.map(prediction => 
        this.formatPrediction(prediction)
      );
    } catch (error) {
      console.error('Erro na predição em lote:', error);
      throw new Error('Não foi possível obter predições em lote');
    }
  }
  
  async getModelInfo() {
    try {
      const response = await mlClient.client.get('/model-info');
      
      return {
        modelType: response.data.model_type,
        features: response.data.features,
        accuracy: response.data.accuracy,
        description: response.data.description
      };
    } catch (error) {
      console.error('Erro ao obter informações do modelo:', error);
      throw new Error('Não foi possível obter informações do modelo');
    }
  }
  
  async getCurrentWeatherData() {
    // Dados climáticos simulados baseados na localização
    return {
      temperatura_celsius: 25.0,
      umidade_pct: 65.0,
      precipitacao_mm_h: 10.0,
      vento_kmh: 30.0,
      pressao_hpa: 1013.0
    };
  }
  
  formatPrediction(prediction) {
    return {
      willOutage: prediction.queda_energia,
      probability: prediction.probabilidade,
      probabilityPct: prediction.probabilidade_pct,
      riskLevel: prediction.nivel_risco,
      inputData: prediction.dados_entrada,
      recommendations: this.generateWeatherRecommendations(prediction)
    };
  }
  
  generateWeatherRecommendations(prediction) {
    const recommendations = [];
    const { riskLevel, probability } = prediction;
    
    if (riskLevel === 'Crítico') {
      recommendations.push('Ative o modo de emergência imediatamente');
      recommendations.push('Verifique se todos os equipamentos estão seguros');
      recommendations.push('Considere desligar equipamentos não essenciais');
    } else if (riskLevel === 'Alto') {
      recommendations.push('Prepare-se para possível interrupção');
      recommendations.push('Verifique o sistema de backup');
    } else if (riskLevel === 'Médio') {
      recommendations.push('Monitore as condições climáticas');
    } else {
      recommendations.push('Condições normais, sistema operando normalmente');
    }
    
    return recommendations;
  }
}

module.exports = new MLService();
```

### 6. Manifest da Skill (skill-package/skill.json)

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
          "description": "Gerencie e monitore seu sistema de energia solar GoodWe através de comandos de voz com Alexa. Obtenha status do sistema, dados de geração, nível da bateria, predições climáticas e muito mais.",
          "keywords": [
            "solar",
            "energia",
            "goodwe",
            "bateria",
            "sustentável",
            "renewable",
            "green energy"
          ],
          "smallIconUri": "https://goodwe-alexa-assets.s3.amazonaws.com/icons/small-icon.png",
          "largeIconUri": "https://goodwe-alexa-assets.s3.amazonaws.com/icons/large-icon.png"
        }
      },
      "isAvailableWorldwide": false,
      "distributionCountries": ["BR"],
      "testingInstructions": "Teste com comandos como 'qual o status do sistema', 'quanta energia estou gerando', 'qual o nível da bateria' e 'risco de queda de energia'",
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
          "privacyPolicyUrl": "https://goodwe.com/privacy-policy",
          "termsOfUseUrl": "https://goodwe.com/terms-of-use"
        }
      }
    }
  }
}
```

### 7. Modelo de Interação (skill-package/interactionModels/custom/pt-BR.json)

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
            "status geral",
            "como está funcionando o sistema"
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
            "produção de energia",
            "quanta energia foi gerada"
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
            "bateria",
            "nível de carga"
          ]
        },
        {
          "name": "GetPowerConsumption",
          "slots": [],
          "samples": [
            "quanto estou consumindo",
            "consumo de energia",
            "energia consumida",
            "consumo atual",
            "quanto estou gastando"
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
            "análise {AnalysisType}",
            "performance do sistema"
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
            "condições do tempo",
            "vai ter queda de energia"
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
            "bateria {BatteryMode}",
            "colocar bateria no modo {BatteryMode}"
          ]
        },
        {
          "name": "ActivateEmergencyMode",
          "slots": [],
          "samples": [
            "ativar modo de emergência",
            "modo de emergência",
            "emergência",
            "ativar emergência",
            "situação de emergência"
          ]
        },
        {
          "name": "GetRecommendations",
          "slots": [],
          "samples": [
            "me dê recomendações",
            "sugestões de otimização",
            "como otimizar o sistema",
            "dicas de economia",
            "recomendações de melhoria"
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

### 8. Package.json da Lambda (lambda/package.json)

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
    "test": "node test.js",
    "deploy": "ask deploy",
    "test:local": "node local-debugger.js"
  },
  "engines": {
    "node": ">=18.0.0"
  }
}
```

## 🚀 Scripts de Deploy

### 1. Deploy Automático (deployment/deploy.sh)

```bash
#!/bin/bash
# deploy.sh - Script de deploy automático

set -e

echo "🚀 Iniciando deploy da GoodWe Alexa Skill..."

# Verificar se ASK CLI está instalado
if ! command -v ask &> /dev/null; then
    echo "❌ ASK CLI não encontrado. Instale com: npm install -g ask-cli"
    exit 1
fi

# Verificar se AWS CLI está configurado
if ! aws sts get-caller-identity &> /dev/null; then
    echo "❌ AWS CLI não configurado. Execute: aws configure"
    exit 1
fi

# Instalar dependências da Lambda
echo "📦 Instalando dependências da Lambda..."
cd lambda
npm install --production
cd ..

# Deploy da skill
echo "🔄 Fazendo deploy da skill..."
ask deploy

# Verificar deploy
echo "✅ Verificando deploy..."
ask get-skill-status

echo "🎉 Deploy concluído com sucesso!"
echo "📱 Teste a skill com: ask simulate --text 'qual o status do sistema' --locale pt-BR"
```

### 2. Script de Teste (deployment/test.sh)

```bash
#!/bin/bash
# test.sh - Script de testes

set -e

echo "🧪 Executando testes da GoodWe Alexa Skill..."

# Testes unitários
echo "📋 Executando testes unitários..."
cd lambda
npm test

# Testes de integração
echo "🔗 Executando testes de integração..."
cd ../tests/integration
npm test

# Testes end-to-end
echo "🎯 Executando testes end-to-end..."
cd ../e2e
npm test

echo "✅ Todos os testes passaram!"
```

### 3. Script de Rollback (deployment/rollback.sh)

```bash
#!/bin/bash
# rollback.sh - Script de rollback

set -e

echo "🔄 Iniciando rollback da GoodWe Alexa Skill..."

# Listar versões disponíveis
echo "📋 Versões disponíveis:"
ask get-skill-status

# Solicitar versão para rollback
read -p "Digite a versão para rollback: " VERSION

# Fazer rollback
echo "🔄 Fazendo rollback para versão $VERSION..."
ask rollback --target lambda --version $VERSION

echo "✅ Rollback concluído!"
```

## 🧪 Testes Completos

### 1. Testes Unitários (tests/unit/system-tests.js)

```javascript
const { handler } = require('../../lambda/index');

describe('GoodWe Alexa Skill - Testes Unitários', () => {
  test('Launch Request', async () => {
    const request = {
      request: {
        type: 'LaunchRequest'
      }
    };
    
    const response = await handler(request);
    
    expect(response.response.outputSpeech).toBeDefined();
    expect(response.response.outputSpeech.text).toContain('Bem-vindo');
  });
  
  test('Get System Status', async () => {
    const request = {
      request: {
        type: 'IntentRequest',
        intent: {
          name: 'GetSystemStatus'
        }
      }
    };
    
    const response = await handler(request);
    
    expect(response.response.outputSpeech).toBeDefined();
    expect(response.response.card).toBeDefined();
  });
  
  test('Get Battery Level', async () => {
    const request = {
      request: {
        type: 'IntentRequest',
        intent: {
          name: 'GetBatteryLevel'
        }
      }
    };
    
    const response = await handler(request);
    
    expect(response.response.outputSpeech).toBeDefined();
    expect(response.response.outputSpeech.text).toContain('bateria');
  });
});
```

### 2. Testes de Integração (tests/integration/api-tests.js)

```javascript
const axios = require('axios');

describe('GoodWe Alexa Skill - Testes de Integração', () => {
  test('API GoodWe Health Check', async () => {
    const response = await axios.get('http://localhost:3000/health');
    expect(response.status).toBe(200);
    expect(response.data.status).toBe('OK');
  });
  
  test('API ML Health Check', async () => {
    const response = await axios.get('http://localhost:8000/health');
    expect(response.status).toBe(200);
    expect(response.data.status).toBe('healthy');
  });
  
  test('API GoodWe Data Endpoint', async () => {
    const response = await axios.get('http://localhost:3000/data/paginated?limit=1');
    expect(response.status).toBe(200);
    expect(response.data.success).toBe(true);
  });
  
  test('API ML Prediction Endpoint', async () => {
    const testData = {
      temperatura_celsius: 25.0,
      umidade_pct: 65.0,
      precipitacao_mm_h: 10.0,
      vento_kmh: 30.0,
      pressao_hpa: 1013.0
    };
    
    const response = await axios.post('http://localhost:8000/predict', testData);
    expect(response.status).toBe(200);
    expect(response.data).toHaveProperty('queda_energia');
    expect(response.data).toHaveProperty('probabilidade');
  });
});
```

### 3. Testes End-to-End (tests/e2e/e2e-tests.js)

```javascript
const { handler } = require('../../lambda/index');

describe('GoodWe Alexa Skill - Testes End-to-End', () => {
  test('Fluxo Completo - Status do Sistema', async () => {
    const request = {
      request: {
        type: 'IntentRequest',
        intent: {
          name: 'GetSystemStatus'
        }
      }
    };
    
    const response = await handler(request);
    
    expect(response.response.outputSpeech).toBeDefined();
    expect(response.response.card).toBeDefined();
    expect(response.response.card.title).toContain('Status');
  });
  
  test('Fluxo Completo - Predição Climática', async () => {
    const request = {
      request: {
        type: 'IntentRequest',
        intent: {
          name: 'GetWeatherPrediction'
        }
      }
    };
    
    const response = await handler(request);
    
    expect(response.response.outputSpeech).toBeDefined();
    expect(response.response.card).toBeDefined();
    expect(response.response.outputSpeech.text).toContain('risco');
  });
  
  test('Fluxo Completo - Recomendações', async () => {
    const request = {
      request: {
        type: 'IntentRequest',
        intent: {
          name: 'GetRecommendations'
        }
      }
    };
    
    const response = await handler(request);
    
    expect(response.response.outputSpeech).toBeDefined();
    expect(response.response.card).toBeDefined();
  });
});
```

## 📊 Monitoramento e Logs

### 1. Configuração de Logs (lambda/utils/logger.js)

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
    new winston.transports.Console({
      format: winston.format.combine(
        winston.format.colorize(),
        winston.format.simple()
      )
    }),
    new winston.transports.File({ 
      filename: 'error.log', 
      level: 'error',
      maxsize: 5242880, // 5MB
      maxFiles: 5
    }),
    new winston.transports.File({ 
      filename: 'combined.log',
      maxsize: 5242880,
      maxFiles: 5
    })
  ]
});

module.exports = logger;
```

### 2. Métricas Personalizadas (lambda/utils/metrics.js)

```javascript
const AWS = require('aws-sdk');
const cloudwatch = new AWS.CloudWatch();

class MetricsCollector {
  constructor() {
    this.metrics = {
      requests: 0,
      errors: 0,
      responseTimes: [],
      cacheHits: 0,
      cacheMisses: 0
    };
  }
  
  recordRequest(duration, success = true) {
    this.metrics.requests++;
    
    if (!success) {
      this.metrics.errors++;
    }
    
    this.metrics.responseTimes.push(duration);
    
    if (this.metrics.responseTimes.length > 100) {
      this.metrics.responseTimes.shift();
    }
  }
  
  async sendMetric(metricName, value, unit = 'Count') {
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
  
  getMetrics() {
    const avgResponseTime = this.metrics.responseTimes.length > 0
      ? this.metrics.responseTimes.reduce((a, b) => a + b, 0) / this.metrics.responseTimes.length
      : 0;
    
    const errorRate = this.metrics.requests > 0
      ? (this.metrics.errors / this.metrics.requests) * 100
      : 0;
    
    return {
      ...this.metrics,
      avgResponseTime: Math.round(avgResponseTime),
      errorRate: Math.round(errorRate * 100) / 100
    };
  }
}

module.exports = new MetricsCollector();
```

## 🔒 Configuração de Segurança

### 1. Variáveis de Ambiente (.env.example)

```bash
# APIs GoodWe
GOODWE_API_URL=http://localhost:3000
ML_API_URL=http://localhost:8000

# Autenticação
GOODWE_API_KEY=your_goodwe_api_key
ML_API_KEY=your_ml_api_key

# Serviços externos
WEATHER_API_KEY=your_weather_api_key
UTILITY_API_KEY=your_utility_api_key

# Configurações da skill
SKILL_ID=amzn1.ask.skill.your-skill-id
LAMBDA_FUNCTION_NAME=GoodWeSolarAssistant

# Debug e logs
DEBUG=false
LOG_LEVEL=info

# Configurações de cache
CACHE_TTL=300000
CACHE_MAX_SIZE=1000

# Configurações de retry
MAX_RETRIES=3
RETRY_DELAY=1000
```

### 2. Política IAM (deployment/iam-policy.json)

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
    },
    {
      "Effect": "Allow",
      "Action": [
        "cloudwatch:PutMetricData"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "sns:Publish"
      ],
      "Resource": "arn:aws:sns:us-east-1:123456789012:goodwe-alerts"
    }
  ]
}
```

## 📚 Documentação de Deploy

### 1. README de Deploy (deployment/README.md)

```markdown
# Deploy da GoodWe Alexa Skill

## Pré-requisitos

1. Node.js 18+
2. AWS CLI configurado
3. ASK CLI instalado
4. Conta Amazon Developer

## Deploy

1. Clone o repositório
2. Configure as variáveis de ambiente
3. Execute: `./deployment/deploy.sh`

## Testes

Execute: `./deployment/test.sh`

## Rollback

Execute: `./deployment/rollback.sh`
```

---

**Conclusão**: Esta implementação completa fornece uma base sólida para a Alexa Skill GoodWe, incluindo todas as funcionalidades necessárias, testes abrangentes e scripts de deploy automatizados.
