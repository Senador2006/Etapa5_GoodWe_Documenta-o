# Energy Optimization - GoodWe Alexa Skill

## 📋 Visão Geral

Este documento detalha o sistema de otimização energética da Alexa Skill GoodWe, incluindo análises preditivas, recomendações inteligentes e otimização automática do sistema solar.

## 🧠 Sistema de Análise Preditiva

### 1. Engine de Predição de Energia

```javascript
// optimization/prediction-engine.js
const PredictionEngine = {
  async predictEnergyGeneration(weatherForecast, historicalData) {
    try {
      const features = this.prepareGenerationFeatures(weatherForecast, historicalData);
      
      const response = await axios.post(`${API_CONFIG.ml.baseUrl}/predict-generation`, {
        features,
        horizon: 24 // 24 horas à frente
      });
      
      return this.formatGenerationPrediction(response.data);
    } catch (error) {
      console.error('Erro na predição de geração:', error);
      throw new Error('Não foi possível obter predição de geração');
    }
  },
  
  async predictEnergyConsumption(userBehavior, historicalData) {
    try {
      const features = this.prepareConsumptionFeatures(userBehavior, historicalData);
      
      const response = await axios.post(`${API_CONFIG.ml.baseUrl}/predict-consumption`, {
        features,
        horizon: 24
      });
      
      return this.formatConsumptionPrediction(response.data);
    } catch (error) {
      console.error('Erro na predição de consumo:', error);
      throw new Error('Não foi possível obter predição de consumo');
    }
  },
  
  async predictOptimalBatteryUsage(generation, consumption, batteryCapacity) {
    try {
      const optimization = await this.optimizeBatteryUsage(generation, consumption, batteryCapacity);
      
      return {
        chargeSchedule: optimization.chargeSchedule,
        dischargeSchedule: optimization.dischargeSchedule,
        expectedSavings: optimization.savings,
        recommendations: optimization.recommendations
      };
    } catch (error) {
      console.error('Erro na otimização da bateria:', error);
      throw new Error('Não foi possível otimizar uso da bateria');
    }
  },
  
  prepareGenerationFeatures(weather, historical) {
    return {
      // Dados climáticos
      temperature: weather.temperature,
      humidity: weather.humidity,
      cloudCover: weather.cloudCover,
      windSpeed: weather.windSpeed,
      precipitation: weather.precipitation,
      
      // Dados históricos
      historicalGeneration: historical.generation,
      seasonalPatterns: historical.seasonalPatterns,
      panelEfficiency: historical.panelEfficiency,
      
      // Características do sistema
      panelCapacity: historical.panelCapacity,
      panelAngle: historical.panelAngle,
      panelOrientation: historical.panelOrientation,
      systemAge: historical.systemAge
    };
  },
  
  prepareConsumptionFeatures(behavior, historical) {
    return {
      // Comportamento do usuário
      dailyPatterns: behavior.dailyPatterns,
      weeklyPatterns: behavior.weeklyPatterns,
      seasonalPatterns: behavior.seasonalPatterns,
      applianceUsage: behavior.applianceUsage,
      
      // Dados históricos
      historicalConsumption: historical.consumption,
      peakHours: historical.peakHours,
      offPeakHours: historical.offPeakHours,
      
      // Fatores externos
      weather: behavior.weather,
      holidays: behavior.holidays,
      specialEvents: behavior.specialEvents
    };
  },
  
  async optimizeBatteryUsage(generation, consumption, capacity) {
    const optimization = {
      chargeSchedule: [],
      dischargeSchedule: [],
      savings: 0,
      recommendations: []
    };
    
    // Algoritmo de otimização da bateria
    for (let hour = 0; hour < 24; hour++) {
      const gen = generation[hour] || 0;
      const cons = consumption[hour] || 0;
      const net = gen - cons;
      
      if (net > 0) {
        // Excesso de geração - carregar bateria
        optimization.chargeSchedule.push({
          hour,
          power: Math.min(net, capacity * 0.1), // Máximo 10% da capacidade por hora
          reason: 'excess_generation'
        });
      } else if (net < 0) {
        // Déficit de geração - descarregar bateria
        optimization.dischargeSchedule.push({
          hour,
          power: Math.min(Math.abs(net), capacity * 0.2), // Máximo 20% da capacidade por hora
          reason: 'demand_peak'
        });
      }
    }
    
    // Calcular economia esperada
    optimization.savings = this.calculateExpectedSavings(optimization);
    
    // Gerar recomendações
    optimization.recommendations = this.generateBatteryRecommendations(optimization);
    
    return optimization;
  },
  
  calculateExpectedSavings(optimization) {
    let savings = 0;
    const gridPrice = 0.65; // R$ por kWh
    const solarPrice = 0.20; // R$ por kWh
    
    for (const discharge of optimization.dischargeSchedule) {
      const energySaved = discharge.power * 1; // 1 hora
      const costSaved = energySaved * (gridPrice - solarPrice);
      savings += costSaved;
    }
    
    return Math.round(savings * 100) / 100;
  },
  
  generateBatteryRecommendations(optimization) {
    const recommendations = [];
    
    if (optimization.chargeSchedule.length > 0) {
      recommendations.push('Configure a bateria para carregar durante os horários de excesso de geração');
    }
    
    if (optimization.dischargeSchedule.length > 0) {
      recommendations.push('Use a bateria durante os picos de consumo para maximizar a economia');
    }
    
    if (optimization.savings > 10) {
      recommendations.push(`Economia esperada de R$ ${optimization.savings} por dia`);
    }
    
    return recommendations;
  }
};
```

### 2. Sistema de Recomendações Inteligentes

```javascript
// optimization/recommendation-engine.js
const RecommendationEngine = {
  async generateRecommendations(systemData, predictions, userProfile) {
    const recommendations = [];
    
    // Análise de eficiência
    const efficiencyRecs = await this.analyzeEfficiency(systemData);
    recommendations.push(...efficiencyRecs);
    
    // Análise de consumo
    const consumptionRecs = await this.analyzeConsumption(systemData, predictions);
    recommendations.push(...consumptionRecs);
    
    // Análise de geração
    const generationRecs = await this.analyzeGeneration(systemData, predictions);
    recommendations.push(...generationRecs);
    
    // Análise de bateria
    const batteryRecs = await this.analyzeBattery(systemData, predictions);
    recommendations.push(...batteryRecs);
    
    // Análise de custos
    const costRecs = await this.analyzeCosts(systemData, predictions);
    recommendations.push(...costRecs);
    
    return this.prioritizeRecommendations(recommendations);
  },
  
  async analyzeEfficiency(systemData) {
    const recommendations = [];
    const efficiency = systemData.efficiency || 0;
    
    if (efficiency < 60) {
      recommendations.push({
        category: 'efficiency',
        priority: 'high',
        title: 'Otimizar Eficiência do Sistema',
        description: `Sua eficiência atual é de ${efficiency}%. Considere as seguintes ações:`,
        actions: [
          'Verificar limpeza dos painéis solares',
          'Verificar sombreamento nos painéis',
          'Revisar ângulo de inclinação dos painéis',
          'Verificar conexões e cabos'
        ],
        expectedImprovement: '15-25%',
        estimatedSavings: 'R$ 50-100/mês'
      });
    } else if (efficiency < 80) {
      recommendations.push({
        category: 'efficiency',
        priority: 'medium',
        title: 'Melhorar Eficiência do Sistema',
        description: `Sua eficiência de ${efficiency}% pode ser melhorada:`,
        actions: [
          'Limpar painéis solares regularmente',
          'Verificar se há novos obstáculos causando sombra',
          'Otimizar horários de uso de equipamentos'
        ],
        expectedImprovement: '5-15%',
        estimatedSavings: 'R$ 20-50/mês'
      });
    } else {
      recommendations.push({
        category: 'efficiency',
        priority: 'low',
        title: 'Eficiência Excelente',
        description: `Parabéns! Sua eficiência de ${efficiency}% está excelente.`,
        actions: [
          'Manter manutenção regular',
          'Continuar monitorando o sistema'
        ],
        expectedImprovement: 'Manter',
        estimatedSavings: 'R$ 0/mês'
      });
    }
    
    return recommendations;
  },
  
  async analyzeConsumption(systemData, predictions) {
    const recommendations = [];
    const consumption = systemData.consumption || {};
    const peakHours = this.identifyPeakHours(consumption);
    
    if (peakHours.length > 0) {
      recommendations.push({
        category: 'consumption',
        priority: 'high',
        title: 'Otimizar Horários de Consumo',
        description: 'Identificamos picos de consumo que podem ser otimizados:',
        actions: [
          'Deslocar uso de equipamentos para horários de maior geração',
          'Usar bateria durante picos de consumo',
          'Configurar automações para otimizar horários',
          'Evitar uso simultâneo de equipamentos de alto consumo'
        ],
        peakHours: peakHours,
        expectedImprovement: '20-30%',
        estimatedSavings: 'R$ 80-150/mês'
      });
    }
    
    // Análise de equipamentos
    const highConsumptionDevices = this.identifyHighConsumptionDevices(consumption);
    if (highConsumptionDevices.length > 0) {
      recommendations.push({
        category: 'consumption',
        priority: 'medium',
        title: 'Otimizar Equipamentos de Alto Consumo',
        description: 'Alguns equipamentos consomem muita energia:',
        actions: [
          'Considerar substituição por modelos mais eficientes',
          'Usar equipamentos em horários de maior geração',
          'Configurar modos de economia de energia',
          'Desligar equipamentos quando não em uso'
        ],
        devices: highConsumptionDevices,
        expectedImprovement: '10-20%',
        estimatedSavings: 'R$ 30-80/mês'
      });
    }
    
    return recommendations;
  },
  
  async analyzeGeneration(systemData, predictions) {
    const recommendations = [];
    const generation = systemData.generation || {};
    const predictions = predictions.generation || {};
    
    // Análise de subutilização
    const underutilization = this.calculateUnderutilization(generation, predictions);
    if (underutilization > 20) {
      recommendations.push({
        category: 'generation',
        priority: 'high',
        title: 'Maximizar Geração Solar',
        description: `Sua geração está ${underutilization}% abaixo do potencial:`,
        actions: [
          'Verificar limpeza e manutenção dos painéis',
          'Ajustar ângulo dos painéis para máxima exposição',
          'Remover obstáculos que causam sombra',
          'Considerar expansão do sistema'
        ],
        underutilization: underutilization,
        expectedImprovement: '15-30%',
        estimatedSavings: 'R$ 100-200/mês'
      });
    }
    
    // Análise de picos de geração
    const peakGeneration = this.identifyPeakGeneration(generation);
    if (peakGeneration.length > 0) {
      recommendations.push({
        category: 'generation',
        priority: 'medium',
        title: 'Aproveitar Picos de Geração',
        description: 'Identificamos horários de alta geração que podem ser melhor aproveitados:',
        actions: [
          'Programar equipamentos para horários de pico de geração',
          'Usar excedente para carregar bateria',
          'Considerar venda de energia excedente',
          'Configurar automações para aproveitar picos'
        ],
        peakHours: peakGeneration,
        expectedImprovement: '10-20%',
        estimatedSavings: 'R$ 50-100/mês'
      });
    }
    
    return recommendations;
  },
  
  async analyzeBattery(systemData, predictions) {
    const recommendations = [];
    const battery = systemData.battery || {};
    const predictions = predictions.battery || {};
    
    // Análise de uso da bateria
    const batteryUtilization = this.calculateBatteryUtilization(battery);
    if (batteryUtilization < 50) {
      recommendations.push({
        category: 'battery',
        priority: 'high',
        title: 'Otimizar Uso da Bateria',
        description: `Sua bateria está sendo utilizada apenas ${batteryUtilization}% do potencial:`,
        actions: [
          'Configurar carregamento durante excesso de geração',
          'Usar bateria durante picos de consumo',
          'Ajustar configurações de carga e descarga',
          'Implementar automações inteligentes'
        ],
        utilization: batteryUtilization,
        expectedImprovement: '30-50%',
        estimatedSavings: 'R$ 60-120/mês'
      });
    }
    
    // Análise de ciclos da bateria
    const batteryCycles = battery.cycles || 0;
    if (batteryCycles > 1000) {
      recommendations.push({
        category: 'battery',
        priority: 'medium',
        title: 'Monitorar Saúde da Bateria',
        description: `Sua bateria já teve ${batteryCycles} ciclos de carga/descarga:`,
        actions: [
          'Verificar capacidade atual da bateria',
          'Ajustar estratégia de uso para prolongar vida útil',
          'Considerar substituição se necessário',
          'Implementar monitoramento de saúde'
        ],
        cycles: batteryCycles,
        expectedImprovement: 'Manter',
        estimatedSavings: 'R$ 0/mês'
      });
    }
    
    return recommendations;
  },
  
  async analyzeCosts(systemData, predictions) {
    const recommendations = [];
    const costs = systemData.costs || {};
    const predictions = predictions.costs || {};
    
    // Análise de tarifas
    const tariffOptimization = this.analyzeTariffOptimization(costs);
    if (tariffOptimization.potential > 0) {
      recommendations.push({
        category: 'costs',
        priority: 'high',
        title: 'Otimizar Tarifas de Energia',
        description: `Identificamos oportunidade de economia de R$ ${tariffOptimization.potential}/mês:`,
        actions: [
          'Usar energia solar durante horários de pico',
          'Deslocar consumo para horários de menor tarifa',
          'Maximizar uso da bateria durante picos tarifários',
          'Considerar mudança de modalidade tarifária'
        ],
        potential: tariffOptimization.potential,
        expectedImprovement: '15-25%',
        estimatedSavings: `R$ ${tariffOptimization.potential}/mês`
      });
    }
    
    return recommendations;
  },
  
  prioritizeRecommendations(recommendations) {
    return recommendations.sort((a, b) => {
      const priorityOrder = { 'high': 3, 'medium': 2, 'low': 1 };
      return priorityOrder[b.priority] - priorityOrder[a.priority];
    });
  }
};
```

### 3. Sistema de Otimização Automática

```javascript
// optimization/auto-optimization.js
const AutoOptimization = {
  async optimizeSystem(systemData, predictions, userPreferences) {
    const optimizations = [];
    
    // Otimização da bateria
    const batteryOptimization = await this.optimizeBattery(systemData, predictions);
    optimizations.push(batteryOptimization);
    
    // Otimização de cargas
    const loadOptimization = await this.optimizeLoads(systemData, predictions);
    optimizations.push(loadOptimization);
    
    // Otimização de geração
    const generationOptimization = await this.optimizeGeneration(systemData, predictions);
    optimizations.push(generationOptimization);
    
    // Aplicar otimizações
    await this.applyOptimizations(optimizations);
    
    return optimizations;
  },
  
  async optimizeBattery(systemData, predictions) {
    const battery = systemData.battery || {};
    const generation = predictions.generation || {};
    const consumption = predictions.consumption || {};
    
    const optimization = {
      type: 'battery',
      actions: [],
      expectedSavings: 0
    };
    
    // Calcular horários ótimos de carga
    const chargeSchedule = this.calculateOptimalChargeSchedule(generation, consumption);
    optimization.actions.push({
      action: 'set_charge_schedule',
      schedule: chargeSchedule
    });
    
    // Calcular horários ótimos de descarga
    const dischargeSchedule = this.calculateOptimalDischargeSchedule(generation, consumption);
    optimization.actions.push({
      action: 'set_discharge_schedule',
      schedule: dischargeSchedule
    });
    
    // Configurar modo da bateria
    const optimalMode = this.calculateOptimalBatteryMode(generation, consumption);
    optimization.actions.push({
      action: 'set_battery_mode',
      mode: optimalMode
    });
    
    optimization.expectedSavings = this.calculateBatterySavings(chargeSchedule, dischargeSchedule);
    
    return optimization;
  },
  
  async optimizeLoads(systemData, predictions) {
    const loads = systemData.loads || [];
    const generation = predictions.generation || {};
    const consumption = predictions.consumption || {};
    
    const optimization = {
      type: 'loads',
      actions: [],
      expectedSavings: 0
    };
    
    for (const load of loads) {
      const optimalSchedule = this.calculateOptimalLoadSchedule(load, generation, consumption);
      
      optimization.actions.push({
        action: 'schedule_load',
        loadId: load.id,
        schedule: optimalSchedule
      });
    }
    
    optimization.expectedSavings = this.calculateLoadSavings(optimization.actions);
    
    return optimization;
  },
  
  async optimizeGeneration(systemData, predictions) {
    const generation = systemData.generation || {};
    const weather = predictions.weather || {};
    
    const optimization = {
      type: 'generation',
      actions: [],
      expectedSavings: 0
    };
    
    // Otimizar ângulo dos painéis baseado na previsão do tempo
    const optimalAngle = this.calculateOptimalPanelAngle(weather);
    optimization.actions.push({
      action: 'adjust_panel_angle',
      angle: optimalAngle
    });
    
    // Configurar limpeza automática se necessário
    const cleaningNeeded = this.checkCleaningNeeded(generation, weather);
    if (cleaningNeeded) {
      optimization.actions.push({
        action: 'schedule_cleaning',
        priority: 'high'
      });
    }
    
    optimization.expectedSavings = this.calculateGenerationSavings(optimization.actions);
    
    return optimization;
  },
  
  async applyOptimizations(optimizations) {
    for (const optimization of optimizations) {
      for (const action of optimization.actions) {
        await this.executeOptimizationAction(action);
      }
    }
  },
  
  async executeOptimizationAction(action) {
    switch (action.action) {
      case 'set_charge_schedule':
        await this.setBatteryChargeSchedule(action.schedule);
        break;
      case 'set_discharge_schedule':
        await this.setBatteryDischargeSchedule(action.schedule);
        break;
      case 'set_battery_mode':
        await this.setBatteryMode(action.mode);
        break;
      case 'schedule_load':
        await this.scheduleLoad(action.loadId, action.schedule);
        break;
      case 'adjust_panel_angle':
        await this.adjustPanelAngle(action.angle);
        break;
      case 'schedule_cleaning':
        await this.scheduleCleaning(action.priority);
        break;
    }
  }
};
```

## 📊 Relatórios de Otimização

### 1. Relatório de Performance

```javascript
// reports/performance-report.js
const PerformanceReport = {
  async generatePerformanceReport(period = 'monthly') {
    const data = await this.getPerformanceData(period);
    
    return {
      period,
      generation: this.analyzeGeneration(data.generation),
      consumption: this.analyzeConsumption(data.consumption),
      efficiency: this.analyzeEfficiency(data.efficiency),
      savings: this.analyzeSavings(data.savings),
      recommendations: await this.generateRecommendations(data),
      trends: this.analyzeTrends(data.trends)
    };
  },
  
  analyzeGeneration(generationData) {
    const total = generationData.total || 0;
    const average = generationData.average || 0;
    const peak = generationData.peak || 0;
    const efficiency = generationData.efficiency || 0;
    
    return {
      total: {
        value: total,
        unit: 'kWh',
        trend: this.calculateTrend(generationData.trends.total)
      },
      average: {
        value: average,
        unit: 'kW',
        trend: this.calculateTrend(generationData.trends.average)
      },
      peak: {
        value: peak,
        unit: 'kW',
        trend: this.calculateTrend(generationData.trends.peak)
      },
      efficiency: {
        value: efficiency,
        unit: '%',
        trend: this.calculateTrend(generationData.trends.efficiency)
      }
    };
  },
  
  analyzeConsumption(consumptionData) {
    const total = consumptionData.total || 0;
    const average = consumptionData.average || 0;
    const peak = consumptionData.peak || 0;
    const solarUsage = consumptionData.solarUsage || 0;
    const gridUsage = consumptionData.gridUsage || 0;
    
    return {
      total: {
        value: total,
        unit: 'kWh',
        trend: this.calculateTrend(consumptionData.trends.total)
      },
      average: {
        value: average,
        unit: 'kW',
        trend: this.calculateTrend(consumptionData.trends.average)
      },
      peak: {
        value: peak,
        unit: 'kW',
        trend: this.calculateTrend(consumptionData.trends.peak)
      },
      solarUsage: {
        value: solarUsage,
        unit: 'kWh',
        percentage: (solarUsage / total) * 100
      },
      gridUsage: {
        value: gridUsage,
        unit: 'kWh',
        percentage: (gridUsage / total) * 100
      }
    };
  },
  
  analyzeSavings(savingsData) {
    const total = savingsData.total || 0;
    const solar = savingsData.solar || 0;
    const battery = savingsData.battery || 0;
    const optimization = savingsData.optimization || 0;
    
    return {
      total: {
        value: total,
        unit: 'R$',
        trend: this.calculateTrend(savingsData.trends.total)
      },
      breakdown: {
        solar: {
          value: solar,
          unit: 'R$',
          percentage: (solar / total) * 100
        },
        battery: {
          value: battery,
          unit: 'R$',
          percentage: (battery / total) * 100
        },
        optimization: {
          value: optimization,
          unit: 'R$',
          percentage: (optimization / total) * 100
        }
      }
    };
  }
};
```

### 2. Dashboard de Otimização

```javascript
// dashboard/optimization-dashboard.js
const OptimizationDashboard = {
  async getDashboardData() {
    const [
      currentStatus,
      predictions,
      recommendations,
      performance
    ] = await Promise.all([
      this.getCurrentStatus(),
      this.getPredictions(),
      this.getRecommendations(),
      this.getPerformanceData()
    ]);
    
    return {
      currentStatus,
      predictions,
      recommendations,
      performance,
      lastUpdate: new Date().toISOString()
    };
  },
  
  async getCurrentStatus() {
    const response = await axios.get(`${API_CONFIG.goodwe.baseUrl}/system/status`);
    return {
      generation: response.data.generation,
      consumption: response.data.consumption,
      battery: response.data.battery,
      efficiency: response.data.efficiency,
      timestamp: response.data.timestamp
    };
  },
  
  async getPredictions() {
    const response = await axios.get(`${API_CONFIG.ml.baseUrl}/predictions`);
    return {
      generation: response.data.generation,
      consumption: response.data.consumption,
      battery: response.data.battery,
      weather: response.data.weather
    };
  },
  
  async getRecommendations() {
    const response = await axios.get(`${API_CONFIG.goodwe.baseUrl}/optimization/recommendations`);
    return {
      efficiency: response.data.efficiency,
      consumption: response.data.consumption,
      generation: response.data.generation,
      battery: response.data.battery,
      costs: response.data.costs
    };
  },
  
  async getPerformanceData() {
    const response = await axios.get(`${API_CONFIG.goodwe.baseUrl}/performance/dashboard`);
    return {
      daily: response.data.daily,
      weekly: response.data.weekly,
      monthly: response.data.monthly,
      trends: response.data.trends
    };
  }
};
```

## 🧪 Testes de Otimização

### 1. Testes de Predição

```javascript
// tests/optimization-tests.js
const OptimizationTests = {
  async testEnergyPrediction() {
    console.log('🧪 Testando predição de energia...');
    
    const weatherForecast = {
      temperature: 25,
      humidity: 65,
      cloudCover: 30,
      windSpeed: 10,
      precipitation: 0
    };
    
    const historicalData = {
      generation: [1000, 1200, 1500],
      consumption: [800, 900, 1000],
      efficiency: 75
    };
    
    const prediction = await PredictionEngine.predictEnergyGeneration(weatherForecast, historicalData);
    
    console.assert(prediction !== null, 'Deve gerar predição');
    console.assert(prediction.horizon === 24, 'Deve prever 24 horas');
    console.log('✅ Predição de energia: PASSOU');
  },
  
  async testBatteryOptimization() {
    console.log('🧪 Testando otimização da bateria...');
    
    const generation = Array(24).fill(1000);
    const consumption = Array(24).fill(800);
    const capacity = 10000;
    
    const optimization = await PredictionEngine.predictOptimalBatteryUsage(generation, consumption, capacity);
    
    console.assert(optimization.chargeSchedule.length > 0, 'Deve ter horários de carga');
    console.assert(optimization.dischargeSchedule.length > 0, 'Deve ter horários de descarga');
    console.log('✅ Otimização da bateria: PASSOU');
  },
  
  async testRecommendationGeneration() {
    console.log('🧪 Testando geração de recomendações...');
    
    const systemData = {
      efficiency: 60,
      consumption: { peak: 2000, average: 1000 },
      generation: { peak: 3000, average: 1500 },
      battery: { utilization: 40, cycles: 500 }
    };
    
    const predictions = {
      generation: Array(24).fill(1500),
      consumption: Array(24).fill(1000)
    };
    
    const userProfile = {
      preferences: { notifications: true },
      behavior: { dailyPatterns: [] }
    };
    
    const recommendations = await RecommendationEngine.generateRecommendations(systemData, predictions, userProfile);
    
    console.assert(recommendations.length > 0, 'Deve gerar recomendações');
    console.assert(recommendations[0].priority, 'Deve ter prioridade');
    console.log('✅ Geração de recomendações: PASSOU');
  }
};
```

---

**Próximo**: [Complete Implementation](./05-complete-implementation.md)
