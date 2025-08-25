# Código Node.js Completo - Alexa Skill

## Implementação Completa em Node.js

Esta seção contém todo o código Node.js necessário para implementar a Alexa Skill de energia solar.

## 1. Configuração do Projeto

### package.json
```json
{
  "name": "alexa-energy-skill",
  "version": "1.0.0",
  "description": "Alexa Skill para monitoramento de energia solar GoodWe",
  "main": "index.js",
  "scripts": {
    "start": "node index.js",
    "dev": "nodemon index.js",
    "test": "jest",
    "test:watch": "jest --watch",
    "mock-server": "node tests/mockServer.js",
    "lint": "eslint .",
    "lint:fix": "eslint . --fix",
    "deploy": "ask deploy",
    "simulate": "ask simulate -l pt-BR"
  },
  "dependencies": {
    "ask-sdk-core": "^2.13.0",
    "ask-sdk-model": "^1.49.0",
    "axios": "^1.6.0",
    "moment": "^2.29.4"
  },
  "devDependencies": {
    "eslint": "^8.0.0",
    "jest": "^29.0.0",
    "nodemon": "^3.0.0"
  },
  "keywords": ["alexa", "energia-solar", "goodwe", "iot"],
  "author": "Seu Nome",
  "license": "MIT"
}
```

## 2. Utilitários

### utils/apiClient.js
```javascript
const axios = require('axios');

class GoodWeAPIClient {
    constructor() {
        this.baseURL = process.env.API_BASE_URL || 'https://api.goodwe.com/v1';
        this.apiKey = process.env.GOODWE_API_KEY;
        this.timeout = 10000;
        
        this.client = axios.create({
            baseURL: this.baseURL,
            timeout: this.timeout,
            headers: {
                'Authorization': `Bearer ${this.apiKey}`,
                'Content-Type': 'application/json'
            }
        });
        
        // Interceptor para logging
        this.client.interceptors.request.use(
            (config) => {
                console.log(`API Request: ${config.method?.toUpperCase()} ${config.url}`);
                return config;
            },
            (error) => {
                console.error('Request Error:', error);
                return Promise.reject(error);
            }
        );
    }

    async makeRequest(endpoint, params = {}) {
        try {
            console.log(`Making request to: ${endpoint}`, { params });
            
            const response = await this.client.get(endpoint, { params });
            
            if (response.status === 200) {
                console.log(`Success: ${endpoint}`, { dataKeys: Object.keys(response.data) });
                return response.data;
            }
            
            console.warn(`Unexpected status: ${response.status} for ${endpoint}`);
            return null;
            
        } catch (error) {
            console.error('API Request Error:', {
                endpoint,
                status: error.response?.status,
                statusText: error.response?.statusText,
                message: error.message,
                data: error.response?.data
            });
            
            if (error.code === 'ECONNABORTED') {
                console.error('Request timeout - API may be slow');
            } else if (error.code === 'ENOTFOUND') {
                console.error('Network error - check connectivity');
            }
            
            return null;
        }
    }

    async getEnergyConsumption(timeFrame = 'today') {
        const params = {
            period: timeFrame,
            metric: 'consumption'
        };
        return await this.makeRequest('/energy/consumption', params);
    }

    async getEnergyProduction(timeFrame = 'today') {
        const params = {
            period: timeFrame,
            metric: 'production'
        };
        return await this.makeRequest('/energy/production', params);
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
        const params = { period: timeFrame };
        return await this.makeRequest('/financial/savings', params);
    }

    async getWeatherImpact() {
        return await this.makeRequest('/weather/impact');
    }

    async getSystemOverview() {
        return await this.makeRequest('/system/overview');
    }
}

module.exports = GoodWeAPIClient;
```

### utils/dataFormatter.js
```javascript
const moment = require('moment');

class DataFormatter {
    
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

    static formatCurrency(value, currency = 'R$') {
        if (!value || isNaN(value)) return 'R$ 0,00';
        return `${currency} ${value.toFixed(2).replace('.', ',')}`;
    }

    static formatPercentage(value) {
        if (!value || isNaN(value)) return '0%';
        return `${value.toFixed(1)}%`;
    }

    static formatPower(watts) {
        if (!watts || isNaN(watts)) return '0 W';
        
        if (watts >= 1000) {
            return `${(watts / 1000).toFixed(1)} kW`;
        } else {
            return `${watts.toFixed(0)} W`;
        }
    }

    static extractSlotValue(slot) {
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

    static formatDateTime(dateTime) {
        if (!dateTime) return '';
        return moment(dateTime).format('DD/MM/YYYY HH:mm');
    }

    static getTimeGreeting() {
        const hour = new Date().getHours();
        
        if (hour < 12) {
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

### utils/responseBuilder.js
```javascript
class ResponseBuilder {
    
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
            'no_data': 'Não encontrei dados para o período solicitado. Tente outro período.',
            'invalid_device': 'Não reconheci o dispositivo mencionado. Tente inversor, painel solar, bateria ou sistema.',
            'timeout': 'A consulta está demorando mais que o esperado. Tente novamente em alguns minutos.',
            'general': 'Ocorreu um erro inesperado. Tente novamente mais tarde.'
        };
        
        const speechText = errorMessages[errorType] || errorMessages.general;
        return this.buildResponse(handlerInput, speechText);
    }

    static buildHelpResponse(handlerInput) {
        const speechText = `
            Eu posso te ajudar a monitorar seu sistema de energia solar. Aqui estão algumas coisas que você pode perguntar:
            
            Para consumo: "Qual o consumo de energia hoje?"
            Para produção: "Quanta energia foi gerada esta semana?"
            Para dispositivos: "Como está o inversor?"
            Para economias: "Quanto economizei este mês?"
            Para clima: "Como o tempo está afetando a geração?"
            
            O que você gostaria de saber?
        `;
        
        return this.buildResponse(handlerInput, speechText, 'Ajuda', 'Use comandos de voz para monitorar seu sistema de energia solar.', false);
    }

    static getRepromptText() {
        const options = [
            'O que mais você gostaria de saber?',
            'Posso ajudar com mais alguma coisa?',
            'Que outras informações você precisa?'
        ];
        
        return options[Math.floor(Math.random() * options.length)];
    }

    static buildWelcomeResponse(handlerInput) {
        const greeting = require('./dataFormatter').getTimeGreeting();
        
        const speechText = `
            ${greeting}! Bem-vindo ao seu assistente de energia solar GoodWe. 
            Eu posso te ajudar a monitorar consumo, produção, status dos dispositivos e economias.
            O que você gostaria de saber?
        `;
        
        return this.buildResponse(handlerInput, speechText, 'Assistente GoodWe', 'Bem-vindo ao monitoramento de energia solar!', false);
    }
}

module.exports = ResponseBuilder;
```

## 3. Handlers

### handlers/launchRequestHandler.js
```javascript
const ResponseBuilder = require('../utils/responseBuilder');

const LaunchRequestHandler = {
    canHandle(handlerInput) {
        return handlerInput.requestEnvelope.request.type === 'LaunchRequest';
    },
    
    handle(handlerInput) {
        return ResponseBuilder.buildWelcomeResponse(handlerInput);
    }
};

module.exports = LaunchRequestHandler;
```

### handlers/energyHandlers.js
```javascript
const GoodWeAPIClient = require('../utils/apiClient');
const DataFormatter = require('../utils/dataFormatter');
const ResponseBuilder = require('../utils/responseBuilder');

const apiClient = new GoodWeAPIClient();

const GetEnergyConsumptionHandler = {
    canHandle(handlerInput) {
        return handlerInput.requestEnvelope.request.type === 'IntentRequest'
            && handlerInput.requestEnvelope.request.intent.name === 'GetEnergyConsumptionIntent';
    },
    
    async handle(handlerInput) {
        try {
            const startTime = Date.now();
            
            // Extrair slot
            const slots = handlerInput.requestEnvelope.request.intent.slots;
            const timeFrameSlot = slots.timeFrame;
            const timeFrameRaw = DataFormatter.extractSlotValue(timeFrameSlot);
            
            // Normalizar período
            const timeFrame = DataFormatter.normalizeTimeFrame(timeFrameRaw);
            console.log(`Getting consumption for period: ${timeFrame}`);
            
            // Buscar dados na API
            const consumptionData = await apiClient.getEnergyConsumption(timeFrame);
            
            if (consumptionData && consumptionData.data) {
                const data = consumptionData.data;
                const consumption = data.total_consumption || 0;
                const unit = data.unit || 'kWh';
                
                // Formatar resposta
                const formattedPeriod = DataFormatter.formatTimeFrame(timeFrame);
                const formattedConsumption = DataFormatter.formatEnergyValue(consumption, unit);
                
                let speechText = `O consumo de energia ${formattedPeriod} foi de ${formattedConsumption}.`;
                
                // Adicionar comparação se disponível
                if (data.comparison) {
                    const percentage = Math.abs(data.comparison.percentage);
                    const formattedPercentage = DataFormatter.formatPercentage(percentage);
                    const trend = data.comparison.percentage > 0 ? "maior" : "menor";
                    speechText += ` Isso é ${formattedPercentage} ${trend} que o período anterior.`;
                }
                
                // Adicionar custo se disponível
                if (data.cost) {
                    const formattedCost = DataFormatter.formatCurrency(data.cost);
                    speechText += ` O custo estimado é de ${formattedCost}.`;
                }
                
                const cardTitle = "Consumo de Energia";
                const cardContent = `Período: ${formattedPeriod}\nConsumo: ${formattedConsumption}`;
                
                const duration = Date.now() - startTime;
                console.log(`Consumption request completed in ${duration}ms`);
                
                return ResponseBuilder.buildResponse(handlerInput, speechText, cardTitle, cardContent);
            } else {
                return ResponseBuilder.buildErrorResponse(handlerInput, 'no_data');
            }
            
        } catch (error) {
            console.error('Error in consumption handler:', error);
            return ResponseBuilder.buildErrorResponse(handlerInput, 'api_error');
        }
    }
};

const GetEnergyProductionHandler = {
    canHandle(handlerInput) {
        return handlerInput.requestEnvelope.request.type === 'IntentRequest'
            && handlerInput.requestEnvelope.request.intent.name === 'GetEnergyProductionIntent';
    },
    
    async handle(handlerInput) {
        try {
            // Extrair slot
            const slots = handlerInput.requestEnvelope.request.intent.slots;
            const timeFrameSlot = slots.timeFrame;
            const timeFrameRaw = DataFormatter.extractSlotValue(timeFrameSlot);
            
            // Normalizar período
            const timeFrame = DataFormatter.normalizeTimeFrame(timeFrameRaw);
            
            // Buscar dados na API
            const productionData = await apiClient.getEnergyProduction(timeFrame);
            
            if (productionData && productionData.data) {
                const data = productionData.data;
                const production = data.total_production || 0;
                const unit = data.unit || 'kWh';
                
                // Formatar resposta
                const formattedPeriod = DataFormatter.formatTimeFrame(timeFrame);
                const formattedProduction = DataFormatter.formatEnergyValue(production, unit);
                
                let speechText = `A produção de energia ${formattedPeriod} foi de ${formattedProduction}.`;
                
                // Adicionar eficiência se disponível
                if (data.efficiency) {
                    const efficiency = DataFormatter.formatPercentage(data.efficiency);
                    speechText += ` A eficiência dos painéis foi de ${efficiency}.`;
                }
                
                // Adicionar comparação com consumo se disponível
                if (data.consumption_comparison) {
                    const ratio = DataFormatter.formatPercentage(data.consumption_comparison);
                    speechText += ` Isso representa ${ratio} do seu consumo.`;
                }
                
                const cardTitle = "Produção de Energia";
                const cardContent = `Período: ${formattedPeriod}\nProdução: ${formattedProduction}`;
                
                return ResponseBuilder.buildResponse(handlerInput, speechText, cardTitle, cardContent);
            } else {
                return ResponseBuilder.buildErrorResponse(handlerInput, 'no_data');
            }
            
        } catch (error) {
            console.error('Error in production handler:', error);
            return ResponseBuilder.buildErrorResponse(handlerInput, 'api_error');
        }
    }
};

const GetSavingsHandler = {
    canHandle(handlerInput) {
        return handlerInput.requestEnvelope.request.type === 'IntentRequest'
            && handlerInput.requestEnvelope.request.intent.name === 'GetSavingsIntent';
    },
    
    async handle(handlerInput) {
        try {
            // Extrair slot
            const slots = handlerInput.requestEnvelope.request.intent.slots;
            const timeFrameSlot = slots.timeFrame;
            const timeFrameRaw = DataFormatter.extractSlotValue(timeFrameSlot);
            
            // Normalizar período
            const timeFrame = DataFormatter.normalizeTimeFrame(timeFrameRaw);
            
            // Buscar dados na API
            const savingsData = await apiClient.getSavings(timeFrame);
            
            if (savingsData && savingsData.data) {
                const data = savingsData.data;
                const savingsAmount = data.total_savings || 0;
                const currency = data.currency || 'R$';
                
                // Formatar resposta
                const formattedPeriod = DataFormatter.formatTimeFrame(timeFrame);
                const formattedSavings = DataFormatter.formatCurrency(savingsAmount, currency);
                
                let speechText = `Você economizou ${formattedSavings} ${formattedPeriod}.`;
                
                // Adicionar detalhes se disponível
                if (data.energy_offset) {
                    const energyOffset = DataFormatter.formatPercentage(data.energy_offset);
                    speechText += ` Isso representa ${energyOffset} da sua conta de energia.`;
                }
                
                // Adicionar economia acumulada se disponível
                if (data.total_accumulated) {
                    const accumulated = DataFormatter.formatCurrency(data.total_accumulated, currency);
                    speechText += ` Sua economia total acumulada é de ${accumulated}.`;
                }
                
                const cardTitle = "Economias";
                const cardContent = `Período: ${formattedPeriod}\nEconomia: ${formattedSavings}`;
                
                return ResponseBuilder.buildResponse(handlerInput, speechText, cardTitle, cardContent);
            } else {
                return ResponseBuilder.buildErrorResponse(handlerInput, 'no_data');
            }
            
        } catch (error) {
            console.error('Error in savings handler:', error);
            return ResponseBuilder.buildErrorResponse(handlerInput, 'api_error');
        }
    }
};

module.exports = {
    GetEnergyConsumptionHandler,
    GetEnergyProductionHandler,
    GetSavingsHandler
};
```

### handlers/deviceHandlers.js
```javascript
const GoodWeAPIClient = require('../utils/apiClient');
const DataFormatter = require('../utils/dataFormatter');
const ResponseBuilder = require('../utils/responseBuilder');

const apiClient = new GoodWeAPIClient();

const GetDeviceStatusHandler = {
    canHandle(handlerInput) {
        return handlerInput.requestEnvelope.request.type === 'IntentRequest'
            && handlerInput.requestEnvelope.request.intent.name === 'GetDeviceStatusIntent';
    },
    
    async handle(handlerInput) {
        try {
            // Extrair slot
            const slots = handlerInput.requestEnvelope.request.intent.slots;
            const deviceSlot = slots.deviceType;
            const deviceType = DataFormatter.extractSlotValue(deviceSlot) || "sistema";
            
            console.log(`Getting status for device: ${deviceType}`);
            
            // Buscar status na API
            const statusData = await apiClient.getDeviceStatus(deviceType);
            
            if (statusData && statusData.data) {
                const data = statusData.data;
                const status = data.status || 'desconhecido';
                
                let speechText;
                let cardContent;
                
                // Formatar resposta baseada no tipo de dispositivo
                switch (deviceType.toLowerCase()) {
                    case "inversor":
                        const power = data.current_power || 0;
                        const efficiency = data.efficiency || 0;
                        const temperature = data.temperature;
                        
                        speechText = `O inversor está ${status}. `;
                        speechText += `Potência atual: ${DataFormatter.formatPower(power)}, `;
                        speechText += `eficiência: ${DataFormatter.formatPercentage(efficiency)}.`;
                        
                        if (temperature) {
                            speechText += ` Temperatura: ${temperature}°C.`;
                        }
                        
                        cardContent = `Status: ${status}\nPotência: ${DataFormatter.formatPower(power)}\nEficiência: ${DataFormatter.formatPercentage(efficiency)}`;
                        break;
                        
                    case "painel solar":
                        const generation = data.current_generation || 0;
                        const weatherFactor = data.weather_factor || 0;
                        const irradiance = data.irradiance;
                        
                        speechText = `Os painéis solares estão ${status}. `;
                        speechText += `Geração atual: ${DataFormatter.formatPower(generation)}. `;
                        speechText += `Fator climático: ${DataFormatter.formatPercentage(weatherFactor)}.`;
                        
                        if (irradiance) {
                            speechText += ` Irradiância solar: ${irradiance} W/m².`;
                        }
                        
                        cardContent = `Status: ${status}\nGeração: ${DataFormatter.formatPower(generation)}\nFator Climático: ${DataFormatter.formatPercentage(weatherFactor)}`;
                        break;
                        
                    case "bateria":
                        const chargeLevel = data.charge_level || 0;
                        const chargeStatus = data.charge_status || 'parado';
                        const capacity = data.capacity;
                        const voltage = data.voltage;
                        
                        speechText = `A bateria está ${status} com ${DataFormatter.formatPercentage(chargeLevel)} de carga. `;
                        speechText += `Status de carregamento: ${chargeStatus}.`;
                        
                        if (capacity) {
                            speechText += ` Capacidade: ${DataFormatter.formatEnergyValue(capacity)}.`;
                        }
                        
                        cardContent = `Status: ${status}\nCarga: ${DataFormatter.formatPercentage(chargeLevel)}\nCarregamento: ${chargeStatus}`;
                        break;
                        
                    default: // sistema completo
                        speechText = `O sistema está ${status}.`;
                        
                        if (data.summary) {
                            const summary = data.summary;
                            const systemGeneration = summary.generation || 0;
                            const systemConsumption = summary.consumption || 0;
                            const batteryLevel = summary.battery_level;
                            
                            speechText += ` Geração atual: ${DataFormatter.formatPower(systemGeneration)}, `;
                            speechText += `consumo: ${DataFormatter.formatPower(systemConsumption)}.`;
                            
                            if (batteryLevel !== undefined) {
                                speechText += ` Bateria: ${DataFormatter.formatPercentage(batteryLevel)}.`;
                            }
                            
                            cardContent = `Status: ${status}\nGeração: ${DataFormatter.formatPower(systemGeneration)}\nConsumo: ${DataFormatter.formatPower(systemConsumption)}`;
                        } else {
                            cardContent = `Status do Sistema: ${status}`;
                        }
                        break;
                }
                
                const cardTitle = `Status - ${deviceType.charAt(0).toUpperCase() + deviceType.slice(1)}`;
                
                return ResponseBuilder.buildResponse(handlerInput, speechText, cardTitle, cardContent);
            } else {
                return ResponseBuilder.buildErrorResponse(handlerInput, 'no_data');
            }
            
        } catch (error) {
            console.error('Error in device status handler:', error);
            return ResponseBuilder.buildErrorResponse(handlerInput, 'api_error');
        }
    }
};

const GetWeatherImpactHandler = {
    canHandle(handlerInput) {
        return handlerInput.requestEnvelope.request.type === 'IntentRequest'
            && handlerInput.requestEnvelope.request.intent.name === 'GetWeatherImpactIntent';
    },
    
    async handle(handlerInput) {
        try {
            // Buscar dados na API
            const weatherData = await apiClient.getWeatherImpact();
            
            if (weatherData && weatherData.data) {
                const data = weatherData.data;
                const weatherCondition = data.condition || 'desconhecida';
                const impactPercentage = data.impact_percentage || 0;
                const cloudCover = data.cloud_cover;
                const temperature = data.temperature;
                
                const formattedImpact = DataFormatter.formatPercentage(impactPercentage);
                
                let speechText = `As condições climáticas estão ${weatherCondition}. `;
                speechText += `O impacto na geração de energia é de ${formattedImpact}.`;
                
                // Adicionar detalhes climáticos
                if (cloudCover !== undefined) {
                    speechText += ` Cobertura de nuvens: ${DataFormatter.formatPercentage(cloudCover)}.`;
                }
                
                if (temperature) {
                    speechText += ` Temperatura ambiente: ${temperature}°C.`;
                }
                
                // Adicionar previsão se disponível
                if (data.forecast) {
                    speechText += ` Para as próximas horas, espera-se ${data.forecast}.`;
                }
                
                const cardTitle = "Impacto Climático";
                const cardContent = `Condições: ${weatherCondition}\nImpacto: ${formattedImpact}`;
                
                return ResponseBuilder.buildResponse(handlerInput, speechText, cardTitle, cardContent);
            } else {
                return ResponseBuilder.buildErrorResponse(handlerInput, 'no_data');
            }
            
        } catch (error) {
            console.error('Error in weather impact handler:', error);
            return ResponseBuilder.buildErrorResponse(handlerInput, 'api_error');
        }
    }
};

module.exports = {
    GetDeviceStatusHandler,
    GetWeatherImpactHandler
};
```

### handlers/builtInHandlers.js
```javascript
const ResponseBuilder = require('../utils/responseBuilder');

const HelpIntentHandler = {
    canHandle(handlerInput) {
        return handlerInput.requestEnvelope.request.type === 'IntentRequest'
            && handlerInput.requestEnvelope.request.intent.name === 'AMAZON.HelpIntent';
    },
    
    handle(handlerInput) {
        return ResponseBuilder.buildHelpResponse(handlerInput);
    }
};

const CancelAndStopIntentHandler = {
    canHandle(handlerInput) {
        return handlerInput.requestEnvelope.request.type === 'IntentRequest'
            && (handlerInput.requestEnvelope.request.intent.name === 'AMAZON.CancelIntent'
                || handlerInput.requestEnvelope.request.intent.name === 'AMAZON.StopIntent');
    },
    
    handle(handlerInput) {
        const farewell = [
            'Até logo! Volte sempre que quiser monitorar sua energia solar.',
            'Tchau! Continue aproveitando sua energia solar limpa.',
            'Até mais! Seu sistema de energia solar estará aqui quando você voltar.'
        ];
        
        const speechText = farewell[Math.floor(Math.random() * farewell.length)];
        return ResponseBuilder.buildResponse(handlerInput, speechText);
    }
};

const SessionEndedRequestHandler = {
    canHandle(handlerInput) {
        return handlerInput.requestEnvelope.request.type === 'SessionEndedRequest';
    },
    
    handle(handlerInput) {
        const reason = handlerInput.requestEnvelope.request.reason;
        console.log(`Session ended. Reason: ${reason}`);
        
        if (reason === 'ERROR') {
            const error = handlerInput.requestEnvelope.request.error;
            console.error(`Session ended with error: ${error.type} - ${error.message}`);
        }
        
        return handlerInput.responseBuilder.getResponse();
    }
};

const FallbackIntentHandler = {
    canHandle(handlerInput) {
        return handlerInput.requestEnvelope.request.type === 'IntentRequest'
            && handlerInput.requestEnvelope.request.intent.name === 'AMAZON.FallbackIntent';
    },
    
    handle(handlerInput) {
        const speechText = `
            Desculpe, não entendi o que você disse. 
            Tente perguntar sobre consumo de energia, produção, status dos dispositivos ou economias.
            Por exemplo: "Qual o consumo de energia hoje?" ou "Como está o inversor?"
        `;
        
        return ResponseBuilder.buildResponse(handlerInput, speechText, null, null, false);
    }
};

const ErrorHandler = {
    canHandle() {
        return true;
    },
    
    handle(handlerInput, error) {
        console.error('Global error handler triggered:', {
            error: error.message,
            stack: error.stack,
            request: handlerInput.requestEnvelope.request
        });
        
        const speechText = 'Desculpe, ocorreu um erro inesperado. Tente novamente mais tarde.';
        return ResponseBuilder.buildResponse(handlerInput, speechText);
    }
};

module.exports = {
    HelpIntentHandler,
    CancelAndStopIntentHandler,
    SessionEndedRequestHandler,
    FallbackIntentHandler,
    ErrorHandler
};
```

## 4. Interceptors

### interceptors/requestInterceptor.js
```javascript
const RequestInterceptor = {
    process(handlerInput) {
        const requestEnvelope = handlerInput.requestEnvelope;
        const sessionId = requestEnvelope.session?.sessionId;
        const userId = requestEnvelope.session?.user?.userId;
        const requestType = requestEnvelope.request.type;
        const requestId = requestEnvelope.request.requestId;
        
        console.log('=== REQUEST START ===');
        console.log(`Request ID: ${requestId}`);
        console.log(`Session ID: ${sessionId}`);
        console.log(`User ID: ${userId ? userId.substring(0, 20) + '...' : 'N/A'}`);
        console.log(`Request Type: ${requestType}`);
        console.log(`Locale: ${requestEnvelope.request.locale}`);
        
        // Log de intents e slots
        if (requestType === 'IntentRequest') {
            const intent = requestEnvelope.request.intent;
            console.log(`Intent: ${intent.name}`);
            console.log(`Confirmation Status: ${intent.confirmationStatus}`);
            
            if (intent.slots) {
                const slots = Object.keys(intent.slots).reduce((acc, key) => {
                    const slot = intent.slots[key];
                    acc[key] = {
                        value: slot.value,
                        confirmationStatus: slot.confirmationStatus,
                        source: slot.source
                    };
                    return acc;
                }, {});
                
                console.log('Slots:', JSON.stringify(slots, null, 2));
            }
        }
        
        console.log('=== REQUEST END ===');
    }
};

module.exports = RequestInterceptor;
```

### interceptors/responseInterceptor.js
```javascript
const ResponseInterceptor = {
    process(handlerInput, response) {
        const requestId = handlerInput.requestEnvelope.request.requestId;
        
        console.log('=== RESPONSE START ===');
        console.log(`Request ID: ${requestId}`);
        
        // Log de métricas básicas
        if (response.outputSpeech) {
            const outputSpeech = response.outputSpeech;
            console.log(`Speech Type: ${outputSpeech.type}`);
            
            if (outputSpeech.ssml) {
                const speechLength = outputSpeech.ssml.length;
                console.log(`Speech Length: ${speechLength} characters`);
            }
        }
        
        if (response.card) {
            console.log(`Card Type: ${response.card.type}`);
            console.log(`Card Title: ${response.card.title}`);
        }
        
        if (response.reprompt) {
            console.log('Has Reprompt: true');
        }
        
        console.log(`Should End Session: ${response.shouldEndSession}`);
        console.log('=== RESPONSE END ===');
    }
};

module.exports = ResponseInterceptor;
```

## 5. Arquivo Principal

### index.js
```javascript
const Alexa = require('ask-sdk-core');

// Importar handlers
const LaunchRequestHandler = require('./handlers/launchRequestHandler');
const { 
    GetEnergyConsumptionHandler, 
    GetEnergyProductionHandler, 
    GetSavingsHandler 
} = require('./handlers/energyHandlers');
const { 
    GetDeviceStatusHandler, 
    GetWeatherImpactHandler 
} = require('./handlers/deviceHandlers');
const { 
    HelpIntentHandler, 
    CancelAndStopIntentHandler, 
    SessionEndedRequestHandler,
    FallbackIntentHandler,
    ErrorHandler 
} = require('./handlers/builtInHandlers');

// Importar interceptors
const RequestInterceptor = require('./interceptors/requestInterceptor');
const ResponseInterceptor = require('./interceptors/responseInterceptor');

// Configurar e exportar skill
const skillBuilder = Alexa.SkillBuilders.custom()
    .addRequestHandlers(
        LaunchRequestHandler,
        GetEnergyConsumptionHandler,
        GetEnergyProductionHandler,
        GetSavingsHandler,
        GetDeviceStatusHandler,
        GetWeatherImpactHandler,
        HelpIntentHandler,
        CancelAndStopIntentHandler,
        FallbackIntentHandler,
        SessionEndedRequestHandler
    )
    .addErrorHandlers(ErrorHandler)
    .addRequestInterceptors(RequestInterceptor)
    .addResponseInterceptors(ResponseInterceptor)
    .withCustomUserAgent('alexa-energy-skill/1.0.0');

// Handler principal para AWS Lambda
exports.handler = skillBuilder.lambda();

// Para desenvolvimento local
if (require.main === module) {
    const express = require('express');
    const { ExpressAdapter } = require('ask-sdk-express-adapter');
    
    const app = express();
    const adapter = new ExpressAdapter(skillBuilder, true, true);
    
    app.post('/', adapter.getRequestHandlers());
    
    const port = process.env.PORT || 3000;
    app.listen(port, () => {
        console.log(`Alexa Skill server running on port ${port}`);
    });
}
```

---

*Este documento contém toda a implementação Node.js necessária para a Alexa Skill de energia solar GoodWe.*
