# Exemplos de Uso - APIs GoodWe

Esta seção fornece exemplos práticos e detalhados de como usar as APIs GoodWe em diferentes cenários e linguagens de programação.

## 📋 Índice de Exemplos

1. [API Excel - Exemplos Básicos](#api-excel---exemplos-básicos)
2. [API Machine Learning - Exemplos Básicos](#api-machine-learning---exemplos-básicos)
3. [Integração entre APIs](#integração-entre-apis)
4. [Exemplos por Linguagem](#exemplos-por-linguagem)
5. [Casos de Uso Avançados](#casos-de-uso-avançados)
6. [Scripts de Automação](#scripts-de-automação)

## 🔧 API Excel - Exemplos Básicos

### 1. Conversão de Arquivo Específico

#### cURL
```bash
# Converter arquivo Potência Vegetal existente
curl -X GET http://localhost:3000/excel/convert \
  -H "Accept: application/json" | jq '.'
```

#### JavaScript (Fetch)
```javascript
// Converter arquivo específico
async function convertExistingFile() {
    try {
        const response = await fetch('http://localhost:3000/excel/convert');
        const data = await response.json();
        
        console.log('Arquivo convertido:', data.originalFile);
        console.log('Planilhas encontradas:', data.sheets);
        console.log('Total de planilhas:', data.totalSheets);
        
        return data;
    } catch (error) {
        console.error('Erro na conversão:', error);
    }
}

// Usar a função
convertExistingFile();
```

#### Python
```python
import requests
import json

def convert_existing_file():
    """Converte arquivo Excel específico"""
    url = 'http://localhost:3000/excel/convert'
    
    try:
        response = requests.get(url)
        response.raise_for_status()
        
        data = response.json()
        print(f"Arquivo convertido: {data['originalFile']}")
        print(f"Planilhas: {', '.join(data['sheets'])}")
        
        return data
    
    except requests.exceptions.RequestException as e:
        print(f"Erro na requisição: {e}")
        return None

# Executar conversão
result = convert_existing_file()
```

### 2. Upload e Conversão de Arquivo

#### cURL
```bash
# Upload de arquivo Excel
curl -X POST http://localhost:3000/excel/upload \
  -H "Content-Type: multipart/form-data" \
  -F "excel=@/path/to/your/file.xlsx" | jq '.'
```

#### JavaScript (FormData)
```javascript
// Upload de arquivo com JavaScript
async function uploadAndConvert(file) {
    const formData = new FormData();
    formData.append('excel', file);
    
    try {
        const response = await fetch('http://localhost:3000/excel/upload', {
            method: 'POST',
            body: formData
        });
        
        if (!response.ok) {
            throw new Error(`HTTP error! status: ${response.status}`);
        }
        
        const data = await response.json();
        console.log('Upload realizado com sucesso:', data);
        
        return data;
    } catch (error) {
        console.error('Erro no upload:', error);
    }
}

// Exemplo com input file
document.getElementById('fileInput').addEventListener('change', (event) => {
    const file = event.target.files[0];
    if (file) {
        uploadAndConvert(file);
    }
});
```

#### Python (requests)
```python
import requests

def upload_excel_file(file_path):
    """Faz upload e converte arquivo Excel"""
    url = 'http://localhost:3000/excel/upload'
    
    try:
        with open(file_path, 'rb') as file:
            files = {'excel': file}
            response = requests.post(url, files=files)
            response.raise_for_status()
            
            data = response.json()
            print(f"Upload concluído: {data['originalFile']}")
            print(f"Tamanho: {data['fileSize']} bytes")
            print(f"Planilhas: {data['totalSheets']}")
            
            return data
    
    except FileNotFoundError:
        print(f"Arquivo não encontrado: {file_path}")
    except requests.exceptions.RequestException as e:
        print(f"Erro na requisição: {e}")
    
    return None

# Fazer upload
result = upload_excel_file('/path/to/file.xlsx')
```

### 3. Listar Arquivos Disponíveis

#### JavaScript
```javascript
async function listExcelFiles() {
    try {
        const response = await fetch('http://localhost:3000/excel/files');
        const data = await response.json();
        
        console.log(`Encontrados ${data.count} arquivos:`);
        data.files.forEach(file => {
            console.log(`- ${file.name} (${file.size} bytes)`);
            console.log(`  Criado: ${new Date(file.created).toLocaleDateString()}`);
            console.log(`  Modificado: ${new Date(file.modified).toLocaleDateString()}`);
        });
        
        return data.files;
    } catch (error) {
        console.error('Erro ao listar arquivos:', error);
    }
}

listExcelFiles();
```

## 🧠 API Machine Learning - Exemplos Básicos

### 1. Predição Individual

#### cURL
```bash
# Fazer predição de queda de energia
curl -X POST http://localhost:8000/predict \
  -H "Content-Type: application/json" \
  -d '{
    "temperatura_celsius": 30.0,
    "umidade_pct": 75.0,
    "precipitacao_mm_h": 20.0,
    "vento_kmh": 60.0,
    "pressao_hpa": 1000.0
  }' | jq '.'
```

#### JavaScript
```javascript
async function predictPowerOutage(weatherData) {
    const url = 'http://localhost:8000/predict';
    
    try {
        const response = await fetch(url, {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
            },
            body: JSON.stringify(weatherData)
        });
        
        if (!response.ok) {
            throw new Error(`HTTP error! status: ${response.status}`);
        }
        
        const prediction = await response.json();
        
        console.log(`Predição: ${prediction.queda_energia ? 'HAVERÁ' : 'NÃO HAVERÁ'} queda de energia`);
        console.log(`Probabilidade: ${prediction.probabilidade_pct}`);
        console.log(`Nível de risco: ${prediction.nivel_risco}`);
        
        return prediction;
    } catch (error) {
        console.error('Erro na predição:', error);
    }
}

// Dados meteorológicos de exemplo
const weatherData = {
    temperatura_celsius: 28.5,
    umidade_pct: 70.0,
    precipitacao_mm_h: 15.0,
    vento_kmh: 45.0,
    pressao_hpa: 1008.0
};

predictPowerOutage(weatherData);
```

#### Python
```python
import requests
import json

def predict_power_outage(weather_data):
    """Faz predição de queda de energia"""
    url = 'http://localhost:8000/predict'
    
    try:
        response = requests.post(
            url,
            json=weather_data,
            headers={'Content-Type': 'application/json'}
        )
        response.raise_for_status()
        
        prediction = response.json()
        
        print(f"Predição: {'HAVERÁ' if prediction['queda_energia'] else 'NÃO HAVERÁ'} queda de energia")
        print(f"Probabilidade: {prediction['probabilidade_pct']}")
        print(f"Nível de risco: {prediction['nivel_risco']}")
        
        return prediction
    
    except requests.exceptions.RequestException as e:
        print(f"Erro na predição: {e}")
        return None

# Dados meteorológicos
weather_conditions = {
    "temperatura_celsius": 32.0,
    "umidade_pct": 80.0,
    "precipitacao_mm_h": 25.0,
    "vento_kmh": 70.0,
    "pressao_hpa": 995.0
}

# Fazer predição
result = predict_power_outage(weather_conditions)
```

### 2. Predição em Lote

#### Python
```python
def predict_batch(weather_data_list):
    """Faz múltiplas predições em uma requisição"""
    url = 'http://localhost:8000/predict-batch'
    
    try:
        response = requests.post(
            url,
            json=weather_data_list,
            headers={'Content-Type': 'application/json'}
        )
        response.raise_for_status()
        
        results = response.json()
        
        print(f"Processadas {results['total']} predições:")
        for i, prediction in enumerate(results['predictions']):
            print(f"Predição {i+1}: {prediction['nivel_risco']} ({prediction['probabilidade_pct']})")
        
        return results
    
    except requests.exceptions.RequestException as e:
        print(f"Erro na predição em lote: {e}")
        return None

# Múltiplas condições meteorológicas
weather_batch = [
    {
        "temperatura_celsius": 25.0,
        "umidade_pct": 60.0,
        "precipitacao_mm_h": 5.0,
        "vento_kmh": 20.0,
        "pressao_hpa": 1015.0
    },
    {
        "temperatura_celsius": 35.0,
        "umidade_pct": 85.0,
        "precipitacao_mm_h": 30.0,
        "vento_kmh": 80.0,
        "pressao_hpa": 990.0
    },
    {
        "temperatura_celsius": 15.0,
        "umidade_pct": 45.0,
        "precipitacao_mm_h": 0.0,
        "vento_kmh": 10.0,
        "pressao_hpa": 1020.0
    }
]

# Processar lote
batch_results = predict_batch(weather_batch)
```

## 🔗 Integração entre APIs

### Fluxo Completo: Excel → Processamento → ML

#### Python - Pipeline Completo
```python
import requests
import pandas as pd
import json
from datetime import datetime

class GoodWeAPIsClient:
    def __init__(self, excel_api_url='http://localhost:3000', ml_api_url='http://localhost:8000'):
        self.excel_api = excel_api_url
        self.ml_api = ml_api_url
    
    def upload_and_convert_excel(self, file_path):
        """Upload e converte arquivo Excel"""
        url = f'{self.excel_api}/excel/upload'
        
        with open(file_path, 'rb') as file:
            files = {'excel': file}
            response = requests.post(url, files=files)
            response.raise_for_status()
            
            return response.json()
    
    def predict_from_excel_data(self, excel_data):
        """Extrai dados meteorológicos do Excel e faz predições"""
        predictions = []
        
        # Assumindo que a primeira planilha contém dados meteorológicos
        first_sheet = list(excel_data['data'].keys())[0]
        sheet_data = excel_data['data'][first_sheet]
        
        # Converter para DataFrame para facilitar manipulação
        df = pd.DataFrame(sheet_data[1:], columns=sheet_data[0])
        
        for index, row in df.iterrows():
            try:
                # Mapear colunas do Excel para formato da API ML
                weather_data = {
                    "temperatura_celsius": float(row.get('Temperatura', 25.0)),
                    "umidade_pct": float(row.get('Umidade', 60.0)),
                    "precipitacao_mm_h": float(row.get('Precipitacao', 0.0)),
                    "vento_kmh": float(row.get('Vento', 20.0)),
                    "pressao_hpa": float(row.get('Pressao', 1013.0))
                }
                
                # Fazer predição
                prediction = self.predict_power_outage(weather_data)
                if prediction:
                    predictions.append({
                        'linha': index + 2,  # +2 porque começamos da linha 1 (header) + index 0
                        'dados_originais': weather_data,
                        'predicao': prediction
                    })
            
            except (ValueError, KeyError) as e:
                print(f"Erro ao processar linha {index + 2}: {e}")
                continue
        
        return predictions
    
    def predict_power_outage(self, weather_data):
        """Faz predição individual"""
        url = f'{self.ml_api}/predict'
        
        try:
            response = requests.post(
                url,
                json=weather_data,
                headers={'Content-Type': 'application/json'}
            )
            response.raise_for_status()
            return response.json()
        
        except requests.exceptions.RequestException as e:
            print(f"Erro na predição: {e}")
            return None
    
    def generate_report(self, predictions):
        """Gera relatório das predições"""
        total = len(predictions)
        high_risk = len([p for p in predictions if p['predicao']['nivel_risco'] in ['Alto', 'Crítico']])
        
        report = {
            'timestamp': datetime.now().isoformat(),
            'total_predicoes': total,
            'alto_risco': high_risk,
            'percentual_risco': (high_risk / total * 100) if total > 0 else 0,
            'predicoes': predictions
        }
        
        return report

# Exemplo de uso
def main():
    client = GoodWeAPIsClient()
    
    # 1. Upload e conversão do Excel
    print("1. Fazendo upload do arquivo Excel...")
    excel_result = client.upload_and_convert_excel('dados_meteorologicos.xlsx')
    print(f"   ✓ Arquivo convertido: {excel_result['originalFile']}")
    
    # 2. Extrair dados e fazer predições
    print("2. Extraindo dados e fazendo predições...")
    predictions = client.predict_from_excel_data(excel_result)
    print(f"   ✓ {len(predictions)} predições realizadas")
    
    # 3. Gerar relatório
    print("3. Gerando relatório...")
    report = client.generate_report(predictions)
    
    # 4. Salvar relatório
    with open(f'relatorio_predicoes_{datetime.now().strftime("%Y%m%d_%H%M%S")}.json', 'w') as f:
        json.dump(report, f, indent=2, ensure_ascii=False)
    
    print(f"   ✓ Relatório salvo")
    print(f"   📊 Total: {report['total_predicoes']} predições")
    print(f"   ⚠️  Alto risco: {report['alto_risco']} ({report['percentual_risco']:.1f}%)")

if __name__ == "__main__":
    main()
```

## 💻 Exemplos por Linguagem

### JavaScript/Node.js - Cliente Completo

```javascript
// goodwe-client.js
const axios = require('axios');
const FormData = require('form-data');
const fs = require('fs');

class GoodWeClient {
    constructor(excelApiUrl = 'http://localhost:3000', mlApiUrl = 'http://localhost:8000') {
        this.excelApi = excelApiUrl;
        this.mlApi = mlApiUrl;
    }

    // Métodos para API Excel
    async convertExistingFile() {
        try {
            const response = await axios.get(`${this.excelApi}/excel/convert`);
            return response.data;
        } catch (error) {
            throw new Error(`Erro na conversão: ${error.message}`);
        }
    }

    async uploadExcelFile(filePath) {
        try {
            const formData = new FormData();
            formData.append('excel', fs.createReadStream(filePath));

            const response = await axios.post(`${this.excelApi}/excel/upload`, formData, {
                headers: formData.getHeaders()
            });

            return response.data;
        } catch (error) {
            throw new Error(`Erro no upload: ${error.message}`);
        }
    }

    async listExcelFiles() {
        try {
            const response = await axios.get(`${this.excelApi}/excel/files`);
            return response.data;
        } catch (error) {
            throw new Error(`Erro ao listar arquivos: ${error.message}`);
        }
    }

    // Métodos para API ML
    async predictPowerOutage(weatherData) {
        try {
            const response = await axios.post(`${this.mlApi}/predict`, weatherData);
            return response.data;
        } catch (error) {
            throw new Error(`Erro na predição: ${error.message}`);
        }
    }

    async predictBatch(weatherDataArray) {
        try {
            const response = await axios.post(`${this.mlApi}/predict-batch`, weatherDataArray);
            return response.data;
        } catch (error) {
            throw new Error(`Erro na predição em lote: ${error.message}`);
        }
    }

    async getModelInfo() {
        try {
            const response = await axios.get(`${this.mlApi}/model-info`);
            return response.data;
        } catch (error) {
            throw new Error(`Erro ao obter info do modelo: ${error.message}`);
        }
    }

    // Métodos combinados
    async processExcelAndPredict(filePath) {
        console.log('🔄 Processando arquivo Excel...');
        const excelData = await this.uploadExcelFile(filePath);
        
        console.log('🧠 Fazendo predições...');
        // Simular extração de dados meteorológicos
        const weatherSamples = [
            { temperatura_celsius: 25, umidade_pct: 65, precipitacao_mm_h: 10, vento_kmh: 30, pressao_hpa: 1013 },
            { temperatura_celsius: 35, umidade_pct: 80, precipitacao_mm_h: 25, vento_kmh: 70, pressao_hpa: 995 }
        ];
        
        const predictions = await this.predictBatch(weatherSamples);
        
        return {
            excelData,
            predictions
        };
    }
}

// Exemplo de uso
async function exemplo() {
    const client = new GoodWeClient();
    
    try {
        // Testar API ML
        console.log('🧪 Testando API Machine Learning...');
        const prediction = await client.predictPowerOutage({
            temperatura_celsius: 30,
            umidade_pct: 75,
            precipitacao_mm_h: 20,
            vento_kmh: 50,
            pressao_hpa: 1005
        });
        
        console.log(`Predição: ${prediction.queda_energia ? 'HAVERÁ' : 'NÃO HAVERÁ'} queda`);
        console.log(`Risco: ${prediction.nivel_risco} (${prediction.probabilidade_pct})`);
        
        // Testar API Excel
        console.log('\n📊 Testando API Excel...');
        const files = await client.listExcelFiles();
        console.log(`Encontrados ${files.count} arquivos Excel`);
        
    } catch (error) {
        console.error('Erro:', error.message);
    }
}

module.exports = GoodWeClient;

// Executar se chamado diretamente
if (require.main === module) {
    exemplo();
}
```

### PHP - Cliente Simples

```php
<?php
// GoodWeClient.php

class GoodWeClient {
    private $excelApiUrl;
    private $mlApiUrl;
    
    public function __construct($excelApiUrl = 'http://localhost:3000', $mlApiUrl = 'http://localhost:8000') {
        $this->excelApiUrl = $excelApiUrl;
        $this->mlApiUrl = $mlApiUrl;
    }
    
    public function predictPowerOutage($weatherData) {
        $url = $this->mlApiUrl . '/predict';
        
        $ch = curl_init();
        curl_setopt($ch, CURLOPT_URL, $url);
        curl_setopt($ch, CURLOPT_POST, true);
        curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($weatherData));
        curl_setopt($ch, CURLOPT_HTTPHEADER, ['Content-Type: application/json']);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        
        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        
        curl_close($ch);
        
        if ($httpCode !== 200) {
            throw new Exception("Erro na API: HTTP $httpCode");
        }
        
        return json_decode($response, true);
    }
    
    public function uploadExcelFile($filePath) {
        $url = $this->excelApiUrl . '/excel/upload';
        
        $ch = curl_init();
        curl_setopt($ch, CURLOPT_URL, $url);
        curl_setopt($ch, CURLOPT_POST, true);
        curl_setopt($ch, CURLOPT_POSTFIELDS, [
            'excel' => new CURLFile($filePath)
        ]);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        
        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        
        curl_close($ch);
        
        if ($httpCode !== 200) {
            throw new Exception("Erro no upload: HTTP $httpCode");
        }
        
        return json_decode($response, true);
    }
}

// Exemplo de uso
try {
    $client = new GoodWeClient();
    
    // Fazer predição
    $weatherData = [
        'temperatura_celsius' => 28.5,
        'umidade_pct' => 70.0,
        'precipitacao_mm_h' => 15.0,
        'vento_kmh' => 45.0,
        'pressao_hpa' => 1008.0
    ];
    
    $prediction = $client->predictPowerOutage($weatherData);
    
    echo "Predição: " . ($prediction['queda_energia'] ? 'HAVERÁ' : 'NÃO HAVERÁ') . " queda de energia\n";
    echo "Probabilidade: " . $prediction['probabilidade_pct'] . "\n";
    echo "Nível de risco: " . $prediction['nivel_risco'] . "\n";
    
} catch (Exception $e) {
    echo "Erro: " . $e->getMessage() . "\n";
}
?>
```

## 🚀 Casos de Uso Avançados

### 1. Dashboard em Tempo Real

```javascript
// dashboard.js - Atualização em tempo real
class PowerOutageDashboard {
    constructor() {
        this.client = new GoodWeClient();
        this.updateInterval = 30000; // 30 segundos
        this.currentWeatherData = null;
    }

    async init() {
        await this.updateDisplay();
        setInterval(() => this.updateDisplay(), this.updateInterval);
    }

    async updateDisplay() {
        try {
            // Simular dados meteorológicos em tempo real
            const weatherData = await this.getCurrentWeather();
            const prediction = await this.client.predictPowerOutage(weatherData);
            
            this.updateUI(weatherData, prediction);
            this.updateAlerts(prediction);
            
        } catch (error) {
            console.error('Erro ao atualizar dashboard:', error);
        }
    }

    async getCurrentWeather() {
        // Em um cenário real, isso viria de uma API meteorológica
        return {
            temperatura_celsius: 20 + Math.random() * 20,
            umidade_pct: 40 + Math.random() * 40,
            precipitacao_mm_h: Math.random() * 30,
            vento_kmh: Math.random() * 80,
            pressao_hpa: 990 + Math.random() * 40
        };
    }

    updateUI(weatherData, prediction) {
        document.getElementById('temperatura').textContent = weatherData.temperatura_celsius.toFixed(1) + '°C';
        document.getElementById('umidade').textContent = weatherData.umidade_pct.toFixed(1) + '%';
        document.getElementById('vento').textContent = weatherData.vento_kmh.toFixed(1) + ' km/h';
        document.getElementById('probabilidade').textContent = prediction.probabilidade_pct;
        document.getElementById('nivel-risco').textContent = prediction.nivel_risco;
        
        // Atualizar cores baseado no risco
        const riskElement = document.getElementById('nivel-risco');
        riskElement.className = `risk-${prediction.nivel_risco.toLowerCase()}`;
    }

    updateAlerts(prediction) {
        const alertContainer = document.getElementById('alerts');
        
        if (prediction.nivel_risco === 'Crítico') {
            alertContainer.innerHTML = `
                <div class="alert alert-danger">
                    ⚠️ ALERTA CRÍTICO: Alta probabilidade de queda de energia (${prediction.probabilidade_pct})
                </div>
            `;
        } else if (prediction.nivel_risco === 'Alto') {
            alertContainer.innerHTML = `
                <div class="alert alert-warning">
                    ⚡ ATENÇÃO: Risco elevado de queda de energia (${prediction.probabilidade_pct})
                </div>
            `;
        } else {
            alertContainer.innerHTML = '';
        }
    }
}

// Inicializar dashboard
document.addEventListener('DOMContentLoaded', () => {
    const dashboard = new PowerOutageDashboard();
    dashboard.init();
});
```

### 2. Sistema de Alertas por Email

```python
# alert_system.py
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
import schedule
import time
from datetime import datetime

class PowerOutageAlertSystem:
    def __init__(self, smtp_server, smtp_port, email, password):
        self.client = GoodWeAPIsClient()
        self.smtp_server = smtp_server
        self.smtp_port = smtp_port
        self.email = email
        self.password = password
        self.alert_recipients = []
    
    def add_recipient(self, email):
        self.alert_recipients.append(email)
    
    def check_conditions_and_alert(self):
        # Obter condições meteorológicas atuais
        weather_data = self.get_current_weather()
        
        # Fazer predição
        prediction = self.client.predict_power_outage(weather_data)
        
        if prediction and prediction['nivel_risco'] in ['Alto', 'Crítico']:
            self.send_alert(weather_data, prediction)
    
    def get_current_weather(self):
        # Em produção, isso viria de uma API meteorológica real
        import random
        return {
            "temperatura_celsius": 20 + random.random() * 20,
            "umidade_pct": 50 + random.random() * 30,
            "precipitacao_mm_h": random.random() * 25,
            "vento_kmh": random.random() * 70,
            "pressao_hpa": 995 + random.random() * 30
        }
    
    def send_alert(self, weather_data, prediction):
        subject = f"🚨 Alerta de Queda de Energia - Risco {prediction['nivel_risco']}"
        
        body = f"""
        ALERTA DE QUEDA DE ENERGIA
        
        Nível de Risco: {prediction['nivel_risco']}
        Probabilidade: {prediction['probabilidade_pct']}
        Timestamp: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}
        
        Condições Meteorológicas:
        - Temperatura: {weather_data['temperatura_celsius']:.1f}°C
        - Umidade: {weather_data['umidade_pct']:.1f}%
        - Precipitação: {weather_data['precipitacao_mm_h']:.1f} mm/h
        - Vento: {weather_data['vento_kmh']:.1f} km/h
        - Pressão: {weather_data['pressao_hpa']:.1f} hPa
        
        Ações Recomendadas:
        - Verificar equipamentos críticos
        - Preparar geradores de backup
        - Notificar equipes de manutenção
        """
        
        try:
            msg = MIMEMultipart()
            msg['From'] = self.email
            msg['Subject'] = subject
            msg.attach(MIMEText(body, 'plain'))
            
            server = smtplib.SMTP(self.smtp_server, self.smtp_port)
            server.starttls()
            server.login(self.email, self.password)
            
            for recipient in self.alert_recipients:
                msg['To'] = recipient
                server.send_message(msg)
                print(f"Alerta enviado para {recipient}")
            
            server.quit()
            
        except Exception as e:
            print(f"Erro ao enviar alerta: {e}")

# Configurar sistema de alertas
def main():
    alert_system = PowerOutageAlertSystem(
        smtp_server='smtp.gmail.com',
        smtp_port=587,
        email='your-email@gmail.com',
        password='your-app-password'
    )
    
    alert_system.add_recipient('admin@company.com')
    alert_system.add_recipient('maintenance@company.com')
    
    # Verificar a cada 15 minutos
    schedule.every(15).minutes.do(alert_system.check_conditions_and_alert)
    
    print("Sistema de alertas iniciado...")
    while True:
        schedule.run_pending()
        time.sleep(60)

if __name__ == "__main__":
    main()
```

## 📜 Scripts de Automação

### Script de Backup e Análise

```bash
#!/bin/bash
# backup_and_analyze.sh

DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backup/goodwe"
LOG_FILE="/var/log/goodwe/analysis_$DATE.log"

echo "🔄 Iniciando backup e análise - $DATE" | tee -a $LOG_FILE

# Criar diretório de backup
mkdir -p $BACKUP_DIR

# Backup de arquivos Excel
echo "📁 Fazendo backup de arquivos Excel..." | tee -a $LOG_FILE
tar -czf $BACKUP_DIR/excel_data_$DATE.tar.gz /app/API_GoodWe/scr/data/

# Testar APIs
echo "🧪 Testando APIs..." | tee -a $LOG_FILE

# Testar API Excel
EXCEL_HEALTH=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/health)
if [ $EXCEL_HEALTH -eq 200 ]; then
    echo "✅ API Excel: OK" | tee -a $LOG_FILE
else
    echo "❌ API Excel: FAIL (HTTP $EXCEL_HEALTH)" | tee -a $LOG_FILE
fi

# Testar API ML
ML_HEALTH=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8000/health)
if [ $ML_HEALTH -eq 200 ]; then
    echo "✅ API ML: OK" | tee -a $LOG_FILE
else
    echo "❌ API ML: FAIL (HTTP $ML_HEALTH)" | tee -a $LOG_FILE
fi

# Análise de dados
echo "📊 Executando análise de dados..." | tee -a $LOG_FILE
python3 << EOF
import requests
import json
from datetime import datetime

# Dados de teste para análise
test_scenarios = [
    {"temperatura_celsius": 25, "umidade_pct": 60, "precipitacao_mm_h": 5, "vento_kmh": 20, "pressao_hpa": 1015},
    {"temperatura_celsius": 35, "umidade_pct": 85, "precipitacao_mm_h": 30, "vento_kmh": 80, "pressao_hpa": 990},
    {"temperatura_celsius": 15, "umidade_pct": 45, "precipitacao_mm_h": 0, "vento_kmh": 10, "pressao_hpa": 1020}
]

results = []
for scenario in test_scenarios:
    try:
        response = requests.post('http://localhost:8000/predict', json=scenario, timeout=10)
        if response.status_code == 200:
            prediction = response.json()
            results.append({
                'scenario': scenario,
                'prediction': prediction
            })
    except Exception as e:
        print(f"Erro na predição: {e}")

# Salvar resultados
with open('/var/log/goodwe/predictions_$DATE.json', 'w') as f:
    json.dump(results, f, indent=2)

print(f"Análise concluída: {len(results)} cenários processados")
EOF

echo "✅ Backup e análise concluídos - $DATE" | tee -a $LOG_FILE
```

### Monitoramento com Cron

```bash
# Adicionar ao crontab
crontab -e

# Backup diário às 2h
0 2 * * * /scripts/backup_and_analyze.sh

# Verificação de saúde a cada 5 minutos
*/5 * * * * /scripts/health_check.sh

# Análise semanal aos domingos às 3h
0 3 * * 0 /scripts/weekly_analysis.sh
```

---

**Próxima seção**: [Troubleshooting](./troubleshooting.md)
