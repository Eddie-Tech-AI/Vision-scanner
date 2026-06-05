# Explicação Detalhada dos 10 Diagramas de Fluxo

## Diagrama 1: Fluxo Geral de Processamento de Imagem

### Visão Geral
Este é o fluxo mais alto nível do aplicativo. Mostra o caminho completo desde o usuário capturando uma foto até receber a análise final.

### Etapas Detalhadas

**1. Usuário tira foto na câmera**
```
O usuário abre o app → Navega para a tela de câmera → Aperta o botão de captura
```
- A câmera do dispositivo é ativada via `expo-camera`
- A foto é capturada em alta resolução (geralmente 1080p ou superior)
- A imagem fica armazenada temporariamente na memória

**2. Imagem convertida para Base64**
```
Imagem JPEG/PNG → Codificação Base64 → String de texto
```
- A imagem é lida do sistema de arquivos
- Convertida para Base64 (aumenta tamanho em ~33%)
- Exemplo: `data:image/jpeg;base64,/9j/4AAQSkZJRg...`

**3. Enviada para servidor tRPC**
```
Cliente (React Native) → HTTP POST → Servidor Node.js
```
- Usa a biblioteca `@trpc/client` para comunicação
- Payload contém: `{ image: string, prompt: string, outputMode: 'text' | 'overlay' }`
- Exemplo de requisição:
```typescript
const result = await client.vision.analyzeImage.mutate({
  image: "data:image/jpeg;base64,...",
  prompt: "Descreva esta imagem",
  outputMode: "text"
});
```

**4. Decisão: Qual provedor de LLM?**
```
Sistema verifica variáveis de ambiente em ordem:
1. OPENAI_API_KEY → Usar OpenAI
2. ANTHROPIC_API_KEY → Usar Anthropic
3. GOOGLE_API_KEY → Usar Google
4. BUILT_IN_FORGE_API_KEY → Usar Forge
```

**5. Processamento com LLM**
- Cada provedor recebe a imagem em seu formato específico
- Processa o prompt do usuário
- Retorna uma resposta em texto

**6. Decisão: Modo de saída?**
```
if (outputMode === 'text') {
  // Exibir resposta como texto simples
} else if (outputMode === 'overlay') {
  // Renderizar anotações visuais sobre a imagem
}
```

**7. Salvar no histórico**
```
Análise armazenada em AsyncStorage:
{
  id: "uuid",
  timestamp: Date.now(),
  image: "data:image/jpeg;base64,...",
  prompt: "Descreva esta imagem",
  response: "Resposta do LLM",
  outputMode: "text",
  provider: "openai"
}
```

### Exemplo Prático Completo

```typescript
// Usuário tira foto
const photo = await cameraRef.takePictureAsync();

// Converter para Base64
const base64 = await FileSystem.readAsStringAsync(photo.uri, {
  encoding: FileSystem.EncodingType.Base64,
});
const imageData = `data:image/jpeg;base64,${base64}`;

// Enviar para servidor
const result = await client.vision.analyzeImage.mutate({
  image: imageData,
  prompt: "Identifique os objetos nesta imagem",
  outputMode: "text"
});

// Resultado
console.log(result.response); // "A imagem contém: um livro, uma caneta, um café..."

// Salvar no histórico
await addToHistory({
  image: imageData,
  prompt: "Identifique os objetos nesta imagem",
  response: result.response,
  outputMode: "text"
});
```

---

## Diagrama 2: Fluxo de Seleção de Provedor

### Visão Geral
Este diagrama mostra como o sistema decide qual provedor de LLM usar. É um sistema de cascata (fallback) onde tenta um provedor por vez.

### Lógica de Decisão

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
  
  // Prioridade 4: Forge
  if (process.env.BUILT_IN_FORGE_API_KEY) {
    return invokeForgeFallback(params);
  }
  
  // Nenhum provedor disponível
  throw new Error("Nenhum provedor de LLM configurado");
}
```

### Ordem de Prioridade

| Posição | Provedor | Razão |
|---------|----------|-------|
| 1 | OpenAI | Mais avançado, melhor qualidade |
| 2 | Anthropic | Bom custo-benefício, contexto grande |
| 3 | Google | Rápido e econômico |
| 4 | Forge | Fallback padrão (sem custo) |

### Casos de Uso

**Caso 1: Apenas OpenAI configurado**
```
OPENAI_API_KEY=sk-proj-xxx
ANTHROPIC_API_KEY=
GOOGLE_API_KEY=
BUILT_IN_FORGE_API_KEY=

Resultado: Usa OpenAI
```

**Caso 2: Múltiplos provedores configurados**
```
OPENAI_API_KEY=sk-proj-xxx
ANTHROPIC_API_KEY=sk-ant-xxx
GOOGLE_API_KEY=AIzaSy-xxx
BUILT_IN_FORGE_API_KEY=xxx

Resultado: Usa OpenAI (prioridade 1)
```

**Caso 3: Nenhum provedor configurado**
```
OPENAI_API_KEY=
ANTHROPIC_API_KEY=
GOOGLE_API_KEY=
BUILT_IN_FORGE_API_KEY=

Resultado: ❌ Erro - Nenhum provedor disponível
```

### Implementação com Fallback Automático

```typescript
export async function invokeLLMWithFallback(params: InvokeParams): Promise<InvokeResult> {
  const providers = [
    { name: 'OpenAI', fn: invokeOpenAI, key: 'OPENAI_API_KEY' },
    { name: 'Anthropic', fn: invokeAnthropic, key: 'ANTHROPIC_API_KEY' },
    { name: 'Google', fn: invokeGoogle, key: 'GOOGLE_API_KEY' },
    { name: 'Forge', fn: invokeForgeFallback, key: 'BUILT_IN_FORGE_API_KEY' },
  ];

  for (const provider of providers) {
    if (process.env[provider.key]) {
      try {
        console.log(`Tentando ${provider.name}...`);
        return await provider.fn(params);
      } catch (error) {
        console.error(`${provider.name} falhou:`, error);
        // Continua para o próximo provedor
      }
    }
  }

  throw new Error("Todos os provedores falharam");
}
```

---

## Diagrama 3: Fluxo de Integração OpenAI

### Visão Geral
Mostra como a imagem é processada especificamente pelo OpenAI GPT-4 Vision.

### Etapas Detalhadas

**1. Converter para formato OpenAI**
```typescript
// Entrada
const params = {
  messages: [
    {
      role: "user",
      content: "Descreva esta imagem"
    },
    {
      role: "user",
      content: {
        type: "image_url",
        image_url: { url: "data:image/jpeg;base64,..." }
      }
    }
  ]
};

// Saída esperada
const openaiMessages = [
  {
    role: "user",
    content: [
      { type: "text", text: "Descreva esta imagem" },
      {
        type: "image_url",
        image_url: {
          url: "data:image/jpeg;base64,...",
          detail: "auto"
        }
      }
    ]
  }
];
```

**2. Estruturar mensagens com vision_url**
```typescript
const openaiPayload = {
  model: "gpt-4-vision-preview",
  messages: openaiMessages,
  max_tokens: 2048,
  temperature: 0.7
};
```

**3. Chamar client.chat.completions.create**
```typescript
const response = await openai.chat.completions.create({
  model: "gpt-4-vision-preview",
  messages: openaiMessages,
  max_tokens: 2048,
});

// Resposta
{
  id: "chatcmpl-8NRx...",
  object: "chat.completion",
  created: 1699564000,
  model: "gpt-4-vision-preview",
  choices: [
    {
      index: 0,
      message: {
        role: "assistant",
        content: "A imagem mostra uma xícara de café quente em uma mesa de madeira..."
      },
      finish_reason: "stop"
    }
  ],
  usage: {
    prompt_tokens: 150,
    completion_tokens: 50,
    total_tokens: 200
  }
}
```

**4. Mapear para InvokeResult**
```typescript
const result: InvokeResult = {
  id: response.id,
  created: response.created,
  model: response.model,
  choices: response.choices.map(choice => ({
    index: choice.index,
    message: {
      role: "assistant",
      content: choice.message.content || ""
    },
    finish_reason: choice.finish_reason
  })),
  usage: {
    prompt_tokens: response.usage.prompt_tokens,
    completion_tokens: response.usage.completion_tokens,
    total_tokens: response.usage.total_tokens
  }
};
```

### Exemplo Completo com OpenAI

```typescript
import OpenAI from "openai";

const client = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY,
});

export async function analyzeImageWithOpenAI(
  imageBase64: string,
  prompt: string
): Promise<string> {
  const response = await client.chat.completions.create({
    model: "gpt-4-vision-preview",
    messages: [
      {
        role: "user",
        content: [
          { type: "text", text: prompt },
          {
            type: "image_url",
            image_url: {
              url: `data:image/jpeg;base64,${imageBase64}`,
              detail: "high"
            }
          }
        ]
      }
    ],
    max_tokens: 2048
  });

  return response.choices[0].message.content || "";
}

// Uso
const result = await analyzeImageWithOpenAI(imageBase64, "Descreva esta imagem");
console.log(result); // "A imagem mostra..."
```

### Tratamento de Erros OpenAI

```typescript
try {
  const response = await client.chat.completions.create({...});
} catch (error) {
  if (error.status === 401) {
    throw new Error("API key inválida");
  } else if (error.status === 429) {
    throw new Error("Rate limit excedido. Aguarde antes de tentar novamente");
  } else if (error.status === 500) {
    throw new Error("Servidor OpenAI indisponível");
  } else {
    throw error;
  }
}
```

---

## Diagrama 4: Fluxo de Integração Anthropic

### Visão Geral
Mostra como a imagem é processada pelo Claude 3.5 Sonnet da Anthropic.

### Diferenças Principais vs OpenAI

| Aspecto | OpenAI | Anthropic |
|--------|--------|-----------|
| Suporte a role "system" | ✅ Sim | ❌ Não |
| Formato de imagem | `image_url` | `image` com source |
| Contexto máximo | 128K tokens | 200K tokens |
| Custo | Mais caro | Mais barato |

### Etapas Detalhadas

**1. Filtrar mensagens de sistema**
```typescript
// Entrada
const messages = [
  { role: "system", content: "Você é um assistente útil" },
  { role: "user", content: "Descreva esta imagem" }
];

// Claude não suporta role "system", então:
const systemMessage = messages
  .filter(msg => msg.role === "system")
  .map(msg => msg.content)
  .join("\n");

const anthropicMessages = messages
  .filter(msg => msg.role !== "system");
```

**2. Converter para formato Anthropic**
```typescript
const anthropicMessages = [
  {
    role: "user",
    content: [
      { type: "text", text: "Descreva esta imagem" },
      {
        type: "image",
        source: {
          type: "base64",
          media_type: "image/jpeg",
          data: imageBase64
        }
      }
    ]
  }
];
```

**3. Chamar client.messages.create**
```typescript
const response = await client.messages.create({
  model: "claude-3-5-sonnet-20241022",
  max_tokens: 2048,
  system: systemMessage,
  messages: anthropicMessages
});

// Resposta
{
  id: "msg_...",
  type: "message",
  role: "assistant",
  content: [
    {
      type: "text",
      text: "A imagem mostra uma xícara de café quente..."
    }
  ],
  model: "claude-3-5-sonnet-20241022",
  stop_reason: "end_turn",
  stop_sequence: null,
  usage: {
    input_tokens: 150,
    output_tokens: 50
  }
}
```

### Exemplo Completo com Anthropic

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic({
  apiKey: process.env.ANTHROPIC_API_KEY,
});

export async function analyzeImageWithAnthropic(
  imageBase64: string,
  prompt: string
): Promise<string> {
  const response = await client.messages.create({
    model: "claude-3-5-sonnet-20241022",
    max_tokens: 2048,
    messages: [
      {
        role: "user",
        content: [
          { type: "text", text: prompt },
          {
            type: "image",
            source: {
              type: "base64",
              media_type: "image/jpeg",
              data: imageBase64
            }
          }
        ]
      }
    ]
  });

  const textContent = response.content.find(block => block.type === "text");
  return textContent ? textContent.text : "";
}

// Uso
const result = await analyzeImageWithAnthropic(imageBase64, "Descreva esta imagem");
console.log(result); // "A imagem mostra..."
```

### Vantagens do Anthropic

1. **Contexto Grande**: 200K tokens vs 128K do OpenAI
2. **Mais Econômico**: Preços mais baixos
3. **Melhor para Análise**: Excelente para documentos longos
4. **Menos Censura**: Respostas mais diretas

---

## Diagrama 5: Fluxo de Integração Google Gemini

### Visão Geral
Mostra como a imagem é processada pelo Google Gemini 2.0 Flash.

### Características Principais

- **Modelo**: Gemini 2.0 Flash (rápido e eficiente)
- **Limite de Contexto**: 1M tokens (maior que todos)
- **Custo**: Mais barato que OpenAI e Anthropic
- **Velocidade**: Muito rápido

### Etapas Detalhadas

**1. Converter para formato Google**
```typescript
// Entrada
const messages = [
  { role: "user", content: "Descreva esta imagem" }
];

// Saída esperada
const googleMessages = [
  {
    role: "user",
    parts: [
      { text: "Descreva esta imagem" },
      {
        inlineData: {
          mimeType: "image/jpeg",
          data: imageBase64
        }
      }
    ]
  }
];
```

**2. Estruturar conteúdo multimodal**
```typescript
const googlePayload = {
  contents: googleMessages,
  generationConfig: {
    maxOutputTokens: 2048,
    temperature: 0.7
  }
};
```

**3. Chamar model.generateContent**
```typescript
const response = await model.generateContent({
  contents: googleMessages,
  generationConfig: {
    maxOutputTokens: 2048
  }
});

// Resposta
{
  candidates: [
    {
      content: {
        parts: [
          {
            text: "A imagem mostra uma xícara de café quente..."
          }
        ],
        role: "model"
      },
      finishReason: "STOP",
      safetyRatings: [...]
    }
  ]
}
```

### Exemplo Completo com Google

```typescript
import { GoogleGenerativeAI } from "@google/generative-ai";

const genAI = new GoogleGenerativeAI(process.env.GOOGLE_API_KEY);

export async function analyzeImageWithGoogle(
  imageBase64: string,
  prompt: string
): Promise<string> {
  const model = genAI.getGenerativeModel({ model: "gemini-2.0-flash" });

  const response = await model.generateContent({
    contents: [
      {
        role: "user",
        parts: [
          { text: prompt },
          {
            inlineData: {
              mimeType: "image/jpeg",
              data: imageBase64
            }
          }
        ]
      }
    ],
    generationConfig: {
      maxOutputTokens: 2048
    }
  });

  return response.response.text();
}

// Uso
const result = await analyzeImageWithGoogle(imageBase64, "Descreva esta imagem");
console.log(result); // "A imagem mostra..."
```

### Vantagens do Google Gemini

1. **Limite Massivo**: 1M tokens de contexto
2. **Muito Rápido**: Respostas instantâneas
3. **Mais Econômico**: Preços muito baixos
4. **Multimodal**: Suporta texto, imagem, áudio, vídeo

---

## Diagrama 6: Fluxo de Tratamento de Erros

### Visão Geral
Mostra como o sistema lida com diferentes tipos de erros e implementa fallback automático.

### Tipos de Erros

**1. Erro de API Key Inválida**
```typescript
try {
  const response = await openai.chat.completions.create({...});
} catch (error) {
  if (error.status === 401) {
    console.error("API key inválida. Verifique OPENAI_API_KEY");
    // Tentar próximo provedor
    return invokeAnthropic(params);
  }
}
```

**2. Erro de Rate Limit**
```typescript
if (error.status === 429) {
  console.error("Rate limit excedido. Aguarde 60 segundos");
  // Implementar retry com backoff exponencial
  await sleep(60000);
  return invokeLLM(params); // Retry
}
```

**3. Erro de Modelo Não Disponível**
```typescript
if (error.message.includes("model not found")) {
  console.error("Modelo não disponível nesta região");
  // Tentar próximo provedor
  return invokeGoogle(params);
}
```

**4. Erro de Timeout**
```typescript
if (error.code === "ETIMEDOUT") {
  console.error("Timeout na requisição. Servidor lento");
  // Aumentar timeout ou tentar novamente
  return invokeLLMWithRetry(params, { timeout: 30000 });
}
```

### Estratégia de Fallback Completa

```typescript
export async function invokeLLMWithFallback(
  params: InvokeParams,
  maxRetries = 3
): Promise<InvokeResult> {
  const providers = [
    { name: 'OpenAI', fn: invokeOpenAI, key: 'OPENAI_API_KEY' },
    { name: 'Anthropic', fn: invokeAnthropic, key: 'ANTHROPIC_API_KEY' },
    { name: 'Google', fn: invokeGoogle, key: 'GOOGLE_API_KEY' },
    { name: 'Forge', fn: invokeForgeFallback, key: 'BUILT_IN_FORGE_API_KEY' },
  ];

  for (const provider of providers) {
    if (!process.env[provider.key]) continue;

    for (let attempt = 0; attempt < maxRetries; attempt++) {
      try {
        console.log(`[${provider.name}] Tentativa ${attempt + 1}/${maxRetries}`);
        return await provider.fn(params);
      } catch (error) {
        console.error(`[${provider.name}] Erro:`, error.message);

        // Se é o último provedor e última tentativa
        if (provider === providers[providers.length - 1] && 
            attempt === maxRetries - 1) {
          throw new Error(`Todos os provedores falharam após ${maxRetries} tentativas`);
        }

        // Rate limit: aguardar antes de retry
        if (error.status === 429) {
          const delay = Math.pow(2, attempt) * 1000;
          console.log(`Aguardando ${delay}ms antes de retry...`);
          await sleep(delay);
        }
      }
    }
  }

  throw new Error("Nenhum provedor disponível");
}
```

### Logging e Monitoramento

```typescript
export async function invokeLLMWithLogging(
  params: InvokeParams
): Promise<InvokeResult> {
  const startTime = Date.now();

  try {
    const result = await invokeLLM(params);
    const duration = Date.now() - startTime;

    console.log({
      event: "llm_success",
      provider: getCurrentProvider(),
      duration,
      tokens: result.usage?.total_tokens
    });

    return result;
  } catch (error) {
    const duration = Date.now() - startTime;

    console.error({
      event: "llm_error",
      provider: getCurrentProvider(),
      duration,
      error: error.message
    });

    throw error;
  }
}
```

---

## Diagrama 7: Fluxo de Configuração de Ambiente

### Visão Geral
Mostra como o sistema carrega e valida as variáveis de ambiente no inicialização.

### Processo de Carregamento

**1. Leitura de Variáveis**
```typescript
// Arquivo: server/_core/env.ts
export const ENV = {
  forgeApiKey: process.env.BUILT_IN_FORGE_API_KEY ?? "",
  forgeApiUrl: process.env.BUILT_IN_FORGE_API_URL ?? "",
  openaiApiKey: process.env.OPENAI_API_KEY ?? "",
  anthropicApiKey: process.env.ANTHROPIC_API_KEY ?? "",
  googleApiKey: process.env.GOOGLE_API_KEY ?? "",
};
```

**2. Validação de Chaves**
```typescript
export function validateLLMConfig(): void {
  const hasOpenAI = ENV.openaiApiKey.length > 0;
  const hasAnthropic = ENV.anthropicApiKey.length > 0;
  const hasGoogle = ENV.googleApiKey.length > 0;
  const hasForge = ENV.forgeApiKey.length > 0;

  if (!hasOpenAI && !hasAnthropic && !hasGoogle && !hasForge) {
    console.warn("⚠️ Nenhum provedor de LLM configurado!");
    console.warn("Configure uma das seguintes variáveis:");
    console.warn("  - OPENAI_API_KEY");
    console.warn("  - ANTHROPIC_API_KEY");
    console.warn("  - GOOGLE_API_KEY");
    console.warn("  - BUILT_IN_FORGE_API_KEY");
  }

  if (hasOpenAI) {
    console.log("✅ OpenAI configurado");
  }
  if (hasAnthropic) {
    console.log("✅ Anthropic configurado");
  }
  if (hasGoogle) {
    console.log("✅ Google Gemini configurado");
  }
  if (hasForge) {
    console.log("✅ Forge configurado");
  }
}
```

**3. Inicialização de Clientes**
```typescript
// Inicializar apenas os clientes com chaves configuradas
let openaiClient: OpenAI | null = null;
let anthropicClient: Anthropic | null = null;
let googleGenAI: GoogleGenerativeAI | null = null;

if (ENV.openaiApiKey) {
  openaiClient = new OpenAI({ apiKey: ENV.openaiApiKey });
}

if (ENV.anthropicApiKey) {
  anthropicClient = new Anthropic({ apiKey: ENV.anthropicApiKey });
}

if (ENV.googleApiKey) {
  googleGenAI = new GoogleGenerativeAI(ENV.googleApiKey);
}
```

### Arquivo .env.local Exemplo

```bash
# Desenvolvimento
NODE_ENV=development

# Escolha UM provedor:

# OpenAI
OPENAI_API_KEY=sk-proj-xxxxxxxxxxxxx

# OU Anthropic
# ANTHROPIC_API_KEY=sk-ant-xxxxxxxxxxxxx

# OU Google
# GOOGLE_API_KEY=AIzaSyxxxxxxxxxxxxx

# OU Forge (padrão)
# BUILT_IN_FORGE_API_KEY=xxxxxxxxxxxxx
```

### Validação na Inicialização

```typescript
// Arquivo: server/_core/index.ts
import { validateLLMConfig } from "./env";

console.log("🚀 Iniciando servidor...");
validateLLMConfig();
console.log("✅ Servidor pronto para receber requisições");
```

---

## Diagrama 8: Fluxo de Decisão de Provedor (Prioridade)

### Visão Geral
Detalha como o sistema escolhe entre múltiplos provedores configurados usando uma estratégia de prioridade.

### Matriz de Decisão

```
┌─────────────────────────────────────────────────────────┐
│ Verificação de Variáveis de Ambiente (em ordem)         │
├─────────────────────────────────────────────────────────┤
│ 1. OPENAI_API_KEY definida?                             │
│    ├─ SIM → Usar OpenAI (STOP)                          │
│    └─ NÃO → Próxima verificação                         │
│                                                          │
│ 2. ANTHROPIC_API_KEY definida?                          │
│    ├─ SIM → Usar Anthropic (STOP)                       │
│    └─ NÃO → Próxima verificação                         │
│                                                          │
│ 3. GOOGLE_API_KEY definida?                             │
│    ├─ SIM → Usar Google (STOP)                          │
│    └─ NÃO → Próxima verificação                         │
│                                                          │
│ 4. BUILT_IN_FORGE_API_KEY definida?                     │
│    ├─ SIM → Usar Forge (STOP)                           │
│    └─ NÃO → Erro                                        │
└─────────────────────────────────────────────────────────┘
```

### Implementação

```typescript
export function selectLLMProvider(): {
  name: string;
  invoke: (params: InvokeParams) => Promise<InvokeResult>;
} {
  // Prioridade 1
  if (process.env.OPENAI_API_KEY) {
    return {
      name: "OpenAI",
      invoke: invokeOpenAI
    };
  }

  // Prioridade 2
  if (process.env.ANTHROPIC_API_KEY) {
    return {
      name: "Anthropic",
      invoke: invokeAnthropic
    };
  }

  // Prioridade 3
  if (process.env.GOOGLE_API_KEY) {
    return {
      name: "Google",
      invoke: invokeGoogle
    };
  }

  // Prioridade 4
  if (process.env.BUILT_IN_FORGE_API_KEY) {
    return {
      name: "Forge",
      invoke: invokeForgeFallback
    };
  }

  throw new Error("Nenhum provedor de LLM configurado");
}

// Uso
const provider = selectLLMProvider();
console.log(`Usando provedor: ${provider.name}`);
const result = await provider.invoke(params);
```

### Cenários de Uso

**Cenário 1: Desenvolvimento Local**
```bash
# .env.local
OPENAI_API_KEY=sk-proj-xxx

# Resultado: Usa OpenAI para testes
```

**Cenário 2: Produção com Múltiplos Provedores**
```bash
# .env (produção)
OPENAI_API_KEY=sk-proj-xxx
ANTHROPIC_API_KEY=sk-ant-xxx (fallback)
GOOGLE_API_KEY=AIzaSy-xxx (fallback)

# Resultado: Usa OpenAI, mas se falhar, tenta Anthropic, depois Google
```

**Cenário 3: Provedor Específico**
```bash
# .env
ANTHROPIC_API_KEY=sk-ant-xxx

# Resultado: Usa Anthropic (mesmo que OpenAI esteja disponível)
```

---

## Diagrama 9: Fluxo Completo de Análise de Imagem

### Visão Geral
Mostra o processo end-to-end completo desde a captura até o resultado final.

### Etapas Detalhadas

**1. Captura de Imagem**
```typescript
// Usuário pressiona botão de captura
const photo = await cameraRef.takePictureAsync({
  quality: 0.8,
  base64: true,
  exif: true
});

// Resultado
{
  uri: "file:///data/user/0/com.example.app/cache/photo.jpg",
  width: 1080,
  height: 1920,
  base64: "/9j/4AAQSkZJRg..."
}
```

**2. Converter para Base64**
```typescript
const base64Image = photo.base64 || 
  await FileSystem.readAsStringAsync(photo.uri, {
    encoding: FileSystem.EncodingType.Base64
  });

const imageData = `data:image/jpeg;base64,${base64Image}`;
```

**3. Enviar para Servidor**
```typescript
const result = await client.vision.analyzeImage.mutate({
  image: imageData,
  prompt: selectedPrompt.text,
  outputMode: selectedPrompt.outputMode
});
```

**4. Validação no Servidor**
```typescript
// Validar tamanho da imagem
if (imageData.length > 20 * 1024 * 1024) {
  throw new Error("Imagem muito grande (máx 20MB)");
}

// Validar prompt
if (!prompt || prompt.trim().length === 0) {
  throw new Error("Prompt não pode estar vazio");
}
```

**5. Selecionar Provedor**
```typescript
const provider = selectLLMProvider();
console.log(`Usando ${provider.name}`);
```

**6. Processar Imagem**
```typescript
const response = await provider.invoke({
  messages: [
    {
      role: "user",
      content: [
        { type: "text", text: prompt },
        { type: "image_url", image_url: { url: imageData } }
      ]
    }
  ]
});
```

**7. Determinar Modo de Saída**
```typescript
if (outputMode === "text") {
  // Retornar resposta em texto
  return {
    type: "text",
    content: response.choices[0].message.content
  };
} else if (outputMode === "overlay") {
  // Gerar anotações visuais
  const annotations = await generateOverlayAnnotations(
    response.choices[0].message.content,
    imageData
  );
  
  return {
    type: "overlay",
    content: response.choices[0].message.content,
    annotations: annotations
  };
}
```

**8. Salvar no Histórico**
```typescript
await addToHistory({
  id: generateUUID(),
  timestamp: Date.now(),
  image: imageData,
  prompt: prompt,
  response: response.choices[0].message.content,
  outputMode: outputMode,
  provider: provider.name,
  tokens: response.usage?.total_tokens
});
```

**9. Retornar ao Cliente**
```typescript
return {
  success: true,
  response: response.choices[0].message.content,
  outputMode: outputMode,
  provider: provider.name,
  timestamp: Date.now()
};
```

**10. Exibir Resultado**
```typescript
// Cliente recebe resultado
if (result.outputMode === "text") {
  navigation.navigate("ResultText", { result });
} else {
  navigation.navigate("ResultOverlay", { result });
}
```

### Exemplo Completo Funcional

```typescript
// Fluxo completo em uma função
export async function analyzeImageComplete(
  imageUri: string,
  prompt: string,
  outputMode: "text" | "overlay"
): Promise<AnalysisResult> {
  try {
    // 1. Converter para Base64
    const base64 = await FileSystem.readAsStringAsync(imageUri, {
      encoding: FileSystem.EncodingType.Base64
    });
    const imageData = `data:image/jpeg;base64,${base64}`;

    // 2. Enviar para servidor
    const response = await client.vision.analyzeImage.mutate({
      image: imageData,
      prompt,
      outputMode
    });

    // 3. Salvar no histórico
    await addToHistory({
      image: imageData,
      prompt,
      response: response.content,
      outputMode,
      timestamp: Date.now()
    });

    // 4. Retornar resultado
    return {
      success: true,
      content: response.content,
      outputMode,
      timestamp: Date.now()
    };
  } catch (error) {
    console.error("Erro na análise:", error);
    return {
      success: false,
      error: error.message,
      timestamp: Date.now()
    };
  }
}
```

---

## Diagrama 10: Arquitetura de Camadas

### Visão Geral
Mostra como o sistema é organizado em camadas distintas, cada uma com responsabilidades específicas.

### Camadas

**1. Frontend (React Native)**
```
┌─────────────────────────────────────┐
│ Tela de Câmera                      │
│ - Captura foto                      │
│ - Mostra preview                    │
│ - Seleciona prompt                  │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│ Tela de Resultado                   │
│ - Exibe resposta em texto           │
│ - Renderiza overlay                 │
│ - Compartilha resultado             │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│ Configurações                       │
│ - Modo de escaneamento              │
│ - Comportamento após resposta       │
│ - Preferências de saída             │
└─────────────────────────────────────┘
```

**2. Cliente tRPC**
```
┌─────────────────────────────────────┐
│ analyzeImage                        │
│ - Envia imagem em Base64            │
│ - Envia prompt                      │
│ - Especifica modo de saída          │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│ getHistory                          │
│ - Recupera análises anteriores      │
│ - Filtra por data/prompt            │
│ - Pagina resultados                 │
└─────────────────────────────────────┘
```

**3. Servidor (Node.js)**
```
┌─────────────────────────────────────┐
│ Router tRPC                         │
│ - Recebe requisições                │
│ - Roteia para handlers              │
│ - Retorna respostas                 │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│ Validação                           │
│ - Valida tamanho da imagem          │
│ - Valida prompt                     │
│ - Valida outputMode                 │
└─────────────────────────────────────┘
```

**4. Camada de LLM**
```
┌─────────────────────────────────────┐
│ invokeLLM                           │
│ - Orquestra chamadas                │
│ - Implementa retry                  │
│ - Normaliza respostas               │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│ Seletor de Provedor                 │
│ - Verifica variáveis de ambiente    │
│ - Escolhe provedor                  │
│ - Implementa fallback               │
└─────────────────────────────────────┘
```

**5. Provedores de LLM**
```
┌──────────────────┐
│ OpenAI           │
│ GPT-4 Vision     │
└──────────────────┘

┌──────────────────┐
│ Anthropic        │
│ Claude 3.5       │
└──────────────────┘

┌──────────────────┐
│ Google           │
│ Gemini 2.0       │
└──────────────────┘

┌──────────────────┐
│ Forge            │
│ Gemini 2.5       │
└──────────────────┘
```

**6. Armazenamento**
```
┌─────────────────────────────────────┐
│ AsyncStorage Local                  │
│ - Prompts salvos                    │
│ - Configurações                     │
│ - Histórico de análises             │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│ Histórico de Análises               │
│ - Imagem original                   │
│ - Prompt usado                      │
│ - Resposta do LLM                   │
│ - Timestamp                         │
│ - Provedor usado                    │
└─────────────────────────────────────┘
```

### Fluxo de Dados Entre Camadas

```
1. Frontend captura imagem
   ↓
2. Envia via tRPC para servidor
   ↓
3. Servidor valida entrada
   ↓
4. Camada de LLM seleciona provedor
   ↓
5. Provedor processa imagem
   ↓
6. Resposta normalizada
   ↓
7. Armazenada no histórico
   ↓
8. Retorna para frontend
   ↓
9. Frontend exibe resultado
```

### Exemplo de Comunicação Entre Camadas

```typescript
// Frontend (React Native)
const result = await client.vision.analyzeImage.mutate({
  image: "data:image/jpeg;base64,...",
  prompt: "Descreva",
  outputMode: "text"
});

// ↓ HTTP POST /trpc/vision.analyzeImage

// Servidor (Node.js)
export const visionRouter = router({
  analyzeImage: publicProcedure
    .input(z.object({
      image: z.string(),
      prompt: z.string(),
      outputMode: z.enum(["text", "overlay"])
    }))
    .mutation(async ({ input }) => {
      // Validar
      validateInput(input);
      
      // Processar com LLM
      const response = await invokeLLM({
        messages: [...]
      });
      
      // Salvar histórico
      await addToHistory({...});
      
      // Retornar
      return { content: response.content };
    })
});

// ↓ HTTP Response

// Frontend recebe resultado
console.log(result.content); // "A imagem mostra..."
```

---

## Resumo Comparativo

| Diagrama | Foco | Complexidade | Uso |
|----------|------|-------------|-----|
| 1 | Fluxo geral | Baixa | Visão geral |
| 2 | Seleção de provedor | Média | Entender prioridades |
| 3 | OpenAI | Alta | Integração OpenAI |
| 4 | Anthropic | Alta | Integração Anthropic |
| 5 | Google | Alta | Integração Google |
| 6 | Tratamento de erros | Alta | Debugging |
| 7 | Configuração | Média | Setup inicial |
| 8 | Decisão de provedor | Média | Lógica de seleção |
| 9 | Fluxo completo | Muito Alta | Entender tudo |
| 10 | Arquitetura | Média | Estrutura geral |

---

## Próximos Passos

1. **Escolha um diagrama** que corresponda ao seu caso de uso
2. **Siga os exemplos de código** fornecidos
3. **Configure as variáveis de ambiente** necessárias
4. **Teste com uma imagem** para validar a integração
5. **Monitore os logs** para entender o fluxo

Boa sorte! 🚀
