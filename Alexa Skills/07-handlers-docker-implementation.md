# Implementação dos Handlers para Docker

## Handlers Específicos para Infraestrutura Docker

Este arquivo complementa a implementação Docker com handlers otimizados para esse ambiente.

## 1. Handlers de Energia

### app/handlers/energyHandlers.js
```javascript
const GoodWeApiService = require('../services/goodweApi');
const AlexaResponseService = require('../services/alexaResponse');
const DataFormatter = require('../utils/dataFormatter');
const metrics = require('../middleware/metrics');

const GetEnergyConsumptionHandler = {
    canHandle(handlerInput) {
        return handlerInput.requestEnvelope.request.type === 'IntentRequest'
            && handlerInput.requestEnvelope.request.intent.name === 'GetEnergyConsumptionIntent';
    },
    
    async handle(handlerInput) {
        const startTime = Date.now();
        
        try {
            // Incrementar métrica
            await metrics.incrementCounter('energy_consumption_requests');
            
            // Extrair slots
            const slots = handlerInput.requestEnvelope.request.intent.slots;
            const timeFrame = DataFormatter.extractSlotValue(slots.timeFrame) || 'hoje';
            
            console.log(`🔋 Getting consumption for: ${timeFrame}`);
            
            // Buscar dados da API
            const apiTimeFrame = DataFormatter.normalizeTimeFrame(timeFrame);
            const consumptionData = await GoodWeApiService.getEnergyConsumption(apiTimeFrame);
            
            if (!consumptionData || !consumptionData.data) {
                await metrics.incrementCounter('api_errors');
                return AlexaResponseService.buildErrorResponse(handlerInput, 'no_data');
            }
            
            const data = consumptionData.data;
            const consumption = data.total_consumption || 0;
            const unit = data.unit || 'kWh';
            
            // Formatar resposta
            const formattedPeriod = DataFormatter.formatTimeFrame(apiTimeFrame);
            const formattedConsumption = DataFormatter.formatEnergyValue(consumption, unit);
            
            let speechText = `O consumo de energia ${formattedPeriod} foi de ${formattedConsumption}.`;
            
            // Adicionar contexto se disponível
            if (data.comparison) {
                const percentage = Math.abs(data.comparison.percentage);
                const trend = data.comparison.percentage > 0 ? "maior" : "menor";
                const formattedPercentage = DataFormatter.formatPercentage(percentage);
                speechText += ` Isso é ${formattedPercentage} ${trend} que o período anterior.`;
            }
            
            if (data.cost) {
                const formattedCost = DataFormatter.formatCurrency(data.cost);
                speechText += ` O custo estimado é de ${formattedCost}.`;
            }
            
            // Log de performance
            const duration = Date.now() - startTime;
            console.log(`⚡ Consumption request completed in ${duration}ms`);
            
            return AlexaResponseService.buildResponse(
                handlerInput, 
                speechText,
                'Consumo de Energia',
                `Período: ${formattedPeriod}\nConsumo: ${formattedConsumption}`
            );
            
        } catch (error) {
            await metrics.incrementCounter('handler_errors');
            console.error('Error in consumption handler:', error);
            return AlexaResponseService.buildErrorResponse(handlerInput, 'api_error');
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
            await metrics.incrementCounter('energy_production_requests');
            
            const slots = handlerInput.requestEnvelope.request.intent.slots;
            const timeFrame = DataFormatter.extractSlotValue(slots.timeFrame) || 'hoje';
            
            console.log(`☀️ Getting production for: ${timeFrame}`);
            
            const apiTimeFrame = DataFormatter.normalizeTimeFrame(timeFrame);
            const productionData = await GoodWeApiService.getEnergyProduction(apiTimeFrame);
            
            if (!productionData || !productionData.data) {
                await metrics.incrementCounter('api_errors');
                return AlexaResponseService.buildErrorResponse(handlerInput, 'no_data');
            }
            
            const data = productionData.data;
            const production = data.total_production || 0;
            const unit = data.unit || 'kWh';
            
            const formattedPeriod = DataFormatter.formatTimeFrame(apiTimeFrame);
            const formattedProduction = DataFormatter.formatEnergyValue(production, unit);
            
            let speechText = `A produção de energia ${formattedPeriod} foi de ${formattedProduction}.`;
            
            if (data.efficiency) {
                const efficiency = DataFormatter.formatPercentage(data.efficiency);
                speechText += ` A eficiência dos painéis foi de ${efficiency}.`;
            }
            
            if (data.weather_impact) {
                const impact = data.weather_impact;
                speechText += ` As condições climáticas estão ${impact.condition}.`;
            }
            
            return AlexaResponseService.buildResponse(
                handlerInput, 
                speechText,
                'Produção de Energia',
                `Período: ${formattedPeriod}\nProdução: ${formattedProduction}`
            );
            
        } catch (error) {
            await metrics.incrementCounter('handler_errors');
            console.error('Error in production handler:', error);
            return AlexaResponseService.buildErrorResponse(handlerInput, 'api_error');
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
            await metrics.incrementCounter('savings_requests');
            
            const slots = handlerInput.requestEnvelope.request.intent.slots;
            const timeFrame = DataFormatter.extractSlotValue(slots.timeFrame) || 'hoje';
            
            console.log(`💰 Getting savings for: ${timeFrame}`);
            
            const apiTimeFrame = DataFormatter.normalizeTimeFrame(timeFrame);
            const savingsData = await GoodWeApiService.getSavings(apiTimeFrame);
            
            if (!savingsData || !savingsData.data) {
                await metrics.incrementCounter('api_errors');
                return AlexaResponseService.buildErrorResponse(handlerInput, 'no_data');
            }
            
            const data = savingsData.data;
            const savingsAmount = data.total_savings || 0;
            const currency = data.currency || 'R$';
            
            const formattedPeriod = DataFormatter.formatTimeFrame(apiTimeFrame);
            const formattedSavings = DataFormatter.formatCurrency(savingsAmount, currency);
            
            let speechText = `Você economizou ${formattedSavings} ${formattedPeriod}.`;
            
            if (data.energy_offset) {
                const energyOffset = DataFormatter.formatPercentage(data.energy_offset);
                speechText += ` Isso representa ${energyOffset} da sua conta de energia.`;
            }
            
            if (data.co2_saved) {
                const co2Saved = DataFormatter.formatWeight(data.co2_saved);
                speechText += ` Você também evitou a emissão de ${co2Saved} de CO2.`;
            }
            
            return AlexaResponseService.buildResponse(
                handlerInput, 
                speechText,
                'Economias',
                `Período: ${formattedPeriod}\nEconomia: ${formattedSavings}`
            );
            
        } catch (error) {
            await metrics.incrementCounter('handler_errors');
            console.error('Error in savings handler:', error);
            return AlexaResponseService.buildErrorResponse(handlerInput, 'api_error');
        }
    }
};

module.exports = {
    GetEnergyConsumptionHandler,
    GetEnergyProductionHandler,
    GetSavingsHandler
};
```

## 2. Handlers de Dispositivos

### app/handlers/deviceHandlers.js
```javascript
const GoodWeApiService = require('../services/goodweApi');
const AlexaResponseService = require('../services/alexaResponse');
const DataFormatter = require('../utils/dataFormatter');
const metrics = require('../middleware/metrics');

const GetDeviceStatusHandler = {
    canHandle(handlerInput) {
        return handlerInput.requestEnvelope.request.type === 'IntentRequest'
            && handlerInput.requestEnvelope.request.intent.name === 'GetDeviceStatusIntent';
    },
    
    async handle(handlerInput) {
        try {
            await metrics.incrementCounter('device_status_requests');
            
            const slots = handlerInput.requestEnvelope.request.intent.slots;
            const deviceType = DataFormatter.extractSlotValue(slots.deviceType) || 'sistema';
            
            console.log(`🔧 Getting status for device: ${deviceType}`);
            
            const statusData = await GoodWeApiService.getDeviceStatus(deviceType);
            
            if (!statusData || !statusData.data) {
                await metrics.incrementCounter('api_errors');
                return AlexaResponseService.buildErrorResponse(handlerInput, 'no_data');
            }
            
            const data = statusData.data;
            const status = data.status || 'desconhecido';
            
            let speechText;
            let cardContent;
            
            switch (deviceType.toLowerCase()) {
                case 'inversor':
                    speechText = this.buildInverterResponse(data, status);
                    cardContent = this.buildInverterCard(data, status);
                    break;
                    
                case 'painel solar':
                case 'painéis solares':
                    speechText = this.buildSolarPanelResponse(data, status);
                    cardContent = this.buildSolarPanelCard(data, status);
                    break;
                    
                case 'bateria':
                    speechText = this.buildBatteryResponse(data, status);
                    cardContent = this.buildBatteryCard(data, status);
                    break;
                    
                default:
                    speechText = this.buildSystemResponse(data, status);
                    cardContent = this.buildSystemCard(data, status);
                    break;
            }
            
            const cardTitle = `Status - ${deviceType.charAt(0).toUpperCase() + deviceType.slice(1)}`;
            
            return AlexaResponseService.buildResponse(
                handlerInput, 
                speechText,
                cardTitle,
                cardContent
            );
            
        } catch (error) {
            await metrics.incrementCounter('handler_errors');
            console.error('Error in device status handler:', error);
            return AlexaResponseService.buildErrorResponse(handlerInput, 'api_error');
        }
    },
    
    buildInverterResponse(data, status) {
        const power = data.current_power || 0;
        const efficiency = data.efficiency || 0;
        const temperature = data.temperature;
        
        let response = `O inversor está ${status}. `;
        response += `Potência atual: ${DataFormatter.formatPower(power)}, `;
        response += `eficiência: ${DataFormatter.formatPercentage(efficiency)}.`;
        
        if (temperature) {
            response += ` Temperatura: ${temperature}°C.`;
        }
        
        if (data.daily_energy) {
            const dailyEnergy = DataFormatter.formatEnergyValue(data.daily_energy);
            response += ` Energia gerada hoje: ${dailyEnergy}.`;
        }
        
        return response;
    },
    
    buildSolarPanelResponse(data, status) {
        const generation = data.current_generation || 0;
        const weatherFactor = data.weather_factor || 0;
        const irradiance = data.irradiance;
        
        let response = `Os painéis solares estão ${status}. `;
        response += `Geração atual: ${DataFormatter.formatPower(generation)}. `;
        response += `Fator climático: ${DataFormatter.formatPercentage(weatherFactor)}.`;
        
        if (irradiance) {
            response += ` Irradiância solar: ${irradiance} W/m².`;
        }
        
        return response;
    },
    
    buildBatteryResponse(data, status) {
        const chargeLevel = data.charge_level || 0;
        const chargeStatus = data.charge_status || 'parado';
        const capacity = data.capacity;
        const power = data.power;
        
        let response = `A bateria está ${status} com ${DataFormatter.formatPercentage(chargeLevel)} de carga. `;
        response += `Status de carregamento: ${chargeStatus}.`;
        
        if (capacity) {
            response += ` Capacidade total: ${DataFormatter.formatEnergyValue(capacity)}.`;
        }
        
        if (power) {
            const powerDirection = power > 0 ? 'carregando' : 'descarregando';
            response += ` Atualmente ${powerDirection} a ${DataFormatter.formatPower(Math.abs(power))}.`;
        }
        
        return response;
    },
    
    buildSystemResponse(data, status) {
        let response = `O sistema está ${status}.`;
        
        if (data.summary) {
            const summary = data.summary;
            const generation = summary.generation || 0;
            const consumption = summary.consumption || 0;
            const batteryLevel = summary.battery_level;
            
            response += ` Geração atual: ${DataFormatter.formatPower(generation)}, `;
            response += `consumo: ${DataFormatter.formatPower(consumption)}.`;
            
            if (batteryLevel !== undefined) {
                response += ` Bateria: ${DataFormatter.formatPercentage(batteryLevel)}.`;
            }
            
            // Indicar se há excesso ou déficit
            const balance = generation - consumption;
            if (Math.abs(balance) > 100) { // Apenas se diferença for significativa
                if (balance > 0) {
                    response += ` Você está gerando ${DataFormatter.formatPower(balance)} a mais do que consome.`;
                } else {
                    response += ` Você está consumindo ${DataFormatter.formatPower(Math.abs(balance))} a mais do que gera.`;
                }
            }
        }
        
        return response;
    },
    
    buildInverterCard(data, status) {
        const lines = [`Status: ${status}`];
        
        if (data.current_power) {
            lines.push(`Potência: ${DataFormatter.formatPower(data.current_power)}`);
        }
        
        if (data.efficiency) {
            lines.push(`Eficiência: ${DataFormatter.formatPercentage(data.efficiency)}`);
        }
        
        if (data.temperature) {
            lines.push(`Temperatura: ${data.temperature}°C`);
        }
        
        return lines.join('\n');
    },
    
    buildSolarPanelCard(data, status) {
        const lines = [`Status: ${status}`];
        
        if (data.current_generation) {
            lines.push(`Geração: ${DataFormatter.formatPower(data.current_generation)}`);
        }
        
        if (data.weather_factor) {
            lines.push(`Fator Climático: ${DataFormatter.formatPercentage(data.weather_factor)}`);
        }
        
        if (data.irradiance) {
            lines.push(`Irradiância: ${data.irradiance} W/m²`);
        }
        
        return lines.join('\n');
    },
    
    buildBatteryCard(data, status) {
        const lines = [`Status: ${status}`];
        
        if (data.charge_level) {
            lines.push(`Carga: ${DataFormatter.formatPercentage(data.charge_level)}`);
        }
        
        if (data.charge_status) {
            lines.push(`Carregamento: ${data.charge_status}`);
        }
        
        if (data.capacity) {
            lines.push(`Capacidade: ${DataFormatter.formatEnergyValue(data.capacity)}`);
        }
        
        return lines.join('\n');
    },
    
    buildSystemCard(data, status) {
        const lines = [`Status: ${status}`];
        
        if (data.summary) {
            const summary = data.summary;
            
            if (summary.generation) {
                lines.push(`Geração: ${DataFormatter.formatPower(summary.generation)}`);
            }
            
            if (summary.consumption) {
                lines.push(`Consumo: ${DataFormatter.formatPower(summary.consumption)}`);
            }
            
            if (summary.battery_level !== undefined) {
                lines.push(`Bateria: ${DataFormatter.formatPercentage(summary.battery_level)}`);
            }
        }
        
        return lines.join('\n');
    }
};

const GetWeatherImpactHandler = {
    canHandle(handlerInput) {
        return handlerInput.requestEnvelope.request.type === 'IntentRequest'
            && handlerInput.requestEnvelope.request.intent.name === 'GetWeatherImpactIntent';
    },
    
    async handle(handlerInput) {
        try {
            await metrics.incrementCounter('weather_requests');
            
            console.log('🌤️ Getting weather impact...');
            
            const weatherData = await GoodWeApiService.getWeatherImpact();
            
            if (!weatherData || !weatherData.data) {
                await metrics.incrementCounter('api_errors');
                return AlexaResponseService.buildErrorResponse(handlerInput, 'no_data');
            }
            
            const data = weatherData.data;
            const condition = data.condition || 'desconhecida';
            const impactPercentage = data.impact_percentage || 0;
            
            let speechText = `As condições climáticas estão ${condition}. `;
            speechText += `O impacto na geração de energia é de ${DataFormatter.formatPercentage(impactPercentage)}.`;
            
            // Detalhes climáticos
            if (data.cloud_cover !== undefined) {
                speechText += ` Cobertura de nuvens: ${DataFormatter.formatPercentage(data.cloud_cover)}.`;
            }
            
            if (data.temperature) {
                speechText += ` Temperatura ambiente: ${data.temperature}°C.`;
            }
            
            if (data.wind_speed) {
                speechText += ` Velocidade do vento: ${data.wind_speed} km/h.`;
            }
            
            // Previsão
            if (data.forecast) {
                speechText += ` Para as próximas horas, espera-se ${data.forecast}.`;
            }
            
            // Recomendações
            if (data.recommendations) {
                const rec = data.recommendations;
                if (rec.maintenance) {
                    speechText += ` Recomendação: ${rec.maintenance}`;
                }
            }
            
            return AlexaResponseService.buildResponse(
                handlerInput,
                speechText,
                'Impacto Climático',
                `Condições: ${condition}\nImpacto: ${DataFormatter.formatPercentage(impactPercentage)}`
            );
            
        } catch (error) {
            await metrics.incrementCounter('handler_errors');
            console.error('Error in weather impact handler:', error);
            return AlexaResponseService.buildErrorResponse(handlerInput, 'api_error');
        }
    }
};

module.exports = {
    GetDeviceStatusHandler,
    GetWeatherImpactHandler
};
```

## 3. Handlers Built-in

### app/handlers/builtInHandlers.js
```javascript
const AlexaResponseService = require('../services/alexaResponse');
const metrics = require('../middleware/metrics');
const { getTimeGreeting } = require('../utils/dataFormatter');

const LaunchRequestHandler = {
    canHandle(handlerInput) {
        return handlerInput.requestEnvelope.request.type === 'LaunchRequest';
    },
    
    async handle(handlerInput) {
        try {
            await metrics.incrementCounter('launch_requests');
            
            const greeting = getTimeGreeting();
            const speechText = `
                ${greeting}! Bem-vindo ao seu assistente de energia solar GoodWe. 
                Eu posso te ajudar a monitorar consumo, produção, status dos dispositivos e economias.
                O que você gostaria de saber?
            `;
            
            return AlexaResponseService.buildResponse(
                handlerInput, 
                speechText,
                'Assistente GoodWe',
                'Bem-vindo ao monitoramento de energia solar!',
                false
            );
            
        } catch (error) {
            console.error('Error in launch handler:', error);
            return AlexaResponseService.buildErrorResponse(handlerInput, 'general');
        }
    }
};

const HelpIntentHandler = {
    canHandle(handlerInput) {
        return handlerInput.requestEnvelope.request.type === 'IntentRequest'
            && handlerInput.requestEnvelope.request.intent.name === 'AMAZON.HelpIntent';
    },
    
    async handle(handlerInput) {
        try {
            await metrics.incrementCounter('help_requests');
            
            const speechText = `
                Eu posso te ajudar a monitorar seu sistema de energia solar. Aqui estão algumas opções:
                
                Para consumo, diga: "Qual o consumo de energia hoje?"
                Para produção, diga: "Quanta energia foi gerada esta semana?"
                Para dispositivos, diga: "Como está o inversor?"
                Para economias, diga: "Quanto economizei este mês?"
                Para clima, diga: "Como o tempo está afetando a geração?"
                
                Você também pode pedir informações sobre períodos específicos como ontem, esta semana, ou este mês.
                
                O que você gostaria de saber?
            `;
            
            return AlexaResponseService.buildResponse(
                handlerInput, 
                speechText,
                'Ajuda - Assistente GoodWe',
                'Use comandos de voz para monitorar sua energia solar',
                false
            );
            
        } catch (error) {
            console.error('Error in help handler:', error);
            return AlexaResponseService.buildErrorResponse(handlerInput, 'general');
        }
    }
};

const CancelAndStopIntentHandler = {
    canHandle(handlerInput) {
        return handlerInput.requestEnvelope.request.type === 'IntentRequest'
            && (handlerInput.requestEnvelope.request.intent.name === 'AMAZON.CancelIntent'
                || handlerInput.requestEnvelope.request.intent.name === 'AMAZON.StopIntent');
    },
    
    async handle(handlerInput) {
        try {
            await metrics.incrementCounter('stop_requests');
            
            const farewells = [
                'Até logo! Continue aproveitando sua energia solar limpa.',
                'Tchau! Volte sempre que quiser monitorar seu sistema.',
                'Até mais! Seu sistema de energia solar estará aqui quando você voltar.',
                'Obrigado por usar o assistente GoodWe. Até a próxima!'
            ];
            
            const speechText = farewells[Math.floor(Math.random() * farewells.length)];
            
            return AlexaResponseService.buildResponse(handlerInput, speechText);
            
        } catch (error) {
            console.error('Error in stop handler:', error);
            return AlexaResponseService.buildResponse(
                handlerInput, 
                'Até logo!'
            );
        }
    }
};

const SessionEndedRequestHandler = {
    canHandle(handlerInput) {
        return handlerInput.requestEnvelope.request.type === 'SessionEndedRequest';
    },
    
    async handle(handlerInput) {
        try {
            await metrics.incrementCounter('session_ended');
            
            const reason = handlerInput.requestEnvelope.request.reason;
            console.log(`📝 Session ended. Reason: ${reason}`);
            
            if (reason === 'ERROR') {
                const error = handlerInput.requestEnvelope.request.error;
                console.error(`🚨 Session ended with error: ${error.type} - ${error.message}`);
                await metrics.incrementCounter('session_errors');
            }
            
            return handlerInput.responseBuilder.getResponse();
            
        } catch (error) {
            console.error('Error in session ended handler:', error);
            return handlerInput.responseBuilder.getResponse();
        }
    }
};

const FallbackIntentHandler = {
    canHandle(handlerInput) {
        return handlerInput.requestEnvelope.request.type === 'IntentRequest'
            && handlerInput.requestEnvelope.request.intent.name === 'AMAZON.FallbackIntent';
    },
    
    async handle(handlerInput) {
        try {
            await metrics.incrementCounter('fallback_requests');
            
            const speechText = `
                Desculpe, não entendi o que você disse. 
                Tente perguntar sobre consumo de energia, produção, status dos dispositivos ou economias.
                
                Por exemplo:
                "Qual o consumo de energia hoje?"
                "Como está o inversor?"
                "Quanta energia foi gerada esta semana?"
                
                O que você gostaria de saber?
            `;
            
            return AlexaResponseService.buildResponse(
                handlerInput, 
                speechText,
                null,
                null,
                false
            );
            
        } catch (error) {
            console.error('Error in fallback handler:', error);
            return AlexaResponseService.buildErrorResponse(handlerInput, 'general');
        }
    }
};

const ErrorHandler = {
    canHandle() {
        return true;
    },
    
    async handle(handlerInput, error) {
        try {
            await metrics.incrementCounter('global_errors');
            
            console.error('🔥 Global error handler triggered:', {
                error: error.message,
                stack: error.stack,
                request: handlerInput.requestEnvelope.request,
                session: handlerInput.requestEnvelope.session?.sessionId
            });
            
            const speechText = 'Desculpe, ocorreu um erro inesperado. Tente novamente mais tarde.';
            
            return AlexaResponseService.buildResponse(handlerInput, speechText);
            
        } catch (handlerError) {
            console.error('Error in error handler:', handlerError);
            
            // Fallback mais básico possível
            return handlerInput.responseBuilder
                .speak('Desculpe, ocorreu um erro.')
                .getResponse();
        }
    }
};

module.exports = {
    LaunchRequestHandler,
    HelpIntentHandler,
    CancelAndStopIntentHandler,
    SessionEndedRequestHandler,
    FallbackIntentHandler,
    ErrorHandler
};
```

## 4. Serviços

### app/services/goodweApi.js
```javascript
const axios = require('axios');
const redis = require('redis');

class GoodWeApiService {
    constructor() {
        this.baseURL = process.env.GOODWE_API_URL || 'https://api.goodwe.com/v1';
        this.apiKey = process.env.GOODWE_API_KEY;
        this.timeout = 7000; // 7 segundos para deixar buffer para resposta da Alexa
        
        // Cache Redis
        this.redis = redis.createClient({
            url: process.env.REDIS_URL || 'redis://localhost:6379'
        });
        
        this.redis.connect().catch(console.error);
        
        this.client = axios.create({
            baseURL: this.baseURL,
            timeout: this.timeout,
            headers: {
                'Authorization': `Bearer ${this.apiKey}`,
                'Content-Type': 'application/json',
                'User-Agent': 'AlexaSkill-GoodWe/1.0.0'
            }
        });
    }
    
    async makeRequest(endpoint, params = {}, cacheTime = 300) {
        const cacheKey = `api:${endpoint}:${JSON.stringify(params)}`;
        
        try {
            // Tentar cache primeiro
            const cached = await this.redis.get(cacheKey);
            if (cached) {
                console.log(`📦 Cache hit for ${endpoint}`);
                return JSON.parse(cached);
            }
            
            console.log(`🌐 API request to ${endpoint}`);
            const response = await this.client.get(endpoint, { params });
            
            if (response.status === 200) {
                // Salvar no cache
                await this.redis.setEx(cacheKey, cacheTime, JSON.stringify(response.data));
                return response.data;
            }
            
            return null;
            
        } catch (error) {
            console.error(`❌ API Error for ${endpoint}:`, {
                status: error.response?.status,
                statusText: error.response?.statusText,
                message: error.message,
                data: error.response?.data
            });
            
            // Em caso de erro, tentar cache mais antigo
            const staleCache = await this.redis.get(`stale:${cacheKey}`);
            if (staleCache) {
                console.log(`🔄 Using stale cache for ${endpoint}`);
                return JSON.parse(staleCache);
            }
            
            return null;
        }
    }
    
    async getEnergyConsumption(timeFrame = 'today') {
        return await this.makeRequest('/energy/consumption', {
            period: timeFrame,
            metric: 'consumption'
        }, 180); // Cache por 3 minutos
    }
    
    async getEnergyProduction(timeFrame = 'today') {
        return await this.makeRequest('/energy/production', {
            period: timeFrame,
            metric: 'production'
        }, 180);
    }
    
    async getDeviceStatus(deviceType = 'system') {
        const deviceEndpoints = {
            'inversor': '/devices/inverter',
            'painel solar': '/devices/solar-panels',
            'bateria': '/devices/battery',
            'sistema': '/devices/system-status'
        };
        
        const endpoint = deviceEndpoints[deviceType] || '/devices/system-status';
        return await this.makeRequest(endpoint, {}, 60); // Cache por 1 minuto
    }
    
    async getSavings(timeFrame = 'today') {
        return await this.makeRequest('/financial/savings', {
            period: timeFrame
        }, 300); // Cache por 5 minutos
    }
    
    async getWeatherImpact() {
        return await this.makeRequest('/weather/impact', {}, 600); // Cache por 10 minutos
    }
}

module.exports = new GoodWeApiService();
```

### app/services/alexaResponse.js
```javascript
class AlexaResponseService {
    
    static buildResponse(handlerInput, speechText, cardTitle = null, cardContent = null, shouldEndSession = true) {
        const responseBuilder = handlerInput.responseBuilder.speak(speechText);
        
        if (cardTitle && cardContent) {
            responseBuilder.withSimpleCard(cardTitle, cardContent);
        }
        
        if (shouldEndSession) {
            responseBuilder.withShouldEndSession(true);
        } else {
            const reprompt = this.getRepromptText();
            responseBuilder.reprompt(reprompt);
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
        return this.buildResponse(handlerInput, speechText);
    }
    
    static getRepromptText() {
        const options = [
            'O que mais você gostaria de saber sobre seu sistema de energia solar?',
            'Posso ajudar com informações sobre consumo, produção, dispositivos ou economias. O que você precisa?',
            'Que outras informações sobre energia solar você gostaria de conhecer?',
            'Como posso te ajudar mais com o monitoramento da sua energia solar?'
        ];
        
        return options[Math.floor(Math.random() * options.length)];
    }
    
    static buildProgressiveResponse(handlerInput, speechText) {
        // Para respostas que podem demorar, enviar uma resposta progressiva
        return handlerInput.responseBuilder
            .speak('Um momento, estou consultando os dados...')
            .addDirective({
                type: 'VoicePlayer.Speak',
                speech: speechText
            })
            .getResponse();
    }
}

module.exports = AlexaResponseService;
```

## 5. Utils Expandidos

### app/utils/dataFormatter.js
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
    
    static formatPower(watts) {
        if (!watts || isNaN(watts)) return '0 W';
        
        if (watts >= 1000000) {
            return `${(watts / 1000000).toFixed(1)} MW`;
        } else if (watts >= 1000) {
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
    
    static formatWeight(value, unit = 'kg') {
        if (!value || isNaN(value)) return `0 ${unit}`;
        
        if (value >= 1000) {
            return `${(value / 1000).toFixed(1)} toneladas`;
        } else {
            return `${value.toFixed(1)} ${unit}`;
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
    
    static getTimeGreeting() {
        const hour = new Date().getHours();
        
        if (hour < 6) {
            return 'Boa madrugada';
        } else if (hour < 12) {
            return 'Bom dia';
        } else if (hour < 18) {
            return 'Boa tarde';
        } else if (hour < 22) {
            return 'Boa noite';
        } else {
            return 'Boa noite';
        }
    }
    
    static formatDateTime(dateTime) {
        if (!dateTime) return '';
        return moment(dateTime).format('DD/MM/YYYY HH:mm');
    }
    
    static formatDuration(seconds) {
        if (!seconds || isNaN(seconds)) return '0 segundos';
        
        const minutes = Math.floor(seconds / 60);
        const remainingSeconds = seconds % 60;
        
        if (minutes > 0) {
            return `${minutes} minuto${minutes > 1 ? 's' : ''} e ${remainingSeconds} segundo${remainingSeconds > 1 ? 's' : ''}`;
        } else {
            return `${seconds} segundo${seconds > 1 ? 's' : ''}`;
        }
    }
}

module.exports = DataFormatter;
```

---

*Este arquivo complementa a implementação Docker com handlers otimizados, cache Redis, métricas e tratamento robusto de erros.*
