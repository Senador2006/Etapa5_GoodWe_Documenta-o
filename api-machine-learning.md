# API Machine Learning - Documentação Técnica

A API Machine Learning é um serviço FastAPI desenvolvido em Python que fornece predições de quedas de energia baseadas em condições climáticas usando um modelo Random Forest treinado.

## 📋 Informações Gerais

- **Nome**: API de Predição de Quedas de Energia
- **Versão**: 1.0.0
- **Porta**: 8000
- **Base URL**: `http://localhost:8000`
- **Tecnologia**: Python + FastAPI
- **Documentação Interativa**: `http://localhost:8000/docs`
  - Essa documentação explica como cada endpoint do ML funciona
- **URL deploydada (Utilizar essa durante a construção da API principal)** : `http://power-outage-prediction-api.onrender.com`

## 🔧 Dependências Principais

```txt
fastapi==0.116.1
uvicorn==0.35.0
scikit-learn==1.7.1
pandas==2.3.1
numpy==2.3.2
pydantic==2.11.7
joblib==1.5.1
```

## 🧠 Modelo de Machine Learning

### Características do Modelo
- **Algoritmo**: Random Forest Classifier
- **Acurácia**: 76.7%
- **Dataset**: 50.000 registros sintéticos
- **Features**: 5 variáveis climáticas

### Importância das Features
1. **Vento (km/h)**: 57.1% - Mais importante
2. **Precipitação (mm/h)**: 30.0% - Muito importante  
3. **Temperatura (°C)**: 4.6% - Pouco importante
4. **Pressão (hPa)**: 4.3% - Pouco importante
5. **Umidade (%)**: 4.0% - Pouco importante

## 🚀 Endpoints Disponíveis

### 1. **GET /** - Página Principal
Retorna informações gerais da API e endpoints disponíveis.

**URL**: `GET /`

**Resposta de Sucesso**:
```json
{
  "message": "API de Predição de Quedas de Energia",
  "version": "1.0.0",
  "status": "ativo",
  "endpoints": {
    "/predict": "POST - Fazer predição de queda de energia",
    "/health": "GET - Verificar status da API",
    "/docs": "GET - Documentação interativa"
  }
}
```

### 2. **GET /health** - Health Check
Verifica se a API e o modelo estão funcionando.

**URL**: `GET /health`

**Resposta de Sucesso**:
```json
{
  "status": "healthy",
  "model_loaded": true,
  "features": [
    "temperatura_celsius",
    "umidade_pct", 
    "precipitacao_mm_h",
    "vento_kmh",
    "pressao_hpa"
  ]
}
```

**Erro - Modelo não carregado**:
```json
{
  "detail": "Modelo não carregado"
}
```
*Status Code: 503*

### 3. **POST /predict** - Predição Individual
Prediz se haverá queda de energia baseado nas condições climáticas.

**URL**: `POST /predict`

**Content-Type**: `application/json`

**Parâmetros de Entrada**:
```json
{
  "temperatura_celsius": 25.0,     // Range: -10 a 50°C
  "umidade_pct": 65.0,            // Range: 0 a 100%
  "precipitacao_mm_h": 10.0,      // Range: 0 a 100 mm/h
  "vento_kmh": 30.0,              // Range: 0 a 150 km/h
  "pressao_hpa": 1013.0           // Range: 950 a 1050 hPa
}
```

**Exemplo de Requisição**:
```bash
curl -X POST \
  http://localhost:8000/predict \
  -H 'Content-Type: application/json' \
  -d '{
    "temperatura_celsius": 25.0,
    "umidade_pct": 65.0,
    "precipitacao_mm_h": 10.0,
    "vento_kmh": 30.0,
    "pressao_hpa": 1013.0
  }'
```

**Resposta de Sucesso**:
```json
{
  "queda_energia": false,
  "probabilidade": 0.2156,
  "probabilidade_pct": "21.6%",
  "nivel_risco": "Baixo",
  "dados_entrada": {
    "temperatura_celsius": 25.0,
    "umidade_pct": 65.0,
    "precipitacao_mm_h": 10.0,
    "vento_kmh": 30.0,
    "pressao_hpa": 1013.0
  }
}
```

### 4. **POST /predict-batch** - Predição em Lote
Permite fazer múltiplas predições em uma única requisição.

**URL**: `POST /predict-batch`

**Content-Type**: `application/json`

**Limite**: Máximo 100 predições por requisição

**Parâmetros de Entrada**:
```json
[
  {
    "temperatura_celsius": 25.0,
    "umidade_pct": 65.0,
    "precipitacao_mm_h": 10.0,
    "vento_kmh": 30.0,
    "pressao_hpa": 1013.0
  },
  {
    "temperatura_celsius": 35.0,
    "umidade_pct": 80.0,
    "precipitacao_mm_h": 25.0,
    "vento_kmh": 75.0,
    "pressao_hpa": 995.0
  }
]
```

**Resposta de Sucesso**:
```json
{
  "predictions": [
    {
      "queda_energia": false,
      "probabilidade": 0.2156,
      "probabilidade_pct": "21.6%",
      "nivel_risco": "Baixo",
      "dados_entrada": {
        "temperatura_celsius": 25.0,
        "umidade_pct": 65.0,
        "precipitacao_mm_h": 10.0,
        "vento_kmh": 30.0,
        "pressao_hpa": 1013.0
      }
    },
    {
      "queda_energia": true,
      "probabilidade": 0.8932,
      "probabilidade_pct": "89.3%",
      "nivel_risco": "Crítico",
      "dados_entrada": {
        "temperatura_celsius": 35.0,
        "umidade_pct": 80.0,
        "precipitacao_mm_h": 25.0,
        "vento_kmh": 75.0,
        "pressao_hpa": 995.0
      }
    }
  ],
  "total": 2
}
```

### 5. **GET /model-info** - Informações do Modelo
Retorna informações detalhadas sobre o modelo de Machine Learning.

**URL**: `GET /model-info`

**Resposta de Sucesso**:
```json
{
  "model_type": "RandomForestClassifier",
  "features": [
    "temperatura_celsius",
    "umidade_pct",
    "precipitacao_mm_h", 
    "vento_kmh",
    "pressao_hpa"
  ],
  "feature_importance": {
    "vento_kmh": "57.1% - Mais importante",
    "precipitacao_mm_h": "30.0% - Muito importante",
    "temperatura_celsius": "4.6% - Pouco importante",
    "pressao_hpa": "4.3% - Pouco importante",
    "umidade_pct": "4.0% - Pouco importante"
  },
  "accuracy": "76.7%",
  "description": "Modelo treinado com 50.000 dados sintéticos de condições climáticas"
}
```

## 🎯 Níveis de Risco

### Classificação por Probabilidade
- **Baixo**: 0% - 25% de probabilidade de queda
- **Médio**: 25% - 50% de probabilidade de queda  
- **Alto**: 50% - 75% de probabilidade de queda
- **Crítico**: 75% - 100% de probabilidade de queda

### Interpretação dos Resultados
```python
if probability < 0.25:
    nivel_risco = "Baixo"      # Condições normais
elif probability < 0.50:
    nivel_risco = "Médio"      # Atenção recomendada
elif probability < 0.75:
    nivel_risco = "Alto"       # Preparação necessária
else:
    nivel_risco = "Crítico"    # Ação imediata
```

## 📊 Modelos de Dados

### Entrada (WeatherData)
```python
class WeatherData(BaseModel):
    temperatura_celsius: float = Field(..., ge=-10, le=50)
    umidade_pct: float = Field(..., ge=0, le=100)
    precipitacao_mm_h: float = Field(..., ge=0, le=100)
    vento_kmh: float = Field(..., ge=0, le=150)
    pressao_hpa: float = Field(..., ge=950, le=1050)
```

### Saída (PredictionResponse)
```python
class PredictionResponse(BaseModel):
    queda_energia: bool
    probabilidade: float
    probabilidade_pct: str
    nivel_risco: str
    dados_entrada: Dict[str, float]
```

## 🔒 Validações e Segurança

### Validação de Entrada
- **Temperatura**: -10°C a 50°C
- **Umidade**: 0% a 100%
- **Precipitação**: 0 a 100 mm/h
- **Vento**: 0 a 150 km/h
- **Pressão**: 950 a 1050 hPa

### Limitações de Segurança
- Máximo 100 predições por requisição batch
- Validação automática de tipos com Pydantic
- Tratamento robusto de erros
- Logs de todas as requisições

## 🚨 Tratamento de Erros

### Códigos de Status HTTP
- **200**: Sucesso
- **400**: Erro de validação
- **422**: Erro de processamento de entidade
- **500**: Erro interno do servidor
- **503**: Serviço indisponível (modelo não carregado)

### Estrutura de Erro Padrão
```json
{
  "detail": "Descrição do erro",
  "type": "validation_error",
  "loc": ["field_name"],
  "msg": "Mensagem específica"
}
```

### Exemplos de Erros Comuns

**Valor fora do range**:
```json
{
  "detail": [
    {
      "type": "greater_than_equal",
      "loc": ["temperatura_celsius"],
      "msg": "Input should be greater than or equal to -10",
      "input": -15
    }
  ]
}
```

**Campo obrigatório ausente**:
```json
{
  "detail": [
    {
      "type": "missing",
      "loc": ["vento_kmh"],
      "msg": "Field required"
    }
  ]
}
```

## 🔧 Configuração e Execução

### Arquivos Necessários
- `power_outage_model.pkl` - Modelo treinado
- `features.json` - Lista de features
- `api.py` - Código principal da API

### Comandos de Execução
```bash
# Instalar dependências
pip install -r requirements.txt

# Treinar modelo (se necessário)
python train_model.py

# Iniciar API
python start_api.py

# Ou diretamente
uvicorn api:app --host 0.0.0.0 --port 8000 --reload
```

### Variáveis de Ambiente
```bash
HOST=0.0.0.0
PORT=8000
RELOAD=true
LOG_LEVEL=info
```

## 🐳 Docker

### Dockerfile
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
EXPOSE 8000
CMD ["python", "start_api.py"]
```

### Docker Compose
```yaml
version: '3.8'
services:
  ml-api:
    build: .
    ports:
      - "8000:8000"
    environment:
      - HOST=0.0.0.0
      - PORT=8000
```

## 🧪 Testes

### Teste Manual via cURL
```bash
# Health check
curl http://localhost:8000/health

# Predição simples
curl -X POST \
  http://localhost:8000/predict \
  -H 'Content-Type: application/json' \
  -d '{
    "temperatura_celsius": 30,
    "umidade_pct": 70,
    "precipitacao_mm_h": 20,
    "vento_kmh": 50,
    "pressao_hpa": 1000
  }'

# Informações do modelo
curl http://localhost:8000/model-info
```

### Teste com Python
```python
import requests

# Dados de teste
data = {
    "temperatura_celsius": 25.0,
    "umidade_pct": 65.0,
    "precipitacao_mm_h": 10.0,
    "vento_kmh": 30.0,
    "pressao_hpa": 1013.0
}

# Fazer predição
response = requests.post(
    "http://localhost:8000/predict", 
    json=data
)

print(response.json())
```

## 📈 Monitoramento

### Métricas Disponíveis
- Status do modelo (carregado/não carregado)
- Tempo de resposta das predições
- Número de predições realizadas
- Distribuição de níveis de risco

### Logs Estruturados
```python
# Exemplo de log de predição
{
  "timestamp": "2025-01-20T10:30:00Z",
  "level": "INFO",
  "event": "prediction_made",
  "data": {
    "probability": 0.2156,
    "risk_level": "Baixo",
    "processing_time_ms": 45
  }
}
```

## 🔮 Casos de Uso

### 1. Sistema de Alerta
```python
# Monitoramento contínuo
if prediction["probabilidade"] > 0.75:
    send_critical_alert()
elif prediction["probabilidade"] > 0.50:
    send_warning_alert()
```

### 2. Dashboard em Tempo Real
```javascript
// Frontend integration
const weatherData = await getWeatherData();
const prediction = await predictPowerOutage(weatherData);
updateDashboard(prediction);
```

### 3. Planejamento de Manutenção
```python
# Predições para próximos dias
weather_forecast = get_weather_forecast(7)  # 7 dias
predictions = []

for day in weather_forecast:
    pred = predict_power_outage(day)
    predictions.append(pred)

# Agendar manutenção preventiva
high_risk_days = [p for p in predictions if p["nivel_risco"] in ["Alto", "Crítico"]]
```

---

**Próxima seção**: [Guia de Deployment](./deployment.md)
