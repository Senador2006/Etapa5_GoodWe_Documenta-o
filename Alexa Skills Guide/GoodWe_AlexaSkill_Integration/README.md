# GoodWe Alexa Skill Integration

## 📋 Visão Geral

Esta seção contém documentação específica para a integração da Alexa Skill com as APIs GoodWe, incluindo configurações detalhadas, exemplos de código e guias de implementação.

## 🏗️ Estrutura da Integração

```
GoodWe_AlexaSkill_Integration/
├── 01-api-integration.md         # Integração com APIs
├── 02-smart-home-integration.md  # Integração Smart Home
├── 03-emergency-management.md    # Gerenciamento de emergências
├── 04-energy-optimization.md     # Otimização energética
├── 05-complete-implementation.md # Implementação completa
└── README.md                     # Este arquivo
```

## 🎯 Funcionalidades da Integração

### APIs Integradas
- **API Principal GoodWe**: Dados de monitoramento e análises
- **API Machine Learning**: Predições climáticas e otimizações
- **APIs Externas**: Serviços de clima e concessionárias

### Recursos Smart Home
- **Descoberta de Dispositivos**: Inversores, baterias, controladores
- **Controle por Voz**: Comandos de ligar/desligar e configuração
- **Automações**: Regras baseadas em dados solares

### Gerenciamento de Emergências
- **Alertas Inteligentes**: Notificações baseadas em condições
- **Modo de Emergência**: Ativação automática de procedimentos
- **Comunicação**: Notificações para múltiplos canais

### Otimização Energética
- **Análises Preditivas**: Previsões de geração e consumo
- **Recomendações**: Sugestões de otimização
- **Relatórios**: Análises detalhadas de performance

## 🚀 Início Rápido

### Pré-requisitos
- APIs GoodWe rodando (portas 3000 e 8000)
- Conta Amazon Developer configurada
- AWS Lambda function criada
- Variáveis de ambiente configuradas

### Configuração Básica
```bash
# Clonar repositório
git clone <repository-url>
cd GoodWe_AlexaSkill_Integration

# Instalar dependências
npm install

# Configurar variáveis de ambiente
cp .env.example .env
# Editar .env com suas configurações

# Deploy da skill
ask deploy
```

### Teste Inicial
```bash
# Testar skill
ask simulate --text "qual o status do sistema" --locale pt-BR

# Verificar logs
ask get-skill-status
```

## 📚 Documentação por Módulo

### 1. Integração com APIs
- [01-api-integration.md](./01-api-integration.md)
- Configuração de endpoints
- Autenticação e segurança
- Tratamento de erros
- Cache e otimização

### 2. Smart Home
- [02-smart-home-integration.md](./02-smart-home-integration.md)
- Descoberta de dispositivos
- Controles por voz
- Automações inteligentes
- Integração com IoT

### 3. Gerenciamento de Emergências
- [03-emergency-management.md](./03-emergency-management.md)
- Sistema de alertas
- Procedimentos de emergência
- Notificações automáticas
- Recuperação de falhas

### 4. Otimização Energética
- [04-energy-optimization.md](./04-energy-optimization.md)
- Análises preditivas
- Recomendações inteligentes
- Relatórios detalhados
- Otimização automática

### 5. Implementação Completa
- [05-complete-implementation.md](./05-complete-implementation.md)
- Código completo da skill
- Configurações finais
- Deploy e testes
- Monitoramento

## 🔧 Configurações

### Variáveis de Ambiente
```bash
# APIs GoodWe
GOODWE_API_URL=http://localhost:3000
ML_API_URL=http://localhost:8000

# Autenticação
GOODWE_API_KEY=your_api_key
ML_API_KEY=your_ml_key

# Serviços externos
WEATHER_API_KEY=your_weather_key
UTILITY_API_KEY=your_utility_key

# Configurações da skill
SKILL_ID=amzn1.ask.skill.your-skill-id
LAMBDA_FUNCTION_NAME=GoodWeSolarAssistant

# Debug e logs
DEBUG=false
LOG_LEVEL=info
```

### Configuração da Lambda
```javascript
// Configuração da função Lambda
const config = {
  goodwe: {
    baseUrl: process.env.GOODWE_API_URL,
    apiKey: process.env.GOODWE_API_KEY,
    timeout: 5000
  },
  ml: {
    baseUrl: process.env.ML_API_URL,
    apiKey: process.env.ML_API_KEY,
    timeout: 10000
  },
  weather: {
    apiKey: process.env.WEATHER_API_KEY,
    timeout: 5000
  }
};
```

## 🧪 Testes

### Testes Unitários
```bash
# Executar testes
npm test

# Testes com cobertura
npm run test:coverage

# Testes de integração
npm run test:integration
```

### Testes de API
```bash
# Testar APIs GoodWe
curl http://localhost:3000/health
curl http://localhost:8000/health

# Testar endpoints específicos
curl http://localhost:3000/data/paginated?limit=1
curl -X POST http://localhost:8000/predict -H "Content-Type: application/json" -d '{"temperatura_celsius":25,"umidade_pct":65,"precipitacao_mm_h":10,"vento_kmh":30,"pressao_hpa":1013}'
```

### Testes da Skill
```bash
# Simular comandos
ask simulate --text "qual o status do sistema" --locale pt-BR
ask simulate --text "quanta energia estou gerando" --locale pt-BR
ask simulate --text "qual o nível da bateria" --locale pt-BR

# Teste de diálogo
ask dialog --locale pt-BR
```

## 📊 Monitoramento

### Métricas Importantes
- **Uptime**: Disponibilidade das APIs
- **Response Time**: Tempo de resposta
- **Error Rate**: Taxa de erros
- **Intent Success**: Taxa de sucesso dos intents

### Logs e Debugging
- **CloudWatch Logs**: Logs da Lambda
- **API Logs**: Logs das APIs GoodWe
- **Debug Mode**: Modo de debug ativado

### Alertas
- **API Down**: APIs indisponíveis
- **High Error Rate**: Taxa de erro alta
- **Slow Response**: Resposta lenta
- **Critical Alerts**: Alertas críticos

## 🔒 Segurança

### Autenticação
- **API Keys**: Chaves de API para autenticação
- **JWT Tokens**: Tokens para sessões
- **Rate Limiting**: Limitação de requisições

### Privacidade
- **Dados Criptografados**: Dados em trânsito e em repouso
- **Logs Anonimizados**: Logs sem dados pessoais
- **Conformidade LGPD**: Conformidade com a lei brasileira

## 🤝 Contribuição

### Como Contribuir
1. Fork do repositório
2. Criar branch para feature
3. Implementar mudanças
4. Testes e documentação
5. Pull request

### Padrões de Código
- **ESLint**: Linting do código JavaScript
- **Prettier**: Formatação do código
- **JSDoc**: Documentação das funções
- **Commits Semânticos**: Padrão de commits

## 📞 Suporte

### Canais de Suporte
- **GitHub Issues**: Para bugs e features
- **Email**: suporte@goodwe-alexa.com
- **Documentação**: [docs.goodwe-alexa.com](https://docs.goodwe-alexa.com)

### Recursos Adicionais
- **Documentação Amazon Alexa**: [developer.amazon.com/alexa](https://developer.amazon.com/alexa)
- **API GoodWe**: [api.goodwe.com/docs](https://api.goodwe.com/docs)
- **Machine Learning API**: [ml-api.goodwe.com/docs](https://ml-api.goodwe.com/docs)

---

**Versão**: 1.0.0  
**Última Atualização**: Janeiro 2025  
**Autor**: Equipe GoodWe Alexa Integration
