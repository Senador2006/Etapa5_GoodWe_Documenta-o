# 3. Sistema de Gestão de Emergências Energéticas

## ⚠️ Visão Geral do Sistema

O sistema de gestão de emergências energéticas monitora continuamente os dados dos dispositivos GoodWe e fornece alertas proativos, sugestões inteligentes e ações automatizadas para prevenir e gerenciar situações críticas de energia.

## 🚨 Tipos de Emergências Detectadas

### 1. Risco de Queda de Energia
- Bateria com nível crítico (< 20%)
- Produção solar insuficiente
- Alto consumo durante horários de pico
- Falha em dispositivos críticos

### 2. Problemas de Sistema
- Inversor com falha
- Bateria com temperatura alta
- Perda de comunicação com dispositivos
- Sobrecarga de consumo

### 3. Condições Meteorológicas
- Previsão de tempestades
- Dias consecutivos sem sol
- Temperaturas extremas
- Ventos fortes

## 🧠 Sistema de Monitoramento Inteligente

### Detector de Anomalias
```javascript
class EmergencyDetectionService {
    constructor(dataService, weatherService, notificationService) {
        this.dataService = dataService;
        this.weatherService = weatherService;
        this.notificationService = notificationService;
        this.thresholds = {
            criticalBatteryLevel: 20,
            lowBatteryLevel: 30,
            highTemperature: 60,
            maxConsumptionRatio: 1.5,
            minProductionRatio: 0.3
        };
    }

    async analyzeEnergyStatus(stationId) {
        try {
            const realtimeData = await this.dataService.getRealtimeData(stationId);
            const historicalData = await this.dataService.getHistoricalData(
                stationId, 
                this.getDateDaysAgo(7), 
                new Date().toISOString().split('T')[0]
            );
            const weatherForecast = await this.weatherService.getForecast(stationId, 3);

            const alerts = [];

            // Verificar nível da bateria
            const batteryAlert = this.checkBatteryStatus(realtimeData);
            if (batteryAlert) alerts.push(batteryAlert);

            // Verificar produção vs consumo
            const balanceAlert = this.checkEnergyBalance(realtimeData, historicalData);
            if (balanceAlert) alerts.push(balanceAlert);

            // Verificar condições do sistema
            const systemAlert = this.checkSystemHealth(realtimeData);
            if (systemAlert) alerts.push(systemAlert);

            // Verificar previsão meteorológica
            const weatherAlert = this.checkWeatherImpact(weatherForecast, historicalData);
            if (weatherAlert) alerts.push(weatherAlert);

            // Processar alertas críticos
            if (alerts.length > 0) {
                await this.processAlerts(stationId, alerts);
            }

            return alerts;

        } catch (error) {
            console.error('Error in emergency detection:', error);
            throw error;
        }
    }

    checkBatteryStatus(data) {
        const batteryLevel = data.battery.soc;
        const batteryPower = data.battery.power;
        
        if (batteryLevel <= this.thresholds.criticalBatteryLevel) {
            return {
                type: 'CRITICAL_BATTERY',
                severity: 'HIGH',
                message: `Bateria em nível crítico: ${batteryLevel}%`,
                currentValue: batteryLevel,
                threshold: this.thresholds.criticalBatteryLevel,
                suggestions: [
                    'Reduza o consumo de aparelhos não essenciais',
                    'Ative o modo de economia de energia',
                    'Considere usar energia da rede elétrica',
                    'Verifique se há produção solar disponível'
                ],
                actions: [
                    'ACTIVATE_ENERGY_SAVING',
                    'SWITCH_TO_GRID_PRIORITY',
                    'DISABLE_NON_CRITICAL_LOADS'
                ]
            };
        } else if (batteryLevel <= this.thresholds.lowBatteryLevel) {
            return {
                type: 'LOW_BATTERY',
                severity: 'MEDIUM',
                message: `Bateria com nível baixo: ${batteryLevel}%`,
                currentValue: batteryLevel,
                threshold: this.thresholds.lowBatteryLevel,
                suggestions: [
                    'Monitore o consumo nas próximas horas',
                    'Considere adiar uso de equipamentos de alto consumo',
                    'Verifique a previsão de produção solar'
                ],
                actions: [
                    'MONITOR_CONSUMPTION',
                    'SUGGEST_OPTIMAL_USAGE_TIMES'
                ]
            };
        }

        return null;
    }

    checkEnergyBalance(realtimeData, historicalData) {
        const currentProduction = realtimeData.production.current;
        const currentConsumption = realtimeData.consumption.current;
        const consumptionRatio = currentConsumption / (currentProduction || 1);

        // Calcular média histórica
        const avgProduction = this.calculateAverage(historicalData, 'production');
        const productionRatio = currentProduction / avgProduction;

        if (consumptionRatio > this.thresholds.maxConsumptionRatio) {
            return {
                type: 'HIGH_CONSUMPTION',
                severity: 'MEDIUM',
                message: `Consumo ${Math.round(consumptionRatio * 100)}% acima da produção atual`,
                currentValue: consumptionRatio,
                threshold: this.thresholds.maxConsumptionRatio,
                suggestions: [
                    'Identifique equipamentos de alto consumo',
                    'Adie uso de máquina de lavar, secadora ou outros equipamentos',
                    'Use a energia da rede durante horários de tarifa baixa',
                    'Considere carregar a bateria durante a madrugada'
                ],
                actions: [
                    'IDENTIFY_HIGH_CONSUMPTION_DEVICES',
                    'SUGGEST_USAGE_SCHEDULE',
                    'ACTIVATE_GRID_CHARGING'
                ]
            };
        }

        if (productionRatio < this.thresholds.minProductionRatio) {
            return {
                type: 'LOW_PRODUCTION',
                severity: 'MEDIUM',
                message: `Produção ${Math.round(productionRatio * 100)}% abaixo da média histórica`,
                currentValue: productionRatio,
                threshold: this.thresholds.minProductionRatio,
                suggestions: [
                    'Verifique se há sombra nos painéis solares',
                    'Considere limpeza dos painéis',
                    'Monitore condições meteorológicas',
                    'Use energia da rede durante baixa produção'
                ],
                actions: [
                    'SCHEDULE_PANEL_INSPECTION',
                    'SWITCH_TO_GRID_PRIORITY',
                    'MONITOR_WEATHER_CONDITIONS'
                ]
            };
        }

        return null;
    }

    checkSystemHealth(data) {
        const inverterTemp = data.inverter.temperature;
        const inverterStatus = data.inverter.status;
        const batteryStatus = data.battery.status;

        if (inverterTemp > this.thresholds.highTemperature) {
            return {
                type: 'HIGH_TEMPERATURE',
                severity: 'HIGH',
                message: `Temperatura do inversor alta: ${inverterTemp}°C`,
                currentValue: inverterTemp,
                threshold: this.thresholds.highTemperature,
                suggestions: [
                    'Verifique ventilação do inversor',
                    'Reduza a carga temporariamente',
                    'Entre em contato com o suporte técnico se persistir'
                ],
                actions: [
                    'REDUCE_INVERTER_LOAD',
                    'INCREASE_VENTILATION',
                    'SCHEDULE_MAINTENANCE'
                ]
            };
        }

        if (inverterStatus === 'fault' || batteryStatus === 'fault') {
            return {
                type: 'SYSTEM_FAULT',
                severity: 'HIGH',
                message: 'Detectada falha no sistema',
                suggestions: [
                    'Verifique conexões dos equipamentos',
                    'Reinicie o sistema se necessário',
                    'Entre em contato com suporte técnico'
                ],
                actions: [
                    'SYSTEM_DIAGNOSTIC',
                    'ATTEMPT_RESTART',
                    'CONTACT_SUPPORT'
                ]
            };
        }

        return null;
    }

    async checkWeatherImpact(forecast, historicalData) {
        // Verificar se há condições meteorológicas que podem afetar a produção
        const badWeatherDays = forecast.filter(day => 
            day.condition === 'storm' || 
            day.condition === 'heavy_rain' ||
            day.cloudCover > 80
        ).length;

        if (badWeatherDays >= 2) {
            return {
                type: 'WEATHER_IMPACT',
                severity: 'MEDIUM',
                message: `Previsão de ${badWeatherDays} dias com baixa produção solar`,
                suggestions: [
                    'Carregue a bateria completamente antes do período',
                    'Planeje uso de equipamentos para antes da tempestade',
                    'Considere usar energia da rede durante o período',
                    'Monitore o nível da bateria mais frequentemente'
                ],
                actions: [
                    'FULL_BATTERY_CHARGE',
                    'SCHEDULE_HIGH_CONSUMPTION',
                    'WEATHER_MONITORING'
                ]
            };
        }

        return null;
    }

    async processAlerts(stationId, alerts) {
        // Determinar prioridade geral
        const highSeverityAlerts = alerts.filter(alert => alert.severity === 'HIGH');
        const isEmergency = highSeverityAlerts.length > 0;

        if (isEmergency) {
            // Executar ações automáticas de emergência
            await this.executeEmergencyActions(stationId, alerts);
            
            // Enviar notificação proativa imediata
            await this.notificationService.sendProactiveNotification(
                stationId,
                'EMERGENCY',
                this.buildEmergencyMessage(alerts)
            );
        } else {
            // Enviar alerta normal
            await this.notificationService.sendAlert(
                stationId,
                'WARNING', 
                this.buildWarningMessage(alerts)
            );
        }

        // Salvar no histórico
        await this.saveAlertsHistory(stationId, alerts);
    }

    buildEmergencyMessage(alerts) {
        const criticalAlerts = alerts.filter(alert => alert.severity === 'HIGH');
        const mainAlert = criticalAlerts[0];

        return {
            title: 'Emergência Energética Detectada',
            message: mainAlert.message,
            suggestions: mainAlert.suggestions.slice(0, 3), // Top 3 sugestões
            urgency: 'HIGH'
        };
    }

    calculateAverage(historicalData, field) {
        if (!historicalData || historicalData.length === 0) return 0;
        
        const sum = historicalData.reduce((acc, day) => acc + (day[field] || 0), 0);
        return sum / historicalData.length;
    }

    getDateDaysAgo(days) {
        const date = new Date();
        date.setDate(date.getDate() - days);
        return date.toISOString().split('T')[0];
    }
}
```

## 🤖 Ações Automáticas de Emergência

```javascript
class EmergencyActionService {
    constructor(controlService, dataService) {
        this.controlService = controlService;
        this.dataService = dataService;
    }

    async executeEmergencyActions(stationId, alerts) {
        const actions = this.extractActions(alerts);
        
        for (const action of actions) {
            try {
                await this.executeAction(stationId, action);
                console.log(`Emergency action executed: ${action}`);
            } catch (error) {
                console.error(`Failed to execute action ${action}:`, error);
            }
        }
    }

    async executeAction(stationId, action) {
        const devices = await this.dataService.getDeviceList(stationId);
        
        switch (action) {
            case 'ACTIVATE_ENERGY_SAVING':
                await this.activateEnergySaving(devices);
                break;
                
            case 'SWITCH_TO_GRID_PRIORITY':
                await this.switchToGridPriority(devices);
                break;
                
            case 'DISABLE_NON_CRITICAL_LOADS':
                await this.disableNonCriticalLoads(devices);
                break;
                
            case 'REDUCE_INVERTER_LOAD':
                await this.reduceInverterLoad(devices);
                break;
                
            case 'FULL_BATTERY_CHARGE':
                await this.scheduleFullBatteryCharge(devices);
                break;
                
            default:
                console.warn(`Unknown action: ${action}`);
        }
    }

    async activateEnergySaving(devices) {
        const batteries = devices.filter(d => d.type === 'battery');
        
        for (const battery of batteries) {
            await this.controlService.setBatteryMode(battery.id, 'eco');
            await this.controlService.setDischargeLimit(battery.id, 15); // Parar descarga em 15%
        }

        const inverters = devices.filter(d => d.type === 'inverter');
        for (const inverter of inverters) {
            await this.controlService.controlDevice(inverter.id, 'set_power_limit', { limit: 80 });
        }
    }

    async switchToGridPriority(devices) {
        const batteries = devices.filter(d => d.type === 'battery');
        
        for (const battery of batteries) {
            await this.controlService.setLoadPriority(battery.id, 'grid_first');
        }
    }

    async disableNonCriticalLoads(devices) {
        const smartSwitches = devices.filter(d => d.type === 'smart_switch' && d.priority === 'low');
        
        for (const switchDevice of smartSwitches) {
            await this.controlService.controlDevice(switchDevice.id, 'power_off');
        }
    }

    async reduceInverterLoad(devices) {
        const inverters = devices.filter(d => d.type === 'inverter');
        
        for (const inverter of inverters) {
            await this.controlService.controlDevice(inverter.id, 'set_power_limit', { limit: 60 });
        }
    }

    async scheduleFullBatteryCharge(devices) {
        const batteries = devices.filter(d => d.type === 'battery');
        
        for (const battery of batteries) {
            // Programar carregamento durante madrugada (tarifa baixa)
            await this.controlService.scheduleOperation(
                battery.id,
                'force_charge',
                '00:00',
                '06:00'
            );
        }
    }

    extractActions(alerts) {
        const actions = new Set();
        
        alerts.forEach(alert => {
            if (alert.actions) {
                alert.actions.forEach(action => actions.add(action));
            }
        });
        
        return Array.from(actions);
    }
}
```

## 📱 Alexa Skill Handlers para Emergências

```javascript
// Handler para verificar riscos de energia
const CheckEnergyRiskHandler = {
    canHandle(handlerInput) {
        return Alexa.getRequestType(handlerInput.requestEnvelope) === 'IntentRequest'
            && Alexa.getIntentName(handlerInput.requestEnvelope) === 'CheckEnergyRiskIntent';
    },
    async handle(handlerInput) {
        try {
            const userId = handlerInput.requestEnvelope.session.user.userId;
            const stationId = await getUserStationId(userId);
            
            const emergencyService = new EmergencyDetectionService(dataService, weatherService, notificationService);
            const alerts = await emergencyService.analyzeEnergyStatus(stationId);
            
            if (alerts.length === 0) {
                return handlerInput.responseBuilder
                    .speak('Sua situação energética está normal. Não há riscos detectados no momento.')
                    .withSimpleCard('Status Energético', 'Sistema funcionando normalmente')
                    .getResponse();
            }

            const criticalAlerts = alerts.filter(alert => alert.severity === 'HIGH');
            const response = formatAlertsResponse(alerts, criticalAlerts.length > 0);
            
            return handlerInput.responseBuilder
                .speak(response.speech)
                .withSimpleCard('Alertas Energéticos', response.card)
                .reprompt('Quer que eu ative alguma ação de economia?')
                .getResponse();

        } catch (error) {
            console.error('Error checking energy risk:', error);
            return handlerInput.responseBuilder
                .speak('Não consegui verificar os riscos energéticos no momento.')
                .getResponse();
        }
    }
};

// Handler para sugestões de economia
const GetEnergySuggestionsHandler = {
    canHandle(handlerInput) {
        return Alexa.getRequestType(handlerInput.requestEnvelope) === 'IntentRequest'
            && Alexa.getIntentName(handlerInput.requestEnvelope) === 'GetEnergySuggestionsIntent';
    },
    async handle(handlerInput) {
        try {
            const userId = handlerInput.requestEnvelope.session.user.userId;
            const stationId = await getUserStationId(userId);
            
            const suggestions = await generateEnergySuggestions(stationId);
            const response = formatSuggestionsResponse(suggestions);
            
            return handlerInput.responseBuilder
                .speak(response.speech)
                .withSimpleCard('Sugestões de Economia', response.card)
                .reprompt('Quer que eu implemente alguma dessas sugestões?')
                .getResponse();

        } catch (error) {
            console.error('Error getting suggestions:', error);
            return handlerInput.responseBuilder
                .speak('Não consegui gerar sugestões no momento.')
                .getResponse();
        }
    }
};

// Handler para ativar modo de emergência
const ActivateEmergencyModeHandler = {
    canHandle(handlerInput) {
        return Alexa.getRequestType(handlerInput.requestEnvelope) === 'IntentRequest'
            && Alexa.getIntentName(handlerInput.requestEnvelope) === 'ActivateEmergencyModeIntent';
    },
    async handle(handlerInput) {
        try {
            const userId = handlerInput.requestEnvelope.session.user.userId;
            const stationId = await getUserStationId(userId);
            
            const actionService = new EmergencyActionService(controlService, dataService);
            
            // Executar ações de emergência predefinidas
            const emergencyActions = [
                'ACTIVATE_ENERGY_SAVING',
                'SWITCH_TO_GRID_PRIORITY',
                'DISABLE_NON_CRITICAL_LOADS'
            ];
            
            for (const action of emergencyActions) {
                await actionService.executeAction(stationId, action);
            }
            
            return handlerInput.responseBuilder
                .speak('Modo de emergência ativado. Implementei economia máxima de energia, priorizei uso da rede elétrica e desliguei cargas não críticas. Sua bateria será preservada.')
                .withSimpleCard('Emergência Energética', 'Modo de emergência ativo - energia sendo conservada')
                .getResponse();

        } catch (error) {
            console.error('Error activating emergency mode:', error);
            return handlerInput.responseBuilder
                .speak('Não consegui ativar o modo de emergência. Tente novamente.')
                .getResponse();
        }
    }
};

function formatAlertsResponse(alerts, isEmergency) {
    const severity = isEmergency ? 'críticos' : 'importantes';
    let speech = `Detectei ${alerts.length} alertas ${severity}. `;
    
    const mainAlert = alerts[0];
    speech += mainAlert.message + '. ';
    
    if (mainAlert.suggestions && mainAlert.suggestions.length > 0) {
        speech += 'Recomendo: ' + mainAlert.suggestions[0];
    }
    
    const card = alerts.map(alert => `• ${alert.message}`).join('\n');
    
    return { speech, card };
}

async function generateEnergySuggestions(stationId) {
    const dataService = new GoodWeDataService();
    const data = await dataService.getRealtimeData(stationId);
    const historical = await dataService.getHistoricalData(stationId, 
        new Date(Date.now() - 7 * 24 * 60 * 60 * 1000).toISOString().split('T')[0],
        new Date().toISOString().split('T')[0]
    );
    
    const suggestions = [];
    
    // Análise de horários ótimos
    const optimalHours = findOptimalUsageHours(historical);
    if (optimalHours.length > 0) {
        suggestions.push({
            type: 'timing',
            message: `Use equipamentos de alto consumo entre ${optimalHours[0]} e ${optimalHours[1]} quando há mais produção solar`,
            priority: 'high'
        });
    }
    
    // Análise de bateria
    if (data.battery.soc < 50) {
        suggestions.push({
            type: 'battery',
            message: 'Carregue a bateria durante a madrugada quando a tarifa elétrica é menor',
            priority: 'medium'
        });
    }
    
    // Análise de consumo
    const avgConsumption = historical.reduce((sum, day) => sum + day.consumption, 0) / historical.length;
    if (data.consumption.current > avgConsumption * 1.2) {
        suggestions.push({
            type: 'consumption',
            message: 'Seu consumo atual está 20% acima da média. Considere desligar equipamentos desnecessários',
            priority: 'high'
        });
    }
    
    return suggestions;
}

function formatSuggestionsResponse(suggestions) {
    if (suggestions.length === 0) {
        return {
            speech: 'Seu sistema está otimizado. Não tenho sugestões específicas no momento.',
            card: 'Sistema otimizado'
        };
    }
    
    const prioritySuggestions = suggestions.filter(s => s.priority === 'high');
    const mainSuggestion = prioritySuggestions[0] || suggestions[0];
    
    let speech = `Tenho ${suggestions.length} sugestões para otimizar sua energia. `;
    speech += mainSuggestion.message;
    
    if (suggestions.length > 1) {
        speech += ` Posso detalhar as outras ${suggestions.length - 1} sugestões se quiser.`;
    }
    
    const card = suggestions.map((s, i) => `${i + 1}. ${s.message}`).join('\n');
    
    return { speech, card };
}

function findOptimalUsageHours(historical) {
    // Análise simples dos melhores horários baseado em dados históricos
    const hourlyProduction = {};
    
    // Simular análise horária (em implementação real, usar dados detalhados)
    // Geralmente, 10h-14h são melhores para energia solar
    return ['10:00', '14:00'];
}
```

## 🔔 Sistema de Notificações Proativas

```javascript
class ProactiveNotificationService {
    constructor() {
        this.eventGateway = new AWS.AlexaEventGateway();
    }

    async sendProactiveNotification(userId, type, message) {
        try {
            const event = this.buildProactiveEvent(userId, type, message);
            await this.eventGateway.sendEvent(event).promise();
            console.log('Proactive notification sent successfully');
        } catch (error) {
            console.error('Error sending proactive notification:', error);
        }
    }

    buildProactiveEvent(userId, type, message) {
        const eventType = this.getEventType(type);
        
        return {
            event: {
                header: {
                    namespace: 'Alexa.ProactiveEvents',
                    name: eventType,
                    payloadVersion: '1',
                    messageId: generateMessageId()
                },
                payload: {
                    alert: {
                        type: type,
                        status: 'ACTIVE'
                    },
                    relevantAudience: {
                        type: 'Unicast',
                        payload: {
                            user: userId
                        }
                    },
                    expiry: {
                        expiryTime: new Date(Date.now() + 24 * 60 * 60 * 1000).toISOString()
                    }
                }
            }
        };
    }

    getEventType(type) {
        const eventMap = {
            'EMERGENCY': 'WeatherAlert',
            'WARNING': 'SportsEvent',
            'INFO': 'MessageAlert'
        };
        
        return eventMap[type] || 'MessageAlert';
    }
}
```

## 📊 Monitoramento Contínuo

```javascript
// Lambda function para monitoramento periódico
exports.monitoringHandler = async (event) => {
    console.log('Starting energy monitoring check...');
    
    try {
        const emergencyService = new EmergencyDetectionService();
        
        // Buscar todas as estações ativas
        const activeStations = await getActiveStations();
        
        for (const station of activeStations) {
            try {
                const alerts = await emergencyService.analyzeEnergyStatus(station.id);
                
                if (alerts.length > 0) {
                    console.log(`Alerts found for station ${station.id}:`, alerts);
                }
                
            } catch (error) {
                console.error(`Error monitoring station ${station.id}:`, error);
            }
        }
        
        console.log('Energy monitoring check completed');
        
    } catch (error) {
        console.error('Error in monitoring process:', error);
    }
};

// Configurar CloudWatch Event para executar a cada 15 minutos
const monitoringSchedule = {
    Type: 'AWS::Events::Rule',
    Properties: {
        ScheduleExpression: 'rate(15 minutes)',
        Targets: [{
            Arn: 'arn:aws:lambda:us-east-1:123456789:function:energy-monitoring',
            Id: 'EnergyMonitoringTarget'
        }]
    }
};
```

## ⚡ Próximos Passos

1. **Configure** o sistema de monitoramento automático
2. **Implemente** as ações de emergência
3. **Teste** cenários de emergência
4. **Avance** para [Energy Optimization](04-energy-optimization.md)

---
**Anterior:** [← Smart Home Integration](02-smart-home-integration.md) | **Próximo:** [Energy Optimization →](04-energy-optimization.md)
