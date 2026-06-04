# Guia de Integração com Múltiplos Provedores de LLM

Este guia explica como integrar diferentes provedores de LLM no Vision Scanner além do Forge padrão.

## Visão Geral da Arquitetura Atual

O app usa um sistema modular onde a lógica de LLM está centralizada em `server/_core/llm.ts`. Atualmente, ele está configurado para usar a Manus Forge API, mas pode ser facilmente adaptado para outros provedores.

### Arquivo Principal: `server/_core/llm.ts`

```typescript
// Localização: server/_core/llm.ts
// Função principal: invokeLLM(params: InvokeParams): Promise<InvokeResult>
```

## Integração com OpenAI (GPT-4 Vision)

### Passo 1: Instalar a biblioteca OpenAI

```bash
npm install openai
```

### Passo 2: Adicionar variável de ambiente

Crie ou edite o arquivo `.env.local`:

```bash
OPENAI_API_KEY=sk-proj-xxxxxxxxxxxxx
```

### Passo 3: Criar um novo arquivo de integração

Crie `server/_core/llm-openai.ts`:

```typescript
import OpenAI from "openai";
import { InvokeParams, InvokeResult } from "./llm";

const client = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY,
});

export async function invokeOpenAI(params: InvokeParams): Promise<InvokeResult> {
  const { messages } = params;

  // Converter mensagens para formato OpenAI
  const openaiMessages = messages.map((msg) => {
    if (typeof msg.content === "string") {
      return {
        role: msg.role as "system" | "user" | "assistant",
        content: msg.content,
      };
    }

    // Suporte a conteúdo multimodal (imagens)
    if (Array.isArray(msg.content)) {
      const content = msg.content.map((part: any) => {
        if (part.type === "text") {
          return {
            type: "text",
            text: part.text,
          };
        }

        if (part.type === "image_url") {
          return {
            type: "image_url",
            image_url: {
              url: part.image_url.url,
              detail: part.image_url.detail || "auto",
            },
          };
        }

        return null;
      }).filter(Boolean);

      return {
        role: msg.role as "system" | "user" | "assistant",
        content,
      };
    }

    return {
      role: msg.role as "system" | "user" | "assistant",
      content: JSON.stringify(msg.content),
    };
  });

  // Chamar API OpenAI
  const response = await client.chat.completions.create({
    model: "gpt-4-vision-preview", // ou "gpt-4-turbo" com visão
    messages: openaiMessages as any,
    max_tokens: params.maxTokens || params.max_tokens || 2048,
  });

  // Converter resposta para formato padrão
  return {
    id: response.id,
    created: response.created,
    model: response.model,
    choices: response.choices.map((choice) => ({
      index: choice.index,
      message: {
        role: "assistant",
        content: choice.message.content || "",
      },
      finish_reason: choice.finish_reason,
    })),
    usage: response.usage
      ? {
          prompt_tokens: response.usage.prompt_tokens,
          completion_tokens: response.usage.completion_tokens,
          total_tokens: response.usage.total_tokens,
        }
      : undefined,
  };
}
```

### Passo 4: Atualizar `server/_core/llm.ts`

Modifique a função `invokeLLM` para usar OpenAI:

```typescript
import { invokeOpenAI } from "./llm-openai";

export async function invokeLLM(params: InvokeParams): Promise<InvokeResult> {
  // Se OPENAI_API_KEY está definida, usar OpenAI
  if (process.env.OPENAI_API_KEY) {
    return invokeOpenAI(params);
  }

  // Caso contrário, usar Forge padrão
  assertApiKey();
  // ... resto do código Forge
}
```

## Integração com Anthropic Claude

### Passo 1: Instalar a biblioteca Anthropic

```bash
npm install @anthropic-ai/sdk
```

### Passo 2: Adicionar variável de ambiente

```bash
ANTHROPIC_API_KEY=sk-ant-xxxxxxxxxxxxx
```

### Passo 3: Criar arquivo de integração

Crie `server/_core/llm-anthropic.ts`:

```typescript
import Anthropic from "@anthropic-ai/sdk";
import { InvokeParams, InvokeResult } from "./llm";

const client = new Anthropic({
  apiKey: process.env.ANTHROPIC_API_KEY,
});

export async function invokeAnthropic(params: InvokeParams): Promise<InvokeResult> {
  const { messages } = params;

  // Converter mensagens para formato Anthropic
  const anthropicMessages = messages
    .filter((msg) => msg.role !== "system") // Claude não suporta role "system"
    .map((msg) => {
      if (typeof msg.content === "string") {
        return {
          role: msg.role as "user" | "assistant",
          content: msg.content,
        };
      }

      // Suporte a conteúdo multimodal
      if (Array.isArray(msg.content)) {
        const content = msg.content.map((part: any) => {
          if (part.type === "text") {
            return {
              type: "text",
              text: part.text,
            };
          }

          if (part.type === "image_url") {
            // Claude requer base64 ou URL pública
            return {
              type: "image",
              source: {
                type: "url",
                url: part.image_url.url,
              },
            };
          }

          return null;
        }).filter(Boolean);

        return {
          role: msg.role as "user" | "assistant",
          content,
        };
      }

      return {
        role: msg.role as "user" | "assistant",
        content: JSON.stringify(msg.content),
      };
    });

  // Extrair mensagem de sistema se existir
  const systemMessage = messages
    .filter((msg) => msg.role === "system")
    .map((msg) => (typeof msg.content === "string" ? msg.content : JSON.stringify(msg.content)))
    .join("\n");

  // Chamar API Anthropic
  const response = await client.messages.create({
    model: "claude-3-5-sonnet-20241022", // ou claude-3-opus-20240229
    max_tokens: params.maxTokens || params.max_tokens || 2048,
    system: systemMessage || undefined,
    messages: anthropicMessages as any,
  });

  // Converter resposta
  return {
    id: response.id,
    created: Math.floor(Date.now() / 1000),
    model: response.model,
    choices: [
      {
        index: 0,
        message: {
          role: "assistant",
          content: response.content
            .filter((block) => block.type === "text")
            .map((block) => (block.type === "text" ? block.text : ""))
            .join(""),
        },
        finish_reason: response.stop_reason,
      },
    ],
    usage: {
      prompt_tokens: response.usage.input_tokens,
      completion_tokens: response.usage.output_tokens,
      total_tokens: response.usage.input_tokens + response.usage.output_tokens,
    },
  };
}
```

### Passo 4: Atualizar `server/_core/llm.ts`

```typescript
import { invokeAnthropic } from "./llm-anthropic";

export async function invokeLLM(params: InvokeParams): Promise<InvokeResult> {
  if (process.env.ANTHROPIC_API_KEY) {
    return invokeAnthropic(params);
  }

  // ... resto do código
}
```

## Integração com Google Gemini

### Passo 1: Instalar a biblioteca Google

```bash
npm install @google/generative-ai
```

### Passo 2: Adicionar variável de ambiente

```bash
GOOGLE_API_KEY=AIzaSyxxxxxxxxxxxxx
```

### Passo 3: Criar arquivo de integração

Crie `server/_core/llm-google.ts`:

```typescript
import { GoogleGenerativeAI } from "@google/generative-ai";
import { InvokeParams, InvokeResult } from "./llm";

const genAI = new GoogleGenerativeAI(process.env.GOOGLE_API_KEY!);

export async function invokeGoogle(params: InvokeParams): Promise<InvokeResult> {
  const { messages } = params;
  const model = genAI.getGenerativeModel({ model: "gemini-2.0-flash" });

  // Converter mensagens para formato Google
  const googleMessages = messages
    .filter((msg) => msg.role !== "system")
    .map((msg) => {
      if (typeof msg.content === "string") {
        return {
          role: msg.role === "user" ? "user" : "model",
          parts: [{ text: msg.content }],
        };
      }

      if (Array.isArray(msg.content)) {
        const parts = msg.content
          .map((part: any) => {
            if (part.type === "text") {
              return { text: part.text };
            }

            if (part.type === "image_url") {
              // Google Gemini suporta URLs públicas
              return {
                inlineData: {
                  mimeType: "image/jpeg",
                  data: part.image_url.url, // Deve ser base64 ou URL
                },
              };
            }

            return null;
          })
          .filter(Boolean);

        return {
          role: msg.role === "user" ? "user" : "model",
          parts,
        };
      }

      return {
        role: msg.role === "user" ? "user" : "model",
        parts: [{ text: JSON.stringify(msg.content) }],
      };
    });

  // Chamar API Google
  const response = await model.generateContent({
    contents: googleMessages as any,
    generationConfig: {
      maxOutputTokens: params.maxTokens || params.max_tokens || 2048,
    },
  });

  const text = response.response.text();

  return {
    id: `google-${Date.now()}`,
    created: Math.floor(Date.now() / 1000),
    model: "gemini-2.0-flash",
    choices: [
      {
        index: 0,
        message: {
          role: "assistant",
          content: text,
        },
        finish_reason: "stop",
      },
    ],
  };
}
```

### Passo 4: Atualizar `server/_core/llm.ts`

```typescript
import { invokeGoogle } from "./llm-google";

export async function invokeLLM(params: InvokeParams): Promise<InvokeResult> {
  if (process.env.GOOGLE_API_KEY) {
    return invokeGoogle(params);
  }

  // ... resto do código
}
```

## Estratégia de Prioridade

Para usar múltiplos provedores com prioridade, atualize `server/_core/llm.ts`:

```typescript
export async function invokeLLM(params: InvokeParams): Promise<InvokeResult> {
  // Prioridade 1: OpenAI
  if (process.env.OPENAI_API_KEY) {
    return invokeOpenAI(params);
  }

  // Prioridade 2: Anthropic
  if (process.env.ANTHROPIC_API_KEY) {
    return invokeAnthropic(params);
  }

  // Prioridade 3: Google
  if (process.env.GOOGLE_API_KEY) {
    return invokeGoogle(params);
  }

  // Fallback: Forge padrão
  assertApiKey();
  // ... código Forge
}
```

## Configuração de Variáveis de Ambiente

### Arquivo `.env.local` (Desenvolvimento)

```bash
# Escolha UM provedor:

# OpenAI
OPENAI_API_KEY=sk-proj-xxxxxxxxxxxxx

# OU Anthropic
ANTHROPIC_API_KEY=sk-ant-xxxxxxxxxxxxx

# OU Google
GOOGLE_API_KEY=AIzaSyxxxxxxxxxxxxx

# OU Forge (padrão)
BUILT_IN_FORGE_API_KEY=xxxxxxxxxxxxx
```

### Deploy (Vercel, Railway, etc.)

1. Vá para as configurações do seu projeto
2. Adicione as variáveis de ambiente
3. Faça o deploy

## Comparação de Provedores

| Provedor | Modelo | Visão | Custo | Limite | Suporte |
|----------|--------|-------|-------|--------|---------|
| OpenAI | GPT-4 Vision | ✅ | $$ | 128K tokens | Excelente |
| Anthropic | Claude 3.5 | ✅ | $$ | 200K tokens | Muito Bom |
| Google | Gemini 2.0 | ✅ | $ | 1M tokens | Bom |
| Forge | Gemini 2.5 | ✅ | Incluído | Ilimitado | Manus |

## Tratamento de Erros

Adicione tratamento robusto de erros:

```typescript
export async function invokeLLM(params: InvokeParams): Promise<InvokeResult> {
  try {
    if (process.env.OPENAI_API_KEY) {
      return await invokeOpenAI(params);
    }

    // ... outros provedores
  } catch (error) {
    console.error("LLM Error:", error);

    // Fallback para Forge
    if (process.env.BUILT_IN_FORGE_API_KEY) {
      console.log("Falling back to Forge API");
      return invokeForgeFallback(params);
    }

    throw new Error("No LLM provider available");
  }
}
```

## Dicas de Otimização

### 1. Cache de Respostas

```typescript
const cache = new Map<string, InvokeResult>();

export async function invokeLLMWithCache(params: InvokeParams): Promise<InvokeResult> {
  const cacheKey = JSON.stringify(params.messages);

  if (cache.has(cacheKey)) {
    return cache.get(cacheKey)!;
  }

  const result = await invokeLLM(params);
  cache.set(cacheKey, result);

  return result;
}
```

### 2. Rate Limiting

```typescript
import pLimit from "p-limit";

const limit = pLimit(10); // Máximo 10 requisições simultâneas

export async function invokeLLMWithLimit(params: InvokeParams): Promise<InvokeResult> {
  return limit(() => invokeLLM(params));
}
```

### 3. Retry com Backoff

```typescript
export async function invokeLLMWithRetry(
  params: InvokeParams,
  maxRetries = 3
): Promise<InvokeResult> {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await invokeLLM(params);
    } catch (error) {
      if (i === maxRetries - 1) throw error;

      // Exponential backoff
      const delay = Math.pow(2, i) * 1000;
      await new Promise((resolve) => setTimeout(resolve, delay));
    }
  }

  throw new Error("Max retries exceeded");
}
```

## Testando a Integração

### Teste Local

```bash
# Instale a dependência do provedor
npm install openai

# Configure a variável de ambiente
export OPENAI_API_KEY=sk-proj-xxxxxxxxxxxxx

# Inicie o servidor
npm run dev

# Tire uma foto no app e veja se funciona
```

### Teste com cURL

```bash
curl -X POST http://localhost:3000/api/trpc/vision.analyzeImage \
  -H "Content-Type: application/json" \
  -d '{
    "image": "data:image/jpeg;base64,...",
    "prompt": "Descreva esta imagem"
  }'
```

## Troubleshooting

### Erro: "API key not found"

- Verifique se a variável de ambiente está definida
- Reinicie o servidor com `npm run dev`
- Verifique o arquivo `.env.local`

### Erro: "Invalid API key"

- Verifique se a chave está correta no painel do provedor
- Certifique-se de que a chave tem permissões para usar a API de visão
- Algumas chaves podem estar desativadas por segurança

### Erro: "Rate limit exceeded"

- Aguarde alguns minutos antes de tentar novamente
- Implemente retry com backoff (veja seção de otimização)
- Considere atualizar seu plano

### Erro: "Model not available"

- Verifique se o modelo está disponível em sua região
- Alguns modelos podem estar em beta ou preview
- Consulte a documentação do provedor

## Recursos Úteis

- [OpenAI API Docs](https://platform.openai.com/docs/api-reference/chat/create)
- [Anthropic Claude API](https://docs.anthropic.com/en/api/getting-started)
- [Google Gemini API](https://ai.google.dev/tutorials/python_quickstart)
- [Manus Forge API](https://forge.manus.im/docs)

## Próximos Passos

1. Escolha um provedor
2. Obtenha uma API key
3. Siga os passos de integração
4. Teste com uma imagem
5. Configure o ambiente de produção

Boa sorte! 🚀
