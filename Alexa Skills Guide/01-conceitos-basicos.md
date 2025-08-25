# 1. Conceitos Básicos - Alexa Skills

## 🎯 O que são Alexa Skills?

Alexa Skills são aplicações de voz que estendem as capacidades da Amazon Alexa. Elas permitem que os usuários interajam com serviços e aplicações através de comandos de voz naturais.

## 📱 Tipos de Skills

### 1. **Custom Skills**
- Maior flexibilidade e controle
- Ideal para funcionalidades específicas
- Requer desenvolvimento completo do backend

### 2. **Smart Home Skills**
- Controle de dispositivos IoT
- Integração com sistema de casa inteligente
- Comandos padronizados ("Alexa, ligue a luz")

### 3. **Flash Briefing Skills**
- Fornecimento de notícias e atualizações
- Formato de áudio ou texto
- Integração com feeds RSS/JSON

### 4. **Music Skills**
- Streaming de música
- Controle de reprodução
- Integração com serviços de música

## 🏗️ Arquitetura de uma Alexa Skill

```
Usuário fala → Alexa Device → Amazon Voice Service → Alexa Skills Kit → Sua Aplicação
                                      ↓
Resposta ← Alexa Device ← Amazon Voice Service ← Alexa Skills Kit ← Sua Aplicação
```

### Componentes Principais:

#### 1. **Interaction Model (Modelo de Interação)**
```json
{
  "intents": [
    {
      "name": "GetWeatherIntent",
      "samples": [
        "qual é o tempo hoje",
        "como está o clima",
        "vai chover hoje"
      ],
      "slots": [
        {
          "name": "city",
          "type": "AMAZON.US_CITY"
        }
      ]
    }
  ]
}
```

#### 2. **Backend Service (AWS Lambda)**
```javascript
const Alexa = require('ask-sdk-core');

const LaunchRequestHandler = {
    canHandle(handlerInput) {
        return Alexa.getRequestType(handlerInput.requestEnvelope) === 'LaunchRequest';
    },
    handle(handlerInput) {
        const speakOutput = 'Olá! Como posso ajudar você hoje?';
        return handlerInput.responseBuilder
            .speak(speakOutput)
            .reprompt(speakOutput)
            .getResponse();
    }
};
```

## 🔧 Configuração do Ambiente

### Pré-requisitos:
1. **Conta Amazon Developer** (gratuita)
2. **AWS Account** (para hospedagem Lambda)
3. **Node.js** ou **Python** instalado
4. **ASK CLI** (Alexa Skills Kit Command Line Interface)

### Instalação do ASK CLI:
```bash
npm install -g ask-cli
ask configure
```

### Estrutura de Projeto:
```
minha-skill/
├── skill-package/
│   ├── interactionModels/
│   │   └── custom/
│   │       └── pt-BR.json
│   └── skill.json
├── lambda/
│   ├── index.js
│   ├── package.json
│   └── util.js
└── .ask/
    └── config
```

## 📝 Conceitos Fundamentais

### **Intents (Intenções)**
Representam ações que o usuário quer realizar:
- `GetWeatherIntent` - obter informações do tempo
- `PlayMusicIntent` - reproduzir música
- `OrderPizzaIntent` - fazer pedido de pizza

### **Utterances (Expressões)**
Frases que os usuários podem falar para acionar um intent:
```
GetWeatherIntent:
- "Como está o tempo?"
- "Vai chover hoje?"
- "Qual a temperatura?"
```

### **Slots (Parâmetros)**
Variáveis dentro das utterances:
```
"Qual o tempo em {city}"
"Toque música de {artist}"
"Defina alarme para {time}"
```

### **Session Attributes**
Dados persistidos durante uma sessão:
```javascript
const sessionAttributes = handlerInput.attributesManager.getSessionAttributes();
sessionAttributes.userName = 'João';
handlerInput.attributesManager.setSessionAttributes(sessionAttributes);
```

## 🎤 Fluxo de Interação

### 1. **Invocação**
```
"Alexa, abra minha skill personalizada"
"Alexa, peça para o assistente do tempo a previsão"
```

### 2. **Processamento**
- Alexa converte voz em texto
- Identifica o intent e extrai slots
- Envia request para seu backend

### 3. **Resposta**
```javascript
return handlerInput.responseBuilder
    .speak('A temperatura hoje é 25 graus')
    .withSimpleCard('Previsão do Tempo', 'Hoje: 25°C, ensolarado')
    .getResponse();
```

## 🛠️ Ferramentas de Desenvolvimento

### **1. Alexa Developer Console**
- Interface web para configurar skills
- Editor de interaction model
- Simulador para testes

### **2. ASK SDK**
- **Node.js**: `ask-sdk-core`
- **Python**: `ask-sdk-core`
- **Java**: `ask-java-sdk-core`

### **3. Alexa Simulator**
- Teste sem dispositivo físico
- Debug de requests/responses
- Visualização de cards

### **4. Developer Tools**
- **ASK CLI** - linha de comando
- **Alexa Skills Toolkit** (VS Code)
- **Bespoken Tools** - testing framework

## 📊 Tipos de Resposta

### **1. Resposta de Voz (Speech)**
```javascript
.speak('Olá, bem-vindo à minha skill!')
```

### **2. Cards Visuais**
```javascript
.withSimpleCard('Título', 'Conteúdo do card')
.withStandardCard('Título', 'Texto', smallImageUrl, largeImageUrl)
```

### **3. Repromt (Para manter sessão ativa)**
```javascript
.reprompt('Como posso ajudar você?')
```

### **4. Diretivas (Para dispositivos com tela)**
```javascript
.addDirective({
    type: 'Alexa.Presentation.APL.RenderDocument',
    document: aplDocument,
    datasources: aplDataSources
})
```

## ⚡ Próximos Passos

1. **Configure seu ambiente** seguindo os passos acima
2. **Crie sua primeira skill** no Developer Console
3. **Implemente o backend** usando AWS Lambda
4. **Teste** usando o simulador
5. **Avance** para [Desenvolvimento de Skills](02-desenvolvimento.md)

## 🔗 Links Úteis

- [Amazon Developer Portal](https://developer.amazon.com/alexa)
- [ASK SDK Documentation](https://ask-sdk-for-nodejs.readthedocs.io/)
- [Alexa Design Guide](https://developer.amazon.com/en-US/docs/alexa/alexa-design/get-started.html)
- [Voice Design Guide](https://developer.amazon.com/en-US/docs/alexa/alexa-design/design-process.html)

---
**Próximo:** [Desenvolvimento de Skills →](02-desenvolvimento.md)
