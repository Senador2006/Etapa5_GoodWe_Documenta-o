# 3. Exemplos Práticos

## 🚀 Skill de Notícias Personalizadas

### Objetivo
Criar uma skill que forneça notícias personalizadas com base nas preferências do usuário.

### Interaction Model
```json
{
  "intents": [
    {
      "name": "ObterNoticiasIntent",
      "slots": [
        {
          "name": "categoria",
          "type": "CategoriaNoticiasType"
        }
      ],
      "samples": [
        "quais são as notícias",
        "me conte as novidades",
        "notícias de {categoria}",
        "últimas notícias sobre {categoria}",
        "o que está acontecendo"
      ]
    },
    {
      "name": "ConfigurarPreferenciasIntent",
      "slots": [
        {
          "name": "categorias",
          "type": "CategoriaNoticiasType"
        }
      ],
      "samples": [
        "configurar preferências",
        "quero notícias de {categorias}",
        "me interesso por {categorias}",
        "adicionar {categorias} às minhas preferências"
      ]
    }
  ],
  "types": [
    {
      "name": "CategoriaNoticiasType", 
      "values": [
        {"name": {"value": "tecnologia"}},
        {"name": {"value": "esportes"}},
        {"name": {"value": "política"}},
        {"name": {"value": "economia"}},
        {"name": {"value": "entretenimento"}},
        {"name": {"value": "saúde"}}
      ]
    }
  ]
}
```

### Implementação Backend
```javascript
const Alexa = require('ask-sdk-core');
const https = require('https');

// Handler para obter notícias
const ObterNoticiasHandler = {
    canHandle(handlerInput) {
        return Alexa.getRequestType(handlerInput.requestEnvelope) === 'IntentRequest'
            && Alexa.getIntentName(handlerInput.requestEnvelope) === 'ObterNoticiasIntent';
    },
    async handle(handlerInput) {
        const categoria = Alexa.getSlotValue(handlerInput.requestEnvelope, 'categoria');
        
        // Obter preferências salvas
        const attributesManager = handlerInput.attributesManager;
        const persistentAttributes = await attributesManager.getPersistentAttributes() || {};
        const preferencias = persistentAttributes.categoriasPreferidas || ['geral'];
        
        const categoriaFinal = categoria || preferencias[0];
        
        try {
            const noticias = await obterNoticias(categoriaFinal);
            
            if (noticias.length === 0) {
                return handlerInput.responseBuilder
                    .speak('Desculpe, não encontrei notícias no momento.')
                    .reprompt('Posso ajudar com algo mais?')
                    .getResponse();
            }
            
            // Construir resposta com as 3 principais notícias
            const resumo = noticias.slice(0, 3).map((noticia, index) => 
                `${index + 1}. ${noticia.title}`
            ).join('. ');
            
            const speakOutput = `Aqui estão as principais notícias de ${categoriaFinal}: ${resumo}. Quer que eu detalhe alguma delas?`;
            
            // Salvar notícias na sessão para detalhamento
            const sessionAttributes = handlerInput.attributesManager.getSessionAttributes();
            sessionAttributes.ultimasNoticias = noticias.slice(0, 3);
            handlerInput.attributesManager.setSessionAttributes(sessionAttributes);
            
            return handlerInput.responseBuilder
                .speak(speakOutput)
                .withSimpleCard(`Notícias - ${categoriaFinal}`, resumo)
                .reprompt('Quer que eu detalhe alguma notícia?')
                .getResponse();
                
        } catch (error) {
            console.error('Erro ao obter notícias:', error);
            return handlerInput.responseBuilder
                .speak('Desculpe, houve um problema ao buscar as notícias.')
                .reprompt('Como posso ajudar você?')
                .getResponse();
        }
    }
};

// Função para buscar notícias de uma API
async function obterNoticias(categoria) {
    return new Promise((resolve, reject) => {
        const apiKey = process.env.NEWS_API_KEY;
        const categoriaMap = {
            'tecnologia': 'technology',
            'esportes': 'sports', 
            'política': 'politics',
            'economia': 'business',
            'entretenimento': 'entertainment',
            'saúde': 'health'
        };
        
        const cat = categoriaMap[categoria] || 'general';
        const url = `https://newsapi.org/v2/top-headlines?country=br&category=${cat}&apiKey=${apiKey}`;
        
        https.get(url, (response) => {
            let data = '';
            
            response.on('data', (chunk) => data += chunk);
            response.on('end', () => {
                try {
                    const result = JSON.parse(data);
                    resolve(result.articles || []);
                } catch (error) {
                    reject(error);
                }
            });
        }).on('error', reject);
    });
}
```

## 🏠 Skill de Controle de Casa Inteligente

### Objetivo
Controlar dispositivos IoT como luzes, temperatura e segurança.

### Smart Home Skill Configuration
```json
{
  "manifest": {
    "publishingInformation": {
      "locales": {
        "pt-BR": {
          "name": "Casa Inteligente"
        }
      }
    },
    "apis": {
      "smartHome": {
        "endpoint": {
          "uri": "arn:aws:lambda:us-east-1:123456789:function:smart-home-handler"
        },
        "regions": {
          "NA": {
            "endpoint": {
              "uri": "arn:aws:lambda:us-east-1:123456789:function:smart-home-handler"
            }
          }
        }
      }
    }
  }
}
```

### Implementação Smart Home
```javascript
const AWS = require('aws-sdk');
const iot = new AWS.IoT();

exports.handler = async (event) => {
    console.log('Smart Home Request:', JSON.stringify(event, null, 2));
    
    const directive = event.directive;
    const header = directive.header;
    const payload = directive.payload;
    
    switch (header.name) {
        case 'Discover':
            return handleDiscovery(event);
        case 'TurnOn':
            return await handleTurnOn(event);
        case 'TurnOff':
            return await handleTurnOff(event);
        case 'SetBrightness':
            return await handleSetBrightness(event);
        case 'SetTargetTemperature':
            return await handleSetTemperature(event);
        default:
            return createErrorResponse('INVALID_DIRECTIVE', 'Comando não suportado');
    }
};

// Descobrir dispositivos disponíveis
function handleDiscovery(event) {
    const response = {
        event: {
            header: {
                namespace: 'Alexa.Discovery',
                name: 'Discover.Response',
                payloadVersion: '3',
                messageId: generateMessageId()
            },
            payload: {
                endpoints: [
                    {
                        endpointId: 'luz-sala',
                        manufacturerName: 'Casa Inteligente',
                        friendlyName: 'Luz da Sala',
                        description: 'Lâmpada LED inteligente da sala',
                        displayCategories: ['LIGHT'],
                        capabilities: [
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
                        ]
                    },
                    {
                        endpointId: 'termostato-casa',
                        manufacturerName: 'Casa Inteligente',
                        friendlyName: 'Termostato',
                        description: 'Controle de temperatura da casa',
                        displayCategories: ['THERMOSTAT'],
                        capabilities: [
                            {
                                type: 'AlexaInterface',
                                interface: 'Alexa.ThermostatController',
                                version: '3',
                                properties: {
                                    supported: [
                                        { name: 'targetSetpoint' },
                                        { name: 'thermostatMode' }
                                    ],
                                    retrievable: true
                                }
                            }
                        ]
                    }
                ]
            }
        }
    };
    
    return response;
}

// Ligar dispositivo
async function handleTurnOn(event) {
    const endpointId = event.directive.endpoint.endpointId;
    
    try {
        // Enviar comando para dispositivo IoT
        await enviarComandoIoT(endpointId, { action: 'turn_on' });
        
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
                        namespace: 'Alexa.PowerController',
                        name: 'powerState',
                        value: 'ON',
                        timeOfSample: new Date().toISOString(),
                        uncertaintyInMilliseconds: 500
                    }
                ]
            }
        };
    } catch (error) {
        return createErrorResponse('INTERNAL_ERROR', 'Falha ao ligar dispositivo');
    }
}

// Enviar comando para dispositivo IoT
async function enviarComandoIoT(deviceId, command) {
    const params = {
        topic: `devices/${deviceId}/commands`,
        payload: JSON.stringify(command),
        qos: 1
    };
    
    return iot.publish(params).promise();
}

function generateMessageId() {
    return 'msg-' + Math.random().toString(36).substr(2, 9);
}
```

## 📊 Skill de Consulta de Dados Empresariais

### Objetivo
Permitir consultas rápidas a métricas de negócio e relatórios.

### Interaction Model
```json
{
  "intents": [
    {
      "name": "ConsultarVendasIntent",
      "slots": [
        {
          "name": "periodo",
          "type": "AMAZON.Date"
        },
        {
          "name": "produto",
          "type": "ProdutoType"
        }
      ],
      "samples": [
        "vendas de hoje",
        "vendas de {periodo}",
        "vendas do produto {produto}",
        "quanto vendemos {periodo}",
        "faturamento de {periodo}"
      ]
    },
    {
      "name": "ConsultarEstoqueIntent",
      "slots": [
        {
          "name": "produto",
          "type": "ProdutoType"
        }
      ],
      "samples": [
        "estoque do {produto}",
        "quantos {produto} temos",
        "verificar estoque",
        "estoque disponível"
      ]
    }
  ]
}
```

### Implementação com Base de Dados
```javascript
const AWS = require('aws-sdk');
const dynamodb = new AWS.DynamoDB.DocumentClient();

const ConsultarVendasHandler = {
    canHandle(handlerInput) {
        return Alexa.getRequestType(handlerInput.requestEnvelope) === 'IntentRequest'
            && Alexa.getIntentName(handlerInput.requestEnvelope) === 'ConsultarVendasIntent';
    },
    async handle(handlerInput) {
        const periodo = Alexa.getSlotValue(handlerInput.requestEnvelope, 'periodo');
        const produto = Alexa.getSlotValue(handlerInput.requestEnvelope, 'produto');
        
        try {
            const filtros = {};
            
            if (periodo) {
                filtros.data = formatarPeriodo(periodo);
            }
            if (produto) {
                filtros.produto = produto;
            }
            
            const vendas = await consultarVendas(filtros);
            
            const totalVendas = vendas.reduce((sum, venda) => sum + venda.valor, 0);
            const quantidadeVendas = vendas.length;
            
            let speakOutput;
            if (produto) {
                speakOutput = `As vendas do produto ${produto} totalizam ${formatarMoeda(totalVendas)} em ${quantidadeVendas} transações.`;
            } else if (periodo) {
                speakOutput = `As vendas de ${periodo} totalizam ${formatarMoeda(totalVendas)} em ${quantidadeVendas} transações.`;
            } else {
                speakOutput = `As vendas totais são de ${formatarMoeda(totalVendas)} em ${quantidadeVendas} transações.`;
            }
            
            return handlerInput.responseBuilder
                .speak(speakOutput)
                .withSimpleCard('Relatório de Vendas', speakOutput)
                .reprompt('Precisa de mais algum relatório?')
                .getResponse();
                
        } catch (error) {
            console.error('Erro ao consultar vendas:', error);
            return handlerInput.responseBuilder
                .speak('Desculpe, não consegui acessar os dados de vendas.')
                .reprompt('Como posso ajudar você?')
                .getResponse();
        }
    }
};

async function consultarVendas(filtros) {
    const params = {
        TableName: 'VendasTable'
    };
    
    if (filtros.data) {
        params.FilterExpression = '#data = :data';
        params.ExpressionAttributeNames = { '#data': 'data' };
        params.ExpressionAttributeValues = { ':data': filtros.data };
    }
    
    if (filtros.produto) {
        if (params.FilterExpression) {
            params.FilterExpression += ' AND produto = :produto';
        } else {
            params.FilterExpression = 'produto = :produto';
        }
        params.ExpressionAttributeValues = {
            ...params.ExpressionAttributeValues,
            ':produto': filtros.produto
        };
    }
    
    const result = await dynamodb.scan(params).promise();
    return result.Items;
}

function formatarMoeda(valor) {
    return new Intl.NumberFormat('pt-BR', {
        style: 'currency',
        currency: 'BRL'
    }).format(valor);
}

function formatarPeriodo(periodo) {
    // Converter período natural para formato de data
    const hoje = new Date();
    
    if (periodo === 'hoje' || periodo === '2024-01-15') {
        return hoje.toISOString().split('T')[0];
    } else if (periodo === 'ontem') {
        const ontem = new Date(hoje);
        ontem.setDate(hoje.getDate() - 1);
        return ontem.toISOString().split('T')[0];
    }
    
    return periodo;
}
```

## 🎵 Skill de Reprodução de Música

### Objetivo
Integrar com serviços de streaming para reprodução de música.

### Audio Player Interface
```javascript
const MusicPlayerHandler = {
    canHandle(handlerInput) {
        return Alexa.getRequestType(handlerInput.requestEnvelope) === 'IntentRequest'
            && Alexa.getIntentName(handlerInput.requestEnvelope) === 'PlayMusicIntent';
    },
    handle(handlerInput) {
        const artista = Alexa.getSlotValue(handlerInput.requestEnvelope, 'artista');
        const musica = Alexa.getSlotValue(handlerInput.requestEnvelope, 'musica');
        
        // Buscar música na sua API de streaming
        const audioUrl = buscarMusica(artista, musica);
        
        if (!audioUrl) {
            return handlerInput.responseBuilder
                .speak('Desculpe, não encontrei essa música.')
                .reprompt('Que outra música posso tocar?')
                .getResponse();
        }
        
        return handlerInput.responseBuilder
            .speak(`Tocando ${musica} de ${artista}`)
            .addAudioPlayerPlayDirective(
                'REPLACE_ALL',
                audioUrl,
                'track-001',
                0,
                null
            )
            .getResponse();
    }
};

// Handler para eventos do Audio Player
const AudioPlayerEventHandler = {
    canHandle(handlerInput) {
        return handlerInput.requestEnvelope.request.type.startsWith('AudioPlayer.');
    },
    handle(handlerInput) {
        const requestType = handlerInput.requestEnvelope.request.type;
        
        switch (requestType) {
            case 'AudioPlayer.PlaybackStarted':
                console.log('Reprodução iniciada');
                break;
            case 'AudioPlayer.PlaybackFinished':
                console.log('Reprodução finalizada');
                break;
            case 'AudioPlayer.PlaybackFailed':
                console.log('Falha na reprodução');
                break;
        }
        
        return handlerInput.responseBuilder.getResponse();
    }
};
```

## 🔧 Integração com APIs Personalizadas

### Webhook para Receber Dados
```javascript
// Endpoint para receber webhooks
const express = require('express');
const app = express();
app.use(express.json());

app.post('/webhook/alexa-data', (req, res) => {
    const { userId, data } = req.body;
    
    // Processar dados recebidos
    processarDadosUsuario(userId, data);
    
    res.json({ success: true });
});

// Notificar usuário via Alexa
async function notificarUsuario(userId, mensagem) {
    const params = {
        TableName: 'AlexaTokensTable',
        Key: { userId: userId }
    };
    
    const user = await dynamodb.get(params).promise();
    
    if (user.Item && user.Item.notificationToken) {
        // Enviar notificação proativa
        await enviarNotificacaoProativa(user.Item.notificationToken, mensagem);
    }
}
```

## 📱 Skill Multimodal (Echo Show)

### APL (Alexa Presentation Language)
```json
{
  "type": "APL",
  "version": "1.6",
  "mainTemplate": {
    "items": [
      {
        "type": "Container",
        "width": "100vw",
        "height": "100vh",
        "items": [
          {
            "type": "Text",
            "text": "${title}",
            "fontSize": "40dp",
            "fontWeight": "bold",
            "textAlign": "center"
          },
          {
            "type": "Image",
            "source": "${imageUrl}",
            "width": "50vw",
            "height": "50vh",
            "scale": "best-fit"
          },
          {
            "type": "Text",
            "text": "${description}",
            "fontSize": "20dp",
            "textAlign": "center"
          }
        ]
      }
    ]
  }
}
```

### Implementação APL
```javascript
const APLHandler = {
    canHandle(handlerInput) {
        return Alexa.getRequestType(handlerInput.requestEnvelope) === 'IntentRequest'
            && Alexa.getIntentName(handlerInput.requestEnvelope) === 'ShowVisualIntent';
    },
    handle(handlerInput) {
        const aplDocument = require('./apl-document.json');
        
        const datasources = {
            "data": {
                "title": "Bem-vindo!",
                "imageUrl": "https://example.com/image.jpg",
                "description": "Esta é uma demonstração visual"
            }
        };
        
        if (Alexa.getSupportedInterfaces(handlerInput.requestEnvelope)['Alexa.Presentation.APL']) {
            return handlerInput.responseBuilder
                .speak('Aqui está a informação visual que você pediu.')
                .addDirective({
                    type: 'Alexa.Presentation.APL.RenderDocument',
                    document: aplDocument,
                    datasources: datasources
                })
                .getResponse();
        } else {
            return handlerInput.responseBuilder
                .speak('Esta funcionalidade requer um dispositivo com tela.')
                .getResponse();
        }
    }
};
```

## ⚡ Próximos Passos

1. **Escolha** um dos exemplos acima para implementar
2. **Adapte** o código para suas necessidades específicas  
3. **Teste** thoroughly no simulador e dispositivos
4. **Refine** baseado no feedback dos usuários
5. **Prepare** para [Deploy e Publicação](04-deployment.md)

---
**Anterior:** [← Desenvolvimento](02-desenvolvimento.md) | **Próximo:** [Deploy e Publicação →](04-deployment.md)
