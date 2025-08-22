# Documentação das APIs do Projeto GoodWe

Este diretório contém toda a documentação técnica das APIs desenvolvidas para o projeto GoodWe. O sistema é composto por duas APIs principais que trabalham em conjunto para fornecer funcionalidades de conversão de dados e predição de quedas de energia.

## 📋 Visão Geral

O projeto GoodWe consiste em:

- **API_GoodWe**: API REST em Node.js para conversão de arquivos Excel
- **Machine_Learning_GoodWe**: API FastAPI em Python para predição de quedas de energia usando Machine Learning

## 📚 Documentação Disponível

### 📋 Navegação
- [**📖 Índice Completo**](./index.md) - Navegação organizada por perfil e funcionalidade
- [**❓ FAQ**](./faq.md) - Perguntas frequentes categorizadas

### 🔧 APIs Principais
- [**📊 API Excel (Node.js)**](./api-excel.md) - Documentação completa da API de conversão Excel
- [**🧠 API Machine Learning (Python)**](./api-machine-learning.md) - Documentação da API de predição de quedas de energia

### 🚀 Deployment e Configuração
- [**🌐 Guia de Deployment**](./deployment.md) - Como configurar e executar as APIs em produção
- [**🐳 Configuração Docker**](./docker-configuration.md) - Setup detalhado com Docker e containers

### 💡 Exemplos e Casos de Uso
- [**💻 Exemplos de Uso**](./examples.md) - Códigos práticos e exemplos de integração
- [**🏭 Casos de Uso**](./use-cases.md) - Cenários reais de aplicação das APIs

### 🛠️ Suporte e Manutenção
- [**🔧 Troubleshooting**](./troubleshooting.md) - Soluções para problemas comuns e debug
- [**❓ FAQ**](./faq.md) - Perguntas frequentes organizadas por categoria

## 🏗️ Arquitetura do Sistema

```
Projeto GoodWe
├── API_GoodWe (Node.js)
│   ├── Conversão Excel → JSON
│   ├── Upload de arquivos
│   └── Gerenciamento de dados
│
├── Machine_Learning_GoodWe (Python)
│   ├── Predição de quedas de energia
│   ├── Análise de condições climáticas
│   └── Modelo Random Forest
│
└── Docs (Documentação)
    ├── APIs endpoints
    ├── Deployment guides
    └── Exemplos práticos
```

## ⚡ Quick Start

### 1. API Excel (Port 3000)
```bash
cd API_GoodWe
npm install
npm start
```
Acesse: http://localhost:3000

### 2. API Machine Learning (Port 8000)
```bash
cd Machine_learning_GoodWe
pip install -r requirements.txt
python start_api.py
```
Acesse: http://localhost:8000

## 🔗 URLs Principais

- **API Excel**: http://localhost:3000
- **API ML**: http://localhost:8000
- **Documentação ML**: http://localhost:8000/docs (Swagger UI)

## 🤝 Integração entre APIs

As duas APIs podem trabalhar em conjunto:

1. **API Excel** converte dados de planilhas para JSON
2. **API ML** utiliza dados climáticos para predições
3. Dados convertidos podem alimentar o modelo de ML

## 📊 Tecnologias Utilizadas

### API Excel (Node.js)
- **Express.js** - Framework web
- **XLSX** - Manipulação de arquivos Excel
- **Multer** - Upload de arquivos
- **Helmet** - Segurança
- **CORS** - Cross-origin requests

### API Machine Learning (Python)
- **FastAPI** - Framework web moderno
- **Scikit-learn** - Machine Learning
- **Pandas** - Manipulação de dados
- **NumPy** - Computação científica
- **Pydantic** - Validação de dados

## 🔐 Segurança

- Validação de tipos de arquivo
- Limitação de tamanho de upload (10MB)
- Headers de segurança com Helmet
- Validação de entrada com Pydantic
- Tratamento robusto de erros

## 📈 Monitoramento

Ambas as APIs incluem:
- Health check endpoints
- Logging estruturado
- Tratamento de erros
- Métricas de performance

---

**Última atualização**: Janeiro 2025  
**Versão**: 1.0.0  
**Status**: Ativo
