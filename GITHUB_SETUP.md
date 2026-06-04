# Configuração do Vision Scanner no GitHub

## Pré-requisitos

- Git instalado
- Conta GitHub
- Uma API key de LLM (OpenAI, Anthropic, Google, etc.)

## Passos para fazer push para o GitHub

### 1. Criar um novo repositório no GitHub

1. Acesse [github.com/new](https://github.com/new)
2. Preencha os campos:
   - **Repository name**: `vision-scanner`
   - **Description**: Vision Scanner - Análise de imagens com IA
   - **Visibility**: Public (ou Private, conforme preferência)
3. Clique em "Create repository"

### 2. Configurar o repositório local

```bash
cd /home/ubuntu/vision-llm-app

# Inicializar git (se ainda não estiver)
git init

# Adicionar o remote do GitHub
git remote add origin https://github.com/SEU_USUARIO/vision-scanner.git

# Adicionar todos os arquivos
git add .

# Fazer o commit inicial
git commit -m "Initial commit: Vision Scanner app with camera, LLM integration, and overlay features"

# Fazer push para o GitHub
git branch -M main
git push -u origin main
```

### 3. Configurar a API Key

O app usa a Manus Forge API por padrão. Para usar sua própria API key:

#### Opção A: Usar Manus Forge API (Recomendado)

Já está configurado! Configure a variável de ambiente:

```bash
BUILT_IN_FORGE_API_URL=https://forge.manus.im/v1/chat/completions
BUILT_IN_FORGE_API_KEY=sua_chave_aqui
```

#### Opção B: Variáveis de Ambiente Local (Desenvolvimento)

1. Crie um arquivo `.env.local` na raiz do projeto:

```bash
# Use a Manus Forge API
BUILT_IN_FORGE_API_URL=https://forge.manus.im/v1/chat/completions
BUILT_IN_FORGE_API_KEY=sua_chave_aqui
```

2. O arquivo `server/_core/llm.ts` já está configurado para usar `BUILT_IN_FORGE_API_KEY`

#### Opção B: Variáveis de Ambiente no Deploy

Se você estiver deployando em uma plataforma (Vercel, Railway, etc.):

1. Vá para as configurações do seu projeto
2. Adicione as variáveis de ambiente:
   - `OPENAI_API_KEY` (ou a chave do seu provedor)
3. Faça o deploy

### 4. Testar a integração

1. Inicie o servidor de desenvolvimento:

```bash
npm run dev
```

2. Abra a câmera e tire uma foto
3. O app deve enviar a imagem para a LLM e receber uma resposta

## Estrutura do Projeto

```
vision-scanner/
├── app/                    # Telas e rotas
├── components/             # Componentes reutilizáveis
├── hooks/                  # Custom hooks
├── server/                 # Backend com tRPC e LLM
├── tests/                  # Testes unitários
├── assets/                 # Imagens e ícones
└── package.json            # Dependências
```

## Provedores de LLM Suportados

### OpenAI (GPT-4 Vision)

```typescript
// Instalar
npm install openai

// Usar
const OpenAI = require("openai");
const client = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY,
});
```

### Anthropic (Claude)

```typescript
// Instalar
npm install @anthropic-ai/sdk

// Usar
import Anthropic from "@anthropic-ai/sdk";
const client = new Anthropic({
  apiKey: process.env.ANTHROPIC_API_KEY,
});
```

### Google (Gemini)

```typescript
// Instalar
npm install @google/generative-ai

// Usar
import { GoogleGenerativeAI } from "@google/generative-ai";
const genAI = new GoogleGenerativeAI(process.env.GOOGLE_API_KEY);
```

## Troubleshooting

### Erro: "API key not found"

- Verifique se o arquivo `.env.local` existe e tem a chave correta
- Reinicie o servidor com `npm run dev`

### Erro: "Invalid API key"

- Verifique se a chave está correta no painel do seu provedor
- Certifique-se de que a chave tem permissões para usar a API de visão

### Erro: "Rate limit exceeded"

- Aguarde alguns minutos antes de tentar novamente
- Considere atualizar seu plano no provedor de LLM

## Recursos Úteis

- [OpenAI API Documentation](https://platform.openai.com/docs)
- [Anthropic Claude API](https://docs.anthropic.com)
- [Google Gemini API](https://ai.google.dev)
- [Expo Documentation](https://docs.expo.dev)
- [React Native Documentation](https://reactnative.dev)

## Licença

MIT

## Suporte

Para dúvidas ou problemas, abra uma issue no repositório GitHub.
