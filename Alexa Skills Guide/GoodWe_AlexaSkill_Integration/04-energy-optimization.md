# 4. Otimização Energética Inteligente

## 🧠 Visão Geral do Sistema de Otimização

O sistema de otimização energética usa inteligência artificial e análise de dados para maximizar a eficiência do sistema solar GoodWe, reduzir custos e otimizar o uso de energia baseado em padrões de consumo, produção solar e tarifas elétricas.

## 📊 Análise Preditiva de Energia

### Motor de Previsão
```javascript
class EnergyForecastService {
    constructor(dataService, weatherService) {
        this.dataService = dataService;
        this.weatherService = weatherService;
        this.mlModel = new EnergyPredictionModel();
    }

    async generateProductionForecast(stationId, days = 7) {
        try {
            // Buscar dados históricos
            const historicalData = await this.dataService.getHistoricalData(
                stationId,
                this.getDateDaysAgo(90), // 3 meses de histórico
                new Date().toISOString().split('T')[0]
            );

            // Buscar previsão meteorológica
            const weatherForecast = await this.weatherService.getDetailedForecast(stationId, days);

            // Buscar dados do sistema
            const systemInfo = await this.dataService.getSystemInfo(stationId);

            // Gerar previsão usando ML
            const forecast = await this.mlModel.predictProduction({
                historical: historicalData,
                weather: weatherForecast,
                system: systemInfo,
                days: days
            });

            return this.enrichForecast(forecast, weatherForecast);

        } catch (error) {
            console.error('Error generating production forecast:', error);
            return this.fallbackForecast(stationId, days);
        }
    }

    async generateConsumptionForecast(stationId, days = 7) {
        try {
            const historicalConsumption = await this.dataService.getConsumptionHistory(stationId, 30);
            const appliances = await this.dataService.getApplianceUsage(stationId);
            
            // Análise de padrões semanais e sazonais
            const patterns = this.analyzeConsumptionPatterns(historicalConsumption);
            
            const forecast = this.mlModel.predictConsumption({
                historical: historicalConsumption,
                patterns: patterns,
                appliances: appliances,
                days: days
            });

            return forecast;

        } catch (error) {
            console.error('Error generating consumption forecast:', error);
            return this.fallbackConsumptionForecast(stationId, days);
        }
    }

    enrichForecast(forecast, weather) {
        return forecast.map((day, index) => ({
            ...day,
            weather: weather[index],
            confidence: this.calculateConfidence(day, weather[index]),
            recommendations: this.generateDayRecommendations(day, weather[index])
        }));
    }

    analyzeConsumptionPatterns(data) {
        const hourlyPatterns = this.calculateHourlyAverages(data);
        const weeklyPatterns = this.calculateWeeklyPatterns(data);
        const seasonalTrends = this.calculateSeasonalTrends(data);

        return {
            hourly: hourlyPatterns,
            weekly: weeklyPatterns,
            seasonal: seasonalTrends,
            peaks: this.identifyPeakUsage(data)
        };
    }

    calculateHourlyAverages(data) {
        const hourlyTotals = new Array(24).fill(0);
        const hourlyCounts = new Array(24).fill(0);

        data.forEach(record => {
            const hour = new Date(record.timestamp).getHours();
            hourlyTotals[hour] += record.consumption;
            hourlyCounts[hour]++;
        });

        return hourlyTotals.map((total, hour) => ({
            hour,
            average: hourlyCounts[hour] > 0 ? total / hourlyCounts[hour] : 0,
            samples: hourlyCounts[hour]
        }));
    }

    calculateConfidence(dayForecast, weather) {
        let confidence = 0.8; // Base confidence

        // Reduzir confiança para condições meteorológicas extremas
        if (weather.cloudCover > 80) confidence -= 0.2;
        if (weather.precipitation > 5) confidence -= 0.1;
        if (weather.windSpeed > 20) confidence -= 0.1;

        // Aumentar confiança para condições ideais
        if (weather.cloudCover < 20 && weather.temperature > 15 && weather.temperature < 35) {
            confidence += 0.1;
        }

        return Math.max(0.3, Math.min(0.95, confidence));
    }

    getDateDaysAgo(days) {
        const date = new Date();
        date.setDate(date.getDate() - days);
        return date.toISOString().split('T')[0];
    }
}
```

### Modelo de Machine Learning
```javascript
class EnergyPredictionModel {
    constructor() {
        this.models = {
            production: null,
            consumption: null
        };
        this.features = [
            'solarIrradiance',
            'temperature',
            'cloudCover',
            'humidity',
            'windSpeed',
            'dayOfYear',
            'dayOfWeek',
            'month',
            'systemCapacity',
            'systemAge'
        ];
    }

    async predictProduction(inputs) {
        // Simplified prediction logic - in production, use AWS SageMaker or similar
        const { historical, weather, system, days } = inputs;
        
        const predictions = [];
        
        for (let i = 0; i < days; i++) {
            const dayWeather = weather[i];
            const prediction = this.calculateDailyProduction(dayWeather, system, historical);
            predictions.push(prediction);
        }

        return predictions;
    }

    calculateDailyProduction(weather, system, historical) {
        // Simplified model - replace with trained ML model
        const baseProduction = system.capacity * 5; // 5 hours effective sun
        
        // Weather adjustments
        let weatherFactor = 1.0;
        weatherFactor *= (100 - weather.cloudCover) / 100;
        weatherFactor *= Math.max(0.1, 1 - (weather.precipitation / 20));
        
        // Temperature adjustment (peak efficiency around 25°C)
        const tempOptimal = 25;
        const tempDiff = Math.abs(weather.temperature - tempOptimal);
        weatherFactor *= Math.max(0.7, 1 - (tempDiff / 50));

        // Seasonal adjustment
        const dayOfYear = new Date().getDayOfYear();
        const seasonalFactor = 0.8 + 0.4 * Math.sin((dayOfYear - 80) * Math.PI / 182.5);

        const production = baseProduction * weatherFactor * seasonalFactor;

        return {
            date: this.getDateOffset(i),
            expectedProduction: Math.max(0, production),
            weatherFactor: weatherFactor,
            seasonalFactor: seasonalFactor,
            confidence: this.calculateProductionConfidence(weather)
        };
    }

    calculateProductionConfidence(weather) {
        let confidence = 0.8;
        
        if (weather.cloudCover < 30) confidence += 0.1;
        if (weather.precipitation < 1) confidence += 0.05;
        if (weather.windSpeed < 15) confidence += 0.05;
        
        return Math.min(0.95, confidence);
    }

    getDateOffset(days) {
        const date = new Date();
        date.setDate(date.getDate() + days);
        return date.toISOString().split('T')[0];
    }
}
```

## ⚡ Sistema de Otimização Automática

### Otimizador de Cargas
```javascript
class LoadOptimizationService {
    constructor(dataService, forecastService, controlService) {
        this.dataService = dataService;
        this.forecastService = forecastService;
        this.controlService = controlService;
        this.tariffService = new TariffService();
    }

    async optimizeEnergyUsage(stationId, timeHorizon = 24) {
        try {
            // Buscar dados necessários
            const productionForecast = await this.forecastService.generateProductionForecast(stationId, 2);
            const consumptionForecast = await this.forecastService.generateConsumptionForecast(stationId, 2);
            const currentBatteryLevel = await this.dataService.getBatteryLevel(stationId);
            const tariffs = await this.tariffService.getTariffSchedule(timeHorizon);
            const appliances = await this.dataService.getControllableAppliances(stationId);

            // Gerar plano de otimização
            const optimizationPlan = this.generateOptimizationPlan({
                production: productionForecast,
                consumption: consumptionForecast,
                battery: currentBatteryLevel,
                tariffs: tariffs,
                appliances: appliances,
                timeHorizon: timeHorizon
            });

            // Executar otimizações imediatas
            await this.executeImmediateOptimizations(stationId, optimizationPlan);

            // Agendar otimizações futuras
            await this.scheduleOptimizations(stationId, optimizationPlan);

            return optimizationPlan;

        } catch (error) {
            console.error('Error in load optimization:', error);
            throw error;
        }
    }

    generateOptimizationPlan(inputs) {
        const { production, consumption, battery, tariffs, appliances, timeHorizon } = inputs;
        
        const plan = {
            summary: {
                totalSavings: 0,
                co2Reduction: 0,
                actions: []
            },
            hourlyPlan: [],
            applianceSchedule: [],
            batteryStrategy: {},
            recommendations: []
        };

        // Análise hora por hora
        for (let hour = 0; hour < timeHorizon; hour++) {
            const hourPlan = this.optimizeHour(hour, inputs);
            plan.hourlyPlan.push(hourPlan);
            plan.summary.totalSavings += hourPlan.savings;
        }

        // Estratégia de bateria
        plan.batteryStrategy = this.optimizeBatteryStrategy(inputs);

        // Agendamento de aparelhos
        plan.applianceSchedule = this.optimizeApplianceSchedule(appliances, inputs);

        // Gerar recomendações
        plan.recommendations = this.generateRecommendations(plan);

        return plan;
    }

    optimizeHour(hour, inputs) {
        const { production, consumption, tariffs } = inputs;
        
        const currentHour = new Date().getHours() + hour;
        const dayIndex = Math.floor(currentHour / 24);
        const hourOfDay = currentHour % 24;

        const expectedProduction = this.getHourlyProduction(production[dayIndex], hourOfDay);
        const expectedConsumption = this.getHourlyConsumption(consumption[dayIndex], hourOfDay);
        const tariff = tariffs[hour] || tariffs[tariffs.length - 1];

        const balance = expectedProduction - expectedConsumption;
        
        let action = 'maintain';
        let savings = 0;

        if (balance > 0) {
            // Excesso de produção
            if (tariff.rate < 0.15) { // Tarifa baixa
                action = 'charge_battery';
                savings = balance * (0.20 - tariff.rate); // Economia futura
            } else {
                action = 'sell_to_grid';
                savings = balance * tariff.sellRate;
            }
        } else if (balance < 0) {
            // Déficit de energia
            if (tariff.rate > 0.25) { // Tarifa alta
                action = 'use_battery';
                savings = Math.abs(balance) * (tariff.rate - 0.15);
            } else {
                action = 'use_grid';
                savings = 0;
            }
        }

        return {
            hour: currentHour,
            production: expectedProduction,
            consumption: expectedConsumption,
            balance: balance,
            tariff: tariff.rate,
            action: action,
            savings: savings
        };
    }

    optimizeBatteryStrategy(inputs) {
        const { battery, tariffs, production } = inputs;
        
        const strategy = {
            currentLevel: battery.level,
            targetLevel: {},
            chargingSchedule: [],
            dischargingSchedule: []
        };

        // Identificar horários de tarifa baixa para carregamento
        const lowTariffHours = tariffs
            .map((tariff, index) => ({ ...tariff, hour: index }))
            .filter(t => t.rate < 0.18)
            .sort((a, b) => a.rate - b.rate);

        // Identificar horários de tarifa alta para descarga
        const highTariffHours = tariffs
            .map((tariff, index) => ({ ...tariff, hour: index }))
            .filter(t => t.rate > 0.25)
            .sort((a, b) => b.rate - a.rate);

        // Programar carregamento durante tarifa baixa
        lowTariffHours.slice(0, 4).forEach(hour => {
            strategy.chargingSchedule.push({
                hour: hour.hour,
                targetCharge: 20, // 20% por hora
                reason: 'low_tariff'
            });
        });

        // Programar descarga durante tarifa alta
        highTariffHours.slice(0, 3).forEach(hour => {
            strategy.dischargingSchedule.push({
                hour: hour.hour,
                maxDischarge: 25, // 25% por hora
                reason: 'high_tariff'
            });
        });

        return strategy;
    }

    optimizeApplianceSchedule(appliances, inputs) {
        const { production, tariffs } = inputs;
        
        const schedule = [];

        appliances.forEach(appliance => {
            const optimalTime = this.findOptimalTime(appliance, inputs);
            
            if (optimalTime) {
                schedule.push({
                    applianceId: appliance.id,
                    name: appliance.name,
                    scheduledTime: optimalTime.hour,
                    duration: appliance.duration,
                    savings: optimalTime.savings,
                    reason: optimalTime.reason
                });
            }
        });

        return schedule.sort((a, b) => b.savings - a.savings);
    }

    findOptimalTime(appliance, inputs) {
        const { production, tariffs } = inputs;
        
        let bestTime = null;
        let maxSavings = 0;

        for (let hour = 0; hour < 24; hour++) {
            // Verificar se o horário é compatível com o aparelho
            if (!this.isTimeCompatible(hour, appliance)) continue;

            const tariff = tariffs[hour];
            const productionAtHour = this.getHourlyProduction(production[0], hour);
            
            let savings = 0;
            let reason = '';

            if (productionAtHour > appliance.consumption) {
                // Usar energia solar
                savings = appliance.consumption * (tariff.rate - 0.05); // Custo evitado
                reason = 'solar_power';
            } else if (tariff.rate < 0.18) {
                // Usar rede com tarifa baixa
                savings = appliance.consumption * (0.25 - tariff.rate);
                reason = 'low_tariff';
            }

            if (savings > maxSavings) {
                maxSavings = savings;
                bestTime = {
                    hour: hour,
                    savings: savings,
                    reason: reason
                };
            }
        }

        return bestTime;
    }

    isTimeCompatible(hour, appliance) {
        // Verificar restrições de uso do aparelho
        if (appliance.type === 'washing_machine' && hour >= 22) return false;
        if (appliance.type === 'dryer' && hour >= 23) return false;
        if (appliance.type === 'dishwasher' && hour >= 1 && hour <= 6) return false;
        
        return true;
    }

    async executeImmediateOptimizations(stationId, plan) {
        const currentHour = new Date().getHours();
        const currentPlan = plan.hourlyPlan[0];
        
        if (!currentPlan) return;

        switch (currentPlan.action) {
            case 'charge_battery':
                await this.controlService.setBatteryMode(stationId, 'force_charge');
                break;
                
            case 'use_battery':
                await this.controlService.setBatteryMode(stationId, 'force_discharge');
                break;
                
            case 'use_grid':
                await this.controlService.setLoadPriority(stationId, 'grid_first');
                break;
                
            default:
                await this.controlService.setBatteryMode(stationId, 'auto');
        }

        // Executar schedule de aparelhos para a próxima hora
        const upcomingAppliances = plan.applianceSchedule.filter(
            item => item.scheduledTime === currentHour + 1
        );

        for (const appliance of upcomingAppliances) {
            await this.scheduleApplianceActivation(appliance);
        }
    }

    generateRecommendations(plan) {
        const recommendations = [];

        // Recomendação de economia
        if (plan.summary.totalSavings > 10) {
            recommendations.push({
                type: 'savings',
                priority: 'high',
                title: 'Economia Significativa Disponível',
                message: `Você pode economizar R$ ${plan.summary.totalSavings.toFixed(2)} seguindo as otimizações sugeridas.`,
                actions: ['IMPLEMENT_OPTIMIZATION']
            });
        }

        // Recomendação de horários
        const bestHours = plan.hourlyPlan
            .filter(h => h.savings > 2)
            .sort((a, b) => b.savings - a.savings)
            .slice(0, 3);

        if (bestHours.length > 0) {
            recommendations.push({
                type: 'timing',
                priority: 'medium',
                title: 'Melhores Horários para Uso',
                message: `Use equipamentos de alto consumo às ${bestHours.map(h => `${h.hour}h`).join(', ')} para máxima economia.`,
                actions: ['SCHEDULE_APPLIANCES']
            });
        }

        // Recomendação de bateria
        if (plan.batteryStrategy.chargingSchedule.length > 0) {
            recommendations.push({
                type: 'battery',
                priority: 'medium',
                title: 'Estratégia de Bateria',
                message: 'Configure carregamento automático durante madrugada para aproveitar tarifa baixa.',
                actions: ['CONFIGURE_BATTERY_SCHEDULE']
            });
        }

        return recommendations;
    }

    getHourlyProduction(dayProduction, hour) {
        // Curva de produção solar típica
        const solarCurve = [
            0, 0, 0, 0, 0, 0.05, 0.15, 0.35, 0.6, 0.8, 0.95, 1.0,
            1.0, 0.95, 0.8, 0.6, 0.35, 0.15, 0.05, 0, 0, 0, 0, 0
        ];

        return dayProduction.expectedProduction * solarCurve[hour];
    }

    getHourlyConsumption(dayConsumption, hour) {
        // Padrão típico de consumo residencial
        const consumptionPattern = [
            0.3, 0.25, 0.2, 0.2, 0.25, 0.4, 0.7, 0.8, 0.6, 0.5, 0.45, 0.5,
            0.6, 0.55, 0.5, 0.6, 0.8, 1.0, 0.9, 0.85, 0.8, 0.7, 0.5, 0.4
        ];

        return dayConsumption.expectedConsumption * consumptionPattern[hour];
    }
}
```

## 💰 Sistema de Tarifas Dinâmicas

```javascript
class TariffService {
    constructor() {
        this.tariffData = new Map();
        this.providers = {
            default: this.getDefaultTariff.bind(this),
            timeOfUse: this.getTimeOfUseTariff.bind(this),
            dynamic: this.getDynamicTariff.bind(this)
        };
    }

    async getTariffSchedule(hours = 24, location = 'BR-SP') {
        try {
            // Em produção, buscar de API da concessionária
            const tariffType = await this.getTariffType(location);
            const provider = this.providers[tariffType] || this.providers.default;
            
            const schedule = [];
            
            for (let i = 0; i < hours; i++) {
                const hour = (new Date().getHours() + i) % 24;
                const tariff = await provider(hour, location);
                schedule.push(tariff);
            }

            return schedule;

        } catch (error) {
            console.error('Error getting tariff schedule:', error);
            return this.getFallbackTariff(hours);
        }
    }

    getTimeOfUseTariff(hour, location) {
        // Tarifa branca típica no Brasil
        let rate, sellRate;

        if (hour >= 18 && hour <= 20) {
            // Horário de ponta
            rate = 0.45;
            sellRate = 0.25;
        } else if ((hour >= 6 && hour <= 7) || (hour >= 21 && hour <= 22)) {
            // Horário intermediário
            rate = 0.28;
            sellRate = 0.18;
        } else {
            // Horário fora de ponta
            rate = 0.18;
            sellRate = 0.15;
        }

        return {
            hour: hour,
            rate: rate,
            sellRate: sellRate,
            type: 'time_of_use',
            currency: 'BRL'
        };
    }

    getDynamicTariff(hour, location) {
        // Simular tarifa dinâmica baseada em oferta/demanda
        const baseTariff = this.getTimeOfUseTariff(hour, location);
        
        // Fatores de ajuste dinâmico
        const demandFactor = this.getCurrentDemandFactor();
        const renewableFactor = this.getRenewableGenerationFactor();
        
        const dynamicRate = baseTariff.rate * demandFactor * renewableFactor;
        
        return {
            ...baseTariff,
            rate: Math.max(0.10, Math.min(0.60, dynamicRate)),
            type: 'dynamic',
            factors: {
                demand: demandFactor,
                renewable: renewableFactor
            }
        };
    }

    getDefaultTariff(hour, location) {
        return {
            hour: hour,
            rate: 0.25, // Tarifa fixa
            sellRate: 0.18,
            type: 'fixed',
            currency: 'BRL'
        };
    }

    getCurrentDemandFactor() {
        // Simular fator de demanda (0.8 - 1.3)
        const hour = new Date().getHours();
        
        if (hour >= 18 && hour <= 20) return 1.3; // Pico de demanda
        if (hour >= 12 && hour <= 14) return 1.1; // Demanda média-alta
        if (hour >= 2 && hour <= 5) return 0.8;   // Demanda baixa
        
        return 1.0;
    }

    getRenewableGenerationFactor() {
        // Simular fator de geração renovável (0.9 - 1.1)
        const hour = new Date().getHours();
        
        if (hour >= 10 && hour <= 15) return 0.9; // Alta geração solar = preço menor
        if (hour >= 19 || hour <= 6) return 1.1; // Baixa geração renovável = preço maior
        
        return 1.0;
    }

    getFallbackTariff(hours) {
        return Array(hours).fill({
            rate: 0.25,
            sellRate: 0.18,
            type: 'fallback',
            currency: 'BRL'
        });
    }
}
```

## 📱 Alexa Handlers para Otimização

```javascript
// Handler para otimização de energia
const OptimizeEnergyHandler = {
    canHandle(handlerInput) {
        return Alexa.getRequestType(handlerInput.requestEnvelope) === 'IntentRequest'
            && Alexa.getIntentName(handlerInput.requestEnvelope) === 'OptimizeEnergyIntent';
    },
    async handle(handlerInput) {
        try {
            const userId = handlerInput.requestEnvelope.session.user.userId;
            const stationId = await getUserStationId(userId);
            
            const optimizationService = new LoadOptimizationService(dataService, forecastService, controlService);
            const plan = await optimizationService.optimizeEnergyUsage(stationId, 24);
            
            const response = formatOptimizationResponse(plan);
            
            return handlerInput.responseBuilder
                .speak(response.speech)
                .withSimpleCard('Otimização Energética', response.card)
                .reprompt('Quer que eu implemente essas otimizações?')
                .getResponse();

        } catch (error) {
            console.error('Error in energy optimization:', error);
            return handlerInput.responseBuilder
                .speak('Não consegui otimizar o uso de energia no momento.')
                .getResponse();
        }
    }
};

// Handler para melhor horário para usar aparelhos
const GetBestUsageTimeHandler = {
    canHandle(handlerInput) {
        return Alexa.getRequestType(handlerInput.requestEnvelope) === 'IntentRequest'
            && Alexa.getIntentName(handlerInput.requestEnvelope) === 'GetBestUsageTimeIntent';
    },
    async handle(handlerInput) {
        try {
            const appliance = Alexa.getSlotValue(handlerInput.requestEnvelope, 'appliance') || 'máquina de lavar';
            const userId = handlerInput.requestEnvelope.session.user.userId;
            const stationId = await getUserStationId(userId);
            
            const optimizationService = new LoadOptimizationService(dataService, forecastService, controlService);
            const optimalTime = await findOptimalApplianceTime(stationId, appliance);
            
            const response = formatUsageTimeResponse(appliance, optimalTime);
            
            return handlerInput.responseBuilder
                .speak(response.speech)
                .withSimpleCard('Melhor Horário', response.card)
                .reprompt('Quer que eu programe automaticamente?')
                .getResponse();

        } catch (error) {
            console.error('Error finding optimal usage time:', error);
            return handlerInput.responseBuilder
                .speak('Não consegui determinar o melhor horário no momento.')
                .getResponse();
        }
    }
};

// Handler para previsão de produção
const GetProductionForecastHandler = {
    canHandle(handlerInput) {
        return Alexa.getRequestType(handlerInput.requestEnvelope) === 'IntentRequest'
            && Alexa.getIntentName(handlerInput.requestEnvelope) === 'GetProductionForecastIntent';
    },
    async handle(handlerInput) {
        try {
            const period = Alexa.getSlotValue(handlerInput.requestEnvelope, 'period') || 'hoje';
            const userId = handlerInput.requestEnvelope.session.user.userId;
            const stationId = await getUserStationId(userId);
            
            const forecastService = new EnergyForecastService(dataService, weatherService);
            const forecast = await forecastService.generateProductionForecast(stationId, parsePeriod(period));
            
            const response = formatForecastResponse(forecast, period);
            
            return handlerInput.responseBuilder
                .speak(response.speech)
                .withSimpleCard('Previsão de Produção', response.card)
                .getResponse();

        } catch (error) {
            console.error('Error getting production forecast:', error);
            return handlerInput.responseBuilder
                .speak('Não consegui obter a previsão de produção no momento.')
                .getResponse();
        }
    }
};

function formatOptimizationResponse(plan) {
    const savings = plan.summary.totalSavings;
    const topRecommendation = plan.recommendations[0];
    
    let speech = `Analisei seu sistema e identifiquei uma economia potencial de R$ ${savings.toFixed(2)} para hoje. `;
    
    if (topRecommendation) {
        speech += topRecommendation.message + ' ';
    }
    
    const schedule = plan.applianceSchedule.slice(0, 2);
    if (schedule.length > 0) {
        speech += `Recomendo usar ${schedule[0].name} às ${schedule[0].scheduledTime}h `;
        if (schedule.length > 1) {
            speech += `e ${schedule[1].name} às ${schedule[1].scheduledTime}h `;
        }
        speech += 'para máxima economia.';
    }
    
    const card = `Economia potencial: R$ ${savings.toFixed(2)}\n` +
                 plan.recommendations.map(r => `• ${r.message}`).join('\n');
    
    return { speech, card };
}

function formatUsageTimeResponse(appliance, optimalTime) {
    if (!optimalTime) {
        return {
            speech: `Não encontrei um horário específico mais vantajoso para ${appliance} hoje. Use quando for mais conveniente.`,
            card: `${appliance}: Use quando conveniente`
        };
    }
    
    const hour = optimalTime.hour;
    const period = hour < 12 ? 'manhã' : hour < 18 ? 'tarde' : 'noite';
    const savings = optimalTime.savings;
    
    let speech = `O melhor horário para usar ${appliance} é às ${hour} horas da ${period}. `;
    
    if (optimalTime.reason === 'solar_power') {
        speech += 'Nesse horário você usará energia solar, ';
    } else if (optimalTime.reason === 'low_tariff') {
        speech += 'Nesse horário a tarifa elétrica é mais baixa, ';
    }
    
    speech += `economizando aproximadamente R$ ${savings.toFixed(2)}.`;
    
    const card = `${appliance}: ${hour}h (economia: R$ ${savings.toFixed(2)})`;
    
    return { speech, card };
}

function formatForecastResponse(forecast, period) {
    const todayForecast = forecast[0];
    const production = todayForecast.expectedProduction;
    const confidence = Math.round(todayForecast.confidence * 100);
    
    let speech = `Para ${period}, prevejo uma produção de ${production.toFixed(1)} quilowatts-hora `;
    speech += `com ${confidence}% de confiança. `;
    
    if (forecast.length > 1) {
        const tomorrowProduction = forecast[1].expectedProduction;
        const change = ((tomorrowProduction - production) / production * 100);
        
        if (Math.abs(change) > 10) {
            const direction = change > 0 ? 'maior' : 'menor';
            speech += `Amanhã a produção será ${Math.abs(change).toFixed(0)}% ${direction}.`;
        }
    }
    
    const card = forecast.map((day, i) => 
        `${i === 0 ? 'Hoje' : 'Amanhã'}: ${day.expectedProduction.toFixed(1)} kWh`
    ).join('\n');
    
    return { speech, card };
}

function parsePeriod(period) {
    const periodMap = {
        'hoje': 1,
        'amanhã': 2,
        'semana': 7,
        'próximos dias': 3
    };
    
    return periodMap[period.toLowerCase()] || 1;
}

async function findOptimalApplianceTime(stationId, applianceName) {
    const applianceMap = {
        'máquina de lavar': { consumption: 2.0, duration: 2, type: 'washing_machine' },
        'secadora': { consumption: 3.5, duration: 1.5, type: 'dryer' },
        'lava louças': { consumption: 1.8, duration: 1, type: 'dishwasher' },
        'ferro de passar': { consumption: 2.2, duration: 0.5, type: 'iron' }
    };
    
    const appliance = applianceMap[applianceName.toLowerCase()];
    if (!appliance) return null;
    
    const optimizationService = new LoadOptimizationService(dataService, forecastService, controlService);
    const forecast = await forecastService.generateProductionForecast(stationId, 1);
    const tariffs = await new TariffService().getTariffSchedule(24);
    
    return optimizationService.findOptimalTime(appliance, {
        production: forecast,
        tariffs: tariffs
    });
}
```

## ⚡ Próximos Passos

1. **Implemente** o sistema de previsão
2. **Configure** otimização automática
3. **Teste** as recomendações
4. **Finalize** com [Complete Implementation](05-complete-implementation.md)

---
**Anterior:** [← Emergency Management](03-emergency-management.md) | **Próximo:** [Complete Implementation →](05-complete-implementation.md)
