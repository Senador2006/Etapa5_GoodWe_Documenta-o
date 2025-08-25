# Conectando Alexa Skills com APIs Externas

## Visão Geral

A integração entre Alexa Skills e APIs externas permite que sua skill acesse dados em tempo real, execute ações em sistemas externos e forneça informações dinâmicas aos usuários.

## Arquitetura de Integração

```
Usuário → Alexa → Skill Backend → API Externa → Banco de Dados/Sistemas
                      ↓
              Processamento/Lógica
                      ↓
              Resposta para Alexa
```

## Configuração do Backend

### 1. AWS Lambda Function (Python)

#### Estrutura Básica
```python
import json
import requests
import logging
from ask_sdk_core.skill_builder import SkillBuilder
from ask_sdk_core.dispatch_components import AbstractRequestHandler
from ask_sdk_core.utils import is_intent_name
from ask_sdk_core.handler_input import HandlerInput
from ask_sdk_model import Response

# Configuração de logging
logger = logging.getLogger(__name__)
logger.setLevel(logging.INFO)

# URL base da sua API
API_BASE_URL = "https://api.goodwe.com/v1"
API_KEY = "sua_api_key_aqui"

sb = SkillBuilder()
```

#### Função para Chamadas de API
```python
def call_api(endpoint, params=None):
    """
    Função genérica para chamar APIs externas
    """
    try:
        headers = {
            'Authorization': f'Bearer {API_KEY}',
            'Content-Type': 'application/json'
        }
        
        url = f"{API_BASE_URL}/{endpoint}"
        response = requests.get(url, headers=headers, params=params, timeout=10)
        
        if response.status_code == 200:
            return response.json()
        else:
            logger.error(f"API Error: {response.status_code} - {response.text}")
            return None
            
    except requests.exceptions.RequestException as e:
        logger.error(f"Request failed: {str(e)}")
        return None
    except Exception as e:
        logger.error(f"Unexpected error: {str(e)}")
        return None
```

### 2. Handlers para Intents com API

#### Intent: Obter Consumo de Energia
```python
@sb.request_handler(can_handle_func=is_intent_name("GetEnergyConsumptionIntent"))
def get_energy_consumption_handler(handler_input):
    """
    Handler para obter dados de consumo de energia
    """
    try:
        # Extrair slots
        slots = handler_input.request_envelope.request.intent.slots
        time_frame = extract_time_frame(slots.get("timeFrame"))
        
        # Chamar API
        params = {
            'period': time_frame,
            'metric': 'consumption'
        }
        
        api_response = call_api('energy/consumption', params)
        
        if api_response:
            consumption = api_response.get('data', {}).get('total_consumption', 0)
            unit = api_response.get('data', {}).get('unit', 'kWh')
            
            speech_text = f"O consumo de energia {format_time_frame(time_frame)} foi de {consumption} {unit}."
            
            # Adicionar informações extras se disponível
            if 'comparison' in api_response.get('data', {}):
                comparison = api_response['data']['comparison']
                speech_text += f" Isso representa {comparison['percentage']}% comparado ao período anterior."
                
        else:
            speech_text = "Desculpe, não consegui obter os dados de consumo no momento. Tente novamente mais tarde."
            
    except Exception as e:
        logger.error(f"Error in get_energy_consumption_handler: {str(e)}")
        speech_text = "Ocorreu um erro ao buscar os dados de consumo."
    
    return handler_input.response_builder.speak(speech_text).response
```

#### Intent: Status de Dispositivos
```python
@sb.request_handler(can_handle_func=is_intent_name("GetDeviceStatusIntent"))
def get_device_status_handler(handler_input):
    """
    Handler para obter status de dispositivos
    """
    try:
        # Extrair slots
        slots = handler_input.request_envelope.request.intent.slots
        device_type = slots.get("deviceType", {}).get("value", "sistema")
        
        # Mapear tipos de dispositivos para endpoints da API
        device_endpoints = {
            "inversor": "devices/inverter",
            "painel solar": "devices/solar-panels", 
            "bateria": "devices/battery",
            "sistema": "devices/system-status"
        }
        
        endpoint = device_endpoints.get(device_type, "devices/system-status")
        api_response = call_api(endpoint)
        
        if api_response and 'data' in api_response:
            device_data = api_response['data']
            status = device_data.get('status', 'unknown')
            
            # Formatar resposta baseada no tipo de dispositivo
            if device_type == "inversor":
                power = device_data.get('current_power', 0)
                speech_text = f"O inversor está {status}. Potência atual: {power} watts."
                
            elif device_type == "painel solar":
                generation = device_data.get('current_generation', 0)
                efficiency = device_data.get('efficiency', 0)
                speech_text = f"Os painéis solares estão {status}. Geração atual: {generation} watts, eficiência: {efficiency}%."
                
            elif device_type == "bateria":
                charge_level = device_data.get('charge_level', 0)
                speech_text = f"A bateria está {status} com {charge_level}% de carga."
                
            else:  # sistema completo
                speech_text = f"O sistema está {status}."
                if 'summary' in device_data:
                    summary = device_data['summary']
                    speech_text += f" Geração atual: {summary.get('generation', 0)} watts, consumo: {summary.get('consumption', 0)} watts."
        else:
            speech_text = f"Não consegui obter o status do {device_type} no momento."
            
    except Exception as e:
        logger.error(f"Error in get_device_status_handler: {str(e)}")
        speech_text = "Ocorreu um erro ao verificar o status do dispositivo."
    
    return handler_input.response_builder.speak(speech_text).response
```

## Funções Auxiliares

### Processamento de Dados
```python
def extract_time_frame(time_slot):
    """
    Extrai e normaliza período de tempo do slot
    """
    if not time_slot or not time_slot.value:
        return "today"
    
    time_value = time_slot.value.lower()
    
    # Mapear valores em português para a API
    time_mappings = {
        "hoje": "today",
        "ontem": "yesterday", 
        "esta semana": "this_week",
        "semana passada": "last_week",
        "este mês": "this_month",
        "mês passado": "last_month"
    }
    
    return time_mappings.get(time_value, "today")

def format_time_frame(time_frame):
    """
    Formata período para resposta em português
    """
    format_mappings = {
        "today": "hoje",
        "yesterday": "ontem",
        "this_week": "esta semana", 
        "last_week": "na semana passada",
        "this_month": "este mês",
        "last_month": "no mês passado"
    }
    
    return format_mappings.get(time_frame, time_frame)
```

### Validação e Tratamento de Erros
```python
def validate_api_response(response):
    """
    Valida resposta da API
    """
    if not response:
        return False, "API não respondeu"
    
    if 'error' in response:
        return False, response['error'].get('message', 'Erro desconhecido')
    
    if 'data' not in response:
        return False, "Dados não encontrados na resposta"
    
    return True, None

def handle_api_error(error_message):
    """
    Gera resposta amigável para erros de API
    """
    error_responses = {
        "timeout": "O serviço está demorando para responder. Tente novamente em alguns minutos.",
        "unauthorized": "Não tenho permissão para acessar esses dados no momento.",
        "not_found": "Os dados solicitados não foram encontrados.",
        "rate_limit": "Muitas consultas foram feitas. Aguarde um momento antes de tentar novamente."
    }
    
    for error_type, response in error_responses.items():
        if error_type in error_message.lower():
            return response
    
    return "Ocorreu um problema ao acessar os dados. Tente novamente mais tarde."
```

## Configuração de Autenticação

### 1. Variáveis de Ambiente no Lambda
```python
import os

API_KEY = os.environ.get('GOODWE_API_KEY')
API_BASE_URL = os.environ.get('API_BASE_URL', 'https://api.goodwe.com/v1')
```

### 2. Configuração no AWS Lambda
```bash
# Através do AWS CLI
aws lambda update-function-configuration \
  --function-name your-alexa-skill-function \
  --environment Variables='{
    "GOODWE_API_KEY": "sua_chave_aqui",
    "API_BASE_URL": "https://api.goodwe.com/v1"
  }'
```

## Otimização e Performance

### 1. Cache de Respostas
```python
import time
from functools import lru_cache

# Cache simples para dados que mudam lentamente
@lru_cache(maxsize=100)
def get_device_info_cached(device_id, cache_time=300):
    """
    Cache de informações de dispositivos por 5 minutos
    """
    current_time = int(time.time())
    cache_key = f"{device_id}_{current_time // cache_time}"
    
    return call_api(f'devices/{device_id}')
```

### 2. Timeout e Retry
```python
import time
from functools import wraps

def retry_api_call(max_retries=3, delay=1):
    """
    Decorator para retry automático em chamadas de API
    """
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_retries):
                try:
                    result = func(*args, **kwargs)
                    if result:
                        return result
                except Exception as e:
                    if attempt == max_retries - 1:
                        raise e
                    time.sleep(delay * (attempt + 1))
            return None
        return wrapper
    return decorator

@retry_api_call(max_retries=2, delay=0.5)
def robust_api_call(endpoint, params=None):
    return call_api(endpoint, params)
```

## Exemplo de API Mock para Testes

### Servidor de Desenvolvimento
```python
from flask import Flask, jsonify, request

app = Flask(__name__)

@app.route('/v1/energy/consumption')
def get_consumption():
    period = request.args.get('period', 'today')
    
    mock_data = {
        'today': {'total_consumption': 25.5, 'unit': 'kWh'},
        'yesterday': {'total_consumption': 28.2, 'unit': 'kWh'},
        'this_week': {'total_consumption': 180.3, 'unit': 'kWh'}
    }
    
    return jsonify({
        'data': mock_data.get(period, mock_data['today']),
        'timestamp': '2024-01-15T10:30:00Z'
    })

@app.route('/v1/devices/system-status')
def get_system_status():
    return jsonify({
        'data': {
            'status': 'operacional',
            'summary': {
                'generation': 1200,
                'consumption': 800,
                'battery_level': 85
            }
        }
    })

if __name__ == '__main__':
    app.run(debug=True, port=5000)
```

## Próximos Passos

1. Implementar exemplos práticos completos
2. Configurar testes automatizados
3. Adicionar logs e monitoramento
4. Otimizar para produção

---

*Este documento explica como conectar Alexa Skills com APIs externas no projeto GoodWe.*
