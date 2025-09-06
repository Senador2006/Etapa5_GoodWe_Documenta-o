# Casos Avançados - Alexa Skills GoodWe

## 📋 Visão Geral

Este documento apresenta casos de uso avançados e integrações complexas para a Alexa Skill GoodWe, incluindo personalizações, otimizações e funcionalidades especializadas.

## 🎯 Caso 1: Sistema Multi-Usuário com Perfis

### Cenário
Sistema solar compartilhado entre múltiplos usuários com perfis personalizados e permissões diferenciadas.

### Implementação

#### 1. Gerenciamento de Perfis
```javascript
const ProfileManager = {
  profiles: new Map(),
  
  async getUserProfile(userId) {
    if (this.profiles.has(userId)) {
      return this.profiles.get(userId);
    }
    
    // Carregar perfil do banco de dados
    const profile = await this.loadProfileFromDB(userId);
    this.profiles.set(userId, profile);
    return profile;
  },
  
  async loadProfileFromDB(userId) {
    const response = await axios.get(`${API_CONFIG.goodwe.baseUrl}/users/${userId}/profile`);
    return response.data;
  },
  
  async updateProfile(userId, updates) {
    const profile = await this.getUserProfile(userId);
    const updatedProfile = { ...profile, ...updates };
    
    await axios.put(`${API_CONFIG.goodwe.baseUrl}/users/${userId}/profile`, updatedProfile);
    this.profiles.set(userId, updatedProfile);
    
    return updatedProfile;
  }
};

// Handler para comandos personalizados
const PersonalizedCommandHandler = {
  canHandle(handlerInput) {
    return Alexa.getRequestType(handlerInput.requestEnvelope) === 'IntentRequest'
      && Alexa.getIntentName(handlerInput.requestEnvelope) === 'PersonalizedCommand';
  },
  
  async handle(handlerInput) {
    const userId = Alexa.getUserId(handlerInput.requestEnvelope);
    const profile = await ProfileManager.getUserProfile(userId);
    
    // Personalizar resposta baseada no perfil
    const personalizedResponse = await this.createPersonalizedResponse(profile);
    
    return handlerInput.responseBuilder
      .speak(personalizedResponse.speech)
      .withSimpleCard(profile.name, personalizedResponse.card)
      .getResponse();
  },
  
  async createPersonalizedResponse(profile) {
    const { name, preferences, systemConfig, role } = profile;
    
    let speech = `Olá ${name}! `;
    
    // Personalizar baseado no papel do usuário
    switch (role) {
      case 'admin':
        speech += await this.getAdminDashboard();
        break;
      case 'monitor':
        speech += await this.getMonitorView();
        break;
      case 'guest':
        speech += await this.getGuestView();
        break;
    }
    
    // Personalizar baseado nas preferências
    if (preferences.notifications) {
      speech += await this.getNotificationSummary();
    }
    
    return {
      speech,
      card: this.formatPersonalizedCard(profile)
    };
  }
};
```

#### 2. Sistema de Permissões
```javascript
const PermissionManager = {
  permissions: {
    'admin': ['read', 'write', 'configure', 'monitor'],
    'monitor': ['read', 'monitor'],
    'guest': ['read']
  },
  
  hasPermission(userRole, action) {
    return this.permissions[userRole]?.includes(action) || false;
  },
  
  async checkPermission(userId, action) {
    const profile = await ProfileManager.getUserProfile(userId);
    return this.hasPermission(profile.role, action);
  }
};

// Middleware de autorização
const AuthorizationMiddleware = {
  async process(handlerInput, next) {
    const userId = Alexa.getUserId(handlerInput.requestEnvelope);
    const intentName = Alexa.getIntentName(handlerInput.requestEnvelope);
    
    const requiredPermission = this.getRequiredPermission(intentName);
    
    if (requiredPermission) {
      const hasPermission = await PermissionManager.checkPermission(userId, requiredPermission);
      
      if (!hasPermission) {
        return handlerInput.responseBuilder
          .speak('Desculpe, você não tem permissão para executar esta ação.')
          .getResponse();
      }
    }
    
    return next(handlerInput);
  },
  
  getRequiredPermission(intentName) {
    const permissionMap = {
      'GetSystemStatus': 'read',
      'SetBatteryMode': 'configure',
      'ActivateEmergencyMode': 'write',
      'GetUserList': 'monitor'
    };
    
    return permissionMap[intentName];
  }
};
```

## 🎯 Caso 2: Integração com IoT e Smart Home

### Cenário
Integração completa com dispositivos IoT para controle automatizado baseado em dados solares.

### Implementação

#### 1. Discovery de Dispositivos
```javascript
const SmartHomeHandler = {
  async handleDiscovery(handlerInput) {
    const userId = Alexa.getUserId(handlerInput.requestEnvelope);
    const devices = await this.discoverDevices(userId);
    
    const discoveryResponse = {
      event: {
        header: {
          namespace: 'Alexa.Discovery',
          name: 'Discover.Response',
          payloadVersion: '3',
          messageId: this.generateMessageId()
        },
        payload: {
          endpoints: devices.map(device => this.formatEndpoint(device))
        }
      }
    };
    
    return discoveryResponse;
  },
  
  async discoverDevices(userId) {
    // Buscar dispositivos do usuário
    const response = await axios.get(`${API_CONFIG.goodwe.baseUrl}/users/${userId}/devices`);
    return response.data.devices;
  },
  
  formatEndpoint(device) {
    return {
      endpointId: device.id,
      manufacturerName: 'GoodWe',
      description: device.description,
      friendlyName: device.name,
      displayCategories: [device.category],
      capabilities: this.getCapabilities(device),
      relationships: this.getRelationships(device)
    };
  },
  
  getCapabilities(device) {
    const capabilities = [];
    
    switch (device.type) {
      case 'inverter':
        capabilities.push({
          type: 'AlexaInterface',
          interface: 'Alexa.PowerController',
          properties: {
            supported: [{ name: 'powerState' }],
            proactivelyReported: true,
            retrievable: true
          }
        });
        break;
        
      case 'battery':
        capabilities.push({
          type: 'AlexaInterface',
          interface: 'Alexa.PercentageController',
          properties: {
            supported: [{ name: 'percentage' }],
            proactivelyReported: true,
            retrievable: true
          }
        });
        break;
        
      case 'load_controller':
        capabilities.push({
          type: 'AlexaInterface',
          interface: 'Alexa.PowerController',
          properties: {
            supported: [{ name: 'powerState' }],
            proactivelyReported: true,
            retrievable: true
          }
        });
        break;
    }
    
    return capabilities;
  }
};
```

#### 2. Controle Automatizado
```javascript
const AutomationEngine = {
  rules: new Map(),
  
  async processSolarData(solarData) {
    const rules = await this.getActiveRules();
    
    for (const rule of rules) {
      if (await this.evaluateRule(rule, solarData)) {
        await this.executeRule(rule, solarData);
      }
    }
  },
  
  async evaluateRule(rule, solarData) {
    const { conditions, logic } = rule;
    
    switch (logic) {
      case 'AND':
        return conditions.every(condition => this.evaluateCondition(condition, solarData));
      case 'OR':
        return conditions.some(condition => this.evaluateCondition(condition, solarData));
      default:
        return false;
    }
  },
  
  evaluateCondition(condition, solarData) {
    const { field, operator, value } = condition;
    const actualValue = this.getFieldValue(field, solarData);
    
    switch (operator) {
      case 'gt':
        return actualValue > value;
      case 'lt':
        return actualValue < value;
      case 'eq':
        return actualValue === value;
      case 'gte':
        return actualValue >= value;
      case 'lte':
        return actualValue <= value;
      default:
        return false;
    }
  },
  
  async executeRule(rule, solarData) {
    const { actions } = rule;
    
    for (const action of actions) {
      await this.executeAction(action, solarData);
    }
  },
  
  async executeAction(action, solarData) {
    switch (action.type) {
      case 'turn_on_device':
        await this.turnOnDevice(action.deviceId);
        break;
      case 'turn_off_device':
        await this.turnOffDevice(action.deviceId);
        break;
      case 'set_battery_mode':
        await this.setBatteryMode(action.mode);
        break;
      case 'send_notification':
        await this.sendNotification(action.message);
        break;
    }
  }
};

// Exemplo de regra de automação
const exampleRule = {
  id: 'high_generation_rule',
  name: 'Alta Geração Solar',
  conditions: [
    { field: 'fv_power', operator: 'gt', value: 3000 },
    { field: 'soc_percentage', operator: 'lt', value: 80 }
  ],
  logic: 'AND',
  actions: [
    { type: 'turn_on_device', deviceId: 'water_heater' },
    { type: 'set_battery_mode', mode: 'fast_charge' },
    { type: 'send_notification', message: 'Alta geração detectada! Carregando bateria e ligando aquecedor.' }
  ]
};
```

## 🎯 Caso 3: Análise Preditiva Avançada

### Cenário
Sistema de análise preditiva que combina dados históricos, clima e Machine Learning para otimização energética.

### Implementação

#### 1. Engine de Predição
```javascript
const PredictiveEngine = {
  models: new Map(),
  
  async initializeModels() {
    // Carregar modelos ML
    const models = await this.loadModels();
    
    for (const model of models) {
      this.models.set(model.type, model);
    }
  },
  
  async predictEnergyGeneration(weatherForecast, historicalData) {
    const model = this.models.get('energy_generation');
    
    if (!model) {
      throw new Error('Modelo de predição de geração não encontrado');
    }
    
    const features = this.prepareFeatures(weatherForecast, historicalData);
    const prediction = await this.runModel(model, features);
    
    return this.formatPrediction(prediction);
  },
  
  async predictEnergyConsumption(historicalData, userBehavior) {
    const model = this.models.get('energy_consumption');
    
    if (!model) {
      throw new Error('Modelo de predição de consumo não encontrado');
    }
    
    const features = this.prepareConsumptionFeatures(historicalData, userBehavior);
    const prediction = await this.runModel(model, features);
    
    return this.formatPrediction(prediction);
  },
  
  async optimizeEnergyFlow(currentState, predictions) {
    const optimizationModel = this.models.get('energy_optimization');
    
    if (!optimizationModel) {
      throw new Error('Modelo de otimização não encontrado');
    }
    
    const optimizationInput = {
      currentState,
      predictions,
      constraints: await this.getOptimizationConstraints()
    };
    
    const optimization = await this.runModel(optimizationModel, optimizationInput);
    
    return this.formatOptimization(optimization);
  },
  
  prepareFeatures(weatherForecast, historicalData) {
    return {
      // Dados climáticos
      temperature: weatherForecast.temperature,
      humidity: weatherForecast.humidity,
      cloud_cover: weatherForecast.cloud_cover,
      wind_speed: weatherForecast.wind_speed,
      
      // Dados históricos
      historical_generation: historicalData.generation,
      historical_consumption: historicalData.consumption,
      seasonal_patterns: historicalData.seasonal_patterns,
      
      // Características do sistema
      panel_efficiency: historicalData.panel_efficiency,
      system_age: historicalData.system_age,
      maintenance_history: historicalData.maintenance_history
    };
  }
};
```

#### 2. Sistema de Recomendações
```javascript
const RecommendationEngine = {
  async generateRecommendations(userProfile, systemData, predictions) {
    const recommendations = [];
    
    // Análise de eficiência
    const efficiencyAnalysis = await this.analyzeEfficiency(systemData);
    if (efficiencyAnalysis.score < 0.7) {
      recommendations.push({
        type: 'efficiency',
        priority: 'high',
        title: 'Otimizar Eficiência do Sistema',
        description: 'Sua eficiência está abaixo do ideal. Considere limpeza dos painéis.',
        actions: [
          'Agendar limpeza dos painéis',
          'Verificar sombreamento',
          'Revisar ângulo de inclinação'
        ]
      });
    }
    
    // Análise de consumo
    const consumptionAnalysis = await this.analyzeConsumption(systemData, predictions);
    if (consumptionAnalysis.peakHours.length > 0) {
      recommendations.push({
        type: 'consumption',
        priority: 'medium',
        title: 'Otimizar Horários de Consumo',
        description: 'Considere deslocar consumo para horários de maior geração.',
        actions: [
          'Programar equipamentos para horário solar',
          'Usar bateria durante picos de consumo',
          'Configurar automações inteligentes'
        ]
      });
    }
    
    // Análise de bateria
    const batteryAnalysis = await this.analyzeBattery(systemData);
    if (batteryAnalysis.health < 0.8) {
      recommendations.push({
        type: 'battery',
        priority: 'high',
        title: 'Manutenção da Bateria',
        description: 'A saúde da bateria está comprometida.',
        actions: [
          'Agendar manutenção da bateria',
          'Verificar conexões',
          'Calibrar sistema de monitoramento'
        ]
      });
    }
    
    return this.prioritizeRecommendations(recommendations);
  },
  
  prioritizeRecommendations(recommendations) {
    return recommendations.sort((a, b) => {
      const priorityOrder = { 'high': 3, 'medium': 2, 'low': 1 };
      return priorityOrder[b.priority] - priorityOrder[a.priority];
    });
  }
};
```

## 🎯 Caso 4: Sistema de Alertas Inteligentes

### Cenário
Sistema de alertas que combina múltiplas fontes de dados para notificações contextuais e personalizadas.

### Implementação

#### 1. Engine de Alertas
```javascript
const AlertEngine = {
  alertRules: new Map(),
  notificationChannels: new Map(),
  
  async initialize() {
    await this.loadAlertRules();
    await this.setupNotificationChannels();
  },
  
  async processData(systemData, weatherData, predictions) {
    const alerts = [];
    
    // Verificar regras de alerta
    for (const [ruleId, rule] of this.alertRules) {
      const alert = await this.evaluateAlertRule(rule, systemData, weatherData, predictions);
      if (alert) {
        alerts.push(alert);
      }
    }
    
    // Processar alertas
    for (const alert of alerts) {
      await this.processAlert(alert);
    }
  },
  
  async evaluateAlertRule(rule, systemData, weatherData, predictions) {
    const { conditions, severity, message, channels } = rule;
    
    // Verificar condições
    const isTriggered = await this.checkConditions(conditions, {
      system: systemData,
      weather: weatherData,
      predictions
    });
    
    if (isTriggered) {
      return {
        id: this.generateAlertId(),
        ruleId: rule.id,
        severity,
        message: this.formatMessage(message, systemData),
        channels,
        timestamp: new Date(),
        data: { systemData, weatherData, predictions }
      };
    }
    
    return null;
  },
  
  async processAlert(alert) {
    // Enviar para canais configurados
    for (const channel of alert.channels) {
      await this.sendToChannel(channel, alert);
    }
    
    // Log do alerta
    await this.logAlert(alert);
    
    // Ações automáticas se configuradas
    await this.executeAlertActions(alert);
  },
  
  async sendToChannel(channel, alert) {
    switch (channel.type) {
      case 'alexa':
        await this.sendAlexaNotification(alert);
        break;
      case 'email':
        await this.sendEmailNotification(alert);
        break;
      case 'sms':
        await this.sendSMSNotification(alert);
        break;
      case 'push':
        await this.sendPushNotification(alert);
        break;
    }
  }
};

// Exemplo de regra de alerta
const criticalBatteryRule = {
  id: 'critical_battery',
  name: 'Bateria Crítica',
  conditions: [
    { field: 'soc_percentage', operator: 'lt', value: 15 },
    { field: 'fv_power', operator: 'lt', value: 100 }
  ],
  severity: 'critical',
  message: 'Bateria com {soc_percentage}% de carga! Sistema pode desligar em breve.',
  channels: ['alexa', 'email', 'sms'],
  actions: [
    { type: 'activate_emergency_mode' },
    { type: 'notify_emergency_contacts' }
  ]
};
```

#### 2. Sistema de Notificações Contextuais
```javascript
const ContextualNotificationSystem = {
  async createContextualAlert(alert, userProfile) {
    const context = await this.analyzeContext(alert, userProfile);
    
    return {
      ...alert,
      contextualMessage: this.createContextualMessage(alert, context),
      suggestedActions: this.getSuggestedActions(alert, context),
      urgency: this.calculateUrgency(alert, context)
    };
  },
  
  async analyzeContext(alert, userProfile) {
    const { location, timezone, preferences, systemConfig } = userProfile;
    
    return {
      timeOfDay: this.getTimeOfDay(timezone),
      weatherConditions: await this.getCurrentWeather(location),
      systemStatus: await this.getSystemStatus(),
      userActivity: await this.getUserActivity(userProfile.id),
      historicalPatterns: await this.getHistoricalPatterns(userProfile.id)
    };
  },
  
  createContextualMessage(alert, context) {
    let message = alert.message;
    
    // Personalizar baseado no horário
    if (context.timeOfDay === 'night') {
      message += ' Recomendo verificar o sistema pela manhã.';
    }
    
    // Personalizar baseado no clima
    if (context.weatherConditions.rain) {
      message += ' Cuidado com equipamentos elétricos devido à chuva.';
    }
    
    // Personalizar baseado na atividade do usuário
    if (context.userActivity === 'away') {
      message += ' Você está fora de casa. Sistema em modo de monitoramento.';
    }
    
    return message;
  }
};
```

## 🎯 Caso 5: Integração com Sistemas Externos

### Cenário
Integração com sistemas de terceiros como concessionárias de energia, serviços de clima e plataformas de IoT.

### Implementação

#### 1. Integração com Concessionária
```javascript
const UtilityIntegration = {
  async getElectricityRates(region) {
    const response = await axios.get(`${API_CONFIG.utility.baseUrl}/rates`, {
      params: { region },
      headers: {
        'Authorization': `Bearer ${process.env.UTILITY_API_KEY}`
      }
    });
    
    return response.data;
  },
  
  async getTimeOfUseRates() {
    const response = await axios.get(`${API_CONFIG.utility.baseUrl}/time-of-use`);
    return response.data;
  },
  
  async calculateSavings(solarGeneration, consumption, rates) {
    const savings = {
      total: 0,
      breakdown: {
        peak: 0,
        offPeak: 0,
        shoulder: 0
      }
    };
    
    for (const hour of consumption.hours) {
      const rate = this.getRateForHour(hour, rates);
      const solarUsed = Math.min(solarGeneration[hour], consumption[hour]);
      const gridUsed = Math.max(0, consumption[hour] - solarUsed);
      
      const hourSavings = gridUsed * rate;
      savings.total += hourSavings;
      savings.breakdown[rate.period] += hourSavings;
    }
    
    return savings;
  }
};
```

#### 2. Integração com Serviços de Clima
```javascript
const WeatherIntegration = {
  providers: ['openweathermap', 'accuweather', 'weatherbit'],
  
  async getWeatherData(location, provider = 'openweathermap') {
    const config = this.getProviderConfig(provider);
    
    const response = await axios.get(config.endpoint, {
      params: {
        ...config.params,
        lat: location.lat,
        lon: location.lon
      },
      headers: {
        'Authorization': `Bearer ${config.apiKey}`
      }
    });
    
    return this.normalizeWeatherData(response.data, provider);
  },
  
  async getWeatherForecast(location, days = 7) {
    const forecasts = [];
    
    for (const provider of this.providers) {
      try {
        const forecast = await this.getWeatherData(location, provider);
        forecasts.push(forecast);
      } catch (error) {
        console.error(`Erro ao obter dados do ${provider}:`, error);
      }
    }
    
    return this.consolidateForecasts(forecasts);
  },
  
  consolidateForecasts(forecasts) {
    // Usar média ponderada dos diferentes provedores
    const consolidated = {};
    
    for (const field of ['temperature', 'humidity', 'precipitation', 'wind_speed']) {
      const values = forecasts.map(f => f[field]).filter(v => v !== undefined);
      consolidated[field] = values.reduce((a, b) => a + b, 0) / values.length;
    }
    
    return consolidated;
  }
};
```

## 🎯 Caso 6: Sistema de Backup e Recuperação

### Cenário
Sistema robusto de backup e recuperação para garantir continuidade do serviço.

### Implementação

#### 1. Sistema de Backup
```javascript
const BackupSystem = {
  async createBackup(type = 'full') {
    const backupId = this.generateBackupId();
    const timestamp = new Date().toISOString();
    
    const backup = {
      id: backupId,
      type,
      timestamp,
      status: 'in_progress',
      components: []
    };
    
    try {
      // Backup da configuração da skill
      const skillConfig = await this.backupSkillConfig();
      backup.components.push(skillConfig);
      
      // Backup dos dados do usuário
      const userData = await this.backupUserData();
      backup.components.push(userData);
      
      // Backup das regras de automação
      const automationRules = await this.backupAutomationRules();
      backup.components.push(automationRules);
      
      // Backup dos modelos ML
      const mlModels = await this.backupMLModels();
      backup.components.push(mlModels);
      
      backup.status = 'completed';
      await this.saveBackup(backup);
      
      return backup;
    } catch (error) {
      backup.status = 'failed';
      backup.error = error.message;
      await this.saveBackup(backup);
      throw error;
    }
  },
  
  async restoreBackup(backupId) {
    const backup = await this.loadBackup(backupId);
    
    if (!backup) {
      throw new Error('Backup não encontrado');
    }
    
    if (backup.status !== 'completed') {
      throw new Error('Backup não está completo');
    }
    
    try {
      // Restaurar cada componente
      for (const component of backup.components) {
        await this.restoreComponent(component);
      }
      
      return { success: true, message: 'Backup restaurado com sucesso' };
    } catch (error) {
      return { success: false, error: error.message };
    }
  }
};
```

## 🧪 Testes dos Casos Avançados

### Script de Teste Completo
```javascript
// test-advanced-cases.js
const { handler } = require('./index');

const advancedTestCases = [
  {
    name: 'Sistema Multi-Usuário',
    test: async () => {
      // Testar diferentes perfis de usuário
      const adminRequest = createTestRequest('GetSystemStatus', 'admin-user');
      const guestRequest = createTestRequest('SetBatteryMode', 'guest-user');
      
      const adminResponse = await handler(adminRequest);
      const guestResponse = await handler(guestRequest);
      
      return adminResponse.response.outputSpeech && 
             guestResponse.response.outputSpeech.includes('permissão');
    }
  },
  {
    name: 'Integração IoT',
    test: async () => {
      // Testar descoberta de dispositivos
      const discoveryRequest = createTestRequest('DiscoverDevices');
      const response = await handler(discoveryRequest);
      
      return response.event && response.event.payload.endpoints.length > 0;
    }
  },
  {
    name: 'Análise Preditiva',
    test: async () => {
      // Testar predições
      const predictionRequest = createTestRequest('GetEnergyPrediction');
      const response = await handler(predictionRequest);
      
      return response.response.outputSpeech.includes('predição') ||
             response.response.outputSpeech.includes('previsão');
    }
  }
];

async function runAdvancedTests() {
  console.log('🧪 Testando casos avançados...\n');
  
  for (const testCase of advancedTestCases) {
    try {
      const result = await testCase.test();
      console.log(`${result ? '✅' : '❌'} ${testCase.name}`);
    } catch (error) {
      console.log(`❌ ${testCase.name}: ${error.message}`);
    }
  }
}

runAdvancedTests();
```

---

**Próximo**: [Troubleshooting](./06-troubleshooting.md)
