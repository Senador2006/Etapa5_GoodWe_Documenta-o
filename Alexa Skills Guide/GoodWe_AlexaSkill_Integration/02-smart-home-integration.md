# 2. Smart Home Integration - GoodWe

## 🏠 Visão Geral Smart Home

A integração Smart Home permite que os dispositivos GoodWe sejam descobertos e controlados automaticamente pelo ecossistema Alexa, aparecendo nativamente no app Alexa e sendo controláveis por comandos naturais como "Alexa, ligue o inversor" ou "Alexa, configure a bateria para modo economia".

## 🔧 Arquitetura Smart Home

```
Alexa Voice Service
        ↓
Smart Home Skill API
        ↓
AWS Lambda (Discovery & Control)
        ↓
GoodWe API Integration
        ↓
Dispositivos GoodWe Físicos
```

## 📋 Configuração do Smart Home Skill

### Skill Manifest
```json
{
    "manifest": {
        "publishingInformation": {
            "locales": {
                "pt-BR": {
                    "summary": "Controle seus dispositivos GoodWe com a voz",
                    "examplePhrases": [
                        "Alexa, ligar o inversor solar",
                        "Alexa, qual o status da bateria solar",
                        "Alexa, ativar modo economia de energia"
                    ],
                    "keywords": [
                        "solar",
                        "energia",
                        "bateria",
                        "inversor",
                        "goodwe"
                    ],
                    "name": "GoodWe Solar",
                    "description": "Controle completo dos seus dispositivos de energia solar GoodWe através de comandos de voz. Monitore produção, gerencie baterias e otimize o consumo energético."
                }
            },
            "category": "SMART_HOME",
            "isAvailableWorldwide": false,
            "testingInstructions": "Para testar, vincule sua conta GoodWe e diga 'Alexa, descobrir dispositivos'",
            "distributionCountries": ["BR"]
        },
        "apis": {
            "smartHome": {
                "endpoint": {
                    "uri": "arn:aws:lambda:us-east-1:123456789:function:goodwe-smart-home"
                },
                "regions": {
                    "NA": {
                        "endpoint": {
                            "uri": "arn:aws:lambda:us-east-1:123456789:function:goodwe-smart-home"
                        }
                    },
                    "EU": {
                        "endpoint": {
                            "uri": "arn:aws:lambda:eu-west-1:123456789:function:goodwe-smart-home"
                        }
                    }
                }
            }
        },
        "permissions": [
            {
                "name": "alexa::profile:email:read"
            }
        ],
        "privacyAndCompliance": {
            "allowsPurchases": false,
            "usesPersonalInfo": true,
            "isChildDirected": false,
            "isExportCompliant": true,
            "containsAds": false,
            "locales": {
                "pt-BR": {
                    "privacyPolicyUrl": "https://goodwe.com/privacy-policy"
                }
            }
        }
    }
}
```

## 🔍 Discovery Handler

```javascript
const GoodWeDataService = require('./services/GoodWeDataService');
const { v4: uuidv4 } = require('uuid');

exports.handler = async (event) => {
    console.log('Smart Home Request:', JSON.stringify(event, null, 2));
    
    const directive = event.directive;
    const header = directive.header;
    
    try {
        switch (header.namespace) {
            case 'Alexa.Discovery':
                return await handleDiscovery(event);
            case 'Alexa.PowerController':
                return await handlePowerControl(event);
            case 'Alexa.BrightnessController':
                return await handleBrightnessControl(event);
            case 'Alexa.ThermostatController':
                return await handleThermostatControl(event);
            case 'Alexa.RangeController':
                return await handleRangeControl(event);
            case 'Alexa.ModeController':
                return await handleModeControl(event);
            default:
                return createErrorResponse('INVALID_DIRECTIVE', 'Diretiva não suportada');
        }
    } catch (error) {
        console.error('Smart Home Error:', error);
        return createErrorResponse('INTERNAL_ERROR', error.message);
    }
};

async function handleDiscovery(event) {
    const accessToken = event.directive.payload.scope.token;
    const userId = await getUserIdFromToken(accessToken);
    
    try {
        const dataService = new GoodWeDataService();
        const stations = await dataService.getDeviceList(userId);
        
        const endpoints = [];
        
        for (const station of stations) {
            // Adicionar estação como endpoint principal
            endpoints.push(createStationEndpoint(station));
            
            // Adicionar dispositivos individuais da estação
            for (const device of station.devices) {
                endpoints.push(createDeviceEndpoint(device, station));
            }
        }
        
        return {
            event: {
                header: {
                    namespace: 'Alexa.Discovery',
                    name: 'Discover.Response',
                    payloadVersion: '3',
                    messageId: generateMessageId()
                },
                payload: {
                    endpoints: endpoints
                }
            }
        };
        
    } catch (error) {
        console.error('Discovery error:', error);
        return createErrorResponse('DISCOVERY_FAILED', 'Falha na descoberta de dispositivos');
    }
}

function createStationEndpoint(station) {
    return {
        endpointId: `station-${station.id}`,
        manufacturerName: 'GoodWe',
        friendlyName: station.name || `Estação Solar ${station.id}`,
        description: `Estação de energia solar com capacidade de ${station.capacity}kW`,
        displayCategories: ['OTHER'],
        capabilities: [
            // Alexa.EndpointHealth
            {
                type: 'AlexaInterface',
                interface: 'Alexa.EndpointHealth',
                version: '3',
                properties: {
                    supported: [
                        { name: 'connectivity' }
                    ],
                    retrievable: true
                }
            },
            // Alexa Interface básica
            {
                type: 'AlexaInterface',
                interface: 'Alexa',
                version: '3'
            },
            // Controle de energia (ligar/desligar sistema)
            {
                type: 'AlexaInterface',
                interface: 'Alexa.PowerController',
                version: '3',
                properties: {
                    supported: [
                        { name: 'powerState' }
                    ],
                    retrievable: true
                }
            },
            // Sensor de energia
            {
                type: 'AlexaInterface',
                interface: 'Alexa.PowerController',
                version: '3',
                properties: {
                    supported: [
                        { name: 'powerState' }
                    ],
                    retrievable: true
                }
            }
        ],
        connections: [
            {
                type: 'TCP_IP',
                value: station.ipAddress || 'unknown'
            }
        ],
        additionalAttributes: {
            manufacturer: 'GoodWe',
            model: station.model || 'Solar Station',
            serialNumber: station.serialNumber,
            firmwareVersion: station.firmwareVersion || '1.0.0',
            softwareVersion: '1.0.0',
            customIdentifier: station.id
        }
    };
}

function createDeviceEndpoint(device, station) {
    const capabilities = getDeviceCapabilities(device);
    
    return {
        endpointId: `device-${device.id}`,
        manufacturerName: 'GoodWe',
        friendlyName: device.name || `${device.type} ${device.id}`,
        description: `${device.type} da estação ${station.name}`,
        displayCategories: getDeviceCategory(device.type),
        capabilities: capabilities,
        additionalAttributes: {
            manufacturer: 'GoodWe',
            model: device.model || device.type,
            serialNumber: device.serialNumber,
            firmwareVersion: device.firmwareVersion || '1.0.0',
            customIdentifier: device.id,
            stationId: station.id
        }
    };
}

function getDeviceCapabilities(device) {
    const baseCapabilities = [
        {
            type: 'AlexaInterface',
            interface: 'Alexa',
            version: '3'
        },
        {
            type: 'AlexaInterface',
            interface: 'Alexa.EndpointHealth',
            version: '3',
            properties: {
                supported: [{ name: 'connectivity' }],
                retrievable: true
            }
        }
    ];
    
    switch (device.type.toLowerCase()) {
        case 'inverter':
            return [
                ...baseCapabilities,
                {
                    type: 'AlexaInterface',
                    interface: 'Alexa.PowerController',
                    version: '3',
                    properties: {
                        supported: [{ name: 'powerState' }],
                        retrievable: true
                    }
                },
                {
                    type: 'AlexaInterface',
                    interface: 'Alexa.BrightnessController',
                    version: '3',
                    properties: {
                        supported: [{ name: 'brightness' }],
                        retrievable: true
                    }
                }
            ];
            
        case 'battery':
            return [
                ...baseCapabilities,
                {
                    type: 'AlexaInterface',
                    interface: 'Alexa.PowerController',
                    version: '3',
                    properties: {
                        supported: [{ name: 'powerState' }],
                        retrievable: true
                    }
                },
                {
                    type: 'AlexaInterface',
                    interface: 'Alexa.RangeController',
                    version: '3',
                    instance: 'Battery.Level',
                    capabilityResources: {
                        friendlyNames: [
                            { '@type': 'text', value: { text: 'Nível da Bateria', locale: 'pt-BR' } },
                            { '@type': 'text', value: { text: 'Carga', locale: 'pt-BR' } }
                        ]
                    },
                    configuration: {
                        supportedRange: {
                            minimumValue: 0,
                            maximumValue: 100,
                            precision: 1
                        },
                        unitOfMeasure: 'Alexa.Unit.Percent'
                    },
                    properties: {
                        supported: [{ name: 'rangeValue' }],
                        retrievable: true
                    }
                },
                {
                    type: 'AlexaInterface',
                    interface: 'Alexa.ModeController',
                    version: '3',
                    instance: 'Battery.Mode',
                    capabilityResources: {
                        friendlyNames: [
                            { '@type': 'text', value: { text: 'Modo da Bateria', locale: 'pt-BR' } }
                        ]
                    },
                    configuration: {
                        ordered: false,
                        supportedModes: [
                            {
                                value: 'Battery.Auto',
                                modeResources: {
                                    friendlyNames: [
                                        { '@type': 'text', value: { text: 'Automático', locale: 'pt-BR' } }
                                    ]
                                }
                            },
                            {
                                value: 'Battery.ForceCharge',
                                modeResources: {
                                    friendlyNames: [
                                        { '@type': 'text', value: { text: 'Carregamento Forçado', locale: 'pt-BR' } }
                                    ]
                                }
                            },
                            {
                                value: 'Battery.ForceDischarge',
                                modeResources: {
                                    friendlyNames: [
                                        { '@type': 'text', value: { text: 'Descarga Forçada', locale: 'pt-BR' } }
                                    ]
                                }
                            },
                            {
                                value: 'Battery.Eco',
                                modeResources: {
                                    friendlyNames: [
                                        { '@type': 'text', value: { text: 'Modo Econômico', locale: 'pt-BR' } }
                                    ]
                                }
                            }
                        ]
                    },
                    properties: {
                        supported: [{ name: 'mode' }],
                        retrievable: true
                    }
                }
            ];
            
        case 'smartmeter':
            return [
                ...baseCapabilities,
                {
                    type: 'AlexaInterface',
                    interface: 'Alexa.PowerController',
                    version: '3',
                    properties: {
                        supported: [{ name: 'powerState' }],
                        retrievable: true
                    }
                }
            ];
            
        default:
            return baseCapabilities;
    }
}

function getDeviceCategory(deviceType) {
    const categoryMap = {
        'inverter': ['OTHER'],
        'battery': ['OTHER'],
        'smartmeter': ['SMARTPLUG'],
        'charger': ['SMARTPLUG'],
        'switch': ['SWITCH']
    };
    
    return categoryMap[deviceType.toLowerCase()] || ['OTHER'];
}
```

## 🔌 Power Control Handler

```javascript
const GoodWeControlService = require('./services/GoodWeControlService');

async function handlePowerControl(event) {
    const endpointId = event.directive.endpoint.endpointId;
    const directiveName = event.directive.header.name;
    const correlationToken = event.directive.header.correlationToken;
    
    try {
        const controlService = new GoodWeControlService();
        const deviceId = extractDeviceId(endpointId);
        
        let powerState;
        
        switch (directiveName) {
            case 'TurnOn':
                await controlService.controlDevice(deviceId, 'power_on');
                powerState = 'ON';
                break;
                
            case 'TurnOff':
                await controlService.controlDevice(deviceId, 'power_off');
                powerState = 'OFF';
                break;
                
            default:
                throw new Error(`Unsupported directive: ${directiveName}`);
        }
        
        return {
            event: {
                header: {
                    namespace: 'Alexa',
                    name: 'Response',
                    payloadVersion: '3',
                    messageId: generateMessageId(),
                    correlationToken: correlationToken
                },
                endpoint: {
                    endpointId: endpointId
                },
                payload: {}
            },
            context: {
                properties: [
                    {
                        namespace: 'Alexa.PowerController',
                        name: 'powerState',
                        value: powerState,
                        timeOfSample: new Date().toISOString(),
                        uncertaintyInMilliseconds: 500
                    }
                ]
            }
        };
        
    } catch (error) {
        console.error('Power control error:', error);
        return createErrorResponse('ENDPOINT_UNREACHABLE', 'Dispositivo não pode ser controlado no momento');
    }
}
```

## 🔋 Battery Mode Control

```javascript
async function handleModeControl(event) {
    const endpointId = event.directive.endpoint.endpointId;
    const directiveName = event.directive.header.name;
    const instance = event.directive.header.instance;
    
    if (instance !== 'Battery.Mode') {
        return createErrorResponse('INVALID_DIRECTIVE', 'Instância de modo não suportada');
    }
    
    try {
        const controlService = new GoodWeControlService();
        const deviceId = extractDeviceId(endpointId);
        
        let mode;
        let modeValue;
        
        switch (directiveName) {
            case 'SetMode':
                mode = event.directive.payload.mode;
                modeValue = await setBatteryMode(controlService, deviceId, mode);
                break;
                
            default:
                throw new Error(`Unsupported directive: ${directiveName}`);
        }
        
        return {
            event: {
                header: {
                    namespace: 'Alexa',
                    name: 'Response',
                    payloadVersion: '3',
                    messageId: generateMessageId(),
                    correlationToken: event.directive.header.correlationToken
                },
                endpoint: {
                    endpointId: endpointId
                },
                payload: {}
            },
            context: {
                properties: [
                    {
                        namespace: 'Alexa.ModeController',
                        instance: 'Battery.Mode',
                        name: 'mode',
                        value: modeValue,
                        timeOfSample: new Date().toISOString(),
                        uncertaintyInMilliseconds: 500
                    }
                ]
            }
        };
        
    } catch (error) {
        console.error('Mode control error:', error);
        return createErrorResponse('ENDPOINT_UNREACHABLE', 'Não foi possível alterar o modo do dispositivo');
    }
}

async function setBatteryMode(controlService, deviceId, alexaMode) {
    const modeMap = {
        'Battery.Auto': 'auto',
        'Battery.ForceCharge': 'force_charge', 
        'Battery.ForceDischarge': 'force_discharge',
        'Battery.Eco': 'eco'
    };
    
    const goodweMode = modeMap[alexaMode];
    
    if (!goodweMode) {
        throw new Error(`Unsupported mode: ${alexaMode}`);
    }
    
    await controlService.setBatteryMode(deviceId, goodweMode);
    return alexaMode;
}
```

## 📊 Range Controller (Níveis)

```javascript
async function handleRangeControl(event) {
    const endpointId = event.directive.endpoint.endpointId;
    const directiveName = event.directive.header.name;
    const instance = event.directive.header.instance;
    
    try {
        const controlService = new GoodWeControlService();
        const deviceId = extractDeviceId(endpointId);
        
        let rangeValue;
        
        switch (directiveName) {
            case 'SetRangeValue':
                rangeValue = event.directive.payload.rangeValue;
                await setDeviceRange(controlService, deviceId, instance, rangeValue);
                break;
                
            case 'AdjustRangeValue':
                const deltaValue = event.directive.payload.rangeValueDelta;
                rangeValue = await adjustDeviceRange(controlService, deviceId, instance, deltaValue);
                break;
                
            default:
                throw new Error(`Unsupported directive: ${directiveName}`);
        }
        
        return {
            event: {
                header: {
                    namespace: 'Alexa',
                    name: 'Response',
                    payloadVersion: '3',
                    messageId: generateMessageId(),
                    correlationToken: event.directive.header.correlationToken
                },
                endpoint: {
                    endpointId: endpointId
                },
                payload: {}
            },
            context: {
                properties: [
                    {
                        namespace: 'Alexa.RangeController',
                        instance: instance,
                        name: 'rangeValue',
                        value: rangeValue,
                        timeOfSample: new Date().toISOString(),
                        uncertaintyInMilliseconds: 500
                    }
                ]
            }
        };
        
    } catch (error) {
        console.error('Range control error:', error);
        return createErrorResponse('VALUE_OUT_OF_RANGE', 'Valor fora do intervalo suportado');
    }
}

async function setDeviceRange(controlService, deviceId, instance, value) {
    switch (instance) {
        case 'Battery.Level':
            // Configurar limite de carga da bateria
            await controlService.setChargeLimit(deviceId, value);
            break;
            
        case 'Inverter.Power':
            // Configurar limite de potência do inversor
            await controlService.controlDevice(deviceId, 'set_power_limit', { limit: value });
            break;
            
        default:
            throw new Error(`Unsupported range instance: ${instance}`);
    }
    
    return value;
}
```

## 🔄 State Reporting

```javascript
async function handleStateReport(event) {
    const endpointId = event.directive.endpoint.endpointId;
    
    try {
        const dataService = new GoodWeDataService();
        const deviceId = extractDeviceId(endpointId);
        
        // Buscar estado atual do dispositivo
        const deviceData = await dataService.getDeviceStatus(deviceId);
        
        const properties = buildStateProperties(deviceData);
        
        return {
            event: {
                header: {
                    namespace: 'Alexa',
                    name: 'StateReport',
                    payloadVersion: '3',
                    messageId: generateMessageId(),
                    correlationToken: event.directive.header.correlationToken
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
        
    } catch (error) {
        console.error('State report error:', error);
        return createErrorResponse('ENDPOINT_UNREACHABLE', 'Não foi possível obter o estado do dispositivo');
    }
}

function buildStateProperties(deviceData) {
    const properties = [
        {
            namespace: 'Alexa.EndpointHealth',
            name: 'connectivity',
            value: {
                value: deviceData.online ? 'OK' : 'UNREACHABLE'
            },
            timeOfSample: new Date().toISOString(),
            uncertaintyInMilliseconds: 500
        }
    ];
    
    // Adicionar propriedades específicas baseadas no tipo de dispositivo
    if (deviceData.type === 'battery') {
        properties.push(
            {
                namespace: 'Alexa.PowerController',
                name: 'powerState',
                value: deviceData.powerState || 'ON',
                timeOfSample: new Date().toISOString(),
                uncertaintyInMilliseconds: 500
            },
            {
                namespace: 'Alexa.RangeController',
                instance: 'Battery.Level',
                name: 'rangeValue',
                value: deviceData.batteryLevel || 0,
                timeOfSample: new Date().toISOString(),
                uncertaintyInMilliseconds: 500
            },
            {
                namespace: 'Alexa.ModeController',
                instance: 'Battery.Mode',
                name: 'mode',
                value: mapBatteryMode(deviceData.batteryMode),
                timeOfSample: new Date().toISOString(),
                uncertaintyInMilliseconds: 500
            }
        );
    }
    
    return properties;
}

function mapBatteryMode(goodweMode) {
    const modeMap = {
        'auto': 'Battery.Auto',
        'force_charge': 'Battery.ForceCharge',
        'force_discharge': 'Battery.ForceDischarge',
        'eco': 'Battery.Eco'
    };
    
    return modeMap[goodweMode] || 'Battery.Auto';
}
```

## 🛠️ Utility Functions

```javascript
function extractDeviceId(endpointId) {
    // Extrair ID do dispositivo do endpointId
    const parts = endpointId.split('-');
    return parts[1]; // device-123 -> 123
}

function generateMessageId() {
    return 'msg-' + Math.random().toString(36).substr(2, 9);
}

function createErrorResponse(errorType, message) {
    return {
        event: {
            header: {
                namespace: 'Alexa',
                name: 'ErrorResponse',
                payloadVersion: '3',
                messageId: generateMessageId()
            },
            payload: {
                type: errorType,
                message: message
            }
        }
    };
}

async function getUserIdFromToken(accessToken) {
    // Validar token OAuth e retornar user ID
    // Implementar validação com serviço de autenticação
    try {
        const response = await fetch('https://api.amazon.com/user/profile', {
            headers: {
                'Authorization': `Bearer ${accessToken}`
            }
        });
        
        const userData = await response.json();
        return userData.user_id;
        
    } catch (error) {
        throw new Error('Invalid access token');
    }
}
```

## 📱 Account Linking

### OAuth Configuration
```json
{
    "accountLinkingRequest": {
        "type": "AUTH_CODE",
        "authorizationUrl": "https://goodwe-auth.com/oauth/authorize",
        "accessTokenUrl": "https://goodwe-auth.com/oauth/token",
        "clientId": "goodwe_alexa_client_id",
        "clientSecret": "goodwe_alexa_client_secret",
        "accessTokenScheme": "HTTP_BASIC",
        "scope": "read:devices control:devices",
        "domains": ["goodwe-auth.com"],
        "redirectUrls": ["https://pitangui.amazon.com/api/skill/link/M2YSKZKQF7J3E2"]
    }
}
```

## 🧪 Testes

### Comandos de Teste
```bash
# Descoberta de dispositivos
"Alexa, descobrir dispositivos"

# Controle de energia
"Alexa, ligar o inversor solar"
"Alexa, desligar a bateria"

# Controle de modo
"Alexa, configurar a bateria para modo econômico"
"Alexa, ativar carregamento forçado"

# Consulta de status
"Alexa, qual o status da bateria solar?"
"Alexa, como está o inversor?"
```

## ⚡ Próximos Passos

1. **Configure** o account linking com OAuth
2. **Implemente** os handlers de controle
3. **Teste** discovery e controle básico
4. **Avance** para [Emergency Management](03-emergency-management.md)

---
**Anterior:** [← API Integration](01-api-integration.md) | **Próximo:** [Emergency Management →](03-emergency-management.md)
