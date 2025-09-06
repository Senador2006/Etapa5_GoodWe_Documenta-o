# Exemplos Práticos - Alexa Skills GoodWe

## 📋 Visão Geral

Este documento apresenta exemplos práticos e casos de uso reais para a integração da Alexa Skill com as APIs GoodWe, incluindo código completo e implementações passo a passo.

## 🎯 Exemplo 1: Monitoramento Básico do Sistema

### Cenário
Usuário quer verificar o status geral do sistema solar através do comando "Alexa, qual o status do sistema?"

### Implementação Completa

#### 1. Intent Handler
```javascript
const GetSystemStatusHandler = {
  canHandle(handlerInput) {
    return Alexa.getRequestType(handlerInput.requestEnvelope) === 'IntentRequest'
      && Alexa.getIntentName(handlerInput.requestEnvelope) === 'GetSystemStatus';
  },
  async handle(handlerInput) {
    try {
      // Obter dados do sistema
      const systemData = await getSystemData();
      
      // Processar e formatar dados
      const status = processSystemStatus(systemData);
      
      // Criar resposta com SSML para melhor experiência
      const speechText = createSystemStatusSSML(status);
      
      return handlerInput.responseBuilder
        .speak(speechText)
        .withSimpleCard('Status do Sistema Solar', formatCardContent(status))
        .getResponse();
    } catch (error) {
      console.error('Erro no GetSystemStatus:', error);
      return createErrorResponse(handlerInput, 'status do sistema');
    }
  }
};

// Função para obter dados do sistema
async function getSystemData() {
  const response = await axios.get(`${API_CONFIG.goodwe.baseUrl}/data/paginated?limit=1`, {
    timeout: API_CONFIG.goodwe.timeout,
    headers: {
      'Authorization': `Bearer ${process.env.GOODWE_API_KEY}`
    }
  });
  
  if (!response.data.success) {
    throw new Error('Falha ao obter dados do sistema');
  }
  
  return response.data.data[0];
}

// Processar status do sistema
function processSystemStatus(data) {
  const now = new Date();
  const lastUpdate = new Date(data.timestamp);
  const timeDiff = now - lastUpdate;
  const minutesAgo = Math.floor(timeDiff / (1000 * 60));
  
  return {
    fvPower: data.fv_power || 0,
    soc: data.soc_percentage || 0,
    batteryPower: data.battery_power || 0,
    gridPower: data.grid_power || 0,
    loadPower: data.load_power || 0,
    lastUpdate: minutesAgo,
    isOnline: minutesAgo < 5,
    efficiency: calculateEfficiency(data)
  };
}

// Calcular eficiência do sistema
function calculateEfficiency(data) {
  const fvPower = data.fv_power || 0;
  const loadPower = data.load_power || 0;
  
  if (fvPower === 0) return 0;
  return Math.round((loadPower / fvPower) * 100);
}

// Criar resposta SSML
function createSystemStatusSSML(status) {
  const { fvPower, soc, batteryPower, loadPower, lastUpdate, isOnline, efficiency } = status;
  
  let speechText = `<speak>`;
  
  // Status de conectividade
  if (!isOnline) {
    speechText += `<prosody rate="slow">Atenção! O sistema está offline há ${lastUpdate} minutos.</prosody> `;
  } else {
    speechText += `Seu sistema solar está funcionando normalmente. `;
  }
  
  // Geração de energia
  if (fvPower > 0) {
    speechText += `A geração atual é de <say-as interpret-as="number">${fvPower}</say-as> watts. `;
  } else {
    speechText += `Não há geração de energia no momento. `;
  }
  
  // Status da bateria
  if (soc > 80) {
    speechText += `A bateria está com <say-as interpret-as="number">${soc}</say-as> por cento de carga, status excelente. `;
  } else if (soc > 50) {
    speechText += `A bateria está com <say-as interpret-as="number">${soc}</say-as> por cento de carga, status bom. `;
  } else if (soc > 20) {
    speechText += `A bateria está com <say-as interpret-as="number">${soc}</say-as> por cento de carga, status baixo. `;
  } else {
    speechText += `<prosody rate="slow" pitch="high">Atenção! A bateria está com apenas <say-as interpret-as="number">${soc}</say-as> por cento de carga.</prosody> `;
  }
  
  // Consumo de energia
  speechText += `O consumo atual é de <say-as interpret-as="number">${loadPower}</say-as> watts. `;
  
  // Eficiência
  if (efficiency > 0) {
    speechText += `A eficiência atual é de <say-as interpret-as="number">${efficiency}</say-as> por cento.`;
  }
  
  speechText += `</speak>`;
  
  return speechText;
}

// Formatar conteúdo do card
function formatCardContent(status) {
  return `🔋 Bateria: ${status.soc}% (${status.batteryPower > 0 ? 'Carregando' : 'Descarregando'})
⚡ Geração: ${status.fvPower}W
🏠 Consumo: ${status.loadPower}W
📊 Eficiência: ${status.efficiency}%
🔄 Última atualização: ${status.lastUpdate} min atrás
${status.isOnline ? '🟢 Online' : '🔴 Offline'}`;
}
```

### Teste do Exemplo
```bash
# Simular comando de predição de quedas
ask simulate --text "risco de queda de energia" --locale pt-BR

# Resposta esperada para risco baixo
"Condições atuais: temperatura de 25 graus, umidade de 65% e ventos de 30 quilômetros por hora. Risco baixo de queda de energia. Probabilidade de 21.6%. Recomendações de prevenção: ✅ RISCO BAIXO: Sistema operando normalmente. 🔋 Bateria pode ser mantida em modo econômico. 💡 Uso normal de energia é seguro. Sistema operando normalmente. Continue monitorando."

# Resposta esperada para risco crítico
"Condições atuais: temperatura de 35 graus, umidade de 80% e ventos de 75 quilômetros por hora. ALERTA! Risco crítico de queda de energia. Probabilidade de 89.3%. Recomendações de prevenção: 🚨 RISCO CRÍTICO: Ative o modo de emergência imediatamente. 🔋 Carregue completamente a bateria do sistema solar. 💡 Desligue equipamentos não essenciais para economizar energia. Dicas adicionais: Mantenha um gerador de emergência carregado. Tenha um plano de evacuação se necessário. Ação imediata necessária! Ative o modo de emergência agora."
```

## 🎯 Exemplo 2: Análise de Eficiência com Gráficos

### Cenário
Usuário solicita análise de eficiência semanal com visualização de dados.

### Implementação

#### 1. Intent Handler para Análise
```javascript
const GetEfficiencyAnalysisHandler = {
  canHandle(handlerInput) {
    return Alexa.getRequestType(handlerInput.requestEnvelope) === 'IntentRequest'
      && Alexa.getIntentName(handlerInput.requestEnvelope) === 'GetEfficiencyAnalysis';
  },
  async handle(handlerInput) {
    try {
      const analysisType = Alexa.getSlotValue(handlerInput.requestEnvelope, 'AnalysisType') || 'daily';
      
      // Obter dados de análise
      const analysisData = await getEfficiencyAnalysis(analysisType);
      
      // Processar análise
      const analysis = processEfficiencyAnalysis(analysisData, analysisType);
      
      // Criar resposta com card visual
      const speechText = createEfficiencySSML(analysis);
      
      return handlerInput.responseBuilder
        .speak(speechText)
        .withStandardCard(
          `Análise de Eficiência ${analysisType}`,
          formatEfficiencyCard(analysis),
          analysis.chartUrl
        )
        .getResponse();
    } catch (error) {
      console.error('Erro na análise de eficiência:', error);
      return createErrorResponse(handlerInput, 'análise de eficiência');
    }
  }
};

// Obter dados de análise
async function getEfficiencyAnalysis(analysisType) {
  const endpoint = analysisType === 'daily' 
    ? `${API_CONFIG.goodwe.baseUrl}/analytics/hourly`
    : `${API_CONFIG.goodwe.baseUrl}/analytics/trends`;
    
  const response = await axios.get(endpoint, {
    timeout: API_CONFIG.goodwe.timeout,
    params: {
      period: analysisType,
      include_charts: true
    }
  });
  
  return response.data;
}

// Processar análise de eficiência
function processEfficiencyAnalysis(data, analysisType) {
  const efficiency = data.efficiency_metrics || {};
  const trends = data.trends || {};
  
  return {
    averageEfficiency: efficiency.average || 0,
    peakEfficiency: efficiency.peak || 0,
    lowEfficiency: efficiency.low || 0,
    trend: trends.efficiency_trend || 'stable',
    recommendations: generateRecommendations(efficiency),
    chartUrl: data.chart_url || null,
    period: analysisType
  };
}

// Gerar recomendações
function generateRecommendations(efficiency) {
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

// Criar resposta SSML
function createEfficiencySSML(analysis) {
  const { averageEfficiency, peakEfficiency, trend, recommendations, period } = analysis;
  
  let speechText = `<speak>`;
  
  speechText += `Análise de eficiência ${period}: `;
  speechText += `A eficiência média é de <say-as interpret-as="number">${averageEfficiency}</say-as> por cento. `;
  speechText += `O pico de eficiência foi de <say-as interpret-as="number">${peakEfficiency}</say-as> por cento. `;
  
  // Tendência
  switch (trend) {
    case 'increasing':
      speechText += `A tendência é de melhoria. `;
      break;
    case 'decreasing':
      speechText += `<prosody rate="slow">Atenção! A tendência é de queda na eficiência.</prosody> `;
      break;
    case 'stable':
      speechText += `A eficiência está estável. `;
      break;
  }
  
  // Recomendações
  if (recommendations.length > 0) {
    speechText += `Recomendações: ${recommendations.join('. ')}.`;
  }
  
  speechText += `</speak>`;
  
  return speechText;
}
```

## 🎯 Exemplo 3: Predição de Quedas de Energia com Sugestões de Prevenção

### Cenário
Usuário quer saber sobre risco de queda de energia baseado na nossa API ML e receber sugestões de prevenção.

### Implementação

#### 1. Intent Handler para Predição
```javascript
const GetWeatherPredictionHandler = {
  canHandle(handlerInput) {
    return Alexa.getRequestType(handlerInput.requestEnvelope) === 'IntentRequest'
      && Alexa.getIntentName(handlerInput.requestEnvelope) === 'GetWeatherPrediction';
  },
  async handle(handlerInput) {
    try {
      // Obter dados climáticos atuais da Alexa
      const weatherData = await getAlexaWeatherData(handlerInput);
      
      // Fazer predição usando nossa API ML
      const prediction = await getWeatherPrediction(weatherData);
      
      // Processar predição e gerar sugestões de prevenção
      const processedPrediction = processWeatherPrediction(prediction, weatherData);
      
      // Criar resposta com alertas e sugestões
      const speechText = createWeatherPredictionSSML(processedPrediction);
      
      return handlerInput.responseBuilder
        .speak(speechText)
        .withStandardCard(
          'Predição de Quedas de Energia',
          formatWeatherCard(processedPrediction),
          processedPrediction.alertImageUrl
        )
        .getResponse();
    } catch (error) {
      console.error('Erro na predição de quedas:', error);
      return createErrorResponse(handlerInput, 'predição de quedas de energia');
    }
  }
};

// Obter dados climáticos da Alexa (usando funcionalidade nativa)
async function getAlexaWeatherData(handlerInput) {
  // A Alexa já fornece dados climáticos básicos
  // Usamos dados padrão baseados na localização do usuário
  const userLocation = await getUserLocation(handlerInput);
  
  // Dados climáticos simulados baseados na localização
  // Em produção, estes dados viriam de APIs de clima integradas
  const weatherData = {
    temperatura_celsius: 25.0, // Seria obtido da localização
    umidade_pct: 65.0,
    precipitacao_mm_h: 10.0,
    vento_kmh: 30.0,
    pressao_hpa: 1013.0
  };
  
  return weatherData;
}

// Obter localização do usuário (simulada)
async function getUserLocation(handlerInput) {
  // Em produção, isso seria obtido através de permissões da Alexa
  // ou configuração do usuário no perfil
  return {
    city: 'São Paulo',
    state: 'SP',
    country: 'BR',
    coordinates: {
      lat: -23.5505,
      lon: -46.6333
    }
  };
}

// Processar predição de quedas de energia
function processWeatherPrediction(prediction, weatherData) {
  const riskLevel = prediction.nivel_risco;
  const probability = prediction.probabilidade;
  
  return {
    riskLevel,
    probability,
    probabilityPct: prediction.probabilidade_pct,
    isHighRisk: riskLevel === 'Alto' || riskLevel === 'Crítico',
    recommendations: generatePreventionRecommendations(riskLevel, weatherData, prediction),
    alertImageUrl: getAlertImageUrl(riskLevel),
    weatherConditions: formatWeatherConditions(weatherData),
    preventionTips: getPreventionTips(riskLevel)
  };
}

// Gerar recomendações de prevenção baseadas no risco de queda
function generatePreventionRecommendations(riskLevel, weatherData, prediction) {
  const recommendations = [];
  
  if (riskLevel === 'Crítico') {
    recommendations.push('🚨 RISCO CRÍTICO: Ative o modo de emergência imediatamente');
    recommendations.push('🔋 Carregue completamente a bateria do sistema solar');
    recommendations.push('💡 Desligue equipamentos não essenciais para economizar energia');
    recommendations.push('📱 Mantenha dispositivos móveis carregados');
    recommendations.push('🕯️ Tenha lanternas e velas prontas para uso');
    recommendations.push('❄️ Prepare geladeira com gelo para manter alimentos');
  } else if (riskLevel === 'Alto') {
    recommendations.push('⚠️ RISCO ALTO: Prepare-se para possível interrupção');
    recommendations.push('🔋 Aumente o nível de carga da bateria para 90%');
    recommendations.push('💡 Reduza o consumo de energia desligando luzes desnecessárias');
    recommendations.push('📱 Carregue dispositivos importantes');
    recommendations.push('🌡️ Ajuste termostatos para economizar energia');
  } else if (riskLevel === 'Médio') {
    recommendations.push('⚡ RISCO MÉDIO: Monitore o sistema de perto');
    recommendations.push('🔋 Mantenha a bateria com pelo menos 70% de carga');
    recommendations.push('💡 Evite usar equipamentos de alto consumo simultaneamente');
    recommendations.push('📊 Verifique o status do sistema regularmente');
  } else {
    recommendations.push('✅ RISCO BAIXO: Sistema operando normalmente');
    recommendations.push('🔋 Bateria pode ser mantida em modo econômico');
    recommendations.push('💡 Uso normal de energia é seguro');
  }
  
  // Sugestões específicas baseadas nos dados da predição
  if (prediction.dados_entrada.vento_kmh > 50) {
    recommendations.push('💨 Ventos fortes: Verifique a fixação dos painéis solares');
  }
  
  if (prediction.dados_entrada.precipitacao_mm_h > 15) {
    recommendations.push('🌧️ Chuva intensa: Proteja equipamentos elétricos externos');
  }
  
  if (prediction.dados_entrada.temperatura_celsius > 35) {
    recommendations.push('🌡️ Temperatura alta: Monitore o sistema de refrigeração');
  }
  
  return recommendations;
}

// Obter dicas de prevenção específicas
function getPreventionTips(riskLevel) {
  const tips = {
    'Crítico': [
      'Mantenha um gerador de emergência carregado',
      'Tenha um plano de evacuação se necessário',
      'Comunique-se com vizinhos sobre a situação',
      'Monitore notícias locais para atualizações'
    ],
    'Alto': [
      'Prepare um kit de emergência básico',
      'Identifique pontos de recarga de dispositivos',
      'Planeje refeições que não precisem de eletricidade',
      'Mantenha contatos de emergência atualizados'
    ],
    'Médio': [
      'Configure alertas automáticos no sistema',
      'Mantenha baterias extras para dispositivos',
      'Planeje atividades que não dependam de energia',
      'Verifique se o seguro cobre danos por quedas'
    ],
    'Baixo': [
      'Mantenha o sistema atualizado',
      'Faça manutenção preventiva regular',
      'Monitore o desempenho do sistema',
      'Tenha um plano de contingência básico'
    ]
  };
  
  return tips[riskLevel] || tips['Baixo'];
}

// Criar resposta SSML com alertas e sugestões de prevenção
function createWeatherPredictionSSML(prediction) {
  const { riskLevel, probabilityPct, isHighRisk, recommendations, weatherConditions, preventionTips } = prediction;
  
  let speechText = `<speak>`;
  
  // Condições climáticas atuais
  speechText += `Condições atuais: ${weatherConditions}. `;
  
  // Nível de risco
  if (isHighRisk) {
    speechText += `<prosody rate="slow" pitch="high">`;
    speechText += `ALERTA! Risco ${riskLevel.toLowerCase()} de queda de energia. `;
    speechText += `Probabilidade de ${probabilityPct}. `;
    speechText += `</prosody>`;
  } else {
    speechText += `Risco ${riskLevel.toLowerCase()} de queda de energia. `;
    speechText += `Probabilidade de ${probabilityPct}. `;
  }
  
  // Recomendações principais
  if (recommendations.length > 0) {
    speechText += `Recomendações de prevenção: `;
    speechText += recommendations.slice(0, 3).join('. ') + '. ';
  }
  
  // Dicas adicionais se for risco alto ou crítico
  if (isHighRisk && preventionTips.length > 0) {
    speechText += `Dicas adicionais: `;
    speechText += preventionTips.slice(0, 2).join('. ') + '. ';
  }
  
  // Sugestão de ação específica
  if (riskLevel === 'Crítico') {
    speechText += `<prosody rate="slow">Ação imediata necessária! Ative o modo de emergência agora.</prosody>`;
  } else if (riskLevel === 'Alto') {
    speechText += `Prepare-se para possível interrupção de energia.`;
  } else if (riskLevel === 'Médio') {
    speechText += `Monitore o sistema e mantenha a bateria carregada.`;
  } else {
    speechText += `Sistema operando normalmente. Continue monitorando.`;
  }
  
  speechText += `</speak>`;
  
  return speechText;
}

// Formatar condições climáticas
function formatWeatherConditions(weatherData) {
  const temp = Math.round(weatherData.temperatura_celsius);
  const humidity = Math.round(weatherData.umidade_pct);
  const wind = Math.round(weatherData.vento_kmh);
  
  return `temperatura de ${temp} graus, umidade de ${humidity}% e ventos de ${wind} quilômetros por hora`;
}

// Formatar conteúdo do card
function formatWeatherCard(prediction) {
  const { riskLevel, probabilityPct, recommendations, preventionTips, weatherConditions } = prediction;
  
  let cardContent = `🌡️ Temperatura: ${weatherConditions.temperature}°C
💧 Umidade: ${weatherConditions.humidity}%
🌧️ Chuva: ${weatherConditions.rain}mm/h
💨 Vento: ${weatherConditions.wind}km/h

⚠️ Risco: ${riskLevel}
📊 Probabilidade: ${probabilityPct}

🔧 AÇÕES RECOMENDADAS:
${recommendations.slice(0, 3).map(rec => `• ${rec}`).join('\n')}`;

  if (prediction.isHighRisk && preventionTips.length > 0) {
    cardContent += `\n\n💡 DICAS ADICIONAIS:
${preventionTips.slice(0, 2).map(tip => `• ${tip}`).join('\n')}`;
  }

  if (riskLevel === 'Crítico') {
    cardContent += `\n\n🚨 AÇÃO IMEDIATA NECESSÁRIA!`;
  } else if (riskLevel === 'Alto') {
    cardContent += `\n\n⚠️ PREPARE-SE PARA POSSÍVEL INTERRUPÇÃO`;
  } else {
    cardContent += `\n\n✅ Continue monitorando o sistema`;
  }

  return cardContent;
}

// Obter URL da imagem de alerta
function getAlertImageUrl(riskLevel) {
  const baseUrl = 'https://goodwe-alexa-assets.s3.amazonaws.com/alerts';
  
  switch (riskLevel) {
    case 'Crítico':
      return `${baseUrl}/critical-alert.png`;
    case 'Alto':
      return `${baseUrl}/high-risk.png`;
    case 'Médio':
      return `${baseUrl}/medium-risk.png`;
    default:
      return `${baseUrl}/low-risk.png`;
  }
}
```

## 🎯 Exemplo 4: Controle de Bateria com Modos

### Cenário
Usuário quer configurar o modo de operação da bateria através de comandos de voz.

### Implementação

#### 1. Intent Handler para Controle de Bateria
```javascript
const SetBatteryModeHandler = {
  canHandle(handlerInput) {
    return Alexa.getRequestType(handlerInput.requestEnvelope) === 'IntentRequest'
      && Alexa.getIntentName(handlerInput.requestEnvelope) === 'SetBatteryMode';
  },
  async handle(handlerInput) {
    try {
      const batteryMode = Alexa.getSlotValue(handlerInput.requestEnvelope, 'BatteryMode');
      
      if (!batteryMode) {
        return handlerInput.responseBuilder
          .speak('Qual modo de bateria você gostaria de configurar? Econômico, balanceado ou performance?')
          .reprompt('Diga o modo desejado para a bateria.')
          .getResponse();
      }
      
      // Configurar modo da bateria
      const result = await setBatteryMode(batteryMode);
      
      // Criar resposta de confirmação
      const speechText = createBatteryModeSSML(batteryMode, result);
      
      return handlerInput.responseBuilder
        .speak(speechText)
        .withSimpleCard(
          'Modo da Bateria Configurado',
          `Modo: ${batteryMode}\nStatus: ${result.success ? 'Sucesso' : 'Falha'}`
        )
        .getResponse();
    } catch (error) {
      console.error('Erro ao configurar bateria:', error);
      return createErrorResponse(handlerInput, 'configuração da bateria');
    }
  }
};

// Configurar modo da bateria
async function setBatteryMode(mode) {
  const modeConfig = getBatteryModeConfig(mode);
  
  const response = await axios.post(`${API_CONFIG.goodwe.baseUrl}/battery/configure`, {
    mode: modeConfig.mode,
    settings: modeConfig.settings
  }, {
    timeout: API_CONFIG.goodwe.timeout,
    headers: {
      'Authorization': `Bearer ${process.env.GOODWE_API_KEY}`,
      'Content-Type': 'application/json'
    }
  });
  
  return response.data;
}

// Obter configuração do modo
function getBatteryModeConfig(mode) {
  const modes = {
    'economy': {
      mode: 'economy',
      settings: {
        maxChargeLevel: 80,
        minDischargeLevel: 20,
        chargeRate: 'slow',
        priority: 'cost_saving'
      }
    },
    'balanced': {
      mode: 'balanced',
      settings: {
        maxChargeLevel: 90,
        minDischargeLevel: 15,
        chargeRate: 'normal',
        priority: 'balanced'
      }
    },
    'performance': {
      mode: 'performance',
      settings: {
        maxChargeLevel: 95,
        minDischargeLevel: 10,
        chargeRate: 'fast',
        priority: 'performance'
      }
    }
  };
  
  return modes[mode] || modes['balanced'];
}

// Criar resposta SSML
function createBatteryModeSSML(mode, result) {
  const modeNames = {
    'economy': 'econômico',
    'balanced': 'balanceado',
    'performance': 'performance'
  };
  
  const modeName = modeNames[mode] || mode;
  
  let speechText = `<speak>`;
  
  if (result.success) {
    speechText += `Modo da bateria configurado para ${modeName} com sucesso. `;
    
    // Explicar características do modo
    switch (mode) {
      case 'economy':
        speechText += `Este modo prioriza economia, carregando até 80% e descarregando até 20%. `;
        break;
      case 'balanced':
        speechText += `Este modo equilibra performance e durabilidade, carregando até 90% e descarregando até 15%. `;
        break;
      case 'performance':
        speechText += `Este modo prioriza performance, carregando até 95% e descarregando até 10%. `;
        break;
    }
    
    speechText += `A configuração será aplicada na próxima operação da bateria.`;
  } else {
    speechText += `Desculpe, não foi possível configurar o modo ${modeName}. `;
    speechText += `Verifique se o sistema está online e tente novamente.`;
  }
  
  speechText += `</speak>`;
  
  return speechText;
}
```

## 🎯 Exemplo 5: Relatório Diário Automático

### Cenário
Usuário quer receber um relatório diário completo do sistema solar.

### Implementação

#### 1. Intent Handler para Relatório
```javascript
const GetDailyReportHandler = {
  canHandle(handlerInput) {
    return Alexa.getRequestType(handlerInput.requestEnvelope) === 'IntentRequest'
      && Alexa.getIntentName(handlerInput.requestEnvelope) === 'GetDailyReport';
  },
  async handle(handlerInput) {
    try {
      // Obter dados do dia
      const dailyData = await getDailyData();
      
      // Processar relatório
      const report = processDailyReport(dailyData);
      
      // Criar resposta completa
      const speechText = createDailyReportSSML(report);
      
      return handlerInput.responseBuilder
        .speak(speechText)
        .withStandardCard(
          `Relatório Diário - ${report.date}`,
          formatDailyReportCard(report),
          report.chartUrl
        )
        .getResponse();
    } catch (error) {
      console.error('Erro no relatório diário:', error);
      return createErrorResponse(handlerInput, 'relatório diário');
    }
  }
};

// Obter dados do dia
async function getDailyData() {
  const today = new Date().toISOString().split('T')[0];
  
  const response = await axios.get(`${API_CONFIG.goodwe.baseUrl}/analytics/daily`, {
    params: {
      date: today,
      include_details: true,
      include_charts: true
    },
    timeout: API_CONFIG.goodwe.timeout
  });
  
  return response.data;
}

// Processar relatório diário
function processDailyReport(data) {
  const summary = data.summary || {};
  const generation = data.generation || {};
  const consumption = data.consumption || {};
  const battery = data.battery || {};
  const efficiency = data.efficiency || {};
  
  return {
    date: new Date().toLocaleDateString('pt-BR'),
    totalGeneration: generation.total || 0,
    totalConsumption: consumption.total || 0,
    peakGeneration: generation.peak || 0,
    peakConsumption: consumption.peak || 0,
    averageEfficiency: efficiency.average || 0,
    batteryCycles: battery.cycles || 0,
    savings: calculateSavings(generation.total, consumption.total),
    recommendations: generateDailyRecommendations(data),
    chartUrl: data.chart_url
  };
}

// Calcular economia
function calculateSavings(generation, consumption) {
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

// Criar relatório SSML
function createDailyReportSSML(report) {
  const { 
    date, totalGeneration, totalConsumption, peakGeneration, 
    averageEfficiency, savings, recommendations 
  } = report;
  
  let speechText = `<speak>`;
  
  speechText += `Relatório diário de ${date}: `;
  
  // Geração total
  speechText += `Geração total de <say-as interpret-as="number">${totalGeneration}</say-as> quilowatts-hora. `;
  
  // Consumo total
  speechText += `Consumo total de <say-as interpret-as="number">${totalConsumption}</say-as> quilowatts-hora. `;
  
  // Pico de geração
  speechText += `Pico de geração de <say-as interpret-as="number">${peakGeneration}</say-as> watts. `;
  
  // Eficiência
  speechText += `Eficiência média de <say-as interpret-as="number">${averageEfficiency}</say-as> por cento. `;
  
  // Economia
  if (savings.totalSavings > 0) {
    speechText += `Economia de <say-as interpret-as="currency">R$ ${savings.totalSavings}</say-as> reais, `;
    speechText += `representando <say-as interpret-as="number">${savings.savingsPercentage}</say-as> por cento de economia. `;
  } else {
    speechText += `Não houve economia significativa hoje. `;
  }
  
  // Recomendações
  if (recommendations.length > 0) {
    speechText += `Recomendações: ${recommendations.join('. ')}.`;
  }
  
  speechText += `</speak>`;
  
  return speechText;
}
```

## 🧪 Testes dos Exemplos

### Script de Teste Completo
```javascript
// test-examples.js
const { handler } = require('./index');

const testCases = [
  {
    name: 'Status do Sistema',
    request: {
      request: {
        type: 'IntentRequest',
        intent: { name: 'GetSystemStatus' }
      }
    },
    expectedKeywords: ['sistema', 'geração', 'bateria']
  },
  {
    name: 'Análise de Eficiência',
    request: {
      request: {
        type: 'IntentRequest',
        intent: { 
          name: 'GetEfficiencyAnalysis',
          slots: {
            AnalysisType: { value: 'daily' }
          }
        }
      }
    },
    expectedKeywords: ['eficiência', 'análise']
  },
  {
    name: 'Predição Climática',
    request: {
      request: {
        type: 'IntentRequest',
        intent: { name: 'GetWeatherPrediction' }
      }
    },
    expectedKeywords: ['risco', 'energia', 'clima']
  }
];

async function runTests() {
  console.log('🧪 Iniciando testes dos exemplos...\n');
  
  for (const testCase of testCases) {
    try {
      console.log(`Testando: ${testCase.name}`);
      
      const response = await handler(testCase.request);
      const speechText = response.response.outputSpeech.ssml || response.response.outputSpeech.text;
      
      // Verificar se contém palavras-chave esperadas
      const hasKeywords = testCase.expectedKeywords.every(keyword => 
        speechText.toLowerCase().includes(keyword.toLowerCase())
      );
      
      if (hasKeywords) {
        console.log(`✅ ${testCase.name}: PASSOU`);
      } else {
        console.log(`❌ ${testCase.name}: FALHOU - Palavras-chave não encontradas`);
      }
      
      console.log(`Resposta: ${speechText.substring(0, 100)}...\n`);
      
    } catch (error) {
      console.error(`❌ ${testCase.name}: ERRO - ${error.message}\n`);
    }
  }
}

runTests();
```

---

**Próximo**: [Guia de Deployment](./04-deployment.md)
