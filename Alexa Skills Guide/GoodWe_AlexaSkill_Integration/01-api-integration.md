# 1. Integração com APIs GoodWe

## 🔗 Visão Geral da Integração

Esta seção detalha como integrar a Alexa Skill com as APIs da GoodWe para acessar dados de inversores solares, sistemas de armazenamento e dispositivos IoT.

## 🏗️ Arquitetura da API

### Endpoints Principais GoodWe
```javascript
const GOODWE_API_CONFIG = {
    baseUrl: 'https://www.goodwe-power.com/api',
    auth: {
        loginUrl: '/v1/Common/CrossLogin',
        tokenUrl: '/v1/oauth/token'
    },
    devices: {
        listUrl: '/v1/PowerStationMonitor/GetMonitorDetailByPowerstationId',
        realtimeUrl: '/v1/PowerStationMonitor/GetRealtimeDataByPowerstationId',
        historyUrl: '/v1/PowerStationMonitor/GetHistoryDataByPowerstationId'
    },
    control: {
        deviceControlUrl: '/v1/Device/Control',
        settingsUrl: '/v1/Device/Settings'
    }
};
```

### Estrutura de Autenticação
```javascript
class GoodWeAuthService {
    constructor() {
        this.baseUrl = process.env.GOODWE_API_BASE_URL;
        this.clientId = process.env.GOODWE_CLIENT_ID;
        this.clientSecret = process.env.GOODWE_CLIENT_SECRET;
        this.accessToken = null;
        this.refreshToken = null;
        this.tokenExpiry = null;
    }

    async authenticate(username, password) {
        try {
            const loginResponse = await fetch(`${this.baseUrl}/v1/Common/CrossLogin`, {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json'
                },
                body: JSON.stringify({
                    account: username,
                    pwd: password,
                    api: 'v1'
                })
            });

            const loginData = await loginResponse.json();
            
            if (loginData.code !== 0) {
                throw new Error(`Login failed: ${loginData.msg}`);
            }

            // Obter token OAuth
            const tokenResponse = await fetch(`${this.baseUrl}/v1/oauth/token`, {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/x-www-form-urlencoded'
                },
                body: new URLSearchParams({
                    grant_type: 'authorization_code',
                    client_id: this.clientId,
                    client_secret: this.clientSecret,
                    code: loginData.data.uid
                })
            });

            const tokenData = await tokenResponse.json();
            
            this.accessToken = tokenData.access_token;
            this.refreshToken = tokenData.refresh_token;
            this.tokenExpiry = Date.now() + (tokenData.expires_in * 1000);

            // Salvar tokens no DynamoDB para persistência
            await this.saveTokens();

            return this.accessToken;
            
        } catch (error) {
            console.error('Authentication error:', error);
            throw error;
        }
    }

    async refreshAccessToken() {
        if (!this.refreshToken) {
            throw new Error('No refresh token available');
        }

        try {
            const response = await fetch(`${this.baseUrl}/v1/oauth/token`, {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/x-www-form-urlencoded'
                },
                body: new URLSearchParams({
                    grant_type: 'refresh_token',
                    refresh_token: this.refreshToken,
                    client_id: this.clientId,
                    client_secret: this.clientSecret
                })
            });

            const tokenData = await response.json();
            
            this.accessToken = tokenData.access_token;
            this.tokenExpiry = Date.now() + (tokenData.expires_in * 1000);

            await this.saveTokens();
            return this.accessToken;

        } catch (error) {
            console.error('Token refresh error:', error);
            throw error;
        }
    }

    async getValidToken() {
        // Verificar se token ainda é válido
        if (this.accessToken && Date.now() < this.tokenExpiry - 60000) { // 1 min buffer
            return this.accessToken;
        }

        // Tentar renovar token
        if (this.refreshToken) {
            return await this.refreshAccessToken();
        }

        // Reautenticar se necessário
        throw new Error('Authentication required');
    }

    async saveTokens() {
        const AWS = require('aws-sdk');
        const dynamodb = new AWS.DynamoDB.DocumentClient();

        await dynamodb.put({
            TableName: 'GoodWeTokens',
            Item: {
                service: 'goodwe_api',
                accessToken: this.accessToken,
                refreshToken: this.refreshToken,
                tokenExpiry: this.tokenExpiry,
                updatedAt: Date.now()
            }
        }).promise();
    }
}
```

## 📊 Classe de Dados de Energia

```javascript
class GoodWeDataService {
    constructor(authService) {
        this.auth = authService;
        this.baseUrl = process.env.GOODWE_API_BASE_URL;
        this.cache = new Map();
        this.CACHE_TTL = 5 * 60 * 1000; // 5 minutos
    }

    async makeAuthenticatedRequest(endpoint, options = {}) {
        const token = await this.auth.getValidToken();
        
        const response = await fetch(`${this.baseUrl}${endpoint}`, {
            ...options,
            headers: {
                'Authorization': `Bearer ${token}`,
                'Content-Type': 'application/json',
                ...options.headers
            }
        });

        if (!response.ok) {
            if (response.status === 401) {
                // Token expirado, tentar renovar
                await this.auth.refreshAccessToken();
                return this.makeAuthenticatedRequest(endpoint, options);
            }
            throw new Error(`API request failed: ${response.status}`);
        }

        return await response.json();
    }

    async getDeviceList(userId) {
        const cacheKey = `devices_${userId}`;
        const cached = this.getCachedData(cacheKey);
        
        if (cached) {
            return cached;
        }

        try {
            const response = await this.makeAuthenticatedRequest(
                `/v1/PowerStationMonitor/GetPowerStationList`,
                {
                    method: 'POST',
                    body: JSON.stringify({
                        account: userId,
                        page_size: 100,
                        page_index: 1
                    })
                }
            );

            if (response.code !== 0) {
                throw new Error(`API Error: ${response.msg}`);
            }

            const devices = response.data.list.map(station => ({
                id: station.id,
                name: station.name,
                type: station.type,
                capacity: station.capacity,
                status: station.status,
                location: {
                    country: station.country,
                    city: station.city,
                    address: station.address
                },
                devices: station.devices || []
            }));

            this.setCachedData(cacheKey, devices);
            return devices;

        } catch (error) {
            console.error('Error fetching device list:', error);
            throw error;
        }
    }

    async getRealtimeData(stationId) {
        const cacheKey = `realtime_${stationId}`;
        const cached = this.getCachedData(cacheKey, 30000); // Cache mais curto para dados em tempo real
        
        if (cached) {
            return cached;
        }

        try {
            const response = await this.makeAuthenticatedRequest(
                `/v1/PowerStationMonitor/GetRealtimeDataByPowerstationId`,
                {
                    method: 'POST',
                    body: JSON.stringify({
                        id: stationId
                    })
                }
            );

            if (response.code !== 0) {
                throw new Error(`API Error: ${response.msg}`);
            }

            const realtimeData = {
                timestamp: new Date().toISOString(),
                production: {
                    current: response.data.pac || 0, // Potência atual em W
                    daily: response.data.eday || 0,  // Energia diária em kWh
                    total: response.data.etotal || 0 // Energia total em kWh
                },
                consumption: {
                    current: response.data.pmeter || 0, // Consumo atual
                    daily: response.data.eload_day || 0 // Consumo diário
                },
                battery: {
                    soc: response.data.soc || 0,        // State of Charge %
                    power: response.data.pbat || 0,     // Potência da bateria
                    voltage: response.data.vbat || 0,   // Voltagem da bateria
                    status: response.data.battery_status || 'unknown'
                },
                grid: {
                    voltage: response.data.vac1 || 0,   // Voltagem da rede
                    frequency: response.data.fac1 || 0, // Frequência da rede
                    power: response.data.pgrid || 0     // Potência da rede
                },
                inverter: {
                    temperature: response.data.tempperature || 0,
                    status: response.data.status || 'unknown',
                    efficiency: response.data.efficiency || 0
                }
            };

            this.setCachedData(cacheKey, realtimeData, 30000);
            return realtimeData;

        } catch (error) {
            console.error('Error fetching realtime data:', error);
            throw error;
        }
    }

    async getHistoricalData(stationId, startDate, endDate, type = 'day') {
        try {
            const response = await this.makeAuthenticatedRequest(
                `/v1/PowerStationMonitor/GetHistoryDataByPowerstationId`,
                {
                    method: 'POST',
                    body: JSON.stringify({
                        id: stationId,
                        date: startDate,
                        date_to: endDate,
                        type: type // 'day', 'month', 'year'
                    })
                }
            );

            if (response.code !== 0) {
                throw new Error(`API Error: ${response.msg}`);
            }

            return response.data.map(item => ({
                date: item.date,
                production: item.eday || 0,
                consumption: item.eload || 0,
                gridFeedIn: item.grid_feed_in || 0,
                gridConsumption: item.grid_consumption || 0,
                batteryCharge: item.battery_charge || 0,
                batteryDischarge: item.battery_discharge || 0
            }));

        } catch (error) {
            console.error('Error fetching historical data:', error);
            throw error;
        }
    }

    async getEnergyForecast(stationId, days = 7) {
        // Integração com serviço de previsão baseado em histórico e clima
        try {
            const historicalData = await this.getHistoricalData(
                stationId, 
                new Date(Date.now() - 30 * 24 * 60 * 60 * 1000).toISOString().split('T')[0],
                new Date().toISOString().split('T')[0]
            );

            // Usar AWS Forecast ou ML simples para previsão
            const forecast = await this.generateForecast(historicalData, days);
            
            return forecast;

        } catch (error) {
            console.error('Error generating forecast:', error);
            return null;
        }
    }

    getCachedData(key, customTTL = null) {
        const cached = this.cache.get(key);
        const ttl = customTTL || this.CACHE_TTL;
        
        if (cached && Date.now() - cached.timestamp < ttl) {
            return cached.data;
        }
        
        this.cache.delete(key);
        return null;
    }

    setCachedData(key, data, customTTL = null) {
        const ttl = customTTL || this.CACHE_TTL;
        this.cache.set(key, {
            data,
            timestamp: Date.now()
        });

        // Limpar cache automaticamente
        setTimeout(() => {
            this.cache.delete(key);
        }, ttl);
    }
}
```

## 🎛️ Controle de Dispositivos

```javascript
class GoodWeControlService {
    constructor(authService) {
        this.auth = authService;
        this.baseUrl = process.env.GOODWE_API_BASE_URL;
    }

    async controlDevice(deviceId, command, parameters = {}) {
        try {
            const token = await this.auth.getValidToken();
            
            const response = await fetch(`${this.baseUrl}/v1/Device/Control`, {
                method: 'POST',
                headers: {
                    'Authorization': `Bearer ${token}`,
                    'Content-Type': 'application/json'
                },
                body: JSON.stringify({
                    device_id: deviceId,
                    command: command,
                    parameters: parameters
                })
            });

            const result = await response.json();
            
            if (result.code !== 0) {
                throw new Error(`Control command failed: ${result.msg}`);
            }

            return result.data;

        } catch (error) {
            console.error('Device control error:', error);
            throw error;
        }
    }

    async setBatteryMode(deviceId, mode) {
        // Modos: 'auto', 'force_charge', 'force_discharge', 'eco'
        return await this.controlDevice(deviceId, 'set_battery_mode', {
            mode: mode
        });
    }

    async setChargeLimit(deviceId, limit) {
        // Limite de carregamento da bateria (0-100%)
        return await this.controlDevice(deviceId, 'set_charge_limit', {
            limit: Math.max(0, Math.min(100, limit))
        });
    }

    async setDischargeLimit(deviceId, limit) {
        // Limite de descarga da bateria (0-100%)
        return await this.controlDevice(deviceId, 'set_discharge_limit', {
            limit: Math.max(0, Math.min(100, limit))
        });
    }

    async setLoadPriority(deviceId, priority) {
        // Prioridade: 'battery_first', 'grid_first', 'pv_first'
        return await this.controlDevice(deviceId, 'set_load_priority', {
            priority: priority
        });
    }

    async scheduleOperation(deviceId, operation, startTime, endTime) {
        // Agendar operações automáticas
        return await this.controlDevice(deviceId, 'schedule_operation', {
            operation: operation,
            start_time: startTime,
            end_time: endTime
        });
    }

    async emergencyMode(deviceId, enable = true) {
        // Ativar/desativar modo de emergência
        return await this.controlDevice(deviceId, 'emergency_mode', {
            enabled: enable
        });
    }
}
```

## 🔄 Integração com Lambda

```javascript
// handler principal para skill
const GoodWeAuthService = require('./services/GoodWeAuthService');
const GoodWeDataService = require('./services/GoodWeDataService');
const GoodWeControlService = require('./services/GoodWeControlService');

// Inicializar serviços
const authService = new GoodWeAuthService();
const dataService = new GoodWeDataService(authService);
const controlService = new GoodWeControlService(authService);

const GetEnergyStatusHandler = {
    canHandle(handlerInput) {
        return Alexa.getRequestType(handlerInput.requestEnvelope) === 'IntentRequest'
            && Alexa.getIntentName(handlerInput.requestEnvelope) === 'GetEnergyStatusIntent';
    },
    async handle(handlerInput) {
        try {
            // Obter ID do usuário e mapear para conta GoodWe
            const userId = handlerInput.requestEnvelope.session.user.userId;
            const goodweUserId = await getUserGoodWeId(userId);
            
            // Buscar dados dos dispositivos
            const devices = await dataService.getDeviceList(goodweUserId);
            
            if (!devices || devices.length === 0) {
                return handlerInput.responseBuilder
                    .speak('Não encontrei nenhum dispositivo GoodWe associado à sua conta.')
                    .getResponse();
            }

            // Pegar dados em tempo real do primeiro dispositivo (ou principal)
            const mainStation = devices[0];
            const realtimeData = await dataService.getRealtimeData(mainStation.id);

            const speakOutput = formatEnergyStatus(realtimeData);
            
            return handlerInput.responseBuilder
                .speak(speakOutput)
                .withSimpleCard('Status de Energia', speakOutput)
                .reprompt('Posso ajudar com mais alguma coisa?')
                .getResponse();

        } catch (error) {
            console.error('Error in GetEnergyStatusHandler:', error);
            
            return handlerInput.responseBuilder
                .speak('Desculpe, não consegui acessar os dados de energia no momento. Tente novamente em alguns instantes.')
                .getResponse();
        }
    }
};

function formatEnergyStatus(data) {
    const production = Math.round(data.production.current / 1000 * 10) / 10; // kW
    const consumption = Math.round(data.consumption.current / 1000 * 10) / 10; // kW
    const batteryLevel = data.battery.soc;
    
    let status = `Atualmente você está produzindo ${production} kilowatts de energia solar `;
    status += `e consumindo ${consumption} kilowatts. `;
    
    if (batteryLevel > 0) {
        status += `Sua bateria está em ${batteryLevel}% de carga. `;
    }
    
    if (production > consumption) {
        const surplus = Math.round((production - consumption) * 10) / 10;
        status += `Você tem um excedente de ${surplus} kilowatts que está sendo `;
        
        if (batteryLevel < 95) {
            status += 'usado para carregar a bateria.';
        } else {
            status += 'injetado na rede elétrica.';
        }
    } else if (consumption > production) {
        const deficit = Math.round((consumption - production) * 10) / 10;
        status += `Você está consumindo ${deficit} kilowatts a mais do que está produzindo, `;
        
        if (batteryLevel > 20) {
            status += 'que está sendo suprido pela bateria.';
        } else {
            status += 'que está sendo suprido pela rede elétrica.';
        }
    }
    
    return status;
}

async function getUserGoodWeId(alexaUserId) {
    // Mapear usuário Alexa para conta GoodWe via DynamoDB
    const AWS = require('aws-sdk');
    const dynamodb = new AWS.DynamoDB.DocumentClient();
    
    const result = await dynamodb.get({
        TableName: 'UserGoodWeMapping',
        Key: { alexaUserId }
    }).promise();
    
    if (!result.Item) {
        throw new Error('GoodWe account not linked');
    }
    
    return result.Item.goodweUserId;
}
```

## 🔧 Configuração de Ambiente

### Variáveis de Ambiente (Lambda)
```bash
GOODWE_API_BASE_URL=https://www.goodwe-power.com/api
GOODWE_CLIENT_ID=your_client_id
GOODWE_CLIENT_SECRET=your_client_secret
LOG_LEVEL=info
CACHE_TTL=300000
```

### Estrutura de Tabelas DynamoDB

#### GoodWeTokens
```json
{
    "TableName": "GoodWeTokens",
    "KeySchema": [
        {
            "AttributeName": "service",
            "KeyType": "HASH"
        }
    ],
    "AttributeDefinitions": [
        {
            "AttributeName": "service",
            "AttributeType": "S"
        }
    ]
}
```

#### UserGoodWeMapping
```json
{
    "TableName": "UserGoodWeMapping", 
    "KeySchema": [
        {
            "AttributeName": "alexaUserId",
            "KeyType": "HASH"
        }
    ],
    "AttributeDefinitions": [
        {
            "AttributeName": "alexaUserId",
            "AttributeType": "S"
        }
    ]
}
```

## ⚡ Próximos Passos

1. **Configure** as credenciais da API GoodWe
2. **Implemente** os serviços de autenticação e dados
3. **Teste** a integração com dados reais
4. **Avance** para [Smart Home Integration](02-smart-home-integration.md)

---
**Anterior:** [← README](README.md) | **Próximo:** [Smart Home Integration →](02-smart-home-integration.md)
