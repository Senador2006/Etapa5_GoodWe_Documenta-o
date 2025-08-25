# Criando e Configurando Intents

## O que são Intents?

Intents representam ações que o usuário deseja realizar. Cada intent define uma funcionalidade específica da sua skill e inclui:

- **Nome do Intent**: Identificador único
- **Sample Utterances**: Frases que ativam o intent
- **Slots**: Variáveis que capturam informações específicas

## Estrutura de um Intent

```json
{
  "name": "GetEnergyConsumptionIntent",
  "slots": [
    {
      "name": "timeFrame",
      "type": "AMAZON.DURATION"
    },
    {
      "name": "deviceType",
      "type": "DeviceTypeSlot"
    }
  ],
  "samples": [
    "qual o consumo de energia {timeFrame}",
    "me diga o gasto de {deviceType} {timeFrame}",
    "quanto gastei de energia hoje",
    "consumo do {deviceType}"
  ]
}
```

## Tipos de Intents

### 1. Built-in Intents
Intents pré-definidos pela Amazon:

```json
{
  "name": "AMAZON.StopIntent",
  "samples": []
}
```

**Intents obrigatórios:**
- `AMAZON.CancelIntent` - Cancelar operação
- `AMAZON.HelpIntent` - Solicitar ajuda
- `AMAZON.StopIntent` - Parar skill

**Intents úteis:**
- `AMAZON.YesIntent` - Confirmação positiva
- `AMAZON.NoIntent` - Confirmação negativa
- `AMAZON.RepeatIntent` - Repetir última resposta

### 2. Custom Intents
Intents específicos para sua aplicação:

```json
{
  "name": "GetDeviceStatusIntent",
  "slots": [
    {
      "name": "deviceName",
      "type": "DeviceNameSlot"
    }
  ],
  "samples": [
    "qual o status do {deviceName}",
    "como está o {deviceName}",
    "me mostre informações sobre {deviceName}",
    "verifique o {deviceName}"
  ]
}
```

## Trabalhando com Slots

### Slot Types Básicos

#### 1. Amazon Built-in Slot Types
```json
{
  "name": "date",
  "type": "AMAZON.DATE"
}
```

**Tipos comuns:**
- `AMAZON.DATE` - Datas
- `AMAZON.TIME` - Horários  
- `AMAZON.DURATION` - Períodos
- `AMAZON.NUMBER` - Números
- `AMAZON.Percentage` - Porcentagens

#### 2. Custom Slot Types
```json
{
  "name": "DeviceTypeSlot",
  "values": [
    {
      "name": {
        "value": "inversor",
        "synonyms": ["inverter", "conversor"]
      }
    },
    {
      "name": {
        "value": "painel solar",
        "synonyms": ["painel", "placa solar", "módulo fotovoltaico"]
      }
    },
    {
      "name": {
        "value": "bateria",
        "synonyms": ["banco de bateria", "armazenamento"]
      }
    }
  ]
}
```

## Boas Práticas para Sample Utterances

### 1. Variedade de Expressões
```json
"samples": [
  "qual o consumo de energia hoje",
  "me diga quanto gastei de energia hoje",
  "quanto foi o gasto energético de hoje",
  "consumo de hoje",
  "gasto de energia do dia",
  "quanto consumi hoje"
]
```

### 2. Diferentes Estruturas
```json
"samples": [
  "ligue o {deviceName}",           // Imperativo direto
  "pode ligar o {deviceName}",      // Pergunta educada
  "quero ligar o {deviceName}",     // Declaração de intenção
  "preciso ligar o {deviceName}"    // Necessidade
]
```

### 3. Com e Sem Slots
```json
"samples": [
  "qual o status dos dispositivos",    // Sem slot - todos
  "qual o status do {deviceName}",     // Com slot - específico
  "me mostre o status",                // Genérico
  "verifique o {deviceName}"          // Específico
]
```

## Exemplo Prático: Sistema de Energia Solar

### Intent Model Completo

```json
{
  "interactionModel": {
    "languageModel": {
      "invocationName": "assistente energia",
      "intents": [
        {
          "name": "GetEnergyConsumptionIntent",
          "slots": [
            {
              "name": "timeFrame",
              "type": "AMAZON.DURATION"
            }
          ],
          "samples": [
            "qual o consumo de energia {timeFrame}",
            "quanto gastei {timeFrame}",
            "me diga o gasto de energia {timeFrame}",
            "consumo {timeFrame}",
            "gasto energético {timeFrame}"
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
            "me mostre informações sobre {deviceType}",
            "verifique o {deviceType}",
            "status do {deviceType}"
          ]
        },
        {
          "name": "GetEnergyProductionIntent",
          "slots": [
            {
              "name": "timeFrame",
              "type": "AMAZON.DURATION"
            }
          ],
          "samples": [
            "quanta energia foi gerada {timeFrame}",
            "produção de energia {timeFrame}",
            "quanto os painéis geraram {timeFrame}",
            "geração {timeFrame}"
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
                "synonyms": ["banco de bateria", "armazenamento", "bateria"]
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

## Testando Intents

### 1. No Alexa Developer Console
- Use o simulador integrado
- Teste diferentes utterances
- Verifique se os slots são capturados corretamente

### 2. Comandos de Teste
```
"Alexa, abra assistente energia"
"qual o consumo de energia hoje"
"como está o inversor"
"quanta energia foi gerada esta semana"
```

## Tratamento de Intents no Backend

### Exemplo em Python (ASK SDK)
```python
@sb.request_handler(can_handle_func=is_intent_name("GetEnergyConsumptionIntent"))
def get_energy_consumption_intent_handler(handler_input):
    # Extrair slots
    slots = handler_input.request_envelope.request.intent.slots
    time_frame = slots["timeFrame"].value if "timeFrame" in slots else "hoje"
    
    # Chamar API para obter dados
    consumption_data = get_consumption_from_api(time_frame)
    
    # Formatar resposta
    speech_text = f"O consumo de energia {time_frame} foi de {consumption_data} kWh"
    
    return handler_input.response_builder.speak(speech_text).response
```

## Próximos Passos

1. Configurar conexão com APIs externas
2. Implementar handlers para cada intent
3. Adicionar validação e tratamento de erros
4. Testar com dados reais

---

*Este documento explica como criar e configurar intents para Alexa Skills no projeto GoodWe.*
