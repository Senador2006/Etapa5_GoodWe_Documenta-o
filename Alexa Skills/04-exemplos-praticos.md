# Exemplos Práticos de Alexa Skills - Node.js

## Projeto Completo: Assistente de Energia Solar (Node.js)

Este arquivo contém exemplos práticos completos para implementar uma Alexa Skill para monitoramento de energia solar usando Node.js e o ASK SDK v2.

## 1. Estrutura Completa do Projeto

```
alexa-energy-skill-nodejs/
├── lambda/
│   ├── package.json
│   ├── index.js
│   ├── handlers/
│   │   ├── launchRequestHandler.js
│   │   ├── energyHandlers.js
│   │   ├── deviceHandlers.js
│   │   └── builtInHandlers.js
│   ├── utils/
│   │   ├── apiClient.js
│   │   ├── responseBuilder.js
│   │   └── dataFormatter.js
│   └── interceptors/
│       ├── requestInterceptor.js
│       └── responseInterceptor.js
├── skill-package/
│   ├── interactionModels/
│   │   └── custom/
│   │       └── pt-BR.json
│   └── skill.json
└── tests/
    ├── unit/
    └── integration/
```

## 2. Interaction Model Completo

### skill-package/interactionModels/custom/pt-BR.json
```json
{
  "interactionModel": {
    "languageModel": {
      "invocationName": "assistente energia",
      "intents": [
        {
          "name": "LaunchIntent",
          "samples": [
            "olá",
            "oi",
            "começar",
            "iniciar"
          ]
        },
        {
          "name": "GetEnergyConsumptionIntent",
          "slots": [
            {
              "name": "timeFrame",
              "type": "TimeFrameSlot"
            }
          ],
          "samples": [
            "qual o consumo de energia {timeFrame}",
            "quanto gastei de energia {timeFrame}",
            "me diga o gasto de energia {timeFrame}",
            "consumo {timeFrame}",
            "gasto energético {timeFrame}",
            "quanto consumi {timeFrame}",
            "qual foi o consumo {timeFrame}"
          ]
        },
        {
          "name": "GetEnergyProductionIntent",
          "slots": [
            {
              "name": "timeFrame",
              "type": "TimeFrameSlot"
            }
          ],
          "samples": [
            "quanta energia foi gerada {timeFrame}",
            "produção de energia {timeFrame}",
            "quanto os painéis geraram {timeFrame}",
            "geração {timeFrame}",
            "quanto foi produzido {timeFrame}",
            "energia gerada {timeFrame}"
          ]
        },
        {
          "name": "GetDeviceStatusIntent",
          "slots": [
            {
              "name": "deviceType",
              "type": "DeviceTypeSlot"
            }
          ],
          "samples": [
            "como está o {deviceType}",
            "qual o status do {deviceType}",
            "me mostre informações sobre o {deviceType}",
            "verifique o {deviceType}",
            "status do {deviceType}",
            "como está funcionando o {deviceType}"
          ]
        },
        {
          "name": "GetSavingsIntent",
          "slots": [
            {
              "name": "timeFrame",
              "type": "TimeFrameSlot"
            }
          ],
          "samples": [
            "quanto economizei {timeFrame}",
            "qual a economia {timeFrame}",
            "me diga a economia {timeFrame}",
            "economias {timeFrame}",
            "quanto pouquei {timeFrame}"
          ]
        },
        {
          "name": "GetWeatherImpactIntent",
          "samples": [
            "como está o clima afetando a geração",
            "impacto do tempo na produção",
            "tempo está bom para energia solar",
            "clima e energia solar"
          ]
        },
        {
          "name": "AMAZON.CancelIntent",
          "samples": []
        },
        {
          "name": "AMAZON.HelpIntent",
          "samples": []
        },
        {
          "name": "AMAZON.StopIntent",
          "samples": []
        }
      ],
      "types": [
        {
          "name": "TimeFrameSlot",
          "values": [
            {
              "name": {
                "value": "hoje",
                "synonyms": ["hoje em dia", "neste dia"]
              }
            },
            {
              "name": {
                "value": "ontem",
                "synonyms": ["dia anterior", "anteontem"]
              }
            },
            {
              "name": {
                "value": "esta semana",
                "synonyms": ["semana atual", "nesta semana"]
              }
            },
            {
              "name": {
                "value": "semana passada",
                "synonyms": ["última semana", "semana anterior"]
              }
            },
            {
              "name": {
                "value": "este mês",
                "synonyms": ["mês atual", "neste mês"]
              }
            },
            {
              "name": {
                "value": "mês passado",
                "synonyms": ["último mês", "mês anterior"]
              }
            }
          ]
        },
        {
          "name": "DeviceTypeSlot",
          "values": [
            {
              "name": {
                "value": "inversor",
                "synonyms": ["inverter", "conversor de energia"]
              }
            },
            {
              "name": {
                "value": "painel solar",
                "synonyms": ["painel", "placa solar", "módulo fotovoltaico", "painéis"]
              }
            },
            {
              "name": {
                "value": "bateria",
                "synonyms": ["banco de bateria", "armazenamento", "baterias"]
              }
            },
            {
              "name": {
                "value": "sistema",
                "synonyms": ["sistema completo", "instalação", "usina solar"]
              }
            }
          ]
        }
      ]
    }
  }
}
```

## 3. Código Lambda Completo

### lambda/requirements.txt
```
ask-sdk-core==1.18.0
requests==2.28.0
boto3==1.26.0
```

> 💡 **Implementação em Node.js**: Todo o código está disponível de forma detalhada no arquivo [`05-codigo-nodejs-completo.md`](./05-codigo-nodejs-completo.md), incluindo todas as classes, handlers e configurações necessárias.

```

## 3. Código Node.js Completo

> 📋 **Nota**: O código completo está disponível no arquivo [`05-codigo-nodejs-completo.md`](./05-codigo-nodejs-completo.md). Aqui está um resumo da estrutura e componentes principais.

### Estrutura Resumida do Projeto
```
alexa-energy-skill-nodejs/
├── package.json              # Dependências e scripts
├── index.js                  # Arquivo principal
├── handlers/                 # Manipuladores de intents
│   ├── launchRequestHandler.js
│   ├── energyHandlers.js
│   ├── deviceHandlers.js
│   └── builtInHandlers.js
├── utils/                    # Utilitários
│   ├── apiClient.js         # Cliente para API GoodWe
│   ├── dataFormatter.js     # Formatação de dados
│   └── responseBuilder.js   # Construção de respostas
└── interceptors/            # Interceptadores
    ├── requestInterceptor.js
    └── responseInterceptor.js
```

### Principais Dependências
```json
{
  "dependencies": {
    "ask-sdk-core": "^2.13.0",
    "ask-sdk-model": "^1.49.0", 
    "axios": "^1.6.0",
    "moment": "^2.29.4"
  }
}
```

### Componentes Principais

#### 1. **API Client** (utils/apiClient.js)
- Gerencia chamadas para a API GoodWe
- Tratamento de erros e timeouts
- Logging automático de requisições

#### 2. **Data Formatter** (utils/dataFormatter.js)
- Formatação de valores de energia, moeda e percentuais
- Normalização de períodos de tempo
- Extração de valores de slots

#### 3. **Response Builder** (utils/responseBuilder.js)
- Construção de respostas padronizadas
- Tratamento de erros
- Geração de cards para dispositivos com tela

#### 4. **Handlers** (handlers/)
- **Energy Handlers**: Consumo, produção e economias
- **Device Handlers**: Status de dispositivos e impacto climático  
- **Built-in Handlers**: Ajuda, cancelar, parar

#### 5. **Interceptors** (interceptors/)
- **Request**: Logging detalhado de requisições
- **Response**: Métricas e logging de respostas

### Exemplo de Handler Simplificado
```javascript
const GetEnergyConsumptionHandler = {
    canHandle(handlerInput) {
        return handlerInput.requestEnvelope.request.type === 'IntentRequest'
            && handlerInput.requestEnvelope.request.intent.name === 'GetEnergyConsumptionIntent';
    },
    
    async handle(handlerInput) {
        try {
            // Extrair período do slot
            const timeFrame = DataFormatter.extractSlotValue(
                handlerInput.requestEnvelope.request.intent.slots.timeFrame
            );
            
            // Buscar dados na API
            const data = await apiClient.getEnergyConsumption(timeFrame);
            
            // Formatar resposta
            const formattedResponse = DataFormatter.formatEnergyValue(
                data.total_consumption, data.unit
            );
            
            const speechText = `O consumo foi de ${formattedResponse}.`;
            
            return ResponseBuilder.buildResponse(handlerInput, speechText);
            
        } catch (error) {
            return ResponseBuilder.buildErrorResponse(handlerInput, 'api_error');
        }
    }
};
```

### Arquivo Principal (index.js)
```javascript
const Alexa = require('ask-sdk-core');

// Importar todos os handlers
const handlers = [...]; 

// Configurar skill
exports.handler = Alexa.SkillBuilders.custom()
    .addRequestHandlers(...handlers)
    .addErrorHandlers(ErrorHandler)
    .addRequestInterceptors(RequestInterceptor)
    .addResponseInterceptors(ResponseInterceptor)
    .lambda();
```

## 4. Configuração de Ambiente e Testes

### Variáveis de Ambiente
```javascript
// Configurar no AWS Lambda Console ou via CLI
const environmentVariables = {
    GOODWE_API_KEY: 'sua_chave_api_aqui',
    API_BASE_URL: 'https://api.goodwe.com/v1',
    LOG_LEVEL: 'INFO'
};
```

### Servidor de Testes Local
```javascript
// tests/mockServer.js
const express = require('express');
const app = express();

app.use(express.json());

// Mock data
const mockData = {
    consumption: {
        today: { total_consumption: 25.5, unit: 'kWh' },
        yesterday: { total_consumption: 28.2, unit: 'kWh' },
        this_week: { total_consumption: 180.3, unit: 'kWh' }
    },
    production: {
        today: { total_production: 45.2, unit: 'kWh', efficiency: 92.5 },
        yesterday: { total_production: 48.1, unit: 'kWh', efficiency: 94.2 }
    },
    devices: {
        inverter: { 
            status: 'operacional', 
            current_power: 1200, 
            efficiency: 95.2 
        },
        battery: { 
            status: 'carregando', 
            charge_level: 85, 
            charge_status: 'carregando' 
        },
        system: {
            status: 'operacional',
            summary: {
                generation: 1200,
                consumption: 800,
                battery_level: 85
            }
        }
    }
};

// Endpoints
app.get('/v1/energy/consumption', (req, res) => {
    const period = req.query.period || 'today';
    const data = mockData.consumption[period] || mockData.consumption.today;
    res.json({ data });
});

app.get('/v1/energy/production', (req, res) => {
    const period = req.query.period || 'today';
    const data = mockData.production[period] || mockData.production.today;
    res.json({ data });
});

app.get('/v1/devices/:type', (req, res) => {
    const type = req.params.type.replace('-', '');
    const data = mockData.devices[type] || mockData.devices.system;
    res.json({ data });
});

app.get('/v1/financial/savings', (req, res) => {
    res.json({ 
        data: { 
            total_savings: 45.80, 
            currency: 'R$',
            energy_offset: 65.2
        } 
    });
});

app.get('/v1/weather/impact', (req, res) => {
    res.json({ 
        data: { 
            condition: 'favoráveis', 
            impact_percentage: 92.5,
            forecast: 'sol durante o dia'
        } 
    });
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
    console.log(`Mock server running on port ${PORT}`);
});

module.exports = app;
```

### Scripts de Teste
```json
// package.json - scripts adicionais
{
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
  }
}
```

### Comandos de Teste
```bash
# Instalar dependências
npm install

# Executar servidor mock para testes
npm run mock-server

# Executar testes unitários
npm test

# Deploy da skill
npm run deploy

# Simular interações
ask simulate -l pt-BR -t "abra assistente energia"
ask simulate -l pt-BR -t "qual o consumo de energia hoje"
ask simulate -l pt-BR -t "como está o inversor"
```

### Frases para Testar
```
# Ativação
"Alexa, abra assistente energia"
"Alexa, abrir assistente energia"

# Consumo
"qual o consumo de energia hoje"
"quanto gastei de energia esta semana"
"consumo de ontem"
"me diga o gasto de energia este mês"

# Produção
"quanta energia foi gerada hoje"
"produção desta semana"
"quanto os painéis geraram ontem"
"geração de energia hoje"

# Status de Dispositivos
"como está o inversor"
"qual o status da bateria"
"verifique o sistema"
"status do painel solar"

# Economias
"quanto economizei este mês"
"qual a economia de hoje"
"economias desta semana"

# Clima
"como está o clima afetando a geração"
"impacto do tempo na produção"

# Ajuda e Navegação
"ajuda"
"o que você pode fazer"
"parar"
"cancelar"
```

## 5. Configuração de Deploy

### skill.json
```json
{
  "manifest": {
    "publishingInformation": {
      "locales": {
        "pt-BR": {
          "name": "Assistente de Energia Solar GoodWe",
          "summary": "Monitore seu sistema de energia solar por voz",
          "description": "Acompanhe consumo, produção, status dos dispositivos e economias do seu sistema de energia solar GoodWe através de comandos de voz simples e intuitivos.",
          "keywords": ["energia solar", "monitoramento", "sustentabilidade", "goodwe", "inversor"],
          "examplePhrases": [
            "Alexa, abra assistente energia",
            "qual o consumo de energia hoje",
            "como está o inversor"
          ],
          "smallIconUri": "https://s3.amazonaws.com/your-bucket/icons/small-icon.png",
          "largeIconUri": "https://s3.amazonaws.com/your-bucket/icons/large-icon.png"
        }
      },
      "isAvailableWorldwide": false,
      "distributionCountries": ["BR"],
      "category": "SMART_HOME"
    },
    "apis": {
      "custom": {
        "endpoint": {
          "uri": "arn:aws:lambda:us-east-1:123456789012:function:alexa-energy-skill"
        },
        "interfaces": []
      }
    },
    "manifestVersion": "1.0"
  }
}
```

### Deploy automatizado
```javascript
// scripts/deploy.js
const { execSync } = require('child_process');
const fs = require('fs');

function deploy() {
    console.log('🚀 Iniciando deploy da Alexa Skill...');
    
    try {
        // Verificar se ASK CLI está configurado
        execSync('ask --version', { stdio: 'pipe' });
        
        // Fazer build do código
        console.log('📦 Instalando dependências...');
        execSync('npm install --production', { stdio: 'inherit' });
        
        // Deploy da skill
        console.log('🎯 Fazendo deploy...');
        execSync('ask deploy', { stdio: 'inherit' });
        
        // Executar testes
        console.log('🧪 Executando testes...');
        execSync('npm test', { stdio: 'inherit' });
        
        console.log('✅ Deploy concluído com sucesso!');
        
    } catch (error) {
        console.error('❌ Erro no deploy:', error.message);
        process.exit(1);
    }
}

if (require.main === module) {
    deploy();
}

module.exports = deploy;
```

### Configuração do AWS Lambda
```yaml
# template.yaml para SAM
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Resources:
  AlexaEnergySkillFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: lambda/
      Handler: index.handler
      Runtime: nodejs18.x
      Timeout: 30
      MemorySize: 256
      Environment:
        Variables:
          GOODWE_API_KEY: !Ref GoodWeApiKey
          API_BASE_URL: !Ref ApiBaseUrl
          LOG_LEVEL: INFO
      Events:
        AlexaSkillEvent:
          Type: AlexaSkill

Parameters:
  GoodWeApiKey:
    Type: String
    Description: API Key for GoodWe API
    NoEcho: true
    
  ApiBaseUrl:
    Type: String
    Default: https://api.goodwe.com/v1
    Description: Base URL for GoodWe API
```

## 6. Monitoramento e Logs

### CloudWatch Logs
```javascript
// Adicionar ao início dos handlers para melhor logging
const logRequest = (handlerInput) => {
    const request = handlerInput.requestEnvelope.request;
    console.log(JSON.stringify({
        timestamp: new Date().toISOString(),
        requestId: handlerInput.requestEnvelope.request.requestId,
        sessionId: handlerInput.requestEnvelope.session?.sessionId,
        userId: handlerInput.requestEnvelope.session?.user?.userId,
        intentName: request.intent?.name,
        locale: handlerInput.requestEnvelope.request.locale
    }));
};

const logResponse = (response, duration) => {
    console.log(JSON.stringify({
        timestamp: new Date().toISOString(),
        responseType: response.outputSpeech?.type,
        speechLength: response.outputSpeech?.ssml?.length || 0,
        hasCard: !!response.card,
        shouldEndSession: response.shouldEndSession,
        duration: `${duration}ms`
    }));
};
```

## 7. Implementações Disponíveis

### 🎯 **IMPLEMENTAÇÃO RECOMENDADA: Estrutura Otimizada**
> 🔥 **NOVO - Otimizada para Console**: [`09-estrutura-otimizada-lambda.md`](./09-estrutura-otimizada-lambda.md)
>
> **Apenas 10 arquivos** vs 27+ da versão completa - ideal para console AWS Lambda
>
> 📋 **Implementação Completa**: [`08-implementacao-aws-lambda-definitiva.md`](./08-implementacao-aws-lambda-definitiva.md)
>
> 🚀 **Quick Start**: [`README-IMPLEMENTACAO.md`](./README-IMPLEMENTACAO.md) - Checklist e resumo executivo

### 🐳 **Abordagem Alternativa: Docker + Endpoints**
> Para maior flexibilidade e controle, você pode implementar usando Docker:
> 
> - [`06-skill-com-docker-endpoints.md`](./06-skill-com-docker-endpoints.md) - Arquitetura completa com Docker
> - [`07-handlers-docker-implementation.md`](./07-handlers-docker-implementation.md) - Implementação detalhada dos handlers

### Vantagens da Abordagem Docker:
- **Controle Total**: Infraestrutura própria
- **Debugging Fácil**: Logs completos e acesso direto  
- **Escalabilidade**: Docker Swarm ou Kubernetes
- **Integração**: Conexão simples com outros serviços
- **Custo**: Potencialmente menor que AWS Lambda
- **Cache Redis**: Melhor performance com cache integrado

### Quando Usar Cada Abordagem:

| Aspecto | AWS Lambda | Docker + Endpoints |
|---------|------------|-------------------|
| **Simplicidade** | ✅ Mais simples | ⚠️ Mais complexo |
| **Controle** | ⚠️ Limitado | ✅ Total |
| **Debugging** | ⚠️ CloudWatch | ✅ Logs diretos |
| **Custo** | ⚠️ Pay-per-use | ✅ Fixo/menor |
| **Escalabilidade** | ✅ Automática | ⚠️ Manual |
| **Manutenção** | ✅ Menor | ⚠️ Maior |

---

*Este documento fornece exemplos práticos completos para implementar uma Alexa Skill de monitoramento de energia solar no projeto GoodWe, com opções tanto para AWS Lambda quanto para Docker.*
