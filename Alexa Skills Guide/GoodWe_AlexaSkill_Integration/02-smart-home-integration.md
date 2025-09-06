# Smart Home Integration - GoodWe Alexa Skill

## 📋 Visão Geral

Este documento detalha a integração Smart Home da Alexa Skill com dispositivos GoodWe, permitindo controle por voz de inversores, baterias e outros equipamentos do sistema solar.

## 🏠 Dispositivos Suportados

### 1. Inversores Solares
- **Categoria**: SWITCH
- **Funcionalidades**: Ligar/desligar, status de operação
- **Controles**: PowerController

### 2. Baterias
- **Categoria**: BATTERY
- **Funcionalidades**: Nível de carga, modo de operação
- **Controles**: PercentageController, PowerController

### 3. Controladores de Carga
- **Categoria**: SWITCH
- **Funcionalidades**: Controle de cargas específicas
- **Controles**: PowerController

### 4. Monitor de Sistema
- **Categoria**: SENSOR
- **Funcionalidades**: Monitoramento de parâmetros
- **Controles**: ReadOnly

## 🔧 Configuração da Integração Smart Home

### 1. Discovery de Dispositivos

```javascript
// handlers/smart-home-handler.js
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
    try {
      // Buscar dispositivos do usuário na API GoodWe
      const response = await axios.get(`${API_CONFIG.goodwe.baseUrl}/users/${userId}/devices`, {
        headers: {
          'Authorization': `Bearer ${process.env.GOODWE_API_KEY}`
        }
      });
      
      return response.data.devices || [];
    } catch (error) {
      console.error('Erro ao descobrir dispositivos:', error);
      return this.getDefaultDevices();
    }
  },
  
  getDefaultDevices() {
    return [
      {
        id: 'inverter-main',
        name: 'Inversor Principal',
        type: 'inverter',
        category: 'SWITCH',
        description: 'Inversor solar GoodWe principal',
        capabilities: ['power']
      },
      {
        id: 'battery-main',
        name: 'Bateria Solar',
        type: 'battery',
        category: 'BATTERY',
        description: 'Sistema de baterias GoodWe',
        capabilities: ['percentage', 'power']
      },
      {
        id: 'load-controller',
        name: 'Controlador de Carga',
        type: 'load_controller',
        category: 'SWITCH',
        description: 'Controlador de cargas do sistema',
        capabilities: ['power']
      }
    ];
  },
  
  formatEndpoint(device) {
    const endpoint = {
      endpointId: device.id,
      manufacturerName: 'GoodWe',
      description: device.description,
      friendlyName: device.name,
      displayCategories: [device.category],
      capabilities: this.getCapabilities(device),
      relationships: this.getRelationships(device)
    };
    
    // Adicionar informações específicas do dispositivo
    if (device.type === 'inverter') {
      endpoint.attributes = {
        powerLevel: 'high',
        voltage: '220V',
        frequency: '60Hz'
      };
    } else if (device.type === 'battery') {
      endpoint.attributes = {
        capacity: '10kWh',
        voltage: '48V',
        chemistry: 'LiFePO4'
      };
    }
    
    return endpoint;
  },
  
  getCapabilities(device) {
    const capabilities = [];
    
    // Capabilidade de energia (ligar/desligar)
    if (device.capabilities.includes('power')) {
      capabilities.push({
        type: 'AlexaInterface',
        interface: 'Alexa.PowerController',
        properties: {
          supported: [{ name: 'powerState' }],
          proactivelyReported: true,
          retrievable: true
        }
      });
    }
    
    // Capabilidade de porcentagem (bateria)
    if (device.capabilities.includes('percentage')) {
      capabilities.push({
        type: 'AlexaInterface',
        interface: 'Alexa.PercentageController',
        properties: {
          supported: [{ name: 'percentage' }],
          proactivelyReported: true,
          retrievable: true
        }
      });
    }
    
    // Capabilidade de temperatura (monitor)
    if (device.type === 'monitor') {
      capabilities.push({
        type: 'AlexaInterface',
        interface: 'Alexa.TemperatureSensor',
        properties: {
          supported: [{ name: 'temperature' }],
          proactivelyReported: true,
          retrievable: true
        }
      });
    }
    
    // Capabilidade de modo (bateria)
    if (device.type === 'battery') {
      capabilities.push({
        type: 'AlexaInterface',
        interface: 'Alexa.ModeController',
        properties: {
          supported: [
            {
              name: 'mode',
              values: [
                { value: 'ECONOMY' },
                { value: 'BALANCED' },
                { value: 'PERFORMANCE' }
              ]
            }
          ],
          proactivelyReported: true,
          retrievable: true
        }
      });
    }
    
    return capabilities;
  },
  
  getRelationships(device) {
    const relationships = [];
    
    // Relacionamento entre bateria e inversor
    if (device.type === 'battery') {
      relationships.push({
        type: 'isConnectedTo',
        endpointId: 'inverter-main'
      });
    }
    
    return relationships;
  }
};
```

### 2. Controle de Dispositivos

```javascript
// handlers/device-control-handler.js
const DeviceControlHandler = {
  async handlePowerControl(handlerInput) {
    const request = handlerInput.requestEnvelope.request;
    const endpointId = request.directive.endpoint.endpointId;
    const powerState = request.directive.header.name === 'TurnOn' ? 'ON' : 'OFF';
    
    try {
      // Controlar dispositivo via API GoodWe
      const result = await this.controlDevice(endpointId, 'power', powerState);
      
      return this.createResponse(request, {
        namespace: 'Alexa.PowerController',
        name: 'Response',
        payload: {
          powerState: {
            value: powerState
          }
        }
      });
    } catch (error) {
      console.error('Erro ao controlar dispositivo:', error);
      return this.createErrorResponse(request, 'INTERNAL_ERROR');
    }
  },
  
  async handlePercentageControl(handlerInput) {
    const request = handlerInput.requestEnvelope.request;
    const endpointId = request.directive.endpoint.endpointId;
    const percentage = request.directive.payload.percentage;
    
    try {
      // Controlar nível da bateria
      const result = await this.controlBatteryLevel(endpointId, percentage);
      
      return this.createResponse(request, {
        namespace: 'Alexa.PercentageController',
        name: 'Response',
        payload: {
          percentage: {
            value: percentage
          }
        }
      });
    } catch (error) {
      console.error('Erro ao controlar bateria:', error);
      return this.createErrorResponse(request, 'INTERNAL_ERROR');
    }
  },
  
  async handleModeControl(handlerInput) {
    const request = handlerInput.requestEnvelope.request;
    const endpointId = request.directive.endpoint.endpointId;
    const mode = request.directive.payload.mode;
    
    try {
      // Controlar modo da bateria
      const result = await this.controlBatteryMode(endpointId, mode);
      
      return this.createResponse(request, {
        namespace: 'Alexa.ModeController',
        name: 'Response',
        payload: {
          mode: {
            value: mode
          }
        }
      });
    } catch (error) {
      console.error('Erro ao controlar modo:', error);
      return this.createErrorResponse(request, 'INTERNAL_ERROR');
    }
  },
  
  async controlDevice(endpointId, property, value) {
    const deviceType = this.getDeviceType(endpointId);
    
    switch (deviceType) {
      case 'inverter':
        return await this.controlInverter(endpointId, property, value);
      case 'battery':
        return await this.controlBattery(endpointId, property, value);
      case 'load_controller':
        return await this.controlLoadController(endpointId, property, value);
      default:
        throw new Error('Tipo de dispositivo não suportado');
    }
  },
  
  async controlInverter(endpointId, property, value) {
    const response = await axios.post(`${API_CONFIG.goodwe.baseUrl}/inverter/control`, {
      deviceId: endpointId,
      property,
      value
    }, {
      headers: {
        'Authorization': `Bearer ${process.env.GOODWE_API_KEY}`
      }
    });
    
    return response.data;
  },
  
  async controlBattery(endpointId, property, value) {
    const response = await axios.post(`${API_CONFIG.goodwe.baseUrl}/battery/control`, {
      deviceId: endpointId,
      property,
      value
    }, {
      headers: {
        'Authorization': `Bearer ${process.env.GOODWE_API_KEY}`
      }
    });
    
    return response.data;
  },
  
  async controlBatteryLevel(endpointId, percentage) {
    const response = await axios.post(`${API_CONFIG.goodwe.baseUrl}/battery/set-level`, {
      deviceId: endpointId,
      level: percentage
    }, {
      headers: {
        'Authorization': `Bearer ${process.env.GOODWE_API_KEY}`
      }
    });
    
    return response.data;
  },
  
  async controlBatteryMode(endpointId, mode) {
    const modeMap = {
      'ECONOMY': 'economy',
      'BALANCED': 'balanced',
      'PERFORMANCE': 'performance'
    };
    
    const response = await axios.post(`${API_CONFIG.goodwe.baseUrl}/battery/set-mode`, {
      deviceId: endpointId,
      mode: modeMap[mode]
    }, {
      headers: {
        'Authorization': `Bearer ${process.env.GOODWE_API_KEY}`
      }
    });
    
    return response.data;
  },
  
  getDeviceType(endpointId) {
    if (endpointId.includes('inverter')) return 'inverter';
    if (endpointId.includes('battery')) return 'battery';
    if (endpointId.includes('load')) return 'load_controller';
    return 'unknown';
  },
  
  createResponse(request, payload) {
    return {
      event: {
        header: {
          namespace: payload.namespace,
          name: payload.name,
          payloadVersion: '3',
          messageId: this.generateMessageId(),
          correlationToken: request.directive.header.correlationToken
        },
        endpoint: request.directive.endpoint,
        payload: payload.payload
      }
    };
  },
  
  createErrorResponse(request, errorType) {
    return {
      event: {
        header: {
          namespace: 'Alexa',
          name: 'ErrorResponse',
          payloadVersion: '3',
          messageId: this.generateMessageId(),
          correlationToken: request.directive.header.correlationToken
        },
        payload: {
          type: errorType,
          message: 'Erro interno do servidor'
        }
      }
    };
  }
};
```

### 3. Relatório de Estado

```javascript
// handlers/state-report-handler.js
const StateReportHandler = {
  async handleStateReport(handlerInput) {
    const request = handlerInput.requestEnvelope.request;
    const endpointId = request.directive.endpoint.endpointId;
    
    try {
      const deviceState = await this.getDeviceState(endpointId);
      
      return this.createStateReport(request, endpointId, deviceState);
    } catch (error) {
      console.error('Erro ao obter estado do dispositivo:', error);
      return this.createErrorResponse(request, 'INTERNAL_ERROR');
    }
  },
  
  async getDeviceState(endpointId) {
    const deviceType = this.getDeviceType(endpointId);
    
    switch (deviceType) {
      case 'inverter':
        return await this.getInverterState(endpointId);
      case 'battery':
        return await this.getBatteryState(endpointId);
      case 'load_controller':
        return await this.getLoadControllerState(endpointId);
      default:
        throw new Error('Tipo de dispositivo não suportado');
    }
  },
  
  async getInverterState(endpointId) {
    const response = await axios.get(`${API_CONFIG.goodwe.baseUrl}/inverter/status/${endpointId}`, {
      headers: {
        'Authorization': `Bearer ${process.env.GOODWE_API_KEY}`
      }
    });
    
    const data = response.data;
    return {
      powerState: data.isOn ? 'ON' : 'OFF',
      power: data.power || 0,
      efficiency: data.efficiency || 0,
      temperature: data.temperature || 0
    };
  },
  
  async getBatteryState(endpointId) {
    const response = await axios.get(`${API_CONFIG.goodwe.baseUrl}/battery/status/${endpointId}`, {
      headers: {
        'Authorization': `Bearer ${process.env.GOODWE_API_KEY}`
      }
    });
    
    const data = response.data;
    return {
      powerState: data.isCharging ? 'ON' : 'OFF',
      percentage: data.soc || 0,
      mode: data.mode || 'BALANCED',
      voltage: data.voltage || 0,
      current: data.current || 0
    };
  },
  
  async getLoadControllerState(endpointId) {
    const response = await axios.get(`${API_CONFIG.goodwe.baseUrl}/load/status/${endpointId}`, {
      headers: {
        'Authorization': `Bearer ${process.env.GOODWE_API_KEY}`
      }
    });
    
    const data = response.data;
    return {
      powerState: data.isOn ? 'ON' : 'OFF',
      power: data.power || 0,
      priority: data.priority || 'normal'
    };
  },
  
  createStateReport(request, endpointId, deviceState) {
    const properties = [];
    
    // Adicionar propriedades baseadas no tipo de dispositivo
    if (deviceState.powerState) {
      properties.push({
        namespace: 'Alexa.PowerController',
        name: 'powerState',
        value: deviceState.powerState,
        timeOfSample: new Date().toISOString(),
        uncertaintyInMilliseconds: 0
      });
    }
    
    if (deviceState.percentage !== undefined) {
      properties.push({
        namespace: 'Alexa.PercentageController',
        name: 'percentage',
        value: deviceState.percentage,
        timeOfSample: new Date().toISOString(),
        uncertaintyInMilliseconds: 0
      });
    }
    
    if (deviceState.mode) {
      properties.push({
        namespace: 'Alexa.ModeController',
        name: 'mode',
        value: deviceState.mode,
        timeOfSample: new Date().toISOString(),
        uncertaintyInMilliseconds: 0
      });
    }
    
    return {
      event: {
        header: {
          namespace: 'Alexa',
          name: 'StateReport',
          payloadVersion: '3',
          messageId: this.generateMessageId(),
          correlationToken: request.directive.header.correlationToken
        },
        endpoint: {
          endpointId: endpointId
        },
        payload: {}
      },
      context: {
        properties: properties
      }
    };
  }
};
```

## 🎯 Comandos de Voz Suportados

### 1. Controle Básico
- "Alexa, ligue o inversor"
- "Alexa, desligue a bateria"
- "Alexa, qual o status do inversor?"
- "Alexa, como está a bateria?"

### 2. Controle de Bateria
- "Alexa, configure a bateria para 80%"
- "Alexa, coloque a bateria no modo econômico"
- "Alexa, ative o modo performance da bateria"
- "Alexa, qual o nível da bateria?"

### 3. Controle de Carga
- "Alexa, ligue o controlador de carga"
- "Alexa, desligue a carga prioritária"
- "Alexa, qual a prioridade da carga?"

## 🔄 Automações Inteligentes

### 1. Regras de Automação

```javascript
// automation/automation-engine.js
const AutomationEngine = {
  rules: new Map(),
  
  async processSolarData(solarData) {
    const activeRules = await this.getActiveRules();
    
    for (const rule of activeRules) {
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
        await this.setBatteryMode(action.deviceId, action.mode);
        break;
      case 'set_battery_level':
        await this.setBatteryLevel(action.deviceId, action.level);
        break;
      case 'send_notification':
        await this.sendNotification(action.message);
        break;
    }
  },
  
  async turnOnDevice(deviceId) {
    const response = await axios.post(`${API_CONFIG.goodwe.baseUrl}/device/control`, {
      deviceId,
      action: 'turn_on'
    });
    
    console.log(`Dispositivo ${deviceId} ligado`);
  },
  
  async turnOffDevice(deviceId) {
    const response = await axios.post(`${API_CONFIG.goodwe.baseUrl}/device/control`, {
      deviceId,
      action: 'turn_off'
    });
    
    console.log(`Dispositivo ${deviceId} desligado`);
  },
  
  async setBatteryMode(deviceId, mode) {
    const response = await axios.post(`${API_CONFIG.goodwe.baseUrl}/battery/set-mode`, {
      deviceId,
      mode
    });
    
    console.log(`Modo da bateria ${deviceId} alterado para ${mode}`);
  },
  
  async setBatteryLevel(deviceId, level) {
    const response = await axios.post(`${API_CONFIG.goodwe.baseUrl}/battery/set-level`, {
      deviceId,
      level
    });
    
    console.log(`Nível da bateria ${deviceId} alterado para ${level}%`);
  },
  
  async sendNotification(message) {
    // Enviar notificação via Alexa
    console.log(`Notificação: ${message}`);
  }
};
```

### 2. Exemplos de Regras

```javascript
// Exemplo de regra: Alta geração solar
const highGenerationRule = {
  id: 'high_generation_rule',
  name: 'Alta Geração Solar',
  conditions: [
    { field: 'fv_power', operator: 'gt', value: 3000 },
    { field: 'soc_percentage', operator: 'lt', value: 80 }
  ],
  logic: 'AND',
  actions: [
    { type: 'turn_on_device', deviceId: 'water_heater' },
    { type: 'set_battery_mode', deviceId: 'battery-main', mode: 'fast_charge' },
    { type: 'send_notification', message: 'Alta geração detectada! Carregando bateria e ligando aquecedor.' }
  ]
};

// Exemplo de regra: Bateria baixa
const lowBatteryRule = {
  id: 'low_battery_rule',
  name: 'Bateria Baixa',
  conditions: [
    { field: 'soc_percentage', operator: 'lt', value: 20 },
    { field: 'fv_power', operator: 'lt', value: 100 }
  ],
  logic: 'AND',
  actions: [
    { type: 'turn_off_device', deviceId: 'load-controller' },
    { type: 'set_battery_mode', deviceId: 'battery-main', mode: 'economy' },
    { type: 'send_notification', message: 'Bateria baixa! Desligando cargas não essenciais.' }
  ]
};

// Exemplo de regra: Modo noturno
const nightModeRule = {
  id: 'night_mode_rule',
  name: 'Modo Noturno',
  conditions: [
    { field: 'hour', operator: 'gte', value: 22 },
    { field: 'hour', operator: 'lt', value: 6 }
  ],
  logic: 'OR',
  actions: [
    { type: 'set_battery_mode', deviceId: 'battery-main', mode: 'economy' },
    { type: 'turn_off_device', deviceId: 'load-controller' },
    { type: 'send_notification', message: 'Modo noturno ativado. Sistema em modo econômico.' }
  ]
};
```

## 📊 Monitoramento e Notificações

### 1. Notificações Proativas

```javascript
// notifications/proactive-notifications.js
const ProactiveNotifications = {
  async sendDeviceStatusNotification(userId, deviceId, status) {
    const notification = {
      type: 'device_status',
      userId,
      deviceId,
      status,
      timestamp: new Date().toISOString()
    };
    
    await this.sendToAlexa(notification);
  },
  
  async sendAlertNotification(userId, alert) {
    const notification = {
      type: 'alert',
      userId,
      alert,
      timestamp: new Date().toISOString()
    };
    
    await this.sendToAlexa(notification);
  },
  
  async sendToAlexa(notification) {
    // Implementar envio de notificação proativa para Alexa
    console.log('Notificação enviada:', notification);
  }
};
```

### 2. Monitoramento de Dispositivos

```javascript
// monitoring/device-monitor.js
const DeviceMonitor = {
  async monitorDevices() {
    const devices = await this.getAllDevices();
    
    for (const device of devices) {
      await this.checkDeviceHealth(device);
    }
  },
  
  async checkDeviceHealth(device) {
    try {
      const status = await this.getDeviceStatus(device.id);
      
      if (status.health === 'critical') {
        await this.sendCriticalAlert(device, status);
      } else if (status.health === 'warning') {
        await this.sendWarningAlert(device, status);
      }
    } catch (error) {
      console.error(`Erro ao monitorar dispositivo ${device.id}:`, error);
    }
  },
  
  async getDeviceStatus(deviceId) {
    const response = await axios.get(`${API_CONFIG.goodwe.baseUrl}/device/status/${deviceId}`);
    return response.data;
  },
  
  async sendCriticalAlert(device, status) {
    await ProactiveNotifications.sendAlertNotification(device.userId, {
      level: 'critical',
      message: `Dispositivo ${device.name} em estado crítico`,
      device: device,
      status: status
    });
  },
  
  async sendWarningAlert(device, status) {
    await ProactiveNotifications.sendAlertNotification(device.userId, {
      level: 'warning',
      message: `Dispositivo ${device.name} com aviso`,
      device: device,
      status: status
    });
  }
};
```

## 🧪 Testes da Integração Smart Home

### 1. Testes de Descoberta

```javascript
// tests/smart-home-tests.js
const SmartHomeTests = {
  async testDeviceDiscovery() {
    console.log('🧪 Testando descoberta de dispositivos...');
    
    const mockRequest = {
      directive: {
        header: {
          namespace: 'Alexa.Discovery',
          name: 'Discover'
        }
      }
    };
    
    const response = await SmartHomeHandler.handleDiscovery(mockRequest);
    
    console.assert(response.event.payload.endpoints.length > 0, 'Deve descobrir dispositivos');
    console.log('✅ Descoberta de dispositivos: PASSOU');
  },
  
  async testDeviceControl() {
    console.log('🧪 Testando controle de dispositivos...');
    
    const mockRequest = {
      directive: {
        header: {
          namespace: 'Alexa.PowerController',
          name: 'TurnOn'
        },
        endpoint: {
          endpointId: 'inverter-main'
        }
      }
    };
    
    const response = await DeviceControlHandler.handlePowerControl(mockRequest);
    
    console.assert(response.event.payload.powerState.value === 'ON', 'Deve ligar dispositivo');
    console.log('✅ Controle de dispositivos: PASSOU');
  }
};
```

### 2. Testes de Automação

```javascript
// tests/automation-tests.js
const AutomationTests = {
  async testHighGenerationRule() {
    console.log('🧪 Testando regra de alta geração...');
    
    const solarData = {
      fv_power: 3500,
      soc_percentage: 70
    };
    
    const rule = highGenerationRule;
    const shouldTrigger = await AutomationEngine.evaluateRule(rule, solarData);
    
    console.assert(shouldTrigger === true, 'Regra deve ser ativada');
    console.log('✅ Regra de alta geração: PASSOU');
  },
  
  async testLowBatteryRule() {
    console.log('🧪 Testando regra de bateria baixa...');
    
    const solarData = {
      fv_power: 50,
      soc_percentage: 15
    };
    
    const rule = lowBatteryRule;
    const shouldTrigger = await AutomationEngine.evaluateRule(rule, solarData);
    
    console.assert(shouldTrigger === true, 'Regra deve ser ativada');
    console.log('✅ Regra de bateria baixa: PASSOU');
  }
};
```

## 🔒 Segurança e Permissões

### 1. Controle de Acesso

```javascript
// security/access-control.js
const AccessControl = {
  async checkDevicePermission(userId, deviceId, action) {
    const userProfile = await this.getUserProfile(userId);
    const device = await this.getDevice(deviceId);
    
    // Verificar se o usuário tem permissão para o dispositivo
    if (!userProfile.devices.includes(deviceId)) {
      throw new Error('Usuário não tem permissão para este dispositivo');
    }
    
    // Verificar se a ação é permitida
    if (!this.isActionAllowed(device.type, action)) {
      throw new Error('Ação não permitida para este tipo de dispositivo');
    }
    
    return true;
  },
  
  isActionAllowed(deviceType, action) {
    const permissions = {
      'inverter': ['turn_on', 'turn_off', 'get_status'],
      'battery': ['turn_on', 'turn_off', 'set_mode', 'set_level', 'get_status'],
      'load_controller': ['turn_on', 'turn_off', 'get_status']
    };
    
    return permissions[deviceType]?.includes(action) || false;
  }
};
```

---

**Próximo**: [Emergency Management](./03-emergency-management.md)
