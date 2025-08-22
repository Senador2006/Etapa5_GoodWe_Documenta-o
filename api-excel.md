# API Excel - Documentação Técnica

A API Excel é um serviço REST desenvolvido em Node.js que fornece funcionalidades para conversão de arquivos Excel (.xls/.xlsx) para formato JSON, com suporte a upload de arquivos e processamento em tempo real.

## 📋 Informações Gerais

- **Nome**: API GoodWe
- **Versão**: 1.0.0
- **Porta**: 3000
- **Base URL**: `http://localhost:3000`
- **Tecnologia**: Node.js + Express.js

## 🔧 Dependências Principais

```json
{
  "express": "^4.18.2",
  "xlsx": "^0.18.5",
  "multer": "^2.0.2",
  "cors": "^2.8.5",
  "helmet": "^7.1.0",
  "morgan": "^1.10.0"
}
```

## 🚀 Endpoints Disponíveis

### 1. **GET /** - Página Principal
Retorna informações gerais da API e lista de endpoints disponíveis.

**URL**: `GET /`

**Resposta de Sucesso**:
```json
{
  "message": "API GoodWe funcionando!",
  "version": "1.0.0",
  "timestamp": "2025-01-20T10:30:00.000Z",
  "availableEndpoints": {
    "GET /": "Esta página",
    "GET /health": "Health check",
    "GET /excel": "Informações do conversor Excel",
    "GET /excel/convert": "Converter arquivo Potência Vegetal",
    "POST /excel/upload": "Upload e conversão de Excel",
    "GET /excel/files": "Listar arquivos Excel"
  }
}
```

### 2. **GET /health** - Health Check
Verifica se a API está funcionando corretamente.

**URL**: `GET /health`

**Resposta de Sucesso**:
```json
{
  "status": "OK",
  "uptime": 3600.123,
  "timestamp": "2025-01-20T10:30:00.000Z"
}
```

### 3. **GET /excel** - Informações do Conversor
Retorna informações sobre o módulo de conversão Excel.

**URL**: `GET /excel`

**Resposta de Sucesso**:
```json
{
  "message": "Conversor Excel para JSON",
  "endpoints": {
    "GET /excel/convert": "Converte o arquivo Potência Vegetal para JSON",
    "POST /excel/upload": "Faz upload e converte arquivo Excel para JSON",
    "GET /excel/files": "Lista arquivos Excel disponíveis",
    "GET /excel": "Esta página de informações"
  },
  "supportedFormats": [".xls", ".xlsx"],
  "maxFileSize": "10MB"
}
```

### 4. **GET /excel/convert** - Converter Arquivo Específico
Converte o arquivo `Potência Vegetal_20250820190939.xls` para JSON.

**URL**: `GET /excel/convert`

**Descrição**: Este endpoint converte um arquivo Excel específico já presente no sistema.

**Resposta de Sucesso**:
```json
{
  "message": "Arquivo Excel convertido com sucesso!",
  "originalFile": "Potência Vegetal_20250820190939.xls",
  "convertedFile": "potencia_vegetal.json",
  "sheets": ["Planilha1", "Planilha2"],
  "data": {
    "Planilha1": [
      ["Coluna1", "Coluna2", "Coluna3"],
      ["Valor1", "Valor2", "Valor3"]
    ],
    "Planilha2": [
      ["Header1", "Header2"],
      ["Data1", "Data2"]
    ]
  },
  "totalSheets": 2,
  "savedAt": "/path/to/potencia_vegetal.json"
}
```

**Possíveis Erros**:
```json
{
  "error": "Arquivo não encontrado",
  "path": "/path/to/file.xls"
}
```

### 5. **POST /excel/upload** - Upload e Conversão
Permite upload de arquivo Excel e conversão para JSON em tempo real.

**URL**: `POST /excel/upload`

**Content-Type**: `multipart/form-data`

**Parâmetros**:
- `excel` (file): Arquivo Excel (.xls ou .xlsx)

**Validações**:
- Tipos aceitos: `application/vnd.ms-excel`, `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`
- Tamanho máximo: 10MB

**Exemplo de Requisição**:
```bash
curl -X POST \
  http://localhost:3000/excel/upload \
  -H 'Content-Type: multipart/form-data' \
  -F 'excel=@arquivo.xlsx'
```

**Resposta de Sucesso**:
```json
{
  "message": "Arquivo Excel convertido com sucesso!",
  "originalFile": "arquivo.xlsx",
  "fileSize": 1048576,
  "sheets": ["Sheet1", "Sheet2"],
  "data": {
    "Sheet1": [
      ["A1", "B1", "C1"],
      ["A2", "B2", "C2"]
    ]
  },
  "totalSheets": 1
}
```

**Possíveis Erros**:
```json
{
  "error": "Nenhum arquivo foi enviado",
  "message": "Envie um arquivo Excel (.xls ou .xlsx)"
}
```

```json
{
  "error": "Apenas arquivos Excel (.xls, .xlsx) são permitidos"
}
```

### 6. **GET /excel/files** - Listar Arquivos
Lista todos os arquivos Excel disponíveis na pasta de dados.

**URL**: `GET /excel/files`

**Resposta de Sucesso**:
```json
{
  "message": "Arquivos Excel encontrados",
  "count": 2,
  "files": [
    {
      "name": "Potência Vegetal_20250820190939.xls",
      "size": 2048576,
      "created": "2025-01-20T08:00:00.000Z",
      "modified": "2025-01-20T09:30:00.000Z"
    },
    {
      "name": "dados_2024.xlsx",
      "size": 1024768,
      "created": "2025-01-19T15:00:00.000Z",
      "modified": "2025-01-19T16:00:00.000Z"
    }
  ],
  "dataPath": "/path/to/data"
}
```

## 🔒 Configurações de Segurança

### Middlewares Aplicados
- **Helmet**: Headers de segurança HTTP
- **CORS**: Configuração de Cross-Origin Resource Sharing
- **Morgan**: Logging de requisições
- **Express.json()**: Parsing de JSON com limite de tamanho
- **Express.urlencoded()**: Parsing de dados de formulário

### Validações de Upload
```javascript
// Configuração Multer
const upload = multer({
  storage: multer.memoryStorage(),
  fileFilter: (req, file, cb) => {
    if (file.mimetype === 'application/vnd.ms-excel' || 
        file.mimetype === 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet') {
      cb(null, true);
    } else {
      cb(new Error('Apenas arquivos Excel (.xls, .xlsx) são permitidos'), false);
    }
  },
  limits: {
    fileSize: 10 * 1024 * 1024 // 10MB
  }
});
```

## 📁 Estrutura de Diretórios

```
API_GoodWe/
├── controllers/
│   └── excelController.js    # Lógica de negócio
├── routes/
│   └── excel.js             # Definição de rotas
├── scr/data/               # Arquivos de dados
│   ├── *.xls               # Arquivos Excel originais
│   └── *.json              # Arquivos convertidos
├── index.js                # Ponto de entrada
└── package.json           # Dependências
```

## 🔄 Fluxo de Processamento

### Conversão Excel → JSON
1. **Leitura**: Arquivo Excel é lido usando a biblioteca XLSX
2. **Parsing**: Cada planilha é convertida para array de arrays
3. **Filtração**: Remoção de linhas completamente vazias
4. **Estruturação**: Organização em objeto JSON por planilha
5. **Salvamento**: Arquivo JSON é salvo (para arquivos específicos)
6. **Resposta**: Dados são retornados via API

### Exemplo de Transformação
**Excel Original**:
```
| Nome     | Idade | Cidade    |
|----------|-------|-----------|
| João     | 30    | São Paulo |
| Maria    | 25    | Rio       |
```

**JSON Resultante**:
```json
{
  "Planilha1": [
    ["Nome", "Idade", "Cidade"],
    ["João", 30, "São Paulo"],
    ["Maria", 25, "Rio"]
  ]
}
```

## 🚨 Tratamento de Erros

### Códigos de Status HTTP
- **200**: Sucesso
- **400**: Erro de validação (arquivo inválido)
- **404**: Recurso não encontrado
- **500**: Erro interno do servidor
- **503**: Serviço indisponível

### Estrutura de Erro Padrão
```json
{
  "error": "Descrição do erro",
  "message": "Mensagem detalhada",
  "details": "Stack trace (apenas em desenvolvimento)"
}
```

## 🔧 Configuração e Execução

### Variáveis de Ambiente
```bash
PORT=3000                    # Porta do servidor
NODE_ENV=development         # Ambiente de execução
```

### Comandos NPM
```bash
npm install                  # Instalar dependências
npm start                   # Iniciar em produção
npm run dev                 # Iniciar em desenvolvimento
```

### Docker
```bash
docker build -t api-goodwe .
docker run -p 3000:3000 api-goodwe
```

## 📊 Monitoramento e Logs

### Health Check
O endpoint `/health` fornece:
- Status da aplicação
- Tempo de atividade
- Timestamp atual

### Logs de Requisição
Formato Morgan 'combined':
```
IP - - [timestamp] "METHOD /path HTTP/version" status size "referer" "user-agent"
```

## 🧪 Testes

### Teste Manual via cURL
```bash
# Health check
curl http://localhost:3000/health

# Upload de arquivo
curl -X POST \
  http://localhost:3000/excel/upload \
  -F 'excel=@test.xlsx'

# Conversão específica
curl http://localhost:3000/excel/convert
```

### Teste com Postman
1. Importar coleção com endpoints
2. Configurar environment (base_url: localhost:3000)
3. Executar testes de cada endpoint

---

**Próxima seção**: [API Machine Learning](./api-machine-learning.md)
