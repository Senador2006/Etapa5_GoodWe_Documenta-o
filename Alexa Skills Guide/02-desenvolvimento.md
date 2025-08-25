# 2. Desenvolvimento de Skills

## 🚀 Criando sua Primeira Skill

### Passo 1: Configuração Inicial

#### Criar Nova Skill no Developer Console
1. Acesse [developer.amazon.com/alexa](https://developer.amazon.com/alexa)
2. Clique em "Create Skill"
3. Configure:
   - **Skill name**: "Minha Primeira Skill"
   - **Primary locale**: Portuguese (BR)
   - **Model**: Custom
   - **Hosting**: Alexa-hosted (Node.js)

### Passo 2: Definindo o Interaction Model

#### Invocation Name
```json
{
  "invocationName": "assistente pessoal"
}
```

#### Intents Básicos
```json
{
  "intents": [
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
    },
    {
      "name": "AMAZON.NavigateHomeIntent",
      "samples": []
    },
    {
      "name": "SaudacaoIntent",
      "slots": [
        {
          "name": "nome",
          "type": "AMAZON.FirstName"
        }
      ],
      "samples": [
        "olá",
        "oi",
        "bom dia",
        "boa tarde", 
        "boa noite",
        "olá meu nome é {nome}",
        "oi eu sou {nome}"
      ]
    },
    {
      "name": "ConsultarHorarioIntent",
      "samples": [
        "que horas são",
        "qual o horário",
        "me diga as horas"
      ]
    }
  ]
}
```

## 💻 Implementação do Backend (AWS Lambda)

### Estrutura Básica do Código

```javascript
const Alexa = require('ask-sdk-core');

// Handler para quando a skill é aberta
const LaunchRequestHandler = {
    canHandle(handlerInput) {
        return Alexa.getRequestType(handlerInput.requestEnvelope) === 'LaunchRequest';
    },
    handle(handlerInput) {
        const speakOutput = 'Olá! Bem-vindo ao seu assistente pessoal. Como posso ajudar?';
        
        return handlerInput.responseBuilder
            .speak(speakOutput)
            .reprompt('Como posso ajudar você hoje?')
            .getResponse();
    }
};

// Handler para saudações
const SaudacaoIntentHandler = {
    canHandle(handlerInput) {
        return Alexa.getRequestType(handlerInput.requestEnvelope) === 'IntentRequest'
            && Alexa.getIntentName(handlerInput.requestEnvelope) === 'SaudacaoIntent';
    },
    handle(handlerInput) {
        const nome = Alexa.getSlotValue(handlerInput.requestEnvelope, 'nome');
        let speakOutput;
        
        if (nome) {
            speakOutput = `Olá ${nome}! É um prazer conhecer você. Como posso ajudar?`;
            
            // Salvar nome na sessão
            const sessionAttributes = handlerInput.attributesManager.getSessionAttributes();
            sessionAttributes.userName = nome;
            handlerInput.attributesManager.setSessionAttributes(sessionAttributes);
        } else {
            speakOutput = 'Olá! Como você está hoje? Como posso ajudar?';
        }
        
        return handlerInput.responseBuilder
            .speak(speakOutput)
            .reprompt('Como posso ajudar você?')
            .getResponse();
    }
};

// Handler para consultar horário
const ConsultarHorarioIntentHandler = {
    canHandle(handlerInput) {
        return Alexa.getRequestType(handlerInput.requestEnvelope) === 'IntentRequest'
            && Alexa.getIntentName(handlerInput.requestEnvelope) === 'ConsultarHorarioIntent';
    },
    handle(handlerInput) {
        const agora = new Date();
        const horas = agora.getHours();
        const minutos = agora.getMinutes();
        
        const sessionAttributes = handlerInput.attributesManager.getSessionAttributes();
        const userName = sessionAttributes.userName;
        
        let saudacao = '';
        if (userName) {
            saudacao = `${userName}, `;
        }
        
        const speakOutput = `${saudacao}agora são ${horas} horas e ${minutos} minutos.`;
        
        return handlerInput.responseBuilder
            .speak(speakOutput)
            .reprompt('Precisa de mais alguma coisa?')
            .getResponse();
    }
};

// Handler para ajuda
const HelpIntentHandler = {
    canHandle(handlerInput) {
        return Alexa.getRequestType(handlerInput.requestEnvelope) === 'IntentRequest'
            && Alexa.getIntentName(handlerInput.requestEnvelope) === 'AMAZON.HelpIntent';
    },
    handle(handlerInput) {
        const speakOutput = `Eu posso ajudar você com várias coisas! 
            Você pode me cumprimentar dizendo "olá" ou "bom dia", 
            perguntar as horas dizendo "que horas são", 
            ou me dizer seu nome. O que gostaria de fazer?`;

        return handlerInput.responseBuilder
            .speak(speakOutput)
            .reprompt(speakOutput)
            .getResponse();
    }
};

// Handlers para cancelar/parar
const CancelAndStopIntentHandler = {
    canHandle(handlerInput) {
        return Alexa.getRequestType(handlerInput.requestEnvelope) === 'IntentRequest'
            && (Alexa.getIntentName(handlerInput.requestEnvelope) === 'AMAZON.CancelIntent'
                || Alexa.getIntentName(handlerInput.requestEnvelope) === 'AMAZON.StopIntent');
    },
    handle(handlerInput) {
        const sessionAttributes = handlerInput.attributesManager.getSessionAttributes();
        const userName = sessionAttributes.userName;
        
        let despedida = 'Até logo!';
        if (userName) {
            despedida = `Até logo, ${userName}!`;
        }
        
        return handlerInput.responseBuilder
            .speak(despedida)
            .getResponse();
    }
};

// Handler para fim de sessão
const SessionEndedRequestHandler = {
    canHandle(handlerInput) {
        return Alexa.getRequestType(handlerInput.requestEnvelope) === 'SessionEndedRequest';
    },
    handle(handlerInput) {
        console.log(`~~~~ Session ended: ${JSON.stringify(handlerInput.requestEnvelope)}`);
        return handlerInput.responseBuilder.getResponse();
    }
};

// Handler para intents não reconhecidos
const IntentReflectorHandler = {
    canHandle(handlerInput) {
        return Alexa.getRequestType(handlerInput.requestEnvelope) === 'IntentRequest';
    },
    handle(handlerInput) {
        const intentName = Alexa.getIntentName(handlerInput.requestEnvelope);
        const speakOutput = `Desculpe, não entendi "${intentName}". Tente novamente.`;

        return handlerInput.responseBuilder
            .speak(speakOutput)
            .reprompt('Como posso ajudar você?')
            .getResponse();
    }
};

// Handler de erro
const ErrorHandler = {
    canHandle() {
        return true;
    },
    handle(handlerInput, error) {
        const speakOutput = 'Desculpe, tive um problema. Tente novamente.';
        console.log(`~~~~ Error handled: ${JSON.stringify(error)}`);

        return handlerInput.responseBuilder
            .speak(speakOutput)
            .reprompt(speakOutput)
            .getResponse();
    }
};

// Função principal
exports.handler = Alexa.SkillBuilders.custom()
    .addRequestHandlers(
        LaunchRequestHandler,
        SaudacaoIntentHandler,
        ConsultarHorarioIntentHandler,
        HelpIntentHandler,
        CancelAndStopIntentHandler,
        SessionEndedRequestHandler,
        IntentReflectorHandler)
    .addErrorHandlers(
        ErrorHandler)
    .withCustomUserAgent('sample/hello-world/v1.2')
    .lambda();
```

## 🔧 Funcionalidades Avançadas

### 1. Integração com APIs Externas

```javascript
const https = require('https');

const ConsultarClimaIntentHandler = {
    canHandle(handlerInput) {
        return Alexa.getRequestType(handlerInput.requestEnvelope) === 'IntentRequest'
            && Alexa.getIntentName(handlerInput.requestEnvelope) === 'ConsultarClimaIntent';
    },
    async handle(handlerInput) {
        const cidade = Alexa.getSlotValue(handlerInput.requestEnvelope, 'cidade') || 'São Paulo';
        
        try {
            const dadosClima = await obterDadosClima(cidade);
            const temperatura = dadosClima.main.temp;
            const descricao = dadosClima.weather[0].description;
            
            const speakOutput = `Em ${cidade}, a temperatura é de ${Math.round(temperatura)}° graus e o tempo está ${descricao}.`;
            
            return handlerInput.responseBuilder
                .speak(speakOutput)
                .withSimpleCard('Previsão do Tempo', `${cidade}: ${Math.round(temperatura)}°C, ${descricao}`)
                .reprompt('Precisa da previsão de outra cidade?')
                .getResponse();
                
        } catch (error) {
            const speakOutput = 'Desculpe, não consegui obter a previsão do tempo agora.';
            return handlerInput.responseBuilder
                .speak(speakOutput)
                .reprompt('Como posso ajudar você?')
                .getResponse();
        }
    }
};

function obterDadosClima(cidade) {
    return new Promise((resolve, reject) => {
        const apiKey = 'SUA_API_KEY'; // Configure no console AWS Lambda
        const url = `https://api.openweathermap.org/data/2.5/weather?q=${cidade}&appid=${apiKey}&units=metric&lang=pt_br`;
        
        https.get(url, (response) => {
            let data = '';
            
            response.on('data', (chunk) => {
                data += chunk;
            });
            
            response.on('end', () => {
                try {
                    const parsedData = JSON.parse(data);
                    resolve(parsedData);
                } catch (error) {
                    reject(error);
                }
            });
        }).on('error', (error) => {
            reject(error);
        });
    });
}
```

### 2. Persistência de Dados (DynamoDB)

```javascript
const persistenceAdapter = require('ask-sdk-dynamodb-persistence-adapter');

// Configurar adaptador de persistência
const skillBuilder = Alexa.SkillBuilders.custom()
    .withPersistenceAdapter(
        new persistenceAdapter.DynamoDbPersistenceAdapter({
            tableName: 'MinhaSkillTable',
            createTable: true,
            dynamoDBClient: undefined
        })
    );

// Salvar preferências do usuário
const SalvarPreferenciaHandler = {
    canHandle(handlerInput) {
        return Alexa.getRequestType(handlerInput.requestEnvelope) === 'IntentRequest'
            && Alexa.getIntentName(handlerInput.requestEnvelope) === 'SalvarPreferenciaIntent';
    },
    async handle(handlerInput) {
        const cor = Alexa.getSlotValue(handlerInput.requestEnvelope, 'cor');
        
        // Obter atributos persistentes
        const attributesManager = handlerInput.attributesManager;
        const persistentAttributes = await attributesManager.getPersistentAttributes() || {};
        
        // Salvar preferência
        persistentAttributes.corFavorita = cor;
        await attributesManager.setPersistentAttributes(persistentAttributes);
        await attributesManager.savePersistentAttributes();
        
        const speakOutput = `Perfeito! Salvei que sua cor favorita é ${cor}.`;
        
        return handlerInput.responseBuilder
            .speak(speakOutput)
            .reprompt('O que mais posso fazer por você?')
            .getResponse();
    }
};
```

### 3. Slots Personalizados

```json
{
  "types": [
    {
      "name": "CoresType",
      "values": [
        {
          "name": {
            "value": "azul",
            "synonyms": ["azul", "cor azul", "azulado"]
          }
        },
        {
          "name": {
            "value": "vermelho", 
            "synonyms": ["vermelho", "cor vermelha", "avermelhado"]
          }
        },
        {
          "name": {
            "value": "verde",
            "synonyms": ["verde", "cor verde", "esverdeado"]
          }
        }
      ]
    }
  ]
}
```

### 4. Validação de Slots

```javascript
const ValidarIdadeHandler = {
    canHandle(handlerInput) {
        return Alexa.getRequestType(handlerInput.requestEnvelope) === 'IntentRequest'
            && Alexa.getIntentName(handlerInput.requestEnvelope) === 'InformarIdadeIntent';
    },
    handle(handlerInput) {
        const idade = parseInt(Alexa.getSlotValue(handlerInput.requestEnvelope, 'idade'));
        
        if (!idade || idade < 0 || idade > 150) {
            return handlerInput.responseBuilder
                .speak('Desculpe, não entendi sua idade. Pode repetir um número válido?')
                .addElicitSlotDirective('idade')
                .getResponse();
        }
        
        let categoria;
        if (idade < 18) {
            categoria = 'menor de idade';
        } else if (idade < 65) {
            categoria = 'adulto';
        } else {
            categoria = 'idoso';
        }
        
        const speakOutput = `Entendi, você tem ${idade} anos e é considerado ${categoria}.`;
        
        return handlerInput.responseBuilder
            .speak(speakOutput)
            .reprompt('Como posso ajudar você?')
            .getResponse();
    }
};
```

## 🧪 Testando sua Skill

### 1. Teste no Console
- Use o simulador integrado
- Teste diferentes frases e cenários
- Verifique responses e cards

### 2. Teste com Dispositivos
```bash
# Habilitar skill para teste
ask api enable-skill --skill-id YOUR_SKILL_ID
```

### 3. Logs e Debug
```javascript
// Adicionar logs detalhados
console.log('Request received:', JSON.stringify(handlerInput.requestEnvelope, null, 2));
console.log('Session attributes:', handlerInput.attributesManager.getSessionAttributes());
```

## 📝 Boas Práticas

### 1. **Design de Conversação**
- Use linguagem natural e conversacional
- Forneça opções claras para o usuário
- Mantenha respostas concisas

### 2. **Tratamento de Erros**
- Sempre tenha fallbacks
- Seja específico sobre o que deu errado
- Ofereça alternativas

### 3. **Performance**
- Mantenha responses rápidas (< 8 segundos)
- Use async/await para APIs externas
- Implemente timeout adequado

### 4. **Acessibilidade**
- Considere usuários com diferentes habilidades
- Forneça múltiplas formas de completar tarefas
- Use linguagem inclusiva

## 🔄 Fluxo de Desenvolvimento

1. **Planejamento**
   - Defina objetivo da skill
   - Mapeie jornadas do usuário
   - Identifique intents necessários

2. **Implementação**
   - Configure interaction model
   - Desenvolva handlers
   - Integre APIs necessárias

3. **Teste**
   - Teste funcionalidades básicas
   - Valide diferentes cenários
   - Teste em dispositivos reais

4. **Refinamento**
   - Ajuste com base nos testes
   - Otimize responses
   - Melhore tratamento de erros

## ⚡ Próximos Passos

1. **Implemente** os códigos acima
2. **Teste** sua skill no simulador
3. **Explore** [Exemplos Práticos](03-exemplos-praticos.md)
4. **Prepare** para [Deploy e Publicação](04-deployment.md)

---
**Anterior:** [← Conceitos Básicos](01-conceitos-basicos.md) | **Próximo:** [Exemplos Práticos →](03-exemplos-praticos.md)
