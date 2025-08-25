# Exemplos Práticos de Alexa Skills

## Projeto Completo: Assistente de Energia Solar

Este arquivo contém exemplos práticos completos para implementar uma Alexa Skill para monitoramento de energia solar.

## 1. Estrutura Completa do Projeto

```
alexa-energy-skill/
├── lambda/
│   ├── requirements.txt
│   ├── lambda_function.py
│   └── utils/
│       ├── api_client.py
│       ├── response_builder.py
│       └── data_formatter.py
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

### lambda/utils/api_client.py
```python
import os
import requests
import logging
from typing import Dict, Optional, Any

logger = logging.getLogger(__name__)

class GoodWeAPIClient:
    def __init__(self):
        self.base_url = os.environ.get('API_BASE_URL', 'https://api.goodwe.com/v1')
        self.api_key = os.environ.get('GOODWE_API_KEY')
        self.timeout = 10
        
    def _make_request(self, endpoint: str, params: Optional[Dict] = None) -> Optional[Dict]:
        """Faz requisição para a API"""
        try:
            headers = {
                'Authorization': f'Bearer {self.api_key}',
                'Content-Type': 'application/json'
            }
            
            url = f"{self.base_url}/{endpoint}"
            response = requests.get(url, headers=headers, params=params, timeout=self.timeout)
            
            if response.status_code == 200:
                return response.json()
            elif response.status_code == 404:
                logger.warning(f"Resource not found: {endpoint}")
                return None
            else:
                logger.error(f"API Error: {response.status_code} - {response.text}")
                return None
                
        except requests.exceptions.Timeout:
            logger.error("API request timeout")
            return None
        except requests.exceptions.RequestException as e:
            logger.error(f"Request failed: {str(e)}")
            return None
        except Exception as e:
            logger.error(f"Unexpected error: {str(e)}")
            return None
    
    def get_energy_consumption(self, time_frame: str = 'today') -> Optional[Dict]:
        """Obter dados de consumo de energia"""
        params = {'period': time_frame, 'metric': 'consumption'}
        return self._make_request('energy/consumption', params)
    
    def get_energy_production(self, time_frame: str = 'today') -> Optional[Dict]:
        """Obter dados de produção de energia"""
        params = {'period': time_frame, 'metric': 'production'}
        return self._make_request('energy/production', params)
    
    def get_device_status(self, device_type: str = 'system') -> Optional[Dict]:
        """Obter status de dispositivos"""
        device_endpoints = {
            'inversor': 'devices/inverter',
            'painel solar': 'devices/solar-panels',
            'bateria': 'devices/battery',
            'sistema': 'devices/system-status'
        }
        
        endpoint = device_endpoints.get(device_type, 'devices/system-status')
        return self._make_request(endpoint)
    
    def get_savings(self, time_frame: str = 'today') -> Optional[Dict]:
        """Obter dados de economia"""
        params = {'period': time_frame}
        return self._make_request('financial/savings', params)
    
    def get_weather_impact(self) -> Optional[Dict]:
        """Obter impacto climático na geração"""
        return self._make_request('weather/impact')
```

### lambda/utils/data_formatter.py
```python
from typing import Dict, Any

class DataFormatter:
    
    @staticmethod
    def format_time_frame(time_frame: str) -> str:
        """Formata período para resposta em português"""
        mappings = {
            'today': 'hoje',
            'yesterday': 'ontem',
            'this_week': 'esta semana',
            'last_week': 'na semana passada',
            'this_month': 'este mês',
            'last_month': 'no mês passado'
        }
        return mappings.get(time_frame, time_frame)
    
    @staticmethod
    def normalize_time_frame(time_value: str) -> str:
        """Normaliza entrada de tempo para formato da API"""
        if not time_value:
            return 'today'
            
        mappings = {
            'hoje': 'today',
            'ontem': 'yesterday',
            'esta semana': 'this_week',
            'semana passada': 'last_week',
            'este mês': 'this_month',
            'mês passado': 'last_month'
        }
        return mappings.get(time_value.lower(), 'today')
    
    @staticmethod
    def format_energy_value(value: float, unit: str = 'kWh') -> str:
        """Formata valores de energia"""
        if value >= 1000:
            return f"{value/1000:.1f} MW{unit[2:]}"
        elif value >= 1:
            return f"{value:.1f} {unit}"
        else:
            return f"{value*1000:.0f} W{unit[2:]}"
    
    @staticmethod
    def format_currency(value: float, currency: str = 'R$') -> str:
        """Formata valores monetários"""
        return f"{currency} {value:.2f}".replace('.', ',')
    
    @staticmethod
    def format_percentage(value: float) -> str:
        """Formata percentuais"""
        return f"{value:.1f}%"
```

### lambda/utils/response_builder.py
```python
from ask_sdk_core.handler_input import HandlerInput
from ask_sdk_model import Response
from ask_sdk_model.ui import SimpleCard

class ResponseBuilder:
    
    @staticmethod
    def build_response(handler_input: HandlerInput, speech_text: str, 
                      card_title: str = None, card_content: str = None,
                      should_end_session: bool = True) -> Response:
        """Constrói resposta padrão"""
        response_builder = handler_input.response_builder.speak(speech_text)
        
        if card_title and card_content:
            response_builder.set_card(SimpleCard(card_title, card_content))
        
        if should_end_session:
            response_builder.set_should_end_session(True)
        else:
            response_builder.ask(speech_text)
            
        return response_builder.response
    
    @staticmethod
    def build_error_response(handler_input: HandlerInput, error_type: str = 'general') -> Response:
        """Constrói resposta de erro"""
        error_messages = {
            'api_error': 'Desculpe, não consegui acessar os dados no momento. Tente novamente mais tarde.',
            'no_data': 'Não encontrei dados para o período solicitado.',
            'invalid_device': 'Não reconheci o dispositivo mencionado. Tente inversor, painel solar, bateria ou sistema.',
            'general': 'Ocorreu um erro inesperado. Tente novamente mais tarde.'
        }
        
        speech_text = error_messages.get(error_type, error_messages['general'])
        return ResponseBuilder.build_response(handler_input, speech_text)
    
    @staticmethod
    def build_help_response(handler_input: HandlerInput) -> Response:
        """Constrói resposta de ajuda"""
        speech_text = """
        Eu posso te ajudar a monitorar seu sistema de energia solar. Você pode perguntar:
        
        "Qual o consumo de energia hoje?"
        "Como está o inversor?"
        "Quanta energia foi gerada esta semana?"
        "Quanto economizei este mês?"
        
        O que você gostaria de saber?
        """
        
        return ResponseBuilder.build_response(
            handler_input, 
            speech_text, 
            should_end_session=False
        )
```

### lambda/lambda_function.py
```python
import logging
from ask_sdk_core.skill_builder import SkillBuilder
from ask_sdk_core.dispatch_components import AbstractRequestHandler, AbstractExceptionHandler
from ask_sdk_core.utils import is_request_type, is_intent_name
from ask_sdk_core.handler_input import HandlerInput
from ask_sdk_model import Response

from utils.api_client import GoodWeAPIClient
from utils.data_formatter import DataFormatter
from utils.response_builder import ResponseBuilder

# Configuração de logging
logger = logging.getLogger(__name__)
logger.setLevel(logging.INFO)

# Inicializar cliente da API
api_client = GoodWeAPIClient()

sb = SkillBuilder()

# Handler para Launch Request
@sb.request_handler(can_handle_func=is_request_type("LaunchRequest"))
def launch_request_handler(handler_input):
    speech_text = """
    Olá! Bem-vindo ao seu assistente de energia solar. 
    Eu posso te ajudar a monitorar consumo, produção, status dos dispositivos e economias.
    O que você gostaria de saber?
    """
    
    return ResponseBuilder.build_response(
        handler_input, 
        speech_text, 
        should_end_session=False
    )

# Handler para Consumo de Energia
@sb.request_handler(can_handle_func=is_intent_name("GetEnergyConsumptionIntent"))
def get_energy_consumption_handler(handler_input):
    try:
        # Extrair slot
        slots = handler_input.request_envelope.request.intent.slots
        time_frame_slot = slots.get("timeFrame", {})
        time_frame_raw = time_frame_slot.value if time_frame_slot else None
        
        # Normalizar período
        time_frame = DataFormatter.normalize_time_frame(time_frame_raw)
        
        # Buscar dados na API
        consumption_data = api_client.get_energy_consumption(time_frame)
        
        if consumption_data and 'data' in consumption_data:
            data = consumption_data['data']
            consumption = data.get('total_consumption', 0)
            unit = data.get('unit', 'kWh')
            
            # Formatar resposta
            formatted_period = DataFormatter.format_time_frame(time_frame)
            formatted_consumption = DataFormatter.format_energy_value(consumption, unit)
            
            speech_text = f"O consumo de energia {formatted_period} foi de {formatted_consumption}."
            
            # Adicionar comparação se disponível
            if 'comparison' in data:
                comparison = data['comparison']
                percentage = DataFormatter.format_percentage(comparison['percentage'])
                trend = "maior" if comparison['percentage'] > 0 else "menor"
                speech_text += f" Isso é {percentage} {trend} que o período anterior."
            
            card_title = "Consumo de Energia"
            card_content = f"Período: {formatted_period}\\nConsumo: {formatted_consumption}"
            
            return ResponseBuilder.build_response(
                handler_input, speech_text, card_title, card_content
            )
        else:
            return ResponseBuilder.build_error_response(handler_input, 'no_data')
            
    except Exception as e:
        logger.error(f"Error in consumption handler: {str(e)}")
        return ResponseBuilder.build_error_response(handler_input, 'api_error')

# Handler para Produção de Energia
@sb.request_handler(can_handle_func=is_intent_name("GetEnergyProductionIntent"))
def get_energy_production_handler(handler_input):
    try:
        # Extrair slot
        slots = handler_input.request_envelope.request.intent.slots
        time_frame_slot = slots.get("timeFrame", {})
        time_frame_raw = time_frame_slot.value if time_frame_slot else None
        
        # Normalizar período
        time_frame = DataFormatter.normalize_time_frame(time_frame_raw)
        
        # Buscar dados na API
        production_data = api_client.get_energy_production(time_frame)
        
        if production_data and 'data' in production_data:
            data = production_data['data']
            production = data.get('total_production', 0)
            unit = data.get('unit', 'kWh')
            
            # Formatar resposta
            formatted_period = DataFormatter.format_time_frame(time_frame)
            formatted_production = DataFormatter.format_energy_value(production, unit)
            
            speech_text = f"A produção de energia {formatted_period} foi de {formatted_production}."
            
            # Adicionar eficiência se disponível
            if 'efficiency' in data:
                efficiency = DataFormatter.format_percentage(data['efficiency'])
                speech_text += f" A eficiência dos painéis foi de {efficiency}."
            
            return ResponseBuilder.build_response(handler_input, speech_text)
        else:
            return ResponseBuilder.build_error_response(handler_input, 'no_data')
            
    except Exception as e:
        logger.error(f"Error in production handler: {str(e)}")
        return ResponseBuilder.build_error_response(handler_input, 'api_error')

# Handler para Status de Dispositivos
@sb.request_handler(can_handle_func=is_intent_name("GetDeviceStatusIntent"))
def get_device_status_handler(handler_input):
    try:
        # Extrair slot
        slots = handler_input.request_envelope.request.intent.slots
        device_slot = slots.get("deviceType", {})
        device_type = device_slot.value if device_slot else "sistema"
        
        # Buscar status na API
        status_data = api_client.get_device_status(device_type)
        
        if status_data and 'data' in status_data:
            data = status_data['data']
            status = data.get('status', 'desconhecido')
            
            # Formatar resposta baseada no tipo de dispositivo
            if device_type == "inversor":
                power = data.get('current_power', 0)
                efficiency = data.get('efficiency', 0)
                speech_text = f"O inversor está {status}. Potência atual: {power} watts, eficiência: {DataFormatter.format_percentage(efficiency)}."
                
            elif device_type == "painel solar":
                generation = data.get('current_generation', 0)
                weather_factor = data.get('weather_factor', 0)
                speech_text = f"Os painéis solares estão {status}. Geração atual: {generation} watts. Fator climático: {DataFormatter.format_percentage(weather_factor)}."
                
            elif device_type == "bateria":
                charge_level = data.get('charge_level', 0)
                charge_status = data.get('charge_status', 'parado')
                speech_text = f"A bateria está {status} com {DataFormatter.format_percentage(charge_level)} de carga. Status: {charge_status}."
                
            else:  # sistema completo
                speech_text = f"O sistema está {status}."
                if 'summary' in data:
                    summary = data['summary']
                    generation = summary.get('generation', 0)
                    consumption = summary.get('consumption', 0)
                    speech_text += f" Geração atual: {generation} watts, consumo: {consumption} watts."
            
            return ResponseBuilder.build_response(handler_input, speech_text)
        else:
            return ResponseBuilder.build_error_response(handler_input, 'no_data')
            
    except Exception as e:
        logger.error(f"Error in device status handler: {str(e)}")
        return ResponseBuilder.build_error_response(handler_input, 'api_error')

# Handler para Economias
@sb.request_handler(can_handle_func=is_intent_name("GetSavingsIntent"))
def get_savings_handler(handler_input):
    try:
        # Extrair slot
        slots = handler_input.request_envelope.request.intent.slots
        time_frame_slot = slots.get("timeFrame", {})
        time_frame_raw = time_frame_slot.value if time_frame_slot else None
        
        # Normalizar período
        time_frame = DataFormatter.normalize_time_frame(time_frame_raw)
        
        # Buscar dados na API
        savings_data = api_client.get_savings(time_frame)
        
        if savings_data and 'data' in savings_data:
            data = savings_data['data']
            savings_amount = data.get('total_savings', 0)
            currency = data.get('currency', 'R$')
            
            # Formatar resposta
            formatted_period = DataFormatter.format_time_frame(time_frame)
            formatted_savings = DataFormatter.format_currency(savings_amount, currency)
            
            speech_text = f"Você economizou {formatted_savings} {formatted_period}."
            
            # Adicionar detalhes se disponível
            if 'energy_offset' in data:
                energy_offset = DataFormatter.format_percentage(data['energy_offset'])
                speech_text += f" Isso representa {energy_offset} da sua conta de energia."
            
            return ResponseBuilder.build_response(handler_input, speech_text)
        else:
            return ResponseBuilder.build_error_response(handler_input, 'no_data')
            
    except Exception as e:
        logger.error(f"Error in savings handler: {str(e)}")
        return ResponseBuilder.build_error_response(handler_input, 'api_error')

# Handler para Impacto Climático
@sb.request_handler(can_handle_func=is_intent_name("GetWeatherImpactIntent"))
def get_weather_impact_handler(handler_input):
    try:
        # Buscar dados na API
        weather_data = api_client.get_weather_impact()
        
        if weather_data and 'data' in weather_data:
            data = weather_data['data']
            weather_condition = data.get('condition', 'desconhecida')
            impact_percentage = data.get('impact_percentage', 0)
            
            formatted_impact = DataFormatter.format_percentage(impact_percentage)
            
            speech_text = f"As condições climáticas estão {weather_condition}. "
            speech_text += f"O impacto na geração de energia é de {formatted_impact}."
            
            # Adicionar previsão se disponível
            if 'forecast' in data:
                forecast = data['forecast']
                speech_text += f" Para as próximas horas, espera-se {forecast}."
            
            return ResponseBuilder.build_response(handler_input, speech_text)
        else:
            return ResponseBuilder.build_error_response(handler_input, 'no_data')
            
    except Exception as e:
        logger.error(f"Error in weather impact handler: {str(e)}")
        return ResponseBuilder.build_error_response(handler_input, 'api_error')

# Handler para Help Intent
@sb.request_handler(can_handle_func=is_intent_name("AMAZON.HelpIntent"))
def help_intent_handler(handler_input):
    return ResponseBuilder.build_help_response(handler_input)

# Handlers para Cancel e Stop
@sb.request_handler(can_handle_func=is_intent_name("AMAZON.CancelIntent"))
@sb.request_handler(can_handle_func=is_intent_name("AMAZON.StopIntent"))
def cancel_and_stop_intent_handler(handler_input):
    speech_text = "Até logo! Volte sempre que quiser monitorar sua energia solar."
    return ResponseBuilder.build_response(handler_input, speech_text)

# Handler para Session Ended
@sb.request_handler(can_handle_func=is_request_type("SessionEndedRequest"))
def session_ended_request_handler(handler_input):
    return handler_input.response_builder.response

# Exception Handler
@sb.exception_handler(can_handle_func=lambda i, e: True)
def all_exception_handler(handler_input, exception):
    logger.error(f"Unexpected error: {str(exception)}")
    speech_text = "Desculpe, ocorreu um erro inesperado. Tente novamente mais tarde."
    return ResponseBuilder.build_response(handler_input, speech_text)

# Lambda handler
def lambda_handler(event, context):
    return sb.lambda_handler()(event, context)
```

## 4. Comandos de Teste

### Frases para Testar
```
# Ativação
"Alexa, abra assistente energia"

# Consumo
"qual o consumo de energia hoje"
"quanto gastei de energia esta semana"
"consumo de ontem"

# Produção
"quanta energia foi gerada hoje"
"produção desta semana"
"quanto os painéis geraram ontem"

# Status
"como está o inversor"
"qual o status da bateria"
"verifique o sistema"

# Economias
"quanto economizei este mês"
"qual a economia de hoje"

# Clima
"como está o clima afetando a geração"

# Ajuda
"ajuda"
"o que você pode fazer"

# Parar
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
          "name": "Assistente de Energia Solar",
          "summary": "Monitore seu sistema de energia solar por voz",
          "description": "Acompanhe consumo, produção, status dos dispositivos e economias do seu sistema de energia solar através de comandos de voz simples.",
          "keywords": ["energia solar", "monitoramento", "sustentabilidade"],
          "examplePhrases": [
            "Alexa, abra assistente energia",
            "qual o consumo de energia hoje",
            "como está o inversor"
          ]
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
        }
      }
    }
  }
}
```

### Deploy com ASK CLI
```bash
# Inicializar projeto
ask new

# Deploy
ask deploy

# Testar
ask simulate -l pt-BR -t "abra assistente energia"
```

---

*Este documento fornece exemplos práticos completos para implementar uma Alexa Skill de monitoramento de energia solar no projeto GoodWe.*
