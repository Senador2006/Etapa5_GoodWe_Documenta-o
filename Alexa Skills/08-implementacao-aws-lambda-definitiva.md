# Implementação Definitiva: Alexa Skill com AWS Lambda

## Visão Geral da Arquitetura

Esta é a implementação definitiva usando AWS Lambda, otimizada para facilidade de deploy e manutenção.

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────────┐
│   Alexa Device  │───▶│  Amazon Alexa    │───▶│   AWS Lambda        │
│   (Echo, etc.)  │    │     Service      │    │   (Node.js)         │
└─────────────────┘    └──────────────────┘    └─────────────────────┘
                                                          │
                                                          ▼
                                               ┌─────────────────────┐
                                               │   API GoodWe        │
                                               │   (Energia Solar)   │
                                               └─────────────────────┘
```

## 1. Estrutura Completa do Projeto

```
alexa-goodwe-skill/
├── package.json                     # Dependências e scripts
├── package-lock.json               # Lock das dependências
├── index.js                        # Arquivo principal - entry point
├── README.md                       # Documentação do projeto
├── .env.example                    # Template de variáveis de ambiente
├── .gitignore                      # Arquivos a ignorar no Git
│
├── handlers/                       # 📁 Manipuladores de intents
│   ├── LaunchHandler.js            # Handler do lançamento da skill
│   ├── EnergyConsumptionHandler.js # Handler para consumo de energia
│   ├── EnergyProductionHandler.js  # Handler para produção de energia
│   ├── DeviceStatusHandler.js      # Handler para status de dispositivos
│   ├── SavingsHandler.js           # Handler para economias
│   ├── WeatherImpactHandler.js     # Handler para impacto climático
│   ├── HelpHandler.js              # Handler de ajuda
│   ├── StopCancelHandler.js        # Handler para parar/cancelar
│   └── ErrorHandler.js             # Handler global de erros
│
├── services/                       # 📁 Serviços de negócio
│   ├── GoodWeApiService.js         # Serviço para API GoodWe
│   ├── ResponseService.js          # Serviço para respostas Alexa
│   └── CacheService.js             # Serviço de cache (opcional)
│
├── utils/                          # 📁 Utilitários
│   ├── DataFormatter.js           # Formatação de dados
│   ├── SlotExtractor.js           # Extração de valores dos slots
│   ├── TimeProcessor.js           # Processamento de períodos
│   └── Constants.js               # Constantes do projeto
│
├── config/                         # 📁 Configurações
│   ├── SkillConfig.js             # Configurações da skill
│   └── ApiConfig.js               # Configurações da API
│
├── skill-package/                  # 📁 Configuração da Skill
│   ├── skill.json                 # Manifest da skill
│   └── interactionModels/
│       └── custom/
│           └── pt-BR.json         # Modelo de interação em português
│
├── lambda/                         # 📁 Configuração AWS Lambda
│   ├── lambda-config.json         # Configuração do Lambda
│   └── environment-variables.json  # Variáveis de ambiente
│
├── tests/                          # 📁 Testes
│   ├── unit/                      # Testes unitários
│   ├── integration/               # Testes de integração
│   └── mock-data/                 # Dados mock para testes
│
└── deploy/                         # 📁 Scripts de deploy
    ├── deploy.sh                  # Script de deploy principal
    ├── zip-lambda.sh             # Script para criar ZIP do Lambda
    └── update-lambda.sh          # Script para atualizar Lambda
```

## 2. Arquivos Principais

### package.json
```json
{
  "name": "alexa-goodwe-skill",
  "version": "1.0.0",
  "description": "Alexa Skill para monitoramento de energia solar GoodWe",
  "main": "index.js",
  "scripts": {
    "start": "node index.js",
    "test": "jest",
    "lint": "eslint .",
    "deploy": "./deploy/deploy.sh",
    "zip": "./deploy/zip-lambda.sh",
    "update": "./deploy/update-lambda.sh",
    "local-test": "sam local start-api"
  },
  "dependencies": {
    "ask-sdk-core": "^2.13.0",
    "ask-sdk-model": "^1.49.0",
    "axios": "^1.6.0",
    "moment": "^2.29.4"
  },
  "devDependencies": {
    "jest": "^29.0.0",
    "eslint": "^8.0.0",
    "@types/jest": "^29.0.0"
  },
  "keywords": ["alexa", "skill", "energia-solar", "goodwe", "aws-lambda"],
  "author": "Seu Nome",
  "license": "MIT"
}
```

### index.js (Entry Point)
```javascript
const Alexa = require('ask-sdk-core');

// Importar todos os handlers
const LaunchHandler = require('./handlers/LaunchHandler');
const EnergyConsumptionHandler = require('./handlers/EnergyConsumptionHandler');
const EnergyProductionHandler = require('./handlers/EnergyProductionHandler');
const DeviceStatusHandler = require('./handlers/DeviceStatusHandler');
const SavingsHandler = require('./handlers/SavingsHandler');
const WeatherImpactHandler = require('./handlers/WeatherImpactHandler');
const HelpHandler = require('./handlers/HelpHandler');
const StopCancelHandler = require('./handlers/StopCancelHandler');
const ErrorHandler = require('./handlers/ErrorHandler');

// Configurar skill
const skillBuilder = Alexa.SkillBuilders.custom()
    .addRequestHandlers(
        LaunchHandler,
        EnergyConsumptionHandler,
        EnergyProductionHandler,
        DeviceStatusHandler,
        SavingsHandler,
        WeatherImpactHandler,
        HelpHandler,
        StopCancelHandler
    )
    .addErrorHandlers(ErrorHandler)
    .addRequestInterceptors({
        process(handlerInput) {
            console.log('=== REQUEST ===');
            console.log(JSON.stringify(handlerInput.requestEnvelope.request, null, 2));
        }
    })
    .addResponseInterceptors({
        process(handlerInput, response) {
            console.log('=== RESPONSE ===');
            console.log(JSON.stringify(response, null, 2));
        }
    })
    .withCustomUserAgent('GoodWe-Alexa-Skill/1.0.0');

// Exportar handler para AWS Lambda
exports.handler = skillBuilder.lambda();
```

## 3. Handlers Específicos

### handlers/LaunchHandler.js
```javascript
const ResponseService = require('../services/ResponseService');
const { getTimeGreeting } = require('../utils/DataFormatter');

const LaunchHandler = {
    canHandle(handlerInput) {
        return handlerInput.requestEnvelope.request.type === 'LaunchRequest';
    },
    
    handle(handlerInput) {
        const greeting = getTimeGreeting();
        
        const speechText = `
            ${greeting}! Bem-vindo ao assistente de energia solar GoodWe. 
            Eu posso te ajudar a monitorar consumo, produção, status dos dispositivos e economias.
            
            Você pode perguntar:
            "Qual o consumo de energia hoje?"
            "Como está o inversor?"
            "Quanta energia foi gerada esta semana?"
            
            O que você gostaria de saber?
        `;
        
        return ResponseService.buildResponse(
            handlerInput,
            speechText,
            'Assistente GoodWe',
            'Bem-vindo ao monitoramento de energia solar!',
            false // Manter sessão aberta
        );
    }
};

module.exports = LaunchHandler;
```

### handlers/EnergyConsumptionHandler.js
```javascript
const GoodWeApiService = require('../services/GoodWeApiService');
const ResponseService = require('../services/ResponseService');
const SlotExtractor = require('../utils/SlotExtractor');
const TimeProcessor = require('../utils/TimeProcessor');
const DataFormatter = require('../utils/DataFormatter');

const EnergyConsumptionHandler = {
    canHandle(handlerInput) {
        return handlerInput.requestEnvelope.request.type === 'IntentRequest'
            && handlerInput.requestEnvelope.request.intent.name === 'GetEnergyConsumptionIntent';
    },
    
    async handle(handlerInput) {
        try {
            console.log('🔋 Processing energy consumption request...');
            
            // Extrair período do slot
            const slots = handlerInput.requestEnvelope.request.intent.slots;
            const timeFrameValue = SlotExtractor.extractValue(slots.timeFrame);
            const timeFrame = TimeProcessor.normalizeTimeFrame(timeFrameValue);
            
            console.log(`Requesting consumption for period: ${timeFrame}`);
            
            // Chamar API GoodWe
            const consumptionData = await GoodWeApiService.getEnergyConsumption(timeFrame);
            
            if (!consumptionData || !consumptionData.data) {
                return ResponseService.buildErrorResponse(handlerInput, 'no_data');
            }
            
            // Processar dados
            const data = consumptionData.data;
            const consumption = data.total_consumption || 0;
            const unit = data.unit || 'kWh';
            const cost = data.cost;
            
            // Formatar resposta
            const formattedPeriod = TimeProcessor.formatTimeFrame(timeFrame);
            const formattedConsumption = DataFormatter.formatEnergyValue(consumption, unit);
            
            let speechText = `O consumo de energia ${formattedPeriod} foi de ${formattedConsumption}.`;
            
            // Adicionar informações extras
            if (cost) {
                const formattedCost = DataFormatter.formatCurrency(cost);
                speechText += ` O custo estimado é de ${formattedCost}.`;
            }
            
            if (data.comparison) {
                const comparison = data.comparison;
                const percentage = Math.abs(comparison.percentage);
                const trend = comparison.percentage > 0 ? "maior" : "menor";
                const formattedPercentage = DataFormatter.formatPercentage(percentage);
                speechText += ` Isso é ${formattedPercentage} ${trend} que o período anterior.`;
            }
            
            // Card para dispositivos com tela
            const cardContent = [
                `Período: ${formattedPeriod}`,
                `Consumo: ${formattedConsumption}`,
                cost ? `Custo: ${DataFormatter.formatCurrency(cost)}` : null
            ].filter(Boolean).join('\n');
            
            return ResponseService.buildResponse(
                handlerInput,
                speechText,
                'Consumo de Energia',
                cardContent
            );
            
        } catch (error) {
            console.error('Error in consumption handler:', error);
            return ResponseService.buildErrorResponse(handlerInput, 'api_error');
        }
    }
};

module.exports = EnergyConsumptionHandler;
```

### handlers/DeviceStatusHandler.js
```javascript
const GoodWeApiService = require('../services/GoodWeApiService');
const ResponseService = require('../services/ResponseService');
const SlotExtractor = require('../utils/SlotExtractor');
const DataFormatter = require('../utils/DataFormatter');

const DeviceStatusHandler = {
    canHandle(handlerInput) {
        return handlerInput.requestEnvelope.request.type === 'IntentRequest'
            && handlerInput.requestEnvelope.request.intent.name === 'GetDeviceStatusIntent';
    },
    
    async handle(handlerInput) {
        try {
            console.log('🔧 Processing device status request...');
            
            // Extrair tipo de dispositivo
            const slots = handlerInput.requestEnvelope.request.intent.slots;
            const deviceType = SlotExtractor.extractValue(slots.deviceType) || 'sistema';
            
            console.log(`Requesting status for device: ${deviceType}`);
            
            // Chamar API GoodWe
            const statusData = await GoodWeApiService.getDeviceStatus(deviceType);
            
            if (!statusData || !statusData.data) {
                return ResponseService.buildErrorResponse(handlerInput, 'no_data');
            }
            
            // Processar resposta baseada no tipo de dispositivo
            const data = statusData.data;
            const response = this.buildDeviceResponse(deviceType, data);
            
            return ResponseService.buildResponse(
                handlerInput,
                response.speechText,
                response.cardTitle,
                response.cardContent
            );
            
        } catch (error) {
            console.error('Error in device status handler:', error);
            return ResponseService.buildErrorResponse(handlerInput, 'api_error');
        }
    },
    
    buildDeviceResponse(deviceType, data) {
        const status = data.status || 'desconhecido';
        
        switch (deviceType.toLowerCase()) {
            case 'inversor':
                return this.buildInverterResponse(data, status);
            
            case 'painel solar':
            case 'painéis solares':
                return this.buildSolarPanelResponse(data, status);
            
            case 'bateria':
                return this.buildBatteryResponse(data, status);
            
            default:
                return this.buildSystemResponse(data, status);
        }
    },
    
    buildInverterResponse(data, status) {
        const power = data.current_power || 0;
        const efficiency = data.efficiency || 0;
        const temperature = data.temperature;
        
        let speechText = `O inversor está ${status}. `;
        speechText += `Potência atual: ${DataFormatter.formatPower(power)}, `;
        speechText += `eficiência: ${DataFormatter.formatPercentage(efficiency)}.`;
        
        if (temperature) {
            speechText += ` Temperatura: ${temperature}°C.`;
        }
        
        const cardContent = [
            `Status: ${status}`,
            `Potência: ${DataFormatter.formatPower(power)}`,
            `Eficiência: ${DataFormatter.formatPercentage(efficiency)}`,
            temperature ? `Temperatura: ${temperature}°C` : null
        ].filter(Boolean).join('\n');
        
        return {
            speechText,
            cardTitle: 'Status do Inversor',
            cardContent
        };
    },
    
    buildSolarPanelResponse(data, status) {
        const generation = data.current_generation || 0;
        const weatherFactor = data.weather_factor || 0;
        const irradiance = data.irradiance;
        
        let speechText = `Os painéis solares estão ${status}. `;
        speechText += `Geração atual: ${DataFormatter.formatPower(generation)}. `;
        speechText += `Fator climático: ${DataFormatter.formatPercentage(weatherFactor)}.`;
        
        if (irradiance) {
            speechText += ` Irradiância solar: ${irradiance} W/m².`;
        }
        
        const cardContent = [
            `Status: ${status}`,
            `Geração: ${DataFormatter.formatPower(generation)}`,
            `Fator Climático: ${DataFormatter.formatPercentage(weatherFactor)}`,
            irradiance ? `Irradiância: ${irradiance} W/m²` : null
        ].filter(Boolean).join('\n');
        
        return {
            speechText,
            cardTitle: 'Status dos Painéis Solares',
            cardContent
        };
    },
    
    buildBatteryResponse(data, status) {
        const chargeLevel = data.charge_level || 0;
        const chargeStatus = data.charge_status || 'parado';
        const capacity = data.capacity;
        
        let speechText = `A bateria está ${status} com ${DataFormatter.formatPercentage(chargeLevel)} de carga. `;
        speechText += `Status de carregamento: ${chargeStatus}.`;
        
        if (capacity) {
            speechText += ` Capacidade total: ${DataFormatter.formatEnergyValue(capacity)}.`;
        }
        
        const cardContent = [
            `Status: ${status}`,
            `Carga: ${DataFormatter.formatPercentage(chargeLevel)}`,
            `Carregamento: ${chargeStatus}`,
            capacity ? `Capacidade: ${DataFormatter.formatEnergyValue(capacity)}` : null
        ].filter(Boolean).join('\n');
        
        return {
            speechText,
            cardTitle: 'Status da Bateria',
            cardContent
        };
    },
    
    buildSystemResponse(data, status) {
        let speechText = `O sistema está ${status}.`;
        
        const cardLines = [`Status: ${status}`];
        
        if (data.summary) {
            const summary = data.summary;
            const generation = summary.generation || 0;
            const consumption = summary.consumption || 0;
            const batteryLevel = summary.battery_level;
            
            speechText += ` Geração atual: ${DataFormatter.formatPower(generation)}, `;
            speechText += `consumo: ${DataFormatter.formatPower(consumption)}.`;
            
            cardLines.push(`Geração: ${DataFormatter.formatPower(generation)}`);
            cardLines.push(`Consumo: ${DataFormatter.formatPower(consumption)}`);
            
            if (batteryLevel !== undefined) {
                speechText += ` Bateria: ${DataFormatter.formatPercentage(batteryLevel)}.`;
                cardLines.push(`Bateria: ${DataFormatter.formatPercentage(batteryLevel)}`);
            }
        }
        
        return {
            speechText,
            cardTitle: 'Status do Sistema',
            cardContent: cardLines.join('\n')
        };
    }
};

module.exports = DeviceStatusHandler;
```

## 4. Serviços

### services/GoodWeApiService.js
```javascript
const axios = require('axios');
const ApiConfig = require('../config/ApiConfig');

class GoodWeApiService {
    constructor() {
        this.client = axios.create({
            baseURL: ApiConfig.BASE_URL,
            timeout: ApiConfig.TIMEOUT,
            headers: {
                'Authorization': `Bearer ${ApiConfig.API_KEY}`,
                'Content-Type': 'application/json',
                'User-Agent': 'Alexa-GoodWe-Skill/1.0.0'
            }
        });
        
        // Interceptor para logging
        this.client.interceptors.request.use(
            (config) => {
                console.log(`🌐 API Request: ${config.method?.toUpperCase()} ${config.url}`);
                return config;
            },
            (error) => {
                console.error('🚨 Request Error:', error);
                return Promise.reject(error);
            }
        );
        
        this.client.interceptors.response.use(
            (response) => {
                console.log(`✅ API Response: ${response.status} ${response.statusText}`);
                return response;
            },
            (error) => {
                console.error('🚨 Response Error:', {
                    status: error.response?.status,
                    statusText: error.response?.statusText,
                    data: error.response?.data
                });
                return Promise.reject(error);
            }
        );
    }
    
    async makeRequest(endpoint, params = {}) {
        try {
            const response = await this.client.get(endpoint, { params });
            return response.data;
        } catch (error) {
            console.error(`API Error for ${endpoint}:`, error.message);
            return null;
        }
    }
    
    async getEnergyConsumption(timeFrame = 'today') {
        return await this.makeRequest('/energy/consumption', {
            period: timeFrame,
            metric: 'consumption'
        });
    }
    
    async getEnergyProduction(timeFrame = 'today') {
        return await this.makeRequest('/energy/production', {
            period: timeFrame,
            metric: 'production'
        });
    }
    
    async getDeviceStatus(deviceType = 'system') {
        const deviceEndpoints = {
            'inversor': '/devices/inverter',
            'painel solar': '/devices/solar-panels',
            'bateria': '/devices/battery',
            'sistema': '/devices/system-status'
        };
        
        const endpoint = deviceEndpoints[deviceType] || '/devices/system-status';
        return await this.makeRequest(endpoint);
    }
    
    async getSavings(timeFrame = 'today') {
        return await this.makeRequest('/financial/savings', {
            period: timeFrame
        });
    }
    
    async getWeatherImpact() {
        return await this.makeRequest('/weather/impact');
    }
}

module.exports = new GoodWeApiService();
```

### services/ResponseService.js
```javascript
class ResponseService {
    
    static buildResponse(handlerInput, speechText, cardTitle = null, cardContent = null, shouldEndSession = true) {
        const responseBuilder = handlerInput.responseBuilder.speak(speechText);
        
        if (cardTitle && cardContent) {
            responseBuilder.withSimpleCard(cardTitle, cardContent);
        }
        
        if (shouldEndSession) {
            responseBuilder.withShouldEndSession(true);
        } else {
            responseBuilder.reprompt(this.getRepromptText());
        }
        
        return responseBuilder.getResponse();
    }
    
    static buildErrorResponse(handlerInput, errorType = 'general') {
        const errorMessages = {
            'api_error': 'Desculpe, não consegui acessar os dados no momento. Tente novamente mais tarde.',
            'no_data': 'Não encontrei dados para o período solicitado. Tente outro período como hoje, ontem, ou esta semana.',
            'invalid_device': 'Não reconheci o dispositivo mencionado. Tente inversor, painel solar, bateria ou sistema.',
            'timeout': 'A consulta está demorando mais que o esperado. Tente novamente em alguns minutos.',
            'general': 'Ocorreu um erro inesperado. Tente novamente mais tarde.'
        };
        
        const speechText = errorMessages[errorType] || errorMessages.general;
        
        return this.buildResponse(
            handlerInput,
            speechText,
            'Erro',
            `Tipo: ${errorType}\nTente novamente mais tarde.`
        );
    }
    
    static getRepromptText() {
        const options = [
            'O que mais você gostaria de saber sobre seu sistema de energia solar?',
            'Posso ajudar com informações sobre consumo, produção, dispositivos ou economias. O que você precisa?',
            'Que outras informações sobre energia solar você gostaria de conhecer?'
        ];
        
        return options[Math.floor(Math.random() * options.length)];
    }
}

module.exports = ResponseService;
```

## 5. Utilitários

### utils/DataFormatter.js
```javascript
class DataFormatter {
    
    static formatEnergyValue(value, unit = 'kWh') {
        if (!value || isNaN(value)) return '0 kWh';
        
        if (value >= 1000) {
            return `${(value / 1000).toFixed(1)} MW${unit.substring(2)}`;
        } else if (value >= 1) {
            return `${value.toFixed(1)} ${unit}`;
        } else {
            return `${(value * 1000).toFixed(0)} W${unit.substring(2)}`;
        }
    }
    
    static formatPower(watts) {
        if (!watts || isNaN(watts)) return '0 W';
        
        if (watts >= 1000) {
            return `${(watts / 1000).toFixed(1)} kW`;
        } else {
            return `${watts.toFixed(0)} W`;
        }
    }
    
    static formatCurrency(value, currency = 'R$') {
        if (!value || isNaN(value)) return 'R$ 0,00';
        return `${currency} ${value.toFixed(2).replace('.', ',')}`;
    }
    
    static formatPercentage(value) {
        if (!value || isNaN(value)) return '0%';
        return `${value.toFixed(1)}%`;
    }
    
    static getTimeGreeting() {
        const hour = new Date().getHours();
        
        if (hour < 6) {
            return 'Boa madrugada';
        } else if (hour < 12) {
            return 'Bom dia';
        } else if (hour < 18) {
            return 'Boa tarde';
        } else {
            return 'Boa noite';
        }
    }
}

module.exports = DataFormatter;
```

### utils/SlotExtractor.js
```javascript
class SlotExtractor {
    
    static extractValue(slot) {
        if (!slot) return null;
        
        // Verificar se há valor resolvido primeiro
        if (slot.resolutions && 
            slot.resolutions.resolutionsPerAuthority && 
            slot.resolutions.resolutionsPerAuthority.length > 0) {
            
            const firstAuthority = slot.resolutions.resolutionsPerAuthority[0];
            if (firstAuthority.status.code === 'ER_SUCCESS_MATCH' && 
                firstAuthority.values && 
                firstAuthority.values.length > 0) {
                return firstAuthority.values[0].value.name;
            }
        }
        
        // Usar valor literal se não houver resolução
        return slot.value;
    }
    
    static extractAllSlots(slots) {
        const result = {};
        
        Object.keys(slots).forEach(slotName => {
            result[slotName] = this.extractValue(slots[slotName]);
        });
        
        return result;
    }
}

module.exports = SlotExtractor;
```

### utils/TimeProcessor.js
```javascript
class TimeProcessor {
    
    static normalizeTimeFrame(timeValue) {
        if (!timeValue) return 'today';
        
        const normalizedValue = timeValue.toLowerCase().trim();
        
        const mappings = {
            'hoje': 'today',
            'ontem': 'yesterday',
            'esta semana': 'this_week',
            'semana passada': 'last_week',
            'última semana': 'last_week',
            'este mês': 'this_month',
            'mês passado': 'last_month',
            'último mês': 'last_month',
            'este ano': 'this_year',
            'ano passado': 'last_year'
        };
        
        return mappings[normalizedValue] || 'today';
    }
    
    static formatTimeFrame(timeFrame) {
        const mappings = {
            'today': 'hoje',
            'yesterday': 'ontem',
            'this_week': 'esta semana',
            'last_week': 'na semana passada',
            'this_month': 'este mês',
            'last_month': 'no mês passado',
            'this_year': 'este ano',
            'last_year': 'no ano passado'
        };
        
        return mappings[timeFrame] || timeFrame;
    }
}

module.exports = TimeProcessor;
```

## 6. Configurações

### config/ApiConfig.js
```javascript
module.exports = {
    BASE_URL: process.env.GOODWE_API_URL || 'https://api.goodwe.com/v1',
    API_KEY: process.env.GOODWE_API_KEY,
    TIMEOUT: 8000, // 8 segundos para dar buffer
    
    // Endpoints específicos
    ENDPOINTS: {
        CONSUMPTION: '/energy/consumption',
        PRODUCTION: '/energy/production',
        SAVINGS: '/financial/savings',
        WEATHER: '/weather/impact',
        DEVICES: {
            INVERTER: '/devices/inverter',
            SOLAR_PANELS: '/devices/solar-panels',
            BATTERY: '/devices/battery',
            SYSTEM: '/devices/system-status'
        }
    }
};
```

### config/SkillConfig.js
```javascript
module.exports = {
    SKILL_ID: process.env.ALEXA_SKILL_ID,
    
    // Configurações de resposta
    DEFAULT_SHOULD_END_SESSION: true,
    DEFAULT_CARD_TITLE: 'Assistente GoodWe',
    
    // Configurações de cache (se implementado)
    CACHE_TTL: {
        ENERGY_DATA: 300,    // 5 minutos
        DEVICE_STATUS: 60,   // 1 minuto
        WEATHER: 600         // 10 minutos
    },
    
    // Configurações de retry
    RETRY_ATTEMPTS: 2,
    RETRY_DELAY: 1000
};
```

## 7. Configuração da Skill no Alexa Developer Console

### skill-package/skill.json
```json
{
  "manifest": {
    "publishingInformation": {
      "locales": {
        "pt-BR": {
          "name": "Assistente de Energia Solar GoodWe",
          "summary": "Monitore seu sistema de energia solar por voz",
          "description": "Acompanhe consumo, produção, status dos dispositivos e economias do seu sistema de energia solar GoodWe através de comandos de voz simples e intuitivos. Pergunte sobre dados em tempo real, históricos e previsões.",
          "keywords": [
            "energia solar",
            "goodwe",
            "monitoramento",
            "sustentabilidade",
            "inversor",
            "painéis solares",
            "bateria",
            "economia",
            "consumo",
            "produção"
          ],
          "examplePhrases": [
            "Alexa, abra assistente energia",
            "qual o consumo de energia hoje",
            "como está o inversor",
            "quanta energia foi gerada esta semana"
          ],
          "smallIconUri": "https://s3.amazonaws.com/seu-bucket/icons/small-icon-108x108.png",
          "largeIconUri": "https://s3.amazonaws.com/seu-bucket/icons/large-icon-512x512.png"
        }
      },
      "isAvailableWorldwide": false,
      "distributionCountries": ["BR"],
      "category": "SMART_HOME",
      "testingInstructions": "Para testar a skill, use comandos como 'qual o consumo de energia hoje' ou 'como está o inversor'. A skill requer acesso à API GoodWe."
    },
    "apis": {
      "custom": {
        "endpoint": {
          "uri": "arn:aws:lambda:us-east-1:123456789012:function:alexa-goodwe-skill"
        },
        "interfaces": [],
        "regions": {
          "NA": {
            "endpoint": {
              "uri": "arn:aws:lambda:us-east-1:123456789012:function:alexa-goodwe-skill"
            }
          }
        }
      }
    },
    "manifestVersion": "1.0",
    "privacyAndCompliance": {
      "allowsPurchases": false,
      "usesPersonalInfo": false,
      "isChildDirected": false,
      "isExportCompliant": true,
      "containsAds": false,
      "locales": {
        "pt-BR": {
          "privacyPolicyUrl": "https://seu-site.com/privacy-policy",
          "termsOfUseUrl": "https://seu-site.com/terms-of-use"
        }
      }
    }
  }
}
```

### skill-package/interactionModels/custom/pt-BR.json
```json
{
  "interactionModel": {
    "languageModel": {
      "invocationName": "assistente energia",
      "intents": [
        {
          "name": "GetEnergyConsumptionIntent",
          "slots": [
            {
              "name": "timeFrame",
              "type": "TimeFrameSlot"
            }
          ],
          "samples": [
            "qual o consumo de energia {timeFrame}",
            "quanto gastei de energia {timeFrame}",
            "me diga o gasto de energia {timeFrame}",
            "consumo {timeFrame}",
            "gasto energético {timeFrame}",
            "quanto consumi {timeFrame}",
            "qual foi o consumo {timeFrame}",
            "consumo de energia {timeFrame}",
            "gasto de energia {timeFrame}"
          ]
        },
        {
          "name": "GetEnergyProductionIntent",
          "slots": [
            {
              "name": "timeFrame",
              "type": "TimeFrameSlot"
            }
          ],
          "samples": [
            "quanta energia foi gerada {timeFrame}",
            "produção de energia {timeFrame}",
            "quanto os painéis geraram {timeFrame}",
            "geração {timeFrame}",
            "quanto foi produzido {timeFrame}",
            "energia gerada {timeFrame}",
            "produção {timeFrame}",
            "quanto geraram os painéis {timeFrame}"
          ]
        },
        {
          "name": "GetDeviceStatusIntent",
          "slots": [
            {
              "name": "deviceType",
              "type": "DeviceTypeSlot"
            }
          ],
          "samples": [
            "como está o {deviceType}",
            "qual o status do {deviceType}",
            "me mostre informações sobre o {deviceType}",
            "verifique o {deviceType}",
            "status do {deviceType}",
            "como está funcionando o {deviceType}",
            "situação do {deviceType}",
            "condições do {deviceType}"
          ]
        },
        {
          "name": "GetSavingsIntent",
          "slots": [
            {
              "name": "timeFrame",
              "type": "TimeFrameSlot"
            }
          ],
          "samples": [
            "quanto economizei {timeFrame}",
            "qual a economia {timeFrame}",
            "me diga a economia {timeFrame}",
            "economias {timeFrame}",
            "quanto poupei {timeFrame}",
            "economia de dinheiro {timeFrame}",
            "valor economizado {timeFrame}"
          ]
        },
        {
          "name": "GetWeatherImpactIntent",
          "samples": [
            "como está o clima afetando a geração",
            "impacto do tempo na produção",
            "tempo está bom para energia solar",
            "clima e energia solar",
            "como o tempo afeta os painéis",
            "influência do clima na geração",
            "condições climáticas e produção"
          ]
        },
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
          "name": "AMAZON.FallbackIntent",
          "samples": []
        }
      ],
      "types": [
        {
          "name": "TimeFrameSlot",
          "values": [
            {
              "name": {
                "value": "hoje",
                "synonyms": ["hoje em dia", "neste dia", "no dia de hoje"]
              }
            },
            {
              "name": {
                "value": "ontem",
                "synonyms": ["dia anterior", "ontem mesmo"]
              }
            },
            {
              "name": {
                "value": "esta semana",
                "synonyms": ["semana atual", "nesta semana", "na semana"]
              }
            },
            {
              "name": {
                "value": "semana passada",
                "synonyms": ["última semana", "semana anterior", "na semana passada"]
              }
            },
            {
              "name": {
                "value": "este mês",
                "synonyms": ["mês atual", "neste mês", "no mês"]
              }
            },
            {
              "name": {
                "value": "mês passado",
                "synonyms": ["último mês", "mês anterior", "no mês passado"]
              }
            }
          ]
        },
        {
          "name": "DeviceTypeSlot",
          "values": [
            {
              "name": {
                "value": "inversor",
                "synonyms": ["inverter", "conversor de energia", "conversor"]
              }
            },
            {
              "name": {
                "value": "painel solar",
                "synonyms": [
                  "painel",
                  "placa solar",
                  "módulo fotovoltaico",
                  "painéis",
                  "painéis solares",
                  "placas solares",
                  "módulos fotovoltaicos"
                ]
              }
            },
            {
              "name": {
                "value": "bateria",
                "synonyms": [
                  "banco de bateria",
                  "armazenamento",
                  "baterias",
                  "sistema de armazenamento"
                ]
              }
            },
            {
              "name": {
                "value": "sistema",
                "synonyms": [
                  "sistema completo",
                  "instalação",
                  "usina solar",
                  "sistema solar",
                  "conjunto"
                ]
              }
            }
          ]
        }
      ]
    },
    "dialog": {
      "intents": [
        {
          "name": "GetEnergyConsumptionIntent",
          "confirmationRequired": false,
          "prompts": {},
          "slots": [
            {
              "name": "timeFrame",
              "type": "TimeFrameSlot",
              "confirmationRequired": false,
              "elicitationRequired": false,
              "prompts": {}
            }
          ]
        }
      ],
      "delegationStrategy": "ALWAYS"
    },
    "prompts": []
  }
}
```

## 8. Deploy e Configuração AWS

### lambda/lambda-config.json
```json
{
  "FunctionName": "alexa-goodwe-skill",
  "Runtime": "nodejs18.x",
  "Role": "arn:aws:iam::123456789012:role/lambda-execution-role",
  "Handler": "index.handler",
  "Description": "Alexa Skill para monitoramento de energia solar GoodWe",
  "Timeout": 30,
  "MemorySize": 256,
  "Environment": {
    "Variables": {
      "NODE_ENV": "production",
      "GOODWE_API_KEY": "sua_chave_api_aqui",
      "GOODWE_API_URL": "https://api.goodwe.com/v1",
      "ALEXA_SKILL_ID": "amzn1.ask.skill.xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
    }
  },
  "Tags": {
    "Project": "GoodWe-Alexa-Skill",
    "Environment": "Production"
  }
}
```

### deploy/deploy.sh
```bash
#!/bin/bash

echo "🚀 Starting deployment process..."

# Verificar se AWS CLI está configurado
if ! aws sts get-caller-identity &> /dev/null; then
    echo "❌ AWS CLI not configured. Please run 'aws configure'"
    exit 1
fi

# Verificar se ASK CLI está configurado
if ! ask --version &> /dev/null; then
    echo "❌ ASK CLI not found. Please install: npm install -g ask-cli"
    exit 1
fi

# 1. Instalar dependências
echo "📦 Installing dependencies..."
npm install --production

# 2. Executar testes
echo "🧪 Running tests..."
npm test

# 3. Criar ZIP do Lambda
echo "📁 Creating Lambda ZIP..."
./deploy/zip-lambda.sh

# 4. Deploy ou atualizar função Lambda
echo "⬆️ Deploying Lambda function..."
if aws lambda get-function --function-name alexa-goodwe-skill &> /dev/null; then
    echo "Updating existing function..."
    aws lambda update-function-code \
        --function-name alexa-goodwe-skill \
        --zip-file fileb://lambda-deployment.zip
else
    echo "Creating new function..."
    aws lambda create-function \
        --cli-input-json file://lambda/lambda-config.json \
        --zip-file fileb://lambda-deployment.zip
fi

# 5. Configurar variáveis de ambiente
echo "🔧 Setting environment variables..."
aws lambda update-function-configuration \
    --function-name alexa-goodwe-skill \
    --environment file://lambda/environment-variables.json

# 6. Deploy da skill
echo "🎯 Deploying Alexa skill..."
ask deploy

# 7. Limpeza
echo "🧹 Cleaning up..."
rm -f lambda-deployment.zip

echo "✅ Deployment completed successfully!"
echo "📝 Don't forget to:"
echo "   1. Test the skill in Alexa Developer Console"
echo "   2. Enable skill testing for development"
echo "   3. Configure account linking if needed"
```

### deploy/zip-lambda.sh
```bash
#!/bin/bash

echo "📁 Creating Lambda deployment package..."

# Remover ZIP anterior se existir
rm -f lambda-deployment.zip

# Criar ZIP com todos os arquivos necessários
zip -r lambda-deployment.zip \
    index.js \
    package.json \
    handlers/ \
    services/ \
    utils/ \
    config/ \
    node_modules/ \
    -x "*.git*" "*.env*" "deploy/*" "tests/*" "*.md"

echo "✅ Lambda package created: lambda-deployment.zip"
```

## 9. Configuração no Alexa Developer Console

### Passos Detalhados:

1. **Criar Nova Skill**
   - Acesse [Alexa Developer Console](https://developer.amazon.com/alexa/console/ask)
   - Create Skill → "Assistente de Energia Solar GoodWe"
   - Choose model: Custom
   - Choose method: Provision your own

2. **Configurar Invocation**
   - Build → Invocation → Skill Invocation Name
   - Nome: "assistente energia"

3. **Importar Interaction Model**
   - Build → Interaction Model → JSON Editor
   - Cole o conteúdo de `pt-BR.json`
   - Save Model → Build Model

4. **Configurar Endpoint**
   - Build → Endpoint
   - Service Endpoint Type: AWS Lambda ARN
   - Default Region: `arn:aws:lambda:us-east-1:123456789012:function:alexa-goodwe-skill`

5. **Testar**
   - Test → Development
   - Teste com: "abra assistente energia"

6. **Distribuição (quando pronto)**
   - Distribution → Complete todas as seções
   - Certification → Submit for review

## 10. Comandos de Teste

### Frases para Testar:
```
# Ativação
"Alexa, abra assistente energia"

# Consumo
"qual o consumo de energia hoje"
"quanto gastei de energia esta semana"
"consumo de ontem"

# Produção
"quanta energia foi gerada hoje"
"produção desta semana"
"quanto os painéis geraram ontem"

# Dispositivos
"como está o inversor"
"qual o status da bateria"
"verifique o sistema"
"como estão os painéis solares"

# Economias
"quanto economizei este mês"
"qual a economia de hoje"

# Clima
"como está o clima afetando a geração"

# Ajuda
"ajuda"
"o que você pode fazer"

# Parar
"parar"
"cancelar"
```

## 11. Próximos Passos

1. **Configure AWS CLI e ASK CLI**
2. **Ajuste as configurações** nos arquivos de config
3. **Execute o deploy** com `npm run deploy`
4. **Configure a skill** no Alexa Developer Console
5. **Teste** com dispositivos reais
6. **Monitore logs** no CloudWatch
7. **Otimize** baseado no uso real

---

*Esta implementação definitiva fornece uma base sólida e escalável para sua Alexa Skill de energia solar GoodWe usando AWS Lambda.*
