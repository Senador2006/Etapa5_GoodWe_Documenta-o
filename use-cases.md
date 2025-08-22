# Casos de Uso - APIs GoodWe

Este documento apresenta cenários reais de aplicação das APIs GoodWe, demonstrando como utilizar as funcionalidades em diferentes contextos empresariais e técnicos.

## 📋 Visão Geral dos Casos de Uso

As APIs GoodWe podem ser aplicadas em diversos cenários:

1. **Gestão de Energia** - Monitoramento e predição de quedas
2. **Análise de Dados** - Processamento de planilhas Excel
3. **Sistemas de Alerta** - Notificações proativas
4. **Dashboards** - Visualização em tempo real
5. **Automação Industrial** - Integração com sistemas SCADA

## ⚡ Caso de Uso 1: Sistema de Monitoramento de Energia

### Cenário
Uma empresa de distribuição de energia precisa monitorar condições climáticas e prever possíveis quedas de energia para otimizar manutenção preventiva e reduzir interrupções.

### Implementação

#### 1. Coleta de Dados Meteorológicos
```python
import requests
import schedule
import time
from datetime import datetime

class EnergyMonitoringSystem:
    def __init__(self):
        self.ml_api_url = 'http://localhost:8000'
        self.weather_stations = [
            {'id': 'ST001', 'location': 'Centro', 'lat': -23.5505, 'lon': -46.6333},
            {'id': 'ST002', 'location': 'Norte', 'lat': -23.4505, 'lon': -46.5333},
            {'id': 'ST003', 'location': 'Sul', 'lat': -23.6505, 'lon': -46.7333}
        ]
        self.alert_threshold = 0.75  # 75% de probabilidade
    
    def collect_weather_data(self, station_id):
        """Simula coleta de dados de estação meteorológica"""
        # Em produção, isso viria de sensores reais ou API meteorológica
        import random
        
        return {
            'station_id': station_id,
            'timestamp': datetime.now().isoformat(),
            'temperatura_celsius': round(15 + random.random() * 25, 1),
            'umidade_pct': round(40 + random.random() * 50, 1),
            'precipitacao_mm_h': round(random.random() * 30, 1),
            'vento_kmh': round(random.random() * 80, 1),
            'pressao_hpa': round(990 + random.random() * 40, 1)
        }
    
    def predict_outage_risk(self, weather_data):
        """Faz predição de risco de queda de energia"""
        try:
            response = requests.post(
                f'{self.ml_api_url}/predict',
                json={
                    'temperatura_celsius': weather_data['temperatura_celsius'],
                    'umidade_pct': weather_data['umidade_pct'],
                    'precipitacao_mm_h': weather_data['precipitacao_mm_h'],
                    'vento_kmh': weather_data['vento_kmh'],
                    'pressao_hpa': weather_data['pressao_hpa']
                }
            )
            return response.json()
        except Exception as e:
            print(f"Erro na predição: {e}")
            return None
    
    def monitor_all_stations(self):
        """Monitora todas as estações e gera alertas"""
        results = []
        
        for station in self.weather_stations:
            # Coletar dados
            weather_data = self.collect_weather_data(station['id'])
            
            # Fazer predição
            prediction = self.predict_outage_risk(weather_data)
            
            if prediction:
                result = {
                    'station': station,
                    'weather': weather_data,
                    'prediction': prediction,
                    'alert_level': self.determine_alert_level(prediction)
                }
                results.append(result)
                
                # Enviar alerta se necessário
                if prediction['probabilidade'] >= self.alert_threshold:
                    self.send_alert(result)
        
        return results
    
    def determine_alert_level(self, prediction):
        """Determina nível de alerta baseado na probabilidade"""
        prob = prediction['probabilidade']
        if prob >= 0.90:
            return 'CRÍTICO'
        elif prob >= 0.75:
            return 'ALTO'
        elif prob >= 0.50:
            return 'MÉDIO'
        else:
            return 'BAIXO'
    
    def send_alert(self, result):
        """Envia alerta para equipes responsáveis"""
        station = result['station']
        prediction = result['prediction']
        
        message = f"""
        🚨 ALERTA DE ENERGIA - {result['alert_level']}
        
        Estação: {station['location']} ({station['id']})
        Probabilidade de queda: {prediction['probabilidade_pct']}
        Nível de risco: {prediction['nivel_risco']}
        
        Condições atuais:
        - Vento: {result['weather']['vento_kmh']} km/h
        - Precipitação: {result['weather']['precipitacao_mm_h']} mm/h
        - Temperatura: {result['weather']['temperatura_celsius']}°C
        
        Ação recomendada: Verificar equipamentos críticos
        """
        
        print(message)
        # Aqui seria integrado com sistema de notificação real
        # (email, SMS, Slack, etc.)

# Executar monitoramento
def run_monitoring():
    monitor = EnergyMonitoringSystem()
    
    # Agendar monitoramento a cada 15 minutos
    schedule.every(15).minutes.do(monitor.monitor_all_stations)
    
    print("🔄 Sistema de monitoramento iniciado...")
    while True:
        schedule.run_pending()
        time.sleep(60)
```

#### 2. Dashboard de Visualização
```html
<!DOCTYPE html>
<html>
<head>
    <title>Dashboard de Energia - GoodWe</title>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <style>
        .station-card {
            border: 1px solid #ddd;
            margin: 10px;
            padding: 15px;
            border-radius: 8px;
        }
        .risk-low { border-left: 4px solid #28a745; }
        .risk-medium { border-left: 4px solid #ffc107; }
        .risk-high { border-left: 4px solid #fd7e14; }
        .risk-critical { border-left: 4px solid #dc3545; }
    </style>
</head>
<body>
    <h1>Dashboard de Monitoramento de Energia</h1>
    
    <div id="stations-container"></div>
    
    <canvas id="riskChart" width="800" height="400"></canvas>
    
    <script>
        class EnergyDashboard {
            constructor() {
                this.stations = [];
                this.chart = null;
                this.updateInterval = 30000; // 30 segundos
                this.initChart();
                this.startUpdates();
            }
            
            async fetchStationData() {
                // Simular dados das estações
                const mockData = [
                    {
                        id: 'ST001',
                        location: 'Centro',
                        weather: {
                            temperatura_celsius: 25 + Math.random() * 10,
                            vento_kmh: Math.random() * 60,
                            precipitacao_mm_h: Math.random() * 20
                        },
                        prediction: {
                            probabilidade: Math.random(),
                            nivel_risco: ['Baixo', 'Médio', 'Alto'][Math.floor(Math.random() * 3)]
                        }
                    },
                    // Mais estações...
                ];
                
                return mockData;
            }
            
            async updateDashboard() {
                const data = await this.fetchStationData();
                this.renderStations(data);
                this.updateChart(data);
            }
            
            renderStations(stations) {
                const container = document.getElementById('stations-container');
                container.innerHTML = '';
                
                stations.forEach(station => {
                    const riskClass = `risk-${station.prediction.nivel_risco.toLowerCase()}`;
                    const card = document.createElement('div');
                    card.className = `station-card ${riskClass}`;
                    card.innerHTML = `
                        <h3>${station.location} (${station.id})</h3>
                        <p><strong>Risco:</strong> ${station.prediction.nivel_risco}</p>
                        <p><strong>Probabilidade:</strong> ${(station.prediction.probabilidade * 100).toFixed(1)}%</p>
                        <p><strong>Temperatura:</strong> ${station.weather.temperatura_celsius.toFixed(1)}°C</p>
                        <p><strong>Vento:</strong> ${station.weather.vento_kmh.toFixed(1)} km/h</p>
                    `;
                    container.appendChild(card);
                });
            }
            
            initChart() {
                const ctx = document.getElementById('riskChart').getContext('2d');
                this.chart = new Chart(ctx, {
                    type: 'line',
                    data: {
                        labels: [],
                        datasets: [{
                            label: 'Risco Médio de Queda de Energia',
                            data: [],
                            borderColor: 'rgb(75, 192, 192)',
                            backgroundColor: 'rgba(75, 192, 192, 0.2)',
                            tension: 0.1
                        }]
                    },
                    options: {
                        responsive: true,
                        scales: {
                            y: {
                                beginAtZero: true,
                                max: 1,
                                ticks: {
                                    callback: function(value) {
                                        return (value * 100).toFixed(0) + '%';
                                    }
                                }
                            }
                        }
                    }
                });
            }
            
            updateChart(stations) {
                const avgRisk = stations.reduce((sum, station) => 
                    sum + station.prediction.probabilidade, 0) / stations.length;
                
                const now = new Date().toLocaleTimeString();
                
                this.chart.data.labels.push(now);
                this.chart.data.datasets[0].data.push(avgRisk);
                
                // Manter apenas últimos 20 pontos
                if (this.chart.data.labels.length > 20) {
                    this.chart.data.labels.shift();
                    this.chart.data.datasets[0].data.shift();
                }
                
                this.chart.update();
            }
            
            startUpdates() {
                this.updateDashboard();
                setInterval(() => this.updateDashboard(), this.updateInterval);
            }
        }
        
        // Inicializar dashboard
        document.addEventListener('DOMContentLoaded', () => {
            new EnergyDashboard();
        });
    </script>
</body>
</html>
```

## 📊 Caso de Uso 2: Análise de Dados de Consumo Energético

### Cenário
Uma empresa precisa analisar grandes volumes de dados de consumo energético armazenados em planilhas Excel para identificar padrões e otimizar distribuição.

### Implementação

```python
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns
from datetime import datetime, timedelta

class EnergyConsumptionAnalyzer:
    def __init__(self, excel_api_url='http://localhost:3000'):
        self.excel_api_url = excel_api_url
        self.data_cache = {}
    
    def upload_and_process_excel(self, file_path):
        """Upload arquivo Excel e processa dados"""
        import requests
        
        with open(file_path, 'rb') as file:
            files = {'excel': file}
            response = requests.post(f'{self.excel_api_url}/excel/upload', files=files)
            
            if response.status_code == 200:
                excel_data = response.json()
                return self.process_consumption_data(excel_data)
            else:
                raise Exception(f"Erro no upload: {response.status_code}")
    
    def process_consumption_data(self, excel_data):
        """Processa dados de consumo do Excel"""
        # Assumindo que a primeira planilha contém dados de consumo
        sheet_name = list(excel_data['data'].keys())[0]
        raw_data = excel_data['data'][sheet_name]
        
        # Converter para DataFrame
        df = pd.DataFrame(raw_data[1:], columns=raw_data[0])
        
        # Limpeza e conversão de dados
        df = self.clean_consumption_data(df)
        
        return df
    
    def clean_consumption_data(self, df):
        """Limpa e normaliza dados de consumo"""
        # Converter colunas numéricas
        numeric_columns = ['Consumo_kWh', 'Demanda_kW', 'Fator_Potencia']
        for col in numeric_columns:
            if col in df.columns:
                df[col] = pd.to_numeric(df[col], errors='coerce')
        
        # Converter datas
        if 'Data' in df.columns:
            df['Data'] = pd.to_datetime(df['Data'], errors='coerce')
        
        # Remover linhas com dados inválidos
        df = df.dropna(subset=['Consumo_kWh'])
        
        return df
    
    def analyze_consumption_patterns(self, df):
        """Analisa padrões de consumo"""
        analysis = {
            'total_consumption': df['Consumo_kWh'].sum(),
            'avg_daily_consumption': df['Consumo_kWh'].mean(),
            'peak_consumption': df['Consumo_kWh'].max(),
            'peak_date': df.loc[df['Consumo_kWh'].idxmax(), 'Data'],
            'consumption_trend': self.calculate_trend(df),
            'efficiency_metrics': self.calculate_efficiency(df)
        }
        
        return analysis
    
    def calculate_trend(self, df):
        """Calcula tendência de consumo"""
        if 'Data' in df.columns:
            df_sorted = df.sort_values('Data')
            # Calcular média móvel de 7 dias
            df_sorted['MA_7'] = df_sorted['Consumo_kWh'].rolling(window=7).mean()
            
            # Calcular tendência (slope)
            import numpy as np
            x = np.arange(len(df_sorted))
            y = df_sorted['Consumo_kWh'].values
            slope, _ = np.polyfit(x, y, 1)
            
            return {
                'slope': slope,
                'direction': 'crescente' if slope > 0 else 'decrescente',
                'moving_average': df_sorted['MA_7'].tolist()
            }
        
        return None
    
    def calculate_efficiency(self, df):
        """Calcula métricas de eficiência energética"""
        if 'Fator_Potencia' in df.columns:
            return {
                'avg_power_factor': df['Fator_Potencia'].mean(),
                'efficiency_score': df['Fator_Potencia'].mean() * 100,
                'low_efficiency_days': len(df[df['Fator_Potencia'] < 0.85])
            }
        return None
    
    def generate_reports(self, df, analysis):
        """Gera relatórios e visualizações"""
        # Relatório em texto
        report = f"""
        RELATÓRIO DE ANÁLISE DE CONSUMO ENERGÉTICO
        ==========================================
        
        Período analisado: {df['Data'].min()} a {df['Data'].max()}
        Total de registros: {len(df)}
        
        CONSUMO:
        - Total: {analysis['total_consumption']:,.2f} kWh
        - Média diária: {analysis['avg_daily_consumption']:,.2f} kWh
        - Pico: {analysis['peak_consumption']:,.2f} kWh ({analysis['peak_date']})
        
        TENDÊNCIA:
        - Direção: {analysis['consumption_trend']['direction'] if analysis['consumption_trend'] else 'N/A'}
        - Taxa de variação: {analysis['consumption_trend']['slope']:.4f} kWh/dia if analysis['consumption_trend'] else 'N/A'}
        
        EFICIÊNCIA:
        - Fator de potência médio: {analysis['efficiency_metrics']['avg_power_factor']:.3f if analysis['efficiency_metrics'] else 'N/A'}
        - Score de eficiência: {analysis['efficiency_metrics']['efficiency_score']:.1f}% if analysis['efficiency_metrics'] else 'N/A'}
        """
        
        # Gerar gráficos
        self.create_visualizations(df, analysis)
        
        return report
    
    def create_visualizations(self, df, analysis):
        """Cria visualizações dos dados"""
        fig, axes = plt.subplots(2, 2, figsize=(15, 10))
        
        # Gráfico 1: Consumo ao longo do tempo
        if 'Data' in df.columns:
            axes[0, 0].plot(df['Data'], df['Consumo_kWh'])
            axes[0, 0].set_title('Consumo de Energia ao Longo do Tempo')
            axes[0, 0].set_xlabel('Data')
            axes[0, 0].set_ylabel('Consumo (kWh)')
            axes[0, 0].tick_params(axis='x', rotation=45)
        
        # Gráfico 2: Distribuição do consumo
        axes[0, 1].hist(df['Consumo_kWh'], bins=30, alpha=0.7)
        axes[0, 1].set_title('Distribuição do Consumo')
        axes[0, 1].set_xlabel('Consumo (kWh)')
        axes[0, 1].set_ylabel('Frequência')
        
        # Gráfico 3: Boxplot por mês
        if 'Data' in df.columns:
            df['Mes'] = df['Data'].dt.month
            df.boxplot(column='Consumo_kWh', by='Mes', ax=axes[1, 0])
            axes[1, 0].set_title('Consumo por Mês')
            axes[1, 0].set_xlabel('Mês')
            axes[1, 0].set_ylabel('Consumo (kWh)')
        
        # Gráfico 4: Fator de potência
        if 'Fator_Potencia' in df.columns:
            axes[1, 1].plot(df['Data'], df['Fator_Potencia'], color='orange')
            axes[1, 1].set_title('Fator de Potência')
            axes[1, 1].set_xlabel('Data')
            axes[1, 1].set_ylabel('Fator de Potência')
            axes[1, 1].tick_params(axis='x', rotation=45)
        
        plt.tight_layout()
        plt.savefig(f'consumption_analysis_{datetime.now().strftime("%Y%m%d")}.png')
        plt.show()

# Exemplo de uso
def main():
    analyzer = EnergyConsumptionAnalyzer()
    
    try:
        # Processar arquivo Excel
        df = analyzer.upload_and_process_excel('dados_consumo.xlsx')
        
        # Analisar dados
        analysis = analyzer.analyze_consumption_patterns(df)
        
        # Gerar relatório
        report = analyzer.generate_reports(df, analysis)
        
        print(report)
        
        # Salvar relatório
        with open(f'relatorio_consumo_{datetime.now().strftime("%Y%m%d")}.txt', 'w') as f:
            f.write(report)
            
    except Exception as e:
        print(f"Erro na análise: {e}")

if __name__ == "__main__":
    main()
```

## 🚨 Caso de Uso 3: Sistema de Alertas Inteligente

### Cenário
Implementar um sistema que monitora múltiplas variáveis e envia alertas personalizados para diferentes stakeholders baseado em níveis de risco.

### Implementação

```python
from dataclasses import dataclass
from typing import List, Dict, Optional
from enum import Enum
import asyncio
import aiohttp
import json

class AlertLevel(Enum):
    INFO = "info"
    WARNING = "warning" 
    CRITICAL = "critical"
    EMERGENCY = "emergency"

@dataclass
class AlertRule:
    name: str
    condition: callable
    level: AlertLevel
    recipients: List[str]
    message_template: str
    cooldown_minutes: int = 30

@dataclass
class Alert:
    rule_name: str
    level: AlertLevel
    message: str
    timestamp: str
    data: Dict

class SmartAlertSystem:
    def __init__(self):
        self.alert_rules = []
        self.alert_history = []
        self.cooldown_tracker = {}
        self.notification_channels = {}
    
    def add_alert_rule(self, rule: AlertRule):
        """Adiciona regra de alerta"""
        self.alert_rules.append(rule)
    
    def add_notification_channel(self, name: str, handler: callable):
        """Adiciona canal de notificação"""
        self.notification_channels[name] = handler
    
    async def evaluate_conditions(self, data: Dict):
        """Avalia todas as condições e gera alertas necessários"""
        alerts = []
        
        for rule in self.alert_rules:
            try:
                if rule.condition(data):
                    # Verificar cooldown
                    if self.check_cooldown(rule.name):
                        alert = self.create_alert(rule, data)
                        alerts.append(alert)
                        await self.send_alert(alert, rule.recipients)
                        
            except Exception as e:
                print(f"Erro ao avaliar regra {rule.name}: {e}")
        
        return alerts
    
    def check_cooldown(self, rule_name: str) -> bool:
        """Verifica se regra está em cooldown"""
        from datetime import datetime, timedelta
        
        last_alert = self.cooldown_tracker.get(rule_name)
        if not last_alert:
            return True
        
        rule = next(r for r in self.alert_rules if r.name == rule_name)
        cooldown_end = last_alert + timedelta(minutes=rule.cooldown_minutes)
        
        return datetime.now() > cooldown_end
    
    def create_alert(self, rule: AlertRule, data: Dict) -> Alert:
        """Cria alerta baseado na regra"""
        from datetime import datetime
        
        message = rule.message_template.format(**data)
        
        alert = Alert(
            rule_name=rule.name,
            level=rule.level,
            message=message,
            timestamp=datetime.now().isoformat(),
            data=data
        )
        
        self.alert_history.append(alert)
        self.cooldown_tracker[rule.name] = datetime.now()
        
        return alert
    
    async def send_alert(self, alert: Alert, recipients: List[str]):
        """Envia alerta para destinatários"""
        for channel_name, handler in self.notification_channels.items():
            try:
                await handler(alert, recipients)
            except Exception as e:
                print(f"Erro ao enviar via {channel_name}: {e}")

# Canais de notificação
class NotificationChannels:
    @staticmethod
    async def email_handler(alert: Alert, recipients: List[str]):
        """Handler para notificação por email"""
        import smtplib
        from email.mime.text import MIMEText
        
        # Configuração SMTP (exemplo)
        smtp_config = {
            'server': 'smtp.gmail.com',
            'port': 587,
            'username': 'your-email@gmail.com',
            'password': 'your-password'
        }
        
        try:
            msg = MIMEText(alert.message)
            msg['Subject'] = f"🚨 Alerta {alert.level.value.upper()} - {alert.rule_name}"
            msg['From'] = smtp_config['username']
            
            server = smtplib.SMTP(smtp_config['server'], smtp_config['port'])
            server.starttls()
            server.login(smtp_config['username'], smtp_config['password'])
            
            for recipient in recipients:
                msg['To'] = recipient
                server.send_message(msg)
                print(f"Email enviado para {recipient}")
            
            server.quit()
            
        except Exception as e:
            print(f"Erro ao enviar email: {e}")
    
    @staticmethod
    async def slack_handler(alert: Alert, recipients: List[str]):
        """Handler para notificação via Slack"""
        webhook_url = "https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK"
        
        # Mapear nível de alerta para cor
        color_map = {
            AlertLevel.INFO: "#36a64f",
            AlertLevel.WARNING: "#ffeb3b", 
            AlertLevel.CRITICAL: "#ff9800",
            AlertLevel.EMERGENCY: "#f44336"
        }
        
        payload = {
            "attachments": [{
                "color": color_map.get(alert.level, "#36a64f"),
                "title": f"Alerta: {alert.rule_name}",
                "text": alert.message,
                "footer": "Sistema GoodWe",
                "ts": int(datetime.fromisoformat(alert.timestamp).timestamp())
            }]
        }
        
        async with aiohttp.ClientSession() as session:
            async with session.post(webhook_url, json=payload) as response:
                if response.status == 200:
                    print("Mensagem Slack enviada")
                else:
                    print(f"Erro no Slack: {response.status}")
    
    @staticmethod
    async def sms_handler(alert: Alert, recipients: List[str]):
        """Handler para notificação via SMS (usando Twilio)"""
        from twilio.rest import Client
        
        account_sid = 'your_account_sid'
        auth_token = 'your_auth_token'
        from_number = '+1234567890'
        
        client = Client(account_sid, auth_token)
        
        for recipient in recipients:
            try:
                message = client.messages.create(
                    body=f"ALERTA: {alert.message}",
                    from_=from_number,
                    to=recipient
                )
                print(f"SMS enviado para {recipient}: {message.sid}")
            except Exception as e:
                print(f"Erro ao enviar SMS: {e}")

# Configuração do sistema de alertas
def setup_alert_system():
    alert_system = SmartAlertSystem()
    
    # Adicionar canais de notificação
    alert_system.add_notification_channel('email', NotificationChannels.email_handler)
    alert_system.add_notification_channel('slack', NotificationChannels.slack_handler)
    alert_system.add_notification_channel('sms', NotificationChannels.sms_handler)
    
    # Regras de alerta para energia
    alert_system.add_alert_rule(AlertRule(
        name="High Risk Power Outage",
        condition=lambda data: data.get('probabilidade', 0) > 0.8,
        level=AlertLevel.EMERGENCY,
        recipients=['admin@company.com', '+5511999999999'],
        message_template="🚨 EMERGÊNCIA: Alta probabilidade ({probabilidade_pct}) de queda de energia. Estação: {station_id}. Condições: Vento {vento_kmh}km/h, Chuva {precipitacao_mm_h}mm/h.",
        cooldown_minutes=15
    ))
    
    alert_system.add_alert_rule(AlertRule(
        name="Medium Risk Power Outage",
        condition=lambda data: 0.5 < data.get('probabilidade', 0) <= 0.8,
        level=AlertLevel.WARNING,
        recipients=['operations@company.com'],
        message_template="⚠️ ATENÇÃO: Risco moderado ({probabilidade_pct}) de queda de energia. Preparar equipes de contingência.",
        cooldown_minutes=30
    ))
    
    alert_system.add_alert_rule(AlertRule(
        name="Equipment Malfunction",
        condition=lambda data: data.get('equipment_status') == 'error',
        level=AlertLevel.CRITICAL,
        recipients=['maintenance@company.com', 'manager@company.com'],
        message_template="🔧 EQUIPAMENTO: Falha detectada no equipamento {equipment_id}. Status: {equipment_status}. Requer atenção imediata.",
        cooldown_minutes=60
    ))
    
    return alert_system

# Exemplo de integração com monitoramento
async def main():
    alert_system = setup_alert_system()
    
    # Simular dados de monitoramento
    monitoring_data = {
        'station_id': 'ST001',
        'probabilidade': 0.85,
        'probabilidade_pct': '85%',
        'vento_kmh': 75,
        'precipitacao_mm_h': 25,
        'equipment_status': 'ok'
    }
    
    # Avaliar condições e enviar alertas
    alerts = await alert_system.evaluate_conditions(monitoring_data)
    
    print(f"Gerados {len(alerts)} alertas")
    for alert in alerts:
        print(f"- {alert.rule_name}: {alert.level.value}")

if __name__ == "__main__":
    asyncio.run(main())
```

## 🏭 Caso de Uso 4: Integração com Sistema SCADA

### Cenário
Integrar as APIs GoodWe com um sistema SCADA existente para automatizar decisões operacionais baseadas em predições de ML.

### Implementação

```python
import asyncio
import json
from datetime import datetime
import logging
from typing import Dict, List

class SCADAIntegration:
    def __init__(self, scada_host: str, goodwe_apis: Dict[str, str]):
        self.scada_host = scada_host
        self.excel_api_url = goodwe_apis['excel']
        self.ml_api_url = goodwe_apis['ml']
        self.logger = logging.getLogger(__name__)
        self.operational_rules = []
        
    async def collect_scada_data(self) -> Dict:
        """Coleta dados do sistema SCADA"""
        # Simular coleta de dados SCADA
        # Em produção, isso seria integração real com protocolo Modbus/OPC-UA
        scada_data = {
            'timestamp': datetime.now().isoformat(),
            'substations': {
                'SUB001': {
                    'voltage': 138.5,
                    'current': 245.8,
                    'power_factor': 0.92,
                    'load_percentage': 78.5,
                    'temperature': 45.2,
                    'status': 'online'
                },
                'SUB002': {
                    'voltage': 137.8,
                    'current': 189.3,
                    'power_factor': 0.89,
                    'load_percentage': 65.2,
                    'temperature': 43.1,
                    'status': 'online'
                }
            },
            'weather_stations': {
                'WS001': {
                    'temperatura_celsius': 28.5,
                    'umidade_pct': 72.0,
                    'precipitacao_mm_h': 15.0,
                    'vento_kmh': 45.0,
                    'pressao_hpa': 1005.0
                }
            },
            'grid_status': {
                'total_generation': 2850.5,  # MW
                'total_demand': 2780.3,      # MW
                'reserve_margin': 70.2,      # MW
                'frequency': 60.02           # Hz
            }
        }
        
        return scada_data
    
    async def predict_grid_stability(self, weather_data: Dict) -> Dict:
        """Prediz estabilidade da rede usando API ML"""
        try:
            import aiohttp
            
            async with aiohttp.ClientSession() as session:
                async with session.post(
                    f'{self.ml_api_url}/predict',
                    json=weather_data
                ) as response:
                    if response.status == 200:
                        return await response.json()
                    else:
                        self.logger.error(f"Erro na predição ML: {response.status}")
                        return None
                        
        except Exception as e:
            self.logger.error(f"Erro ao conectar com API ML: {e}")
            return None
    
    async def execute_operational_actions(self, scada_data: Dict, predictions: List[Dict]):
        """Executa ações operacionais baseadas em predições"""
        for prediction in predictions:
            risk_level = prediction.get('nivel_risco', 'Baixo')
            probability = prediction.get('probabilidade', 0)
            
            if risk_level == 'Crítico' or probability > 0.8:
                await self.emergency_procedures(scada_data, prediction)
            elif risk_level == 'Alto' or probability > 0.6:
                await self.preventive_procedures(scada_data, prediction)
            elif risk_level == 'Médio' or probability > 0.4:
                await self.monitoring_procedures(scada_data, prediction)
    
    async def emergency_procedures(self, scada_data: Dict, prediction: Dict):
        """Procedimentos de emergência"""
        actions_taken = []
        
        # 1. Ativar geradores de backup
        backup_generators = await self.activate_backup_generators()
        if backup_generators:
            actions_taken.append("Geradores de backup ativados")
        
        # 2. Redistribuir carga
        load_redistribution = await self.redistribute_load(scada_data)
        if load_redistribution:
            actions_taken.append("Carga redistribuída entre subestações")
        
        # 3. Alertar operadores
        await self.alert_operators("EMERGENCY", prediction, actions_taken)
        
        # 4. Registrar ações no SCADA
        await self.log_scada_actions("EMERGENCY_RESPONSE", actions_taken)
        
        self.logger.critical(f"Procedimentos de emergência executados: {actions_taken}")
    
    async def preventive_procedures(self, scada_data: Dict, prediction: Dict):
        """Procedimentos preventivos"""
        actions_taken = []
        
        # 1. Preparar geradores
        await self.prepare_backup_systems()
        actions_taken.append("Sistemas de backup preparados")
        
        # 2. Otimizar distribuição de carga
        await self.optimize_load_distribution(scada_data)
        actions_taken.append("Distribuição de carga otimizada")
        
        # 3. Notificar equipes
        await self.notify_maintenance_teams(prediction)
        actions_taken.append("Equipes de manutenção notificadas")
        
        self.logger.warning(f"Procedimentos preventivos executados: {actions_taken}")
    
    async def monitoring_procedures(self, scada_data: Dict, prediction: Dict):
        """Procedimentos de monitoramento intensivo"""
        # Aumentar frequência de monitoramento
        await self.increase_monitoring_frequency()
        
        # Verificar status de equipamentos críticos
        await self.check_critical_equipment()
        
        self.logger.info("Monitoramento intensivo ativado")
    
    async def activate_backup_generators(self) -> bool:
        """Ativa geradores de backup"""
        # Simulação de comando SCADA
        self.logger.info("Comando SCADA: Ativando geradores de backup")
        # Em produção: enviar comando via protocolo SCADA
        return True
    
    async def redistribute_load(self, scada_data: Dict) -> bool:
        """Redistribui carga entre subestações"""
        substations = scada_data.get('substations', {})
        
        # Encontrar subestações com menor carga
        sorted_subs = sorted(
            substations.items(),
            key=lambda x: x[1]['load_percentage']
        )
        
        # Lógica de redistribuição
        redistribution_plan = []
        for sub_id, sub_data in sorted_subs:
            if sub_data['load_percentage'] < 70:
                redistribution_plan.append(f"Aumentar carga em {sub_id}")
            elif sub_data['load_percentage'] > 85:
                redistribution_plan.append(f"Reduzir carga em {sub_id}")
        
        self.logger.info(f"Plano de redistribuição: {redistribution_plan}")
        return len(redistribution_plan) > 0
    
    async def generate_operational_report(self, scada_data: Dict, predictions: List[Dict], actions: List[str]):
        """Gera relatório operacional"""
        report = {
            'timestamp': datetime.now().isoformat(),
            'scada_summary': {
                'total_substations': len(scada_data.get('substations', {})),
                'online_substations': sum(1 for sub in scada_data.get('substations', {}).values() if sub['status'] == 'online'),
                'avg_load': sum(sub['load_percentage'] for sub in scada_data.get('substations', {}).values()) / len(scada_data.get('substations', {})),
                'grid_frequency': scada_data.get('grid_status', {}).get('frequency', 0)
            },
            'ml_predictions': predictions,
            'actions_taken': actions,
            'system_status': 'STABLE' if all(p.get('probabilidade', 0) < 0.5 for p in predictions) else 'AT_RISK'
        }
        
        # Salvar relatório
        filename = f"operational_report_{datetime.now().strftime('%Y%m%d_%H%M%S')}.json"
        with open(filename, 'w') as f:
            json.dump(report, f, indent=2, ensure_ascii=False)
        
        return report

# Exemplo de execução do sistema integrado
async def run_scada_integration():
    # Configurar sistema
    scada_system = SCADAIntegration(
        scada_host='192.168.1.100',
        goodwe_apis={
            'excel': 'http://localhost:3000',
            'ml': 'http://localhost:8000'
        }
    )
    
    while True:
        try:
            # 1. Coletar dados SCADA
            scada_data = await scada_system.collect_scada_data()
            
            # 2. Fazer predições para cada estação meteorológica
            predictions = []
            for ws_id, weather_data in scada_data['weather_stations'].items():
                prediction = await scada_system.predict_grid_stability(weather_data)
                if prediction:
                    prediction['station_id'] = ws_id
                    predictions.append(prediction)
            
            # 3. Executar ações operacionais
            await scada_system.execute_operational_actions(scada_data, predictions)
            
            # 4. Gerar relatório
            actions = []  # Seria preenchido pelas funções de ação
            report = await scada_system.generate_operational_report(scada_data, predictions, actions)
            
            print(f"Ciclo executado - Status: {report['system_status']}")
            
            # Aguardar próximo ciclo (5 minutos)
            await asyncio.sleep(300)
            
        except Exception as e:
            logging.error(f"Erro no ciclo SCADA: {e}")
            await asyncio.sleep(60)  # Retry em 1 minuto

if __name__ == "__main__":
    logging.basicConfig(level=logging.INFO)
    asyncio.run(run_scada_integration())
```

## 📈 Benefícios dos Casos de Uso

### Caso 1 - Monitoramento de Energia
- **Redução de downtime**: 30-50% menos interrupções não planejadas
- **Economia de custos**: Manutenção preventiva vs. corretiva
- **Melhoria na qualidade**: Menos variações na entrega de energia

### Caso 2 - Análise de Consumo
- **Otimização de recursos**: Identificação de padrões de desperdício
- **Planejamento estratégico**: Previsão de demanda baseada em dados
- **Compliance**: Relatórios automáticos para órgãos reguladores

### Caso 3 - Sistema de Alertas
- **Resposta rápida**: Alertas em tempo real
- **Escalabilidade**: Diferentes níveis para diferentes stakeholders
- **Integração**: Múltiplos canais de comunicação

### Caso 4 - Integração SCADA
- **Automação**: Decisões operacionais automatizadas
- **Confiabilidade**: Redução de erro humano
- **Eficiência**: Otimização contínua da rede elétrica

---

**Seção anterior**: [Exemplos de Uso](./examples.md)  
**Próxima seção**: [Troubleshooting](./troubleshooting.md)  
**Voltar ao início**: [README](./README.md)
