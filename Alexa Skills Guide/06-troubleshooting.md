# Troubleshooting - Alexa Skills GoodWe

## 📋 Visão Geral

Este guia fornece soluções para problemas comuns, debugging avançado e procedimentos de recuperação para a Alexa Skill GoodWe.

## 🔍 Problemas Comuns e Soluções

### 1. Problemas de Conectividade

#### Problema: "Desculpe, não consegui obter o status do sistema"

**Possíveis Causas:**
- API GoodWe indisponível
- Timeout na requisição
- Credenciais inválidas
- Problemas de rede

**Soluções:**

```javascript
// 1. Verificar status da API
async function checkAPIStatus() {
  try {
    const response = await axios.get(`${API_CONFIG.goodwe.baseUrl}/health`, {
      timeout: 5000
    });
    console.log('API Status:', response.data);
    return response.data.status === 'OK';
  } catch (error) {
    console.error('API Error:', error.message);
    return false;
  }
}

// 2. Implementar retry com backoff
async function apiCallWithRetry(url, options = {}, maxRetries = 3) {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      const response = await axios.get(url, {
        ...options,
        timeout: 5000 * attempt // Aumentar timeout a cada tentativa
      });
      return response;
    } catch (error) {
      console.error(`Tentativa ${attempt} falhou:`, error.message);
      
      if (attempt === maxRetries) {
        throw error;
      }
      
      // Backoff exponencial
      await new Promise(resolve => setTimeout(resolve, 1000 * Math.pow(2, attempt - 1)));
    }
  }
}

// 3. Fallback para dados em cache
async function getSystemStatusWithFallback() {
  try {
    const data = await apiCallWithRetry(`${API_CONFIG.goodwe.baseUrl}/data/paginated?limit=1`);
    return data.data[0];
  } catch (error) {
    console.error('API falhou, usando cache:', error);
    
    // Usar dados em cache se disponível
    const cachedData = await getCachedData('system_status');
    if (cachedData) {
      return cachedData;
    }
    
    throw new Error('Sistema temporariamente indisponível');
  }
}
```

#### Problema: "Erro de autenticação"

**Soluções:**

```javascript
// Verificar e renovar token
async function refreshAPIToken() {
  try {
    const response = await axios.post(`${API_CONFIG.goodwe.baseUrl}/auth/refresh`, {
      refresh_token: process.env.GOODWE_REFRESH_TOKEN
    });
    
    process.env.GOODWE_API_KEY = response.data.access_token;
    return true;
  } catch (error) {
    console.error('Erro ao renovar token:', error);
    return false;
  }
}

// Middleware de autenticação
const authMiddleware = async (req, res, next) => {
  try {
    const response = await axios.get(`${API_CONFIG.goodwe.baseUrl}/verify`, {
      headers: {
        'Authorization': `Bearer ${process.env.GOODWE_API_KEY}`
      }
    });
    
    if (response.status === 401) {
      const refreshed = await refreshAPIToken();
      if (!refreshed) {
        throw new Error('Falha na autenticação');
      }
    }
    
    next();
  } catch (error) {
    console.error('Erro de autenticação:', error);
    throw error;
  }
};
```

### 2. Problemas de Performance

#### Problema: Resposta lenta da skill

**Diagnóstico:**

```javascript
// Middleware de monitoramento de performance
const performanceMiddleware = (handlerInput, next) => {
  const startTime = Date.now();
  
  return next(handlerInput).then(response => {
    const duration = Date.now() - startTime;
    
    // Log de performance
    console.log(`Performance: ${duration}ms`);
    
    // Enviar métrica para CloudWatch
    sendMetric('ResponseTime', duration, 'Milliseconds');
    
    // Alertar se muito lento
    if (duration > 5000) {
      console.warn(`Resposta lenta detectada: ${duration}ms`);
      sendAlert('SlowResponse', { duration });
    }
    
    return response;
  });
};

// Otimização de cache
const cacheManager = {
  cache: new Map(),
  ttl: 300000, // 5 minutos
  
  get(key) {
    const item = this.cache.get(key);
    if (item && Date.now() - item.timestamp < this.ttl) {
      return item.data;
    }
    this.cache.delete(key);
    return null;
  },
  
  set(key, data) {
    this.cache.set(key, {
      data,
      timestamp: Date.now()
    });
  }
};
```

**Soluções:**

```javascript
// 1. Implementar cache inteligente
async function getCachedSystemData() {
  const cacheKey = 'system_data';
  let data = cacheManager.get(cacheKey);
  
  if (!data) {
    data = await getSystemData();
    cacheManager.set(cacheKey, data);
  }
  
  return data;
}

// 2. Paralelizar requisições
async function getMultipleData() {
  const [systemData, weatherData, predictions] = await Promise.all([
    getSystemData(),
    getWeatherData(),
    getPredictions()
  ]);
  
  return { systemData, weatherData, predictions };
}

// 3. Otimizar queries de banco
async function getOptimizedData() {
  const query = `
    SELECT fv_power, soc_percentage, battery_power, grid_power, load_power, timestamp
    FROM system_data 
    WHERE timestamp >= NOW() - INTERVAL '1 hour'
    ORDER BY timestamp DESC 
    LIMIT 1
  `;
  
  const result = await db.query(query);
  return result.rows[0];
}
```

### 3. Problemas de Intents

#### Problema: Intent não reconhecido

**Diagnóstico:**

```javascript
// Debug de intents
const debugIntent = (handlerInput) => {
  const request = handlerInput.requestEnvelope.request;
  
  console.log('Request Type:', request.type);
  console.log('Intent Name:', request.intent?.name);
  console.log('Slots:', JSON.stringify(request.intent?.slots, null, 2));
  console.log('Raw Request:', JSON.stringify(request, null, 2));
};

// Handler de fallback
const FallbackHandler = {
  canHandle(handlerInput) {
    return Alexa.getRequestType(handlerInput.requestEnvelope) === 'IntentRequest';
  },
  
  handle(handlerInput) {
    debugIntent(handlerInput);
    
    const speechText = 'Desculpe, não entendi. Você pode perguntar sobre o status do sistema, geração de energia ou nível da bateria.';
    
    return handlerInput.responseBuilder
      .speak(speechText)
      .reprompt('Como posso ajudar?')
      .getResponse();
  }
};
```

**Soluções:**

```javascript
// 1. Melhorar utterances
const improvedUtterances = [
  "qual o status do sistema",
  "como está o sistema solar",
  "status do sistema solar",
  "sistema solar status",
  "verificar sistema",
  "checar sistema"
];

// 2. Implementar fuzzy matching
const fuzzyMatch = (input, patterns) => {
  const similarity = (a, b) => {
    const longer = a.length > b.length ? a : b;
    const shorter = a.length > b.length ? b : a;
    
    if (longer.length === 0) return 1.0;
    
    const editDistance = levenshteinDistance(longer, shorter);
    return (longer.length - editDistance) / longer.length;
  };
  
  let bestMatch = null;
  let bestScore = 0;
  
  for (const pattern of patterns) {
    const score = similarity(input.toLowerCase(), pattern.toLowerCase());
    if (score > bestScore && score > 0.7) {
      bestScore = score;
      bestMatch = pattern;
    }
  }
  
  return bestMatch;
};

// 3. Log de intents para análise
const logIntent = (intentName, slots, userId) => {
  console.log(JSON.stringify({
    timestamp: new Date().toISOString(),
    intent: intentName,
    slots,
    userId,
    type: 'intent_usage'
  }));
};
```

### 4. Problemas de Dados

#### Problema: Dados inconsistentes ou inválidos

**Validação de Dados:**

```javascript
// Validador de dados do sistema
const validateSystemData = (data) => {
  const errors = [];
  
  // Validar fv_power
  if (typeof data.fv_power !== 'number' || data.fv_power < 0 || data.fv_power > 10000) {
    errors.push('fv_power inválido');
  }
  
  // Validar soc_percentage
  if (typeof data.soc_percentage !== 'number' || data.soc_percentage < 0 || data.soc_percentage > 100) {
    errors.push('soc_percentage inválido');
  }
  
  // Validar timestamp
  if (!data.timestamp || isNaN(new Date(data.timestamp).getTime())) {
    errors.push('timestamp inválido');
  }
  
  return {
    isValid: errors.length === 0,
    errors
  };
};

// Sanitização de dados
const sanitizeData = (data) => {
  return {
    fv_power: Math.max(0, Math.min(10000, data.fv_power || 0)),
    soc_percentage: Math.max(0, Math.min(100, data.soc_percentage || 0)),
    battery_power: Math.max(-5000, Math.min(5000, data.battery_power || 0)),
    grid_power: Math.max(-5000, Math.min(5000, data.grid_power || 0)),
    load_power: Math.max(0, Math.min(10000, data.load_power || 0)),
    timestamp: data.timestamp || new Date().toISOString()
  };
};
```

**Tratamento de Dados Corrompidos:**

```javascript
// Detecção de anomalias
const detectAnomalies = (data) => {
  const anomalies = [];
  
  // Verificar se fv_power é muito alto para o horário
  const hour = new Date(data.timestamp).getHours();
  if (hour < 6 || hour > 18) {
    if (data.fv_power > 100) {
      anomalies.push('Geração solar alta durante a noite');
    }
  }
  
  // Verificar se soc_percentage mudou drasticamente
  const previousData = getPreviousData();
  if (previousData) {
    const socChange = Math.abs(data.soc_percentage - previousData.soc_percentage);
    if (socChange > 20) {
      anomalies.push('Mudança drástica no nível da bateria');
    }
  }
  
  return anomalies;
};

// Correção automática de dados
const correctData = (data, anomalies) => {
  let correctedData = { ...data };
  
  for (const anomaly of anomalies) {
    if (anomaly.includes('Geração solar alta durante a noite')) {
      correctedData.fv_power = 0;
    }
    
    if (anomaly.includes('Mudança drástica no nível da bateria')) {
      // Usar média móvel para suavizar
      const previousData = getPreviousData();
      correctedData.soc_percentage = (data.soc_percentage + previousData.soc_percentage) / 2;
    }
  }
  
  return correctedData;
};
```

## 🔧 Debugging Avançado

### 1. Logging Estruturado

```javascript
// Configuração de logging avançado
const winston = require('winston');

const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  transports: [
    new winston.transports.Console({
      format: winston.format.combine(
        winston.format.colorize(),
        winston.format.simple()
      )
    }),
    new winston.transports.File({ 
      filename: 'error.log', 
      level: 'error',
      maxsize: 5242880, // 5MB
      maxFiles: 5
    }),
    new winston.transports.File({ 
      filename: 'combined.log',
      maxsize: 5242880,
      maxFiles: 5
    })
  ]
});

// Logging contextual
const logContext = (message, context = {}) => {
  logger.info(message, {
    ...context,
    timestamp: new Date().toISOString(),
    requestId: context.requestId || generateRequestId()
  });
};
```

### 2. Tracing de Requisições

```javascript
// Sistema de tracing
const traceManager = {
  traces: new Map(),
  
  startTrace(requestId) {
    const trace = {
      requestId,
      startTime: Date.now(),
      steps: [],
      errors: []
    };
    
    this.traces.set(requestId, trace);
    return trace;
  },
  
  addStep(requestId, step, data = {}) {
    const trace = this.traces.get(requestId);
    if (trace) {
      trace.steps.push({
        step,
        timestamp: Date.now(),
        data
      });
    }
  },
  
  addError(requestId, error, context = {}) {
    const trace = this.traces.get(requestId);
    if (trace) {
      trace.errors.push({
        error: error.message,
        stack: error.stack,
        context,
        timestamp: Date.now()
      });
    }
  },
  
  endTrace(requestId) {
    const trace = this.traces.get(requestId);
    if (trace) {
      trace.endTime = Date.now();
      trace.duration = trace.endTime - trace.startTime;
      
      // Log do trace completo
      logger.info('Request trace completed', trace);
      
      this.traces.delete(requestId);
    }
  }
};

// Middleware de tracing
const tracingMiddleware = (handlerInput, next) => {
  const requestId = generateRequestId();
  traceManager.startTrace(requestId);
  
  return next(handlerInput).then(response => {
    traceManager.endTrace(requestId);
    return response;
  }).catch(error => {
    traceManager.addError(requestId, error);
    traceManager.endTrace(requestId);
    throw error;
  });
};
```

### 3. Monitoramento em Tempo Real

```javascript
// Dashboard de monitoramento
const monitoringDashboard = {
  metrics: {
    requests: 0,
    errors: 0,
    avgResponseTime: 0,
    activeUsers: new Set()
  },
  
  updateMetrics(request, response, duration) {
    this.metrics.requests++;
    
    if (response.statusCode >= 400) {
      this.metrics.errors++;
    }
    
    // Atualizar tempo médio de resposta
    this.metrics.avgResponseTime = 
      (this.metrics.avgResponseTime * (this.metrics.requests - 1) + duration) / 
      this.metrics.requests;
    
    // Adicionar usuário ativo
    if (request.userId) {
      this.metrics.activeUsers.add(request.userId);
    }
  },
  
  getHealthStatus() {
    const errorRate = this.metrics.errors / this.metrics.requests;
    
    return {
      status: errorRate > 0.1 ? 'unhealthy' : 'healthy',
      metrics: this.metrics,
      errorRate,
      uptime: process.uptime()
    };
  }
};
```

## 🚨 Procedimentos de Emergência

### 1. Falha Total do Sistema

```javascript
// Procedimento de recuperação de emergência
const emergencyRecovery = {
  async execute() {
    console.log('🚨 Iniciando recuperação de emergência...');
    
    try {
      // 1. Verificar status dos serviços
      const services = await this.checkServices();
      
      // 2. Ativar modo de degradação
      await this.activateDegradedMode();
      
      // 3. Notificar administradores
      await this.notifyAdministrators();
      
      // 4. Tentar recuperação automática
      const recovered = await this.attemptRecovery();
      
      if (recovered) {
        console.log('✅ Recuperação bem-sucedida');
        await this.notifyRecovery();
      } else {
        console.log('❌ Recuperação falhou, ativando procedimentos manuais');
        await this.activateManualProcedures();
      }
      
    } catch (error) {
      console.error('Erro na recuperação de emergência:', error);
      await this.activateManualProcedures();
    }
  },
  
  async checkServices() {
    const services = ['api-goodwe', 'api-ml', 'database', 'redis'];
    const status = {};
    
    for (const service of services) {
      try {
        const response = await axios.get(`${service}/health`, { timeout: 5000 });
        status[service] = response.status === 200 ? 'up' : 'down';
      } catch (error) {
        status[service] = 'down';
      }
    }
    
    return status;
  },
  
  async activateDegradedMode() {
    // Ativar respostas básicas sem integração com APIs
    process.env.DEGRADED_MODE = 'true';
    
    // Usar dados em cache
    await this.loadCachedData();
    
    console.log('Modo degradado ativado');
  }
};
```

### 2. Rollback de Emergência

```bash
#!/bin/bash
# emergency-rollback.sh

echo "🚨 Iniciando rollback de emergência..."

# 1. Parar skill atual
aws lambda update-function-configuration \
  --function-name GoodWeSolarAssistant \
  --environment Variables='{"EMERGENCY_MODE":"true"}'

# 2. Deploy da versão anterior
aws lambda update-function-code \
  --function-name GoodWeSolarAssistant \
  --s3-bucket goodwe-lambda-backups \
  --s3-key previous-version.zip

# 3. Verificar saúde
sleep 30
aws lambda invoke \
  --function-name GoodWeSolarAssistant \
  --payload '{"request":{"type":"LaunchRequest"}}' \
  response.json

# 4. Notificar equipe
curl -X POST -H 'Content-type: application/json' \
  --data '{"text":"🚨 Rollback de emergência executado"}' \
  $SLACK_WEBHOOK

echo "✅ Rollback concluído"
```

## 📊 Análise de Problemas

### 1. Análise de Logs

```javascript
// Analisador de logs
const logAnalyzer = {
  async analyzeErrors(timeRange = '1h') {
    const logs = await this.getLogs(timeRange);
    const errors = logs.filter(log => log.level === 'error');
    
    const analysis = {
      totalErrors: errors.length,
      errorTypes: this.groupBy(errors, 'error.type'),
      topErrors: this.getTopErrors(errors),
      trends: this.analyzeTrends(errors)
    };
    
    return analysis;
  },
  
  async analyzePerformance(timeRange = '1h') {
    const logs = await this.getLogs(timeRange);
    const performanceLogs = logs.filter(log => log.metric === 'ResponseTime');
    
    const analysis = {
      avgResponseTime: this.calculateAverage(performanceLogs, 'value'),
      p95ResponseTime: this.calculatePercentile(performanceLogs, 95),
      slowRequests: performanceLogs.filter(log => log.value > 5000)
    };
    
    return analysis;
  }
};
```

### 2. Alertas Inteligentes

```javascript
// Sistema de alertas inteligentes
const intelligentAlerts = {
  async processLogs(logs) {
    const patterns = await this.detectPatterns(logs);
    
    for (const pattern of patterns) {
      if (pattern.severity === 'critical') {
        await this.sendCriticalAlert(pattern);
      } else if (pattern.severity === 'warning') {
        await this.sendWarningAlert(pattern);
      }
    }
  },
  
  async detectPatterns(logs) {
    const patterns = [];
    
    // Detectar picos de erro
    const errorSpike = this.detectErrorSpike(logs);
    if (errorSpike) {
      patterns.push({
        type: 'error_spike',
        severity: 'critical',
        description: 'Pico de erros detectado',
        data: errorSpike
      });
    }
    
    // Detectar degradação de performance
    const performanceDegradation = this.detectPerformanceDegradation(logs);
    if (performanceDegradation) {
      patterns.push({
        type: 'performance_degradation',
        severity: 'warning',
        description: 'Degradação de performance detectada',
        data: performanceDegradation
      });
    }
    
    return patterns;
  }
};
```

## 🛠️ Ferramentas de Debug

### 1. Debug Mode

```javascript
// Modo de debug
const debugMode = {
  enabled: process.env.DEBUG === 'true',
  
  log(message, data = {}) {
    if (this.enabled) {
      console.log(`[DEBUG] ${message}`, JSON.stringify(data, null, 2));
    }
  },
  
  trace(functionName, args = {}) {
    if (this.enabled) {
      console.log(`[TRACE] ${functionName}`, args);
    }
  }
};

// Uso do debug mode
const processSystemData = (data) => {
  debugMode.trace('processSystemData', { input: data });
  
  const processed = validateSystemData(data);
  debugMode.log('Validation result', processed);
  
  return processed;
};
```

### 2. Teste de Conectividade

```javascript
// Teste de conectividade
const connectivityTest = {
  async runFullTest() {
    const results = {
      timestamp: new Date().toISOString(),
      tests: []
    };
    
    // Teste API GoodWe
    const goodweTest = await this.testAPI('GoodWe', API_CONFIG.goodwe.baseUrl);
    results.tests.push(goodweTest);
    
    // Teste API ML
    const mlTest = await this.testAPI('ML', API_CONFIG.ml.baseUrl);
    results.tests.push(mlTest);
    
    // Teste banco de dados
    const dbTest = await this.testDatabase();
    results.tests.push(dbTest);
    
    // Teste Redis
    const redisTest = await this.testRedis();
    results.tests.push(redisTest);
    
    return results;
  },
  
  async testAPI(name, url) {
    const startTime = Date.now();
    
    try {
      const response = await axios.get(`${url}/health`, { timeout: 5000 });
      const duration = Date.now() - startTime;
      
      return {
        name,
        status: 'success',
        duration,
        responseTime: response.data.uptime
      };
    } catch (error) {
      return {
        name,
        status: 'failed',
        error: error.message,
        duration: Date.now() - startTime
      };
    }
  }
};
```

---

**Próximo**: [Integração com APIs](./GoodWe_AlexaSkill_Integration/01-api-integration.md)
