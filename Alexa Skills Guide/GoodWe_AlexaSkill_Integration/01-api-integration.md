# Integração com APIs - GoodWe Alexa Skill

## 📋 Visão Geral

Este documento detalha a integração da Alexa Skill com as APIs GoodWe, incluindo configuração, autenticação, tratamento de erros e otimizações.

## 🔌 APIs Integradas

### 1. API Principal GoodWe
- **URL**: `http://localhost:3000` (desenvolvimento) / `https://api.goodwe.com` (produção)
- **Funcionalidades**: Dados de monitoramento, análises estatísticas, busca avançada
- **Endpoints Principais**: `/data`, `/analytics`, `/search`

### 2. API Machine Learning
- **URL**: `http://localhost:8000` (desenvolvimento) / `https://ml-api.goodwe.com` (produção)
- **Funcionalidades**: Predições climáticas, análise de quedas de energia
- **Endpoints Principais**: `/predict`, `/predict-batch`, `/model-info`

### 3. APIs Externas
- **Clima**: OpenWeatherMap, AccuWeather
- **Concessionárias**: APIs de tarifas e horários
- **IoT**: APIs de dispositivos conectados

## 🛠️ Configuração da Integração

### 1. Configuração Base

```javascript
// config/api-config.js
const API_CONFIG = {
  goodwe: {
    baseUrl: process.env.GOODWE_API_URL || 'http://localhost:3000',
    apiKey: process.env.GOODWE_API_KEY,
    timeout: 5000,
    retries: 3,
    retryDelay: 1000
  },
  ml: {
    baseUrl: process.env.ML_API_URL || 'http://localhost:8000',
    apiKey: process.env.ML_API_KEY,
    timeout: 10000,
    retries: 2,
    retryDelay: 2000
  },
  weather: {
    baseUrl: 'https://api.openweathermap.org/data/2.5',
    apiKey: process.env.WEATHER_API_KEY,
    timeout: 5000
  },
  utility: {
    baseUrl: process.env.UTILITY_API_URL,
    apiKey: process.env.UTILITY_API_KEY,
    timeout: 5000
  }
};
```

### 2. Cliente HTTP Configurado

```javascript
// utils/http-client.js
const axios = require('axios');
const { API_CONFIG } = require('../config/api-config');

class APIClient {
  constructor(config) {
    this.client = axios.create({
      baseURL: config.baseUrl,
      timeout: config.timeout,
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${config.apiKey}`
      }
    });
    
    this.setupInterceptors();
  }
  
  setupInterceptors() {
    // Interceptor de requisição
    this.client.interceptors.request.use(
      (config) => {
        console.log(`[API] ${config.method.toUpperCase()} ${config.url}`);
        return config;
      },
      (error) => {
        console.error('[API] Request error:', error);
        return Promise.reject(error);
      }
    );
    
    // Interceptor de resposta
    this.client.interceptors.response.use(
      (response) => {
        console.log(`[API] ${response.status} ${response.config.url}`);
        return response;
      },
      async (error) => {
        console.error('[API] Response error:', error.message);
        
        // Retry automático para erros de rede
        if (this.shouldRetry(error)) {
          return this.retryRequest(error.config);
        }
        
        return Promise.reject(error);
      }
    );
  }
  
  shouldRetry(error) {
    return error.code === 'ECONNREFUSED' || 
           error.code === 'ETIMEDOUT' ||
           (error.response && error.response.status >= 500);
  }
  
  async retryRequest(config) {
    const maxRetries = 3;
    let retryCount = 0;
    
    while (retryCount < maxRetries) {
      try {
        retryCount++;
        console.log(`[API] Retry ${retryCount}/${maxRetries} for ${config.url}`);
        
        await new Promise(resolve => setTimeout(resolve, 1000 * retryCount));
        return await this.client.request(config);
      } catch (error) {
        if (retryCount === maxRetries) {
          throw error;
        }
      }
    }
  }
}

// Instâncias dos clientes
const goodweClient = new APIClient(API_CONFIG.goodwe);
const mlClient = new APIClient(API_CONFIG.ml);
const weatherClient = new APIClient(API_CONFIG.weather);
const utilityClient = new APIClient(API_CONFIG.utility);

module.exports = {
  goodweClient,
  mlClient,
  weatherClient,
  utilityClient
};
```

## 📊 Integração com API Principal GoodWe

### 1. Dados de Monitoramento

```javascript
// services/goodwe-service.js
const { goodweClient } = require('../utils/http-client');

class GoodWeService {
  async getSystemStatus() {
    try {
      const response = await goodweClient.client.get('/data/paginated?limit=1');
      
      if (!response.data.success) {
        throw new Error('Falha ao obter dados do sistema');
      }
      
      return this.formatSystemData(response.data.data[0]);
    } catch (error) {
      console.error('Erro ao obter status do sistema:', error);
      throw new Error('Sistema temporariamente indisponível');
    }
  }
  
  async getEnergyGeneration(timePeriod = 'now') {
    try {
      let endpoint = '/data/paginated?limit=1';
      
      if (timePeriod !== 'now') {
        endpoint = '/analytics/hourly';
      }
      
      const response = await goodweClient.client.get(endpoint);
      return this.formatEnergyData(response.data, timePeriod);
    } catch (error) {
      console.error('Erro ao obter dados de geração:', error);
      throw new Error('Não foi possível obter dados de geração');
    }
  }
  
  async getBatteryStatus() {
    try {
      const response = await goodweClient.client.get('/data/paginated?limit=1');
      const data = response.data.data[0];
      
      return {
        level: data.soc_percentage || 0,
        power: data.battery_power || 0,
        status: this.getBatteryStatusText(data.soc_percentage, data.battery_power),
        isCharging: data.battery_power > 0,
        isDischarging: data.battery_power < 0
      };
    } catch (error) {
      console.error('Erro ao obter status da bateria:', error);
      throw new Error('Não foi possível obter status da bateria');
    }
  }
  
  async getEfficiencyAnalysis(analysisType = 'daily') {
    try {
      const endpoint = analysisType === 'daily' 
        ? '/analytics/hourly' 
        : '/analytics/trends';
      
      const response = await goodweClient.client.get(endpoint, {
        params: { period: analysisType }
      });
      
      return this.formatEfficiencyData(response.data, analysisType);
    } catch (error) {
      console.error('Erro na análise de eficiência:', error);
      throw new Error('Não foi possível obter análise de eficiência');
    }
  }
  
  formatSystemData(data) {
    return {
      fvPower: data.fv_power || 0,
      soc: data.soc_percentage || 0,
      batteryPower: data.battery_power || 0,
      gridPower: data.grid_power || 0,
      loadPower: data.load_power || 0,
      timestamp: data.timestamp,
      efficiency: this.calculateEfficiency(data)
    };
  }
  
  formatEnergyData(data, timePeriod) {
    if (timePeriod === 'now') {
      return {
        current: data.data[0].fv_power || 0,
        unit: 'watts',
        timestamp: data.data[0].timestamp
      };
    } else {
      return {
        total: data.total_generation || 0,
        average: data.average_generation || 0,
        peak: data.peak_generation || 0,
        unit: 'watts',
        period: timePeriod
      };
    }
  }
  
  formatEfficiencyData(data, analysisType) {
    const efficiency = data.efficiency_metrics || {};
    
    return {
      average: efficiency.average || 0,
      peak: efficiency.peak || 0,
      low: efficiency.low || 0,
      trend: data.trends?.efficiency_trend || 'stable',
      period: analysisType,
      recommendations: this.generateEfficiencyRecommendations(efficiency)
    };
  }
  
  getBatteryStatusText(level, power) {
    if (level > 80) return 'excelente';
    if (level > 50) return 'boa';
    if (level > 20) return 'baixa';
    return 'crítica';
  }
  
  calculateEfficiency(data) {
    const fvPower = data.fv_power || 0;
    const loadPower = data.load_power || 0;
    
    if (fvPower === 0) return 0;
    return Math.round((loadPower / fvPower) * 100);
  }
  
  generateEfficiencyRecommendations(efficiency) {
    const recommendations = [];
    
    if (efficiency.average < 60) {
      recommendations.push('Considere verificar a limpeza dos painéis solares');
    }
    
    if (efficiency.peak < 80) {
      recommendations.push('Verifique se há sombreamento nos painéis');
    }
    
    if (efficiency.average > 85) {
      recommendations.push('Excelente! Seu sistema está operando com alta eficiência');
    }
    
    return recommendations;
  }
}

module.exports = new GoodWeService();
```

### 2. Análises e Relatórios

```javascript
// services/analytics-service.js
const { goodweClient } = require('../utils/http-client');

class AnalyticsService {
  async getDailyReport(date = null) {
    try {
      const targetDate = date || new Date().toISOString().split('T')[0];
      
      const response = await goodweClient.client.get('/analytics/daily', {
        params: { date: targetDate }
      });
      
      return this.formatDailyReport(response.data, targetDate);
    } catch (error) {
      console.error('Erro no relatório diário:', error);
      throw new Error('Não foi possível gerar relatório diário');
    }
  }
  
  async getWeeklyReport() {
    try {
      const response = await goodweClient.client.get('/analytics/trends', {
        params: { period: 'weekly' }
      });
      
      return this.formatWeeklyReport(response.data);
    } catch (error) {
      console.error('Erro no relatório semanal:', error);
      throw new Error('Não foi possível gerar relatório semanal');
    }
  }
  
  async getMonthlyReport() {
    try {
      const response = await goodweClient.client.get('/analytics/trends', {
        params: { period: 'monthly' }
      });
      
      return this.formatMonthlyReport(response.data);
    } catch (error) {
      console.error('Erro no relatório mensal:', error);
      throw new Error('Não foi possível gerar relatório mensal');
    }
  }
  
  formatDailyReport(data, date) {
    const summary = data.summary || {};
    const generation = data.generation || {};
    const consumption = data.consumption || {};
    const efficiency = data.efficiency || {};
    
    return {
      date,
      totalGeneration: generation.total || 0,
      totalConsumption: consumption.total || 0,
      peakGeneration: generation.peak || 0,
      peakConsumption: consumption.peak || 0,
      averageEfficiency: efficiency.average || 0,
      savings: this.calculateSavings(generation.total, consumption.total),
      recommendations: this.generateDailyRecommendations(data)
    };
  }
  
  calculateSavings(generation, consumption) {
    const gridPrice = 0.65; // R$ por kWh
    const solarPrice = 0.20; // R$ por kWh
    
    const solarUsage = Math.min(generation, consumption);
    const gridUsage = Math.max(0, consumption - generation);
    
    const solarCost = solarUsage * solarPrice;
    const gridCost = gridUsage * gridPrice;
    const totalSavings = gridCost - solarCost;
    
    return {
      solarUsage,
      gridUsage,
      totalSavings: Math.round(totalSavings * 100) / 100,
      savingsPercentage: Math.round((totalSavings / gridCost) * 100)
    };
  }
}

module.exports = new AnalyticsService();
```

## 🤖 Integração com API Machine Learning

### 1. Predições Climáticas

```javascript
// services/ml-service.js
const { mlClient } = require('../utils/http-client');

class MLService {
  async getWeatherPrediction(weatherData = null) {
    try {
      // Se não fornecido, obter dados climáticos atuais
      if (!weatherData) {
        weatherData = await this.getCurrentWeatherData();
      }
      
      const response = await mlClient.client.post('/predict', weatherData);
      
      return this.formatPrediction(response.data);
    } catch (error) {
      console.error('Erro na predição climática:', error);
      throw new Error('Não foi possível obter predição climática');
    }
  }
  
  async getBatchPrediction(weatherDataArray) {
    try {
      const response = await mlClient.client.post('/predict-batch', weatherDataArray);
      
      return response.data.predictions.map(prediction => 
        this.formatPrediction(prediction)
      );
    } catch (error) {
      console.error('Erro na predição em lote:', error);
      throw new Error('Não foi possível obter predições em lote');
    }
  }
  
  async getModelInfo() {
    try {
      const response = await mlClient.client.get('/model-info');
      
      return {
        modelType: response.data.model_type,
        features: response.data.features,
        accuracy: response.data.accuracy,
        description: response.data.description
      };
    } catch (error) {
      console.error('Erro ao obter informações do modelo:', error);
      throw new Error('Não foi possível obter informações do modelo');
    }
  }
  
  async getCurrentWeatherData() {
    // Integração com serviço de clima
    const { weatherClient } = require('../utils/http-client');
    
    try {
      const response = await weatherClient.client.get('/weather', {
        params: {
          q: 'São Paulo,BR',
          appid: process.env.WEATHER_API_KEY,
          units: 'metric'
        }
      });
      
      const weather = response.data;
      
      return {
        temperatura_celsius: weather.main.temp,
        umidade_pct: weather.main.humidity,
        precipitacao_mm_h: weather.rain ? weather.rain['1h'] || 0 : 0,
        vento_kmh: weather.wind.speed * 3.6,
        pressao_hpa: weather.main.pressure
      };
    } catch (error) {
      console.error('Erro ao obter dados climáticos:', error);
      
      // Dados padrão em caso de erro
      return {
        temperatura_celsius: 25.0,
        umidade_pct: 65.0,
        precipitacao_mm_h: 0.0,
        vento_kmh: 10.0,
        pressao_hpa: 1013.0
      };
    }
  }
  
  formatPrediction(prediction) {
    return {
      willOutage: prediction.queda_energia,
      probability: prediction.probabilidade,
      probabilityPct: prediction.probabilidade_pct,
      riskLevel: prediction.nivel_risco,
      inputData: prediction.dados_entrada,
      recommendations: this.generateWeatherRecommendations(prediction)
    };
  }
  
  generateWeatherRecommendations(prediction) {
    const recommendations = [];
    const { riskLevel, probability } = prediction;
    
    if (riskLevel === 'Crítico') {
      recommendations.push('Ative o modo de emergência imediatamente');
      recommendations.push('Verifique se todos os equipamentos estão seguros');
      recommendations.push('Considere desligar equipamentos não essenciais');
    } else if (riskLevel === 'Alto') {
      recommendations.push('Prepare-se para possível interrupção');
      recommendations.push('Verifique o sistema de backup');
    } else if (riskLevel === 'Médio') {
      recommendations.push('Monitore as condições climáticas');
    } else {
      recommendations.push('Condições normais, sistema operando normalmente');
    }
    
    return recommendations;
  }
}

module.exports = new MLService();
```

## 🌐 Integração com APIs Externas

### 1. Serviços de Clima

```javascript
// services/weather-service.js
const { weatherClient } = require('../utils/http-client');

class WeatherService {
  async getCurrentWeather(location) {
    try {
      const response = await weatherClient.client.get('/weather', {
        params: {
          q: location,
          appid: process.env.WEATHER_API_KEY,
          units: 'metric',
          lang: 'pt_br'
        }
      });
      
      return this.formatWeatherData(response.data);
    } catch (error) {
      console.error('Erro ao obter dados climáticos:', error);
      throw new Error('Não foi possível obter dados climáticos');
    }
  }
  
  async getWeatherForecast(location, days = 7) {
    try {
      const response = await weatherClient.client.get('/forecast', {
        params: {
          q: location,
          appid: process.env.WEATHER_API_KEY,
          units: 'metric',
          lang: 'pt_br',
          cnt: days * 8 // 8 previsões por dia
        }
      });
      
      return this.formatForecastData(response.data);
    } catch (error) {
      console.error('Erro na previsão do tempo:', error);
      throw new Error('Não foi possível obter previsão do tempo');
    }
  }
  
  formatWeatherData(data) {
    return {
      temperature: Math.round(data.main.temp),
      humidity: data.main.humidity,
      pressure: data.main.pressure,
      windSpeed: Math.round(data.wind.speed * 3.6), // m/s para km/h
      windDirection: data.wind.deg,
      description: data.weather[0].description,
      icon: data.weather[0].icon,
      visibility: data.visibility / 1000, // m para km
      cloudiness: data.clouds.all,
      rain: data.rain ? data.rain['1h'] || 0 : 0,
      snow: data.snow ? data.snow['1h'] || 0 : 0
    };
  }
  
  formatForecastData(data) {
    return data.list.map(item => ({
      datetime: new Date(item.dt * 1000),
      temperature: Math.round(item.main.temp),
      humidity: item.main.humidity,
      pressure: item.main.pressure,
      windSpeed: Math.round(item.wind.speed * 3.6),
      description: item.weather[0].description,
      icon: item.weather[0].icon,
      rain: item.rain ? item.rain['3h'] || 0 : 0
    }));
  }
}

module.exports = new WeatherService();
```

### 2. Concessionárias de Energia

```javascript
// services/utility-service.js
const { utilityClient } = require('../utils/http-client');

class UtilityService {
  async getElectricityRates(region) {
    try {
      const response = await utilityClient.client.get('/rates', {
        params: { region }
      });
      
      return this.formatRatesData(response.data);
    } catch (error) {
      console.error('Erro ao obter tarifas:', error);
      throw new Error('Não foi possível obter tarifas de energia');
    }
  }
  
  async getTimeOfUseRates() {
    try {
      const response = await utilityClient.client.get('/time-of-use');
      
      return this.formatTimeOfUseData(response.data);
    } catch (error) {
      console.error('Erro ao obter tarifas por horário:', error);
      throw new Error('Não foi possível obter tarifas por horário');
    }
  }
  
  async calculateSavings(solarGeneration, consumption, rates) {
    const savings = {
      total: 0,
      breakdown: {
        peak: 0,
        offPeak: 0,
        shoulder: 0
      },
      details: []
    };
    
    for (const hour of consumption.hours) {
      const rate = this.getRateForHour(hour, rates);
      const solarUsed = Math.min(solarGeneration[hour], consumption[hour]);
      const gridUsed = Math.max(0, consumption[hour] - solarUsed);
      
      const hourSavings = gridUsed * rate.price;
      savings.total += hourSavings;
      savings.breakdown[rate.period] += hourSavings;
      
      savings.details.push({
        hour,
        solarUsed,
        gridUsed,
        rate: rate.price,
        savings: hourSavings
      });
    }
    
    return savings;
  }
  
  formatRatesData(data) {
    return {
      baseRate: data.base_rate,
      distributionRate: data.distribution_rate,
      transmissionRate: data.transmission_rate,
      taxes: data.taxes,
      totalRate: data.total_rate,
      currency: data.currency,
      unit: data.unit
    };
  }
  
  formatTimeOfUseData(data) {
    return data.periods.map(period => ({
      name: period.name,
      startTime: period.start_time,
      endTime: period.end_time,
      rate: period.rate,
      days: period.days
    }));
  }
  
  getRateForHour(hour, rates) {
    // Lógica para determinar a tarifa baseada no horário
    if (hour >= 6 && hour < 18) {
      return { period: 'peak', price: rates.peakRate };
    } else if (hour >= 18 && hour < 22) {
      return { period: 'shoulder', price: rates.shoulderRate };
    } else {
      return { period: 'offPeak', price: rates.offPeakRate };
    }
  }
}

module.exports = new UtilityService();
```

## 🔄 Cache e Otimização

### 1. Sistema de Cache

```javascript
// utils/cache-manager.js
class CacheManager {
  constructor() {
    this.cache = new Map();
    this.defaultTTL = 300000; // 5 minutos
  }
  
  get(key) {
    const item = this.cache.get(key);
    
    if (!item) {
      return null;
    }
    
    if (Date.now() - item.timestamp > item.ttl) {
      this.cache.delete(key);
      return null;
    }
    
    return item.data;
  }
  
  set(key, data, ttl = this.defaultTTL) {
    this.cache.set(key, {
      data,
      timestamp: Date.now(),
      ttl
    });
  }
  
  delete(key) {
    this.cache.delete(key);
  }
  
  clear() {
    this.cache.clear();
  }
  
  // Cache específico para diferentes tipos de dados
  getSystemData() {
    return this.get('system_data');
  }
  
  setSystemData(data) {
    this.set('system_data', data, 60000); // 1 minuto
  }
  
  getWeatherData() {
    return this.get('weather_data');
  }
  
  setWeatherData(data) {
    this.set('weather_data', data, 300000); // 5 minutos
  }
  
  getPredictionData() {
    return this.get('prediction_data');
  }
  
  setPredictionData(data) {
    this.set('prediction_data', data, 600000); // 10 minutos
  }
}

module.exports = new CacheManager();
```

### 2. Middleware de Cache

```javascript
// middleware/cache-middleware.js
const cacheManager = require('../utils/cache-manager');

const cacheMiddleware = (ttl = 300000) => {
  return async (req, res, next) => {
    const cacheKey = `${req.method}:${req.url}:${JSON.stringify(req.query)}`;
    
    // Tentar obter do cache
    const cachedData = cacheManager.get(cacheKey);
    if (cachedData) {
      console.log(`[CACHE] Hit for ${cacheKey}`);
      return res.json(cachedData);
    }
    
    // Interceptar resposta para cachear
    const originalJson = res.json;
    res.json = function(data) {
      cacheManager.set(cacheKey, data, ttl);
      console.log(`[CACHE] Set for ${cacheKey}`);
      return originalJson.call(this, data);
    };
    
    next();
  };
};

module.exports = cacheMiddleware;
```

## 🚨 Tratamento de Erros

### 1. Error Handler Centralizado

```javascript
// utils/error-handler.js
class ErrorHandler {
  static handle(error, context = {}) {
    console.error(`[ERROR] ${context.operation || 'Unknown'}:`, error);
    
    const errorInfo = {
      message: error.message,
      code: error.code,
      context,
      timestamp: new Date().toISOString(),
      stack: error.stack
    };
    
    // Log para CloudWatch
    this.logToCloudWatch(errorInfo);
    
    // Retornar erro amigável para o usuário
    return this.getUserFriendlyError(error);
  }
  
  static getUserFriendlyError(error) {
    if (error.code === 'ECONNREFUSED') {
      return 'Serviço temporariamente indisponível. Tente novamente em alguns minutos.';
    }
    
    if (error.code === 'ETIMEDOUT') {
      return 'A requisição demorou muito para responder. Tente novamente.';
    }
    
    if (error.response?.status === 401) {
      return 'Erro de autenticação. Verifique as configurações.';
    }
    
    if (error.response?.status === 404) {
      return 'Recurso não encontrado.';
    }
    
    if (error.response?.status >= 500) {
      return 'Erro interno do servidor. Tente novamente mais tarde.';
    }
    
    return 'Ocorreu um erro inesperado. Tente novamente.';
  }
  
  static logToCloudWatch(errorInfo) {
    // Implementar logging para CloudWatch
    console.log(JSON.stringify({
      level: 'ERROR',
      ...errorInfo
    }));
  }
}

module.exports = ErrorHandler;
```

### 2. Retry com Circuit Breaker

```javascript
// utils/circuit-breaker.js
class CircuitBreaker {
  constructor(threshold = 5, timeout = 60000) {
    this.threshold = threshold;
    this.timeout = timeout;
    this.failureCount = 0;
    this.lastFailureTime = null;
    this.state = 'CLOSED'; // CLOSED, OPEN, HALF_OPEN
  }
  
  async execute(operation) {
    if (this.state === 'OPEN') {
      if (Date.now() - this.lastFailureTime > this.timeout) {
        this.state = 'HALF_OPEN';
      } else {
        throw new Error('Circuit breaker is OPEN');
      }
    }
    
    try {
      const result = await operation();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }
  
  onSuccess() {
    this.failureCount = 0;
    this.state = 'CLOSED';
  }
  
  onFailure() {
    this.failureCount++;
    this.lastFailureTime = Date.now();
    
    if (this.failureCount >= this.threshold) {
      this.state = 'OPEN';
    }
  }
}

module.exports = CircuitBreaker;
```

## 📊 Monitoramento e Métricas

### 1. Métricas de API

```javascript
// utils/metrics-collector.js
class MetricsCollector {
  constructor() {
    this.metrics = {
      requests: 0,
      errors: 0,
      responseTimes: [],
      cacheHits: 0,
      cacheMisses: 0
    };
  }
  
  recordRequest(duration, success = true) {
    this.metrics.requests++;
    
    if (!success) {
      this.metrics.errors++;
    }
    
    this.metrics.responseTimes.push(duration);
    
    // Manter apenas os últimos 100 tempos de resposta
    if (this.metrics.responseTimes.length > 100) {
      this.metrics.responseTimes.shift();
    }
  }
  
  recordCacheHit() {
    this.metrics.cacheHits++;
  }
  
  recordCacheMiss() {
    this.metrics.cacheMisses++;
  }
  
  getMetrics() {
    const avgResponseTime = this.metrics.responseTimes.length > 0
      ? this.metrics.responseTimes.reduce((a, b) => a + b, 0) / this.metrics.responseTimes.length
      : 0;
    
    const errorRate = this.metrics.requests > 0
      ? (this.metrics.errors / this.metrics.requests) * 100
      : 0;
    
    const cacheHitRate = (this.metrics.cacheHits + this.metrics.cacheMisses) > 0
      ? (this.metrics.cacheHits / (this.metrics.cacheHits + this.metrics.cacheMisses)) * 100
      : 0;
    
    return {
      ...this.metrics,
      avgResponseTime: Math.round(avgResponseTime),
      errorRate: Math.round(errorRate * 100) / 100,
      cacheHitRate: Math.round(cacheHitRate * 100) / 100
    };
  }
}

module.exports = new MetricsCollector();
```

---

**Próximo**: [Smart Home Integration](./02-smart-home-integration.md)
