# Introdução às Alexa Skills

## O que são Alexa Skills?

As Alexa Skills são aplicações de voz que estendem as capacidades da Amazon Alexa. Elas permitem que os usuários interajam com serviços e funcionalidades personalizadas através de comandos de voz.

## Arquitetura Básica

```
Usuário → Alexa Device → Amazon Voice Service → Alexa Skills Kit → Sua Aplicação/API
```

### Componentes Principais

1. **Frontend (Interaction Model)**
   - Define como os usuários irão interagir com sua skill
   - Inclui intents, utterances e slots

2. **Backend (Skill Service)**
   - Processa as requisições da Alexa
   - Contém a lógica de negócio
   - Pode ser um AWS Lambda ou endpoint HTTPS

## Conceitos Fundamentais

### 1. Invocation Name
- Nome que o usuário fala para ativar sua skill
- Exemplo: "Alexa, abra meu assistente de energia"

### 2. Intents
- Representam ações que o usuário quer realizar
- Cada intent tem utterances (frases que o usuário pode falar)

### 3. Utterances
- Exemplos de frases que ativam um intent
- Devem cobrir diferentes formas de expressar a mesma intenção

### 4. Slots
- Variáveis nas utterances
- Capturam informações específicas do usuário

### 5. Responses
- Respostas que a Alexa dá ao usuário
- Podem incluir texto, áudio, cards visuais

## Fluxo de Funcionamento

1. **Usuário fala**: "Alexa, pergunte ao meu assistente de energia qual o consumo hoje"
2. **Alexa processa** a fala e identifica:
   - Invocation name: "meu assistente de energia"
   - Intent: "GetEnergyConsumption"
   - Possíveis slots: "hoje"
3. **Alexa envia requisição** para seu backend
4. **Backend processa** e retorna resposta
5. **Alexa fala** a resposta para o usuário

## Tipos de Skills

### 1. Custom Skills
- Mais flexível
- Você define os intents e utterances
- Adequado para casos específicos

### 2. Smart Home Skills
- Para controle de dispositivos IoT
- Intents predefinidos (ligar, desligar, etc.)

### 3. Flash Briefing Skills
- Para notícias e atualizações
- Formato mais simples

## Plataformas de Desenvolvimento

### 1. Alexa Developer Console
- Interface web para configurar skills
- Define interaction model
- Testa skills

### 2. AWS Lambda
- Serviço serverless para backend
- Integração nativa com Alexa
- Escalabilidade automática

### 3. Alexa Skills Kit SDK
- Bibliotecas para desenvolvimento
- Disponível em Python, Node.js, Java
- Facilita desenvolvimento do backend

## Próximos Passos

1. Entender como criar e configurar intents (próximo arquivo)
2. Aprender a conectar com APIs externas
3. Implementar exemplos práticos
4. Testar e publicar sua skill

---

*Este documento faz parte da documentação de desenvolvimento de Alexa Skills para o projeto GoodWe.*
