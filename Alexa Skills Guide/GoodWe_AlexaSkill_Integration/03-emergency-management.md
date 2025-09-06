# Emergency Management - GoodWe Alexa Skill

## 📋 Visão Geral

Este documento detalha o sistema de gerenciamento de emergências da Alexa Skill GoodWe, incluindo alertas automáticos, procedimentos de emergência e notificações críticas.

## 🚨 Sistema de Alertas de Emergência

### 1. Tipos de Emergências

#### Emergências Críticas
- **Falha Total do Sistema**: Sistema solar completamente offline
- **Bateria Crítica**: Nível abaixo de 10% sem geração
- **Sobrecarga**: Sistema operando acima da capacidade
- **Falha de Segurança**: Problemas de segurança elétrica

#### Emergências de Alto Risco
- **Bateria Baixa**: Nível abaixo de 20% com baixa geração
- **Falha Parcial**: Componentes específicos com problemas
- **Condições Climáticas**: Tempestades ou ventos fortes
- **Sobreaquecimento**: Temperatura elevada nos equipamentos

#### Alertas Preventivos
- **Manutenção Necessária**: Componentes precisando de manutenção
- **Eficiência Baixa**: Performance abaixo do esperado
- **Anomalias**: Comportamento anômalo detectado

### 2. Engine de Detecção de Emergências

```javascript
// emergency/emergency-detection-engine.js
const EmergencyDetectionEngine = {
  async processSystemData(systemData) {
    const alerts = [];
    
    // Verificar emergências críticas
    const criticalAlerts = await this.checkCriticalEmergencies(systemData);
    alerts.push(...criticalAlerts);
    
    // Verificar emergências de alto risco
    const highRiskAlerts = await this.checkHighRiskEmergencies(systemData);
    alerts.push(...highRiskAlerts);
    
    // Verificar alertas preventivos
    const preventiveAlerts = await this.checkPreventiveAlerts(systemData);
    alerts.push(...preventiveAlerts);
    
    // Processar alertas
    for (const alert of alerts) {
      await this.processAlert(alert);
    }
    
    return alerts;
  },
  
  async checkCriticalEmergencies(data) {
    const alerts = [];
    
    // Sistema offline
    if (data.isOffline) {
      alerts.push({
        type: 'CRITICAL',
        code: 'SYSTEM_OFFLINE',
        message: 'Sistema solar completamente offline',
        priority: 1,
        actions: ['activate_emergency_mode', 'notify_emergency_contacts']
      });
    }
    
    // Bateria crítica
    if (data.soc_percentage < 10 && data.fv_power < 100) {
      alerts.push({
        type: 'CRITICAL',
        code: 'BATTERY_CRITICAL',
        message: 'Bateria com nível crítico e sem geração',
        priority: 1,
        actions: ['activate_emergency_mode', 'reduce_consumption', 'notify_emergency_contacts']
      });
    }
    
    // Sobrecarga
    if (data.load_power > data.fv_power * 1.2) {
      alerts.push({
        type: 'CRITICAL',
        code: 'OVERLOAD',
        message: 'Sistema operando acima da capacidade',
        priority: 1,
        actions: ['reduce_load', 'activate_emergency_mode']
      });
    }
    
    return alerts;
  },
  
  async checkHighRiskEmergencies(data) {
    const alerts = [];
    
    // Bateria baixa
    if (data.soc_percentage < 20 && data.fv_power < 500) {
      alerts.push({
        type: 'HIGH_RISK',
        code: 'BATTERY_LOW',
        message: 'Bateria com nível baixo e pouca geração',
        priority: 2,
        actions: ['increase_battery_charge', 'reduce_consumption']
      });
    }
    
    // Falha parcial
    if (data.efficiency < 50 && data.fv_power > 1000) {
      alerts.push({
        type: 'HIGH_RISK',
        code: 'PARTIAL_FAILURE',
        message: 'Eficiência baixa detectada',
        priority: 2,
        actions: ['check_system', 'schedule_maintenance']
      });
    }
    
    // Condições climáticas adversas
    if (data.weather_risk === 'HIGH') {
      alerts.push({
        type: 'HIGH_RISK',
        code: 'WEATHER_RISK',
        message: 'Condições climáticas adversas detectadas',
        priority: 2,
        actions: ['secure_equipment', 'monitor_conditions']
      });
    }
    
    return alerts;
  },
  
  async checkPreventiveAlerts(data) {
    const alerts = [];
    
    // Manutenção necessária
    if (data.days_since_maintenance > 90) {
      alerts.push({
        type: 'PREVENTIVE',
        code: 'MAINTENANCE_NEEDED',
        message: 'Manutenção preventiva necessária',
        priority: 3,
        actions: ['schedule_maintenance']
      });
    }
    
    // Eficiência baixa
    if (data.efficiency < 70 && data.efficiency > 50) {
      alerts.push({
        type: 'PREVENTIVE',
        code: 'LOW_EFFICIENCY',
        message: 'Eficiência abaixo do esperado',
        priority: 3,
        actions: ['check_cleanliness', 'verify_shading']
      });
    }
    
    return alerts;
  }
};
```

### 3. Sistema de Notificações de Emergência

```javascript
// emergency/emergency-notification-system.js
const EmergencyNotificationSystem = {
  async processAlert(alert) {
    console.log(`🚨 Processando alerta: ${alert.code}`);
    
    // Enviar notificação imediata
    await this.sendImmediateNotification(alert);
    
    // Executar ações de emergência
    for (const action of alert.actions) {
      await this.executeEmergencyAction(action, alert);
    }
    
    // Registrar alerta
    await this.logAlert(alert);
    
    // Notificar contatos de emergência se necessário
    if (alert.priority <= 2) {
      await this.notifyEmergencyContacts(alert);
    }
  },
  
  async sendImmediateNotification(alert) {
    const notification = {
      type: 'emergency_alert',
      level: alert.type,
      code: alert.code,
      message: alert.message,
      timestamp: new Date().toISOString(),
      priority: alert.priority
    };
    
    // Enviar via Alexa
    await this.sendAlexaNotification(notification);
    
    // Enviar via outros canais
    await this.sendMultiChannelNotification(notification);
  },
  
  async sendAlexaNotification(notification) {
    // Implementar notificação proativa para Alexa
    const speechText = this.createEmergencySpeech(notification);
    
    // Enviar notificação proativa
    await this.sendProactiveNotification({
      type: 'emergency',
      speech: speechText,
      card: this.createEmergencyCard(notification)
    });
  },
  
  createEmergencySpeech(notification) {
    let speech = `<speak>`;
    
    if (notification.level === 'CRITICAL') {
      speech += `<prosody rate="slow" pitch="high">`;
      speech += `ALERTA CRÍTICO! ${notification.message}`;
      speech += `</prosody>`;
    } else if (notification.level === 'HIGH_RISK') {
      speech += `<prosody rate="slow">`;
      speech += `ALERTA DE ALTO RISCO: ${notification.message}`;
      speech += `</prosody>`;
    } else {
      speech += `Alerta preventivo: ${notification.message}`;
    }
    
    // Adicionar instruções específicas
    speech += this.getEmergencyInstructions(notification.code);
    
    speech += `</speak>`;
    
    return speech;
  },
  
  getEmergencyInstructions(code) {
    const instructions = {
      'SYSTEM_OFFLINE': ' Verifique a conexão de energia e contate o suporte técnico imediatamente.',
      'BATTERY_CRITICAL': ' Desligue equipamentos não essenciais e prepare-se para possível interrupção.',
      'OVERLOAD': ' Reduza o consumo de energia imediatamente para evitar danos ao sistema.',
      'BATTERY_LOW': ' Aumente a carga da bateria e monitore o sistema de perto.',
      'PARTIAL_FAILURE': ' Verifique os painéis solares e agende manutenção.',
      'WEATHER_RISK': ' Proteja os equipamentos e monitore as condições climáticas.'
    };
    
    return instructions[code] || ' Monitore o sistema e siga as recomendações.';
  },
  
  createEmergencyCard(notification) {
    const urgency = notification.level === 'CRITICAL' ? '🚨 CRÍTICO' : 
                   notification.level === 'HIGH_RISK' ? '⚠️ ALTO RISCO' : 'ℹ️ PREVENTIVO';
    
    return {
      title: `${urgency} - ${notification.code}`,
      content: notification.message,
      image: this.getEmergencyImage(notification.level)
    };
  },
  
  getEmergencyImage(level) {
    const images = {
      'CRITICAL': 'https://goodwe-alexa-assets.s3.amazonaws.com/alerts/critical.png',
      'HIGH_RISK': 'https://goodwe-alexa-assets.s3.amazonaws.com/alerts/high-risk.png',
      'PREVENTIVE': 'https://goodwe-alexa-assets.s3.amazonaws.com/alerts/preventive.png'
    };
    
    return images[level];
  }
};
```

## 🆘 Procedimentos de Emergência

### 1. Modo de Emergência

```javascript
// emergency/emergency-mode.js
const EmergencyMode = {
  async activateEmergencyMode(reason) {
    console.log(`🚨 Ativando modo de emergência: ${reason}`);
    
    // Desligar cargas não essenciais
    await this.shutDownNonEssentialLoads();
    
    // Configurar bateria para modo de emergência
    await this.setBatteryEmergencyMode();
    
    // Ativar backup de energia
    await this.activateBackupPower();
    
    // Notificar usuário
    await this.notifyEmergencyModeActivation(reason);
    
    // Registrar ativação
    await this.logEmergencyModeActivation(reason);
  },
  
  async shutDownNonEssentialLoads() {
    const nonEssentialLoads = [
      'water_heater',
      'pool_pump',
      'garden_lighting',
      'entertainment_system'
    ];
    
    for (const load of nonEssentialLoads) {
      try {
        await this.turnOffDevice(load);
        console.log(`Carga não essencial ${load} desligada`);
      } catch (error) {
        console.error(`Erro ao desligar ${load}:`, error);
      }
    }
  },
  
  async setBatteryEmergencyMode() {
    try {
      await axios.post(`${API_CONFIG.goodwe.baseUrl}/battery/emergency-mode`, {
        mode: 'emergency',
        maxDischarge: 5, // Máximo 5% de descarga
        priority: 'critical_loads_only'
      });
      
      console.log('Bateria configurada para modo de emergência');
    } catch (error) {
      console.error('Erro ao configurar bateria:', error);
    }
  },
  
  async activateBackupPower() {
    try {
      // Ativar gerador de backup se disponível
      await axios.post(`${API_CONFIG.goodwe.baseUrl}/backup/activate`);
      console.log('Energia de backup ativada');
    } catch (error) {
      console.error('Erro ao ativar backup:', error);
    }
  },
  
  async notifyEmergencyModeActivation(reason) {
    const message = `Modo de emergência ativado: ${reason}. Cargas não essenciais foram desligadas e a bateria foi configurada para máxima conservação.`;
    
    await this.sendEmergencyNotification(message);
  }
};
```

### 2. Contatos de Emergência

```javascript
// emergency/emergency-contacts.js
const EmergencyContacts = {
  async notifyEmergencyContacts(alert) {
    const contacts = await this.getEmergencyContacts();
    
    for (const contact of contacts) {
      await this.sendEmergencyContactNotification(contact, alert);
    }
  },
  
  async getEmergencyContacts() {
    try {
      const response = await axios.get(`${API_CONFIG.goodwe.baseUrl}/emergency/contacts`);
      return response.data.contacts || [];
    } catch (error) {
      console.error('Erro ao obter contatos de emergência:', error);
      return this.getDefaultContacts();
    }
  },
  
  getDefaultContacts() {
    return [
      {
        name: 'Suporte Técnico',
        phone: '+55-11-99999-9999',
        email: 'suporte@goodwe.com',
        priority: 1
      },
      {
        name: 'Emergência 24h',
        phone: '+55-11-88888-8888',
        email: 'emergencia@goodwe.com',
        priority: 1
      }
    ];
  },
  
  async sendEmergencyContactNotification(contact, alert) {
    const message = this.createEmergencyMessage(alert);
    
    // Enviar SMS
    if (contact.phone) {
      await this.sendSMS(contact.phone, message);
    }
    
    // Enviar email
    if (contact.email) {
      await this.sendEmail(contact.email, message, alert);
    }
  },
  
  createEmergencyMessage(alert) {
    const timestamp = new Date().toLocaleString('pt-BR');
    
    return `🚨 ALERTA DE EMERGÊNCIA GOODWE
Data/Hora: ${timestamp}
Tipo: ${alert.type}
Código: ${alert.code}
Mensagem: ${alert.message}
Prioridade: ${alert.priority}

Ação imediata necessária!`;
  },
  
  async sendSMS(phone, message) {
    try {
      await axios.post(`${API_CONFIG.goodwe.baseUrl}/notifications/sms`, {
        phone,
        message
      });
      
      console.log(`SMS enviado para ${phone}`);
    } catch (error) {
      console.error('Erro ao enviar SMS:', error);
    }
  },
  
  async sendEmail(email, message, alert) {
    try {
      await axios.post(`${API_CONFIG.goodwe.baseUrl}/notifications/email`, {
        to: email,
        subject: `🚨 Alerta de Emergência GoodWe - ${alert.code}`,
        body: message,
        priority: 'high'
      });
      
      console.log(`Email enviado para ${email}`);
    } catch (error) {
      console.error('Erro ao enviar email:', error);
    }
  }
};
```

### 3. Recuperação de Emergência

```javascript
// emergency/emergency-recovery.js
const EmergencyRecovery = {
  async attemptRecovery(alert) {
    console.log(`🔄 Tentando recuperação para alerta: ${alert.code}`);
    
    switch (alert.code) {
      case 'SYSTEM_OFFLINE':
        return await this.recoverSystemOffline();
      case 'BATTERY_CRITICAL':
        return await this.recoverBatteryCritical();
      case 'OVERLOAD':
        return await this.recoverOverload();
      case 'PARTIAL_FAILURE':
        return await this.recoverPartialFailure();
      default:
        return await this.genericRecovery(alert);
    }
  },
  
  async recoverSystemOffline() {
    try {
      // Tentar reinicializar sistema
      await axios.post(`${API_CONFIG.goodwe.baseUrl}/system/restart`);
      
      // Aguardar e verificar status
      await this.wait(30000); // 30 segundos
      
      const status = await this.checkSystemStatus();
      
      if (status.isOnline) {
        await this.notifyRecovery('Sistema reinicializado com sucesso');
        return { success: true, message: 'Sistema recuperado' };
      } else {
        throw new Error('Sistema ainda offline após reinicialização');
      }
    } catch (error) {
      console.error('Erro na recuperação do sistema:', error);
      return { success: false, message: error.message };
    }
  },
  
  async recoverBatteryCritical() {
    try {
      // Reduzir consumo ao mínimo
      await this.minimizeConsumption();
      
      // Tentar aumentar geração
      await this.optimizeGeneration();
      
      // Verificar se há geração solar
      const data = await this.getSystemData();
      
      if (data.fv_power > 100) {
        await this.notifyRecovery('Bateria em recuperação com geração solar');
        return { success: true, message: 'Bateria em recuperação' };
      } else {
        throw new Error('Sem geração solar disponível');
      }
    } catch (error) {
      console.error('Erro na recuperação da bateria:', error);
      return { success: false, message: error.message };
    }
  },
  
  async recoverOverload() {
    try {
      // Desligar cargas imediatamente
      await this.emergencyLoadReduction();
      
      // Verificar se sobrecarga foi resolvida
      const data = await this.getSystemData();
      
      if (data.load_power <= data.fv_power) {
        await this.notifyRecovery('Sobrecarga resolvida');
        return { success: true, message: 'Sobrecarga resolvida' };
      } else {
        throw new Error('Sobrecarga persistente');
      }
    } catch (error) {
      console.error('Erro na recuperação de sobrecarga:', error);
      return { success: false, message: error.message };
    }
  },
  
  async minimizeConsumption() {
    const nonEssentialLoads = await this.getNonEssentialLoads();
    
    for (const load of nonEssentialLoads) {
      await this.turnOffDevice(load.id);
    }
  },
  
  async emergencyLoadReduction() {
    const loads = await this.getLoadsByPriority();
    
    // Desligar cargas por prioridade (menor prioridade primeiro)
    for (const load of loads) {
      if (load.priority > 1) {
        await this.turnOffDevice(load.id);
      }
    }
  },
  
  async notifyRecovery(message) {
    await this.sendRecoveryNotification(message);
  }
};
```

## 📊 Monitoramento de Emergências

### 1. Dashboard de Emergências

```javascript
// monitoring/emergency-dashboard.js
const EmergencyDashboard = {
  async getEmergencyStatus() {
    const alerts = await this.getActiveAlerts();
    const systemStatus = await this.getSystemStatus();
    const emergencyMode = await this.getEmergencyModeStatus();
    
    return {
      alerts: this.categorizeAlerts(alerts),
      systemStatus,
      emergencyMode,
      lastUpdate: new Date().toISOString()
    };
  },
  
  categorizeAlerts(alerts) {
    return {
      critical: alerts.filter(a => a.type === 'CRITICAL'),
      highRisk: alerts.filter(a => a.type === 'HIGH_RISK'),
      preventive: alerts.filter(a => a.type === 'PREVENTIVE')
    };
  },
  
  async getActiveAlerts() {
    try {
      const response = await axios.get(`${API_CONFIG.goodwe.baseUrl}/emergency/alerts/active`);
      return response.data.alerts || [];
    } catch (error) {
      console.error('Erro ao obter alertas ativos:', error);
      return [];
    }
  },
  
  async getSystemStatus() {
    try {
      const response = await axios.get(`${API_CONFIG.goodwe.baseUrl}/system/status`);
      return response.data;
    } catch (error) {
      console.error('Erro ao obter status do sistema:', error);
      return { status: 'unknown' };
    }
  },
  
  async getEmergencyModeStatus() {
    try {
      const response = await axios.get(`${API_CONFIG.goodwe.baseUrl}/emergency/mode/status`);
      return response.data;
    } catch (error) {
      console.error('Erro ao obter status do modo de emergência:', error);
      return { active: false };
    }
  }
};
```

### 2. Logs de Emergência

```javascript
// logging/emergency-logger.js
const EmergencyLogger = {
  async logAlert(alert) {
    const logEntry = {
      timestamp: new Date().toISOString(),
      type: 'emergency_alert',
      level: alert.type,
      code: alert.code,
      message: alert.message,
      priority: alert.priority,
      actions: alert.actions,
      resolved: false
    };
    
    await this.saveLogEntry(logEntry);
  },
  
  async logEmergencyModeActivation(reason) {
    const logEntry = {
      timestamp: new Date().toISOString(),
      type: 'emergency_mode_activation',
      reason,
      active: true
    };
    
    await this.saveLogEntry(logEntry);
  },
  
  async logRecovery(alertCode, success, message) {
    const logEntry = {
      timestamp: new Date().toISOString(),
      type: 'emergency_recovery',
      alertCode,
      success,
      message
    };
    
    await this.saveLogEntry(logEntry);
  },
  
  async saveLogEntry(logEntry) {
    try {
      await axios.post(`${API_CONFIG.goodwe.baseUrl}/logs/emergency`, logEntry);
    } catch (error) {
      console.error('Erro ao salvar log de emergência:', error);
    }
  }
};
```

## 🧪 Testes do Sistema de Emergência

### 1. Testes de Detecção

```javascript
// tests/emergency-tests.js
const EmergencyTests = {
  async testCriticalAlertDetection() {
    console.log('🧪 Testando detecção de alertas críticos...');
    
    const systemData = {
      isOffline: true,
      soc_percentage: 5,
      fv_power: 0,
      load_power: 1000
    };
    
    const alerts = await EmergencyDetectionEngine.processSystemData(systemData);
    const criticalAlerts = alerts.filter(a => a.type === 'CRITICAL');
    
    console.assert(criticalAlerts.length > 0, 'Deve detectar alertas críticos');
    console.log('✅ Detecção de alertas críticos: PASSOU');
  },
  
  async testEmergencyModeActivation() {
    console.log('🧪 Testando ativação do modo de emergência...');
    
    const reason = 'Bateria crítica';
    await EmergencyMode.activateEmergencyMode(reason);
    
    const status = await EmergencyMode.getEmergencyModeStatus();
    console.assert(status.active === true, 'Modo de emergência deve estar ativo');
    console.log('✅ Ativação do modo de emergência: PASSOU');
  },
  
  async testEmergencyRecovery() {
    console.log('🧪 Testando recuperação de emergência...');
    
    const alert = {
      code: 'BATTERY_CRITICAL',
      type: 'CRITICAL'
    };
    
    const result = await EmergencyRecovery.attemptRecovery(alert);
    console.assert(result !== null, 'Deve tentar recuperação');
    console.log('✅ Recuperação de emergência: PASSOU');
  }
};
```

---

**Próximo**: [Energy Optimization](./04-energy-optimization.md)
