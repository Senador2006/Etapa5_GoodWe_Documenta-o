# 5. Casos de Uso Avançados

## 📱 Skills Multimodais (Echo Show)

### 1. APL (Alexa Presentation Language) Avançado

#### Template Responsivo
```json
{
    "type": "APL",
    "version": "1.7",
    "import": [
        {
            "name": "alexa-layouts",
            "version": "1.4.0"
        }
    ],
    "mainTemplate": {
        "parameters": [
            "payload"
        ],
        "items": [
            {
                "type": "AlexaHeadline",
                "primaryText": "${payload.title}",
                "secondaryText": "${payload.subtitle}",
                "headerBackgroundColor": "@colorPrimary",
                "headerDivider": true,
                "footerHint": "${payload.hintText}"
            },
            {
                "type": "Container",
                "direction": "row",
                "width": "100%",
                "height": "70vh",
                "items": [
                    {
                        "type": "ScrollView",
                        "width": "60%",
                        "height": "100%",
                        "item": {
                            "type": "Container",
                            "items": [
                                {
                                    "type": "Text",
                                    "text": "${payload.content}",
                                    "fontSize": "20dp",
                                    "fontWeight": "300",
                                    "color": "@colorText",
                                    "paddingLeft": "40dp",
                                    "paddingRight": "40dp"
                                }
                            ]
                        }
                    },
                    {
                        "type": "Container",
                        "width": "40%",
                        "height": "100%",
                        "items": [
                            {
                                "type": "Image",
                                "source": "${payload.imageUrl}",
                                "width": "100%",
                                "height": "50%",
                                "scale": "best-fit",
                                "align": "center"
                            },
                            {
                                "type": "Container",
                                "width": "100%",
                                "height": "50%",
                                "paddingLeft": "20dp",
                                "paddingRight": "20dp",
                                "items": [
                                    {
                                        "type": "Text",
                                        "text": "Informações Adicionais",
                                        "fontSize": "24dp",
                                        "fontWeight": "bold",
                                        "color": "@colorPrimary",
                                        "paddingBottom": "10dp"
                                    },
                                    {
                                        "type": "Text",
                                        "text": "${payload.additionalInfo}",
                                        "fontSize": "16dp",
                                        "color": "@colorText"
                                    }
                                ]
                            }
                        ]
                    }
                ]
            }
        ]
    }
}
```

#### Implementação com Animações
```javascript
const APLAnimationHandler = {
    canHandle(handlerInput) {
        return Alexa.getRequestType(handlerInput.requestEnvelope) === 'IntentRequest'
            && Alexa.getIntentName(handlerInput.requestEnvelope) === 'ShowAnimatedContentIntent';
    },
    handle(handlerInput) {
        const aplDocument = {
            "type": "APL",
            "version": "1.7",
            "mainTemplate": {
                "items": [
                    {
                        "type": "Frame",
                        "width": "100vw",
                        "height": "100vh",
                        "backgroundColor": "black",
                        "items": [
                            {
                                "type": "Container",
                                "width": "100%",
                                "height": "100%",
                                "justifyContent": "center",
                                "alignItems": "center",
                                "items": [
                                    {
                                        "type": "Text",
                                        "id": "animatedText",
                                        "text": "Bem-vindo!",
                                        "fontSize": "60dp",
                                        "color": "white",
                                        "transform": [
                                            {
                                                "scale": 0.1
                                            }
                                        ]
                                    }
                                ]
                            }
                        ]
                    }
                ]
            },
            "commands": {
                "ScaleInAnimation": {
                    "type": "AnimateItem",
                    "componentId": "animatedText",
                    "duration": 2000,
                    "easing": "ease-in-out",
                    "value": [
                        {
                            "property": "transform",
                            "from": [{ "scale": 0.1 }],
                            "to": [{ "scale": 1.0 }]
                        },
                        {
                            "property": "opacity",
                            "from": 0,
                            "to": 1
                        }
                    ]
                }
            },
            "onMount": [
                {
                    "type": "ScaleInAnimation"
                }
            ]
        };

        return handlerInput.responseBuilder
            .speak('Bem-vindo! Veja essa animação especial.')
            .addDirective({
                type: 'Alexa.Presentation.APL.RenderDocument',
                document: aplDocument
            })
            .getResponse();
    }
};
```

### 2. Touch Interactions

```javascript
const TouchEventHandler = {
    canHandle(handlerInput) {
        return Alexa.getRequestType(handlerInput.requestEnvelope) === 'Alexa.Presentation.APL.UserEvent';
    },
    handle(handlerInput) {
        const userEvent = handlerInput.requestEnvelope.request;
        const arguments = userEvent.arguments;
        
        switch (arguments[0]) {
            case 'buttonPressed':
                const buttonId = arguments[1];
                return handleButtonPress(handlerInput, buttonId);
            
            case 'itemSelected':
                const itemIndex = arguments[1];
                return handleItemSelection(handlerInput, itemIndex);
                
            default:
                return handlerInput.responseBuilder
                    .speak('Evento não reconhecido')
                    .getResponse();
        }
    }
};

function handleButtonPress(handlerInput, buttonId) {
    const responses = {
        'playButton': 'Iniciando reprodução...',
        'pauseButton': 'Pausando...',
        'nextButton': 'Próximo item...',
        'prevButton': 'Item anterior...'
    };
    
    const response = responses[buttonId] || 'Botão não reconhecido';
    
    return handlerInput.responseBuilder
        .speak(response)
        .getResponse();
}
```

## 🏠 Integração Smart Home Avançada

### 1. Skill Smart Home com Múltiplos Dispositivos

```javascript
// Smart Home Skill com suporte a cenários
exports.handler = async (event) => {
    const directive = event.directive;
    const namespace = directive.header.namespace;
    
    switch (namespace) {
        case 'Alexa.Discovery':
            return await handleDiscovery(event);
        case 'Alexa.PowerController':
            return await handlePowerControl(event);
        case 'Alexa.BrightnessController':
            return await handleBrightnessControl(event);
        case 'Alexa.ThermostatController':
            return await handleThermostatControl(event);
        case 'Alexa.SceneController':
            return await handleSceneControl(event);
        default:
            return createErrorResponse('INVALID_DIRECTIVE');
    }
};

// Descoberta de dispositivos avançada
async function handleDiscovery(event) {
    const endpoints = await getDevicesFromDatabase();
    
    const discoveryResponse = {
        event: {
            header: {
                namespace: 'Alexa.Discovery',
                name: 'Discover.Response',
                payloadVersion: '3',
                messageId: generateMessageId()
            },
            payload: {
                endpoints: endpoints.map(device => ({
                    endpointId: device.id,
                    manufacturerName: device.manufacturer,
                    friendlyName: device.name,
                    description: device.description,
                    displayCategories: [device.category],
                    capabilities: getCapabilitiesForDevice(device),
                    connections: device.connections || [],
                    additionalAttributes: {
                        manufacturer: device.manufacturer,
                        model: device.model,
                        serialNumber: device.serialNumber,
                        firmwareVersion: device.firmwareVersion
                    }
                }))
            }
        }
    };
    
    return discoveryResponse;
}

// Controle de cenários
async function handleSceneControl(event) {
    const endpointId = event.directive.endpoint.endpointId;
    const directiveName = event.directive.header.name;
    
    if (directiveName === 'Activate') {
        await activateScene(endpointId);
        
        return {
            event: {
                header: {
                    namespace: 'Alexa.SceneController',
                    name: 'ActivationStarted',
                    payloadVersion: '3',
                    messageId: generateMessageId(),
                    correlationToken: event.directive.header.correlationToken
                },
                endpoint: {
                    endpointId: endpointId
                },
                payload: {
                    cause: {
                        type: 'VOICE_INTERACTION'
                    },
                    timestamp: new Date().toISOString()
                }
            }
        };
    }
}

async function activateScene(sceneId) {
    const scene = await getSceneConfiguration(sceneId);
    
    // Executar ações do cenário em paralelo
    const actions = scene.actions.map(action => {
        switch (action.type) {
            case 'light':
                return controlLight(action.deviceId, action.state);
            case 'thermostat':
                return controlThermostat(action.deviceId, action.temperature);
            case 'media':
                return controlMedia(action.deviceId, action.command);
        }
    });
    
    await Promise.all(actions);
}
```

### 2. Proactive Events

```javascript
// Envio de eventos proativos
const AWS = require('aws-sdk');
const alexaEventGateway = new AWS.AlexaEventGateway({
    endpoint: 'https://api.amazonalexa.com'
});

async function sendProactiveEvent(userId, deviceId, eventType, eventData) {
    const event = {
        event: {
            header: {
                namespace: 'Alexa.Security.SecurityDeviceController',
                name: eventType, // 'AlarmActivated', 'BurglaryAlarm', etc.
                payloadVersion: '3',
                messageId: generateMessageId()
            },
            endpoint: {
                scope: {
                    type: 'BearerToken',
                    token: await getAccessTokenForUser(userId)
                },
                endpointId: deviceId
            },
            payload: eventData
        }
    };
    
    try {
        await alexaEventGateway.sendEvent(event).promise();
        console.log('Evento proativo enviado com sucesso');
    } catch (error) {
        console.error('Erro ao enviar evento proativo:', error);
    }
}

// Uso para alarme de segurança
await sendProactiveEvent(userId, 'security-system-001', 'AlarmActivated', {
    cause: {
        type: 'PHYSICAL_INTERACTION'
    },
    alarm: {
        value: 'BURGLARY',
        cause: 'MOTION_DETECTED'
    }
});
```

## 💼 Skills Empresariais

### 1. Autenticação e Autorização

```javascript
// Account Linking com OAuth2
const AccountLinkingHandler = {
    canHandle(handlerInput) {
        const request = handlerInput.requestEnvelope.request;
        return request.type === 'LaunchRequest' && !request.user.accessToken;
    },
    handle(handlerInput) {
        return handlerInput.responseBuilder
            .speak('Para usar essa funcionalidade, você precisa vincular sua conta. Verifique o app Alexa.')
            .withLinkAccountCard()
            .getResponse();
    }
};

// Validação de token de acesso
async function validateAccessToken(accessToken) {
    try {
        const response = await fetch('https://api.empresa.com/validate', {
            headers: {
                'Authorization': `Bearer ${accessToken}`
            }
        });
        
        if (response.ok) {
            const userData = await response.json();
            return userData;
        }
        
        return null;
    } catch (error) {
        console.error('Erro na validação do token:', error);
        return null;
    }
}

// Handler com autorização
const SecureHandler = {
    canHandle(handlerInput) {
        return Alexa.getRequestType(handlerInput.requestEnvelope) === 'IntentRequest'
            && Alexa.getIntentName(handlerInput.requestEnvelope) === 'GetCompanyDataIntent';
    },
    async handle(handlerInput) {
        const accessToken = handlerInput.requestEnvelope.session.user.accessToken;
        
        if (!accessToken) {
            return handlerInput.responseBuilder
                .speak('Você precisa vincular sua conta primeiro.')
                .withLinkAccountCard()
                .getResponse();
        }
        
        const userData = await validateAccessToken(accessToken);
        
        if (!userData) {
            return handlerInput.responseBuilder
                .speak('Sua sessão expirou. Por favor, vincule sua conta novamente.')
                .withLinkAccountCard()
                .getResponse();
        }
        
        // Verificar permissões específicas
        if (!userData.permissions.includes('read_company_data')) {
            return handlerInput.responseBuilder
                .speak('Você não tem permissão para acessar esses dados.')
                .getResponse();
        }
        
        // Processar request autorizado
        const companyData = await getCompanyData(userData.company_id);
        
        return handlerInput.responseBuilder
            .speak(`Aqui estão os dados da empresa: ${companyData.summary}`)
            .getResponse();
    }
};
```

### 2. Integração com Sistemas Empresariais

```javascript
// Integração com CRM (Salesforce)
const SalesforceIntegration = {
    async getAccountInfo(accountId, accessToken) {
        const salesforceUrl = `https://yourinstance.salesforce.com/services/data/v52.0/sobjects/Account/${accountId}`;
        
        const response = await fetch(salesforceUrl, {
            headers: {
                'Authorization': `Bearer ${accessToken}`,
                'Content-Type': 'application/json'
            }
        });
        
        if (response.ok) {
            return await response.json();
        }
        
        throw new Error('Erro ao acessar Salesforce');
    },
    
    async createLead(leadData, accessToken) {
        const salesforceUrl = 'https://yourinstance.salesforce.com/services/data/v52.0/sobjects/Lead';
        
        const response = await fetch(salesforceUrl, {
            method: 'POST',
            headers: {
                'Authorization': `Bearer ${accessToken}`,
                'Content-Type': 'application/json'
            },
            body: JSON.stringify(leadData)
        });
        
        if (response.ok) {
            return await response.json();
        }
        
        throw new Error('Erro ao criar lead no Salesforce');
    }
};

// Handler para consultas de vendas
const SalesQueryHandler = {
    canHandle(handlerInput) {
        return Alexa.getRequestType(handlerInput.requestEnvelope) === 'IntentRequest'
            && Alexa.getIntentName(handlerInput.requestEnvelope) === 'GetSalesInfoIntent';
    },
    async handle(handlerInput) {
        const accessToken = handlerInput.requestEnvelope.session.user.accessToken;
        const accountName = Alexa.getSlotValue(handlerInput.requestEnvelope, 'account');
        
        try {
            // Buscar informações da conta
            const accountInfo = await SalesforceIntegration.getAccountInfo(accountName, accessToken);
            
            const speakOutput = `A conta ${accountInfo.Name} tem ${accountInfo.NumberOfEmployees} funcionários e faturamento anual de ${formatCurrency(accountInfo.AnnualRevenue)}.`;
            
            return handlerInput.responseBuilder
                .speak(speakOutput)
                .withSimpleCard('Informações da Conta', speakOutput)
                .getResponse();
                
        } catch (error) {
            return handlerInput.responseBuilder
                .speak('Não consegui acessar as informações da conta no momento.')
                .getResponse();
        }
    }
};
```

## 💰 Monetização com In-Skill Purchasing

### 1. Configuração de Produtos

```json
{
    "version": "1.0",
    "products": [
        {
            "referenceName": "premium_features",
            "type": "SUBSCRIPTION",
            "name": {
                "en-US": "Premium Features",
                "pt-BR": "Recursos Premium"
            },
            "summary": {
                "en-US": "Unlock advanced features",
                "pt-BR": "Desbloqueie recursos avançados"
            },
            "description": {
                "en-US": "Get access to advanced analytics, custom reports, and priority support.",
                "pt-BR": "Tenha acesso a análises avançadas, relatórios personalizados e suporte prioritário."
            },
            "purchasePrompt": {
                "en-US": "Would you like to subscribe to Premium Features for $9.99 per month?",
                "pt-BR": "Gostaria de assinar os Recursos Premium por R$ 29,90 por mês?"
            },
            "boughtCardDescription": {
                "en-US": "You now have access to all premium features!",
                "pt-BR": "Agora você tem acesso a todos os recursos premium!"
            },
            "subscriptionTerms": {
                "en-US": "This is a monthly subscription that will automatically renew.",
                "pt-BR": "Esta é uma assinatura mensal que será renovada automaticamente."
            }
        }
    ]
}
```

### 2. Implementação de Compras

```javascript
const PurchaseHandler = {
    canHandle(handlerInput) {
        return Alexa.getRequestType(handlerInput.requestEnvelope) === 'IntentRequest'
            && Alexa.getIntentName(handlerInput.requestEnvelope) === 'BuyPremiumIntent';
    },
    handle(handlerInput) {
        return handlerInput.responseBuilder
            .addDirective({
                type: 'Connections.SendRequest',
                name: 'Buy',
                payload: {
                    InSkillProduct: {
                        productId: 'amzn1.adg.product.premium_features'
                    }
                },
                token: 'purchase-premium'
            })
            .getResponse();
    }
};

// Handler para resultado da compra
const PurchaseResponseHandler = {
    canHandle(handlerInput) {
        return Alexa.getRequestType(handlerInput.requestEnvelope) === 'Connections.Response'
            && handlerInput.requestEnvelope.request.name === 'Buy';
    },
    async handle(handlerInput) {
        const purchaseResult = handlerInput.requestEnvelope.request.payload.purchaseResult;
        
        switch (purchaseResult) {
            case 'ACCEPTED':
                // Ativar recursos premium para o usuário
                await activatePremiumFeatures(handlerInput.requestEnvelope.session.user.userId);
                
                return handlerInput.responseBuilder
                    .speak('Obrigado pela compra! Seus recursos premium foram ativados.')
                    .getResponse();
                    
            case 'DECLINED':
                return handlerInput.responseBuilder
                    .speak('Tudo bem, você pode comprar os recursos premium a qualquer momento.')
                    .getResponse();
                    
            case 'ALREADY_PURCHASED':
                return handlerInput.responseBuilder
                    .speak('Você já tem acesso aos recursos premium!')
                    .getResponse();
                    
            default:
                return handlerInput.responseBuilder
                    .speak('Houve um problema com a compra. Tente novamente.')
                    .getResponse();
        }
    }
};

// Verificar status de compra
async function checkPurchaseStatus(handlerInput, productId) {
    const locale = handlerInput.requestEnvelope.request.locale;
    const ms = handlerInput.serviceClientFactory.getMonetizationServiceClient();
    
    try {
        const product = await ms.getInSkillProduct(locale, productId);
        return product.entitled === 'ENTITLED';
    } catch (error) {
        console.error('Erro ao verificar status de compra:', error);
        return false;
    }
}
```

## 🔊 Audio Streaming Avançado

### 1. Reprodução de Música com Playlist

```javascript
const MusicStreamingHandler = {
    canHandle(handlerInput) {
        return Alexa.getRequestType(handlerInput.requestEnvelope) === 'IntentRequest'
            && Alexa.getIntentName(handlerInput.requestEnvelope) === 'PlayPlaylistIntent';
    },
    async handle(handlerInput) {
        const playlistName = Alexa.getSlotValue(handlerInput.requestEnvelope, 'playlist');
        
        try {
            const playlist = await getPlaylist(playlistName);
            
            if (!playlist || playlist.tracks.length === 0) {
                return handlerInput.responseBuilder
                    .speak('Não encontrei essa playlist.')
                    .getResponse();
            }
            
            // Iniciar reprodução da primeira música
            const firstTrack = playlist.tracks[0];
            
            // Salvar playlist na sessão/persistência
            const attributes = handlerInput.attributesManager.getSessionAttributes();
            attributes.currentPlaylist = playlist;
            attributes.currentTrackIndex = 0;
            handlerInput.attributesManager.setSessionAttributes(attributes);
            
            return handlerInput.responseBuilder
                .speak(`Reproduzindo playlist ${playlistName}`)
                .addAudioPlayerPlayDirective(
                    'REPLACE_ALL',
                    firstTrack.url,
                    firstTrack.id,
                    0,
                    null,
                    {
                        title: firstTrack.title,
                        subtitle: firstTrack.artist,
                        art: {
                            sources: [
                                {
                                    url: firstTrack.artworkUrl
                                }
                            ]
                        },
                        backgroundImage: {
                            sources: [
                                {
                                    url: firstTrack.backgroundUrl
                                }
                            ]
                        }
                    }
                )
                .getResponse();
                
        } catch (error) {
            console.error('Erro ao reproduzir playlist:', error);
            return handlerInput.responseBuilder
                .speak('Não consegui reproduzir a playlist no momento.')
                .getResponse();
        }
    }
};

// Handler para próxima música
const NextTrackHandler = {
    canHandle(handlerInput) {
        return Alexa.getRequestType(handlerInput.requestEnvelope) === 'AudioPlayer.PlaybackNearlyFinished';
    },
    async handle(handlerInput) {
        const attributes = handlerInput.attributesManager.getSessionAttributes();
        const playlist = attributes.currentPlaylist;
        const currentIndex = attributes.currentTrackIndex || 0;
        
        if (playlist && currentIndex < playlist.tracks.length - 1) {
            const nextIndex = currentIndex + 1;
            const nextTrack = playlist.tracks[nextIndex];
            
            // Atualizar índice
            attributes.currentTrackIndex = nextIndex;
            handlerInput.attributesManager.setSessionAttributes(attributes);
            
            return handlerInput.responseBuilder
                .addAudioPlayerPlayDirective(
                    'ENQUEUE',
                    nextTrack.url,
                    nextTrack.id,
                    0,
                    handlerInput.requestEnvelope.request.token,
                    {
                        title: nextTrack.title,
                        subtitle: nextTrack.artist,
                        art: {
                            sources: [{ url: nextTrack.artworkUrl }]
                        }
                    }
                )
                .getResponse();
        }
        
        return handlerInput.responseBuilder.getResponse();
    }
};
```

### 2. Controles de Reprodução Avançados

```javascript
// Handler para comandos de controle
const PlaybackControlHandler = {
    canHandle(handlerInput) {
        const request = handlerInput.requestEnvelope.request;
        return request.type === 'IntentRequest' && [
            'AMAZON.PauseIntent',
            'AMAZON.ResumeIntent',
            'AMAZON.NextIntent',
            'AMAZON.PreviousIntent',
            'AMAZON.ShuffleOnIntent',
            'AMAZON.ShuffleOffIntent',
            'AMAZON.LoopOnIntent',
            'AMAZON.LoopOffIntent'
        ].includes(request.intent.name);
    },
    handle(handlerInput) {
        const intentName = Alexa.getIntentName(handlerInput.requestEnvelope);
        
        switch (intentName) {
            case 'AMAZON.PauseIntent':
                return handlerInput.responseBuilder
                    .addAudioPlayerStopDirective()
                    .getResponse();
                    
            case 'AMAZON.ResumeIntent':
                return resumePlayback(handlerInput);
                
            case 'AMAZON.NextIntent':
                return playNextTrack(handlerInput);
                
            case 'AMAZON.PreviousIntent':
                return playPreviousTrack(handlerInput);
                
            case 'AMAZON.ShuffleOnIntent':
                return enableShuffle(handlerInput);
                
            default:
                return handlerInput.responseBuilder.getResponse();
        }
    }
};

function enableShuffle(handlerInput) {
    const attributes = handlerInput.attributesManager.getSessionAttributes();
    const playlist = attributes.currentPlaylist;
    
    if (playlist) {
        // Embaralhar playlist
        const shuffledTracks = [...playlist.tracks].sort(() => Math.random() - 0.5);
        attributes.currentPlaylist.tracks = shuffledTracks;
        attributes.shuffleEnabled = true;
        handlerInput.attributesManager.setSessionAttributes(attributes);
        
        return handlerInput.responseBuilder
            .speak('Modo aleatório ativado')
            .getResponse();
    }
    
    return handlerInput.responseBuilder
        .speak('Nenhuma playlist ativa para embaralhar')
        .getResponse();
}
```

## ⚡ Próximos Passos

1. **Escolha** o caso avançado mais relevante para seu projeto
2. **Implemente** seguindo os exemplos fornecidos
3. **Teste** extensivamente em diferentes cenários
4. **Monitore** performance e uso
5. **Consulte** [Troubleshooting](06-troubleshooting.md) se necessário

---
**Anterior:** [← Deploy e Publicação](04-deployment.md) | **Próximo:** [Troubleshooting →](06-troubleshooting.md)
