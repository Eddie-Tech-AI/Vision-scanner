# Diagrama de Fluxo - Integração de LLMs

## 1. Fluxo Geral de Processamento de Imagem

```mermaid
graph TD
    A["📱 Usuário tira foto na câmera"] --> B["Imagem convertida para Base64"]
    B --> C["Enviada para servidor tRPC"]
    C --> D{"Qual provedor\nde LLM?"}
    
    D -->|OpenAI| E["🔑 Verifica OPENAI_API_KEY"]
    D -->|Anthropic| F["🔑 Verifica ANTHROPIC_API_KEY"]
    D -->|Google| G["🔑 Verifica GOOGLE_API_KEY"]
    D -->|Nenhum| H["🔑 Usa BUILT_IN_FORGE_API_KEY"]
    
    E --> I["Processa com OpenAI GPT-4"]
    F --> J["Processa com Claude 3.5"]
    G --> K["Processa com Gemini 2.0"]
    H --> L["Processa com Forge API"]
    
    I --> M["Resposta recebida"]
    J --> M
    K --> M
    L --> M
    
    M --> N{"Modo de\nsaída?"}
    N -->|Texto| O["Exibe resposta em texto"]
    N -->|Overlay| P["Renderiza anotações visuais"]
    
    O --> Q["Salva no histórico"]
    P --> Q
    Q --> R["✅ Análise concluída"]
```

## 2. Fluxo de Seleção de Provedor

```mermaid
graph TD
    A["Iniciar invokeLLM"] --> B{"OPENAI_API_KEY\ndefinida?"}
    B -->|Sim| C["Usar OpenAI"]
    B -->|Não| D{"ANTHROPIC_API_KEY\ndefinida?"}
    
    D -->|Sim| E["Usar Anthropic"]
    D -->|Não| F{"GOOGLE_API_KEY\ndefinida?"}
    
    F -->|Sim| G["Usar Google"]
    F -->|Não| H{"BUILT_IN_FORGE_API_KEY\ndefinida?"}
    
    H -->|Sim| I["Usar Forge"]
    H -->|Não| J["❌ Erro: Nenhum provedor disponível"]
    
    C --> K["Chamar invokeOpenAI"]
    E --> L["Chamar invokeAnthropic"]
    G --> M["Chamar invokeGoogle"]
    I --> N["Chamar invokeForgeFallback"]
    
    K --> O["Retornar InvokeResult"]
    L --> O
    M --> O
    N --> O
```

## 3. Fluxo de Integração OpenAI

```mermaid
graph TD
    A["Requisição com imagem"] --> B["Converter para formato OpenAI"]
    B --> C["Estruturar mensagens com vision_url"]
    C --> D["Chamar client.chat.completions.create"]
    
    D --> E{"Resposta\nrecebida?"}
    E -->|Sim| F["Mapear para InvokeResult"]
    E -->|Não| G["Capturar erro"]
    
    F --> H["Retornar resposta"]
    G --> I["Tentar próximo provedor ou falhar"]
    
    H --> J["✅ Sucesso"]
    I --> K["❌ Erro"]
```

## 4. Fluxo de Integração Anthropic

```mermaid
graph TD
    A["Requisição com imagem"] --> B["Filtrar mensagens de sistema"]
    B --> C["Converter para formato Anthropic"]
    C --> D["Estruturar conteúdo multimodal"]
    D --> E["Chamar client.messages.create"]
    
    E --> F{"Resposta\nrecebida?"}
    F -->|Sim| G["Extrair texto da resposta"]
    F -->|Não| H["Capturar erro"]
    
    G --> I["Mapear para InvokeResult"]
    H --> J["Tentar próximo provedor"]
    
    I --> K["Retornar resposta"]
    J --> L["❌ Erro"]
    
    K --> M["✅ Sucesso"]
```

## 5. Fluxo de Integração Google Gemini

```mermaid
graph TD
    A["Requisição com imagem"] --> B["Converter para formato Google"]
    B --> C["Estruturar parts com imagem"]
    C --> D["Chamar model.generateContent"]
    
    D --> E{"Resposta\nrecebida?"}
    E -->|Sim| F["Extrair texto"]
    E -->|Não| G["Capturar erro"]
    
    F --> H["Mapear para InvokeResult"]
    G --> I["Tentar próximo provedor"]
    
    H --> J["Retornar resposta"]
    I --> K["❌ Erro"]
    
    J --> L["✅ Sucesso"]
```

## 6. Fluxo de Tratamento de Erros

```mermaid
graph TD
    A["Erro na chamada LLM"] --> B{"Tipo de\nerro?"}
    
    B -->|API Key inválida| C["Log: Verificar credenciais"]
    B -->|Rate limit| D["Log: Aguardar antes de retry"]
    B -->|Modelo não disponível| E["Log: Verificar disponibilidade"]
    B -->|Timeout| F["Log: Aumentar timeout"]
    
    C --> G{"Tentar\nfallback?"}
    D --> G
    E --> G
    F --> G
    
    G -->|Sim| H["Tentar próximo provedor"]
    G -->|Não| I["Retornar erro ao usuário"]
    
    H --> J{"Provedor\ndisponível?"}
    J -->|Sim| K["Chamar novo provedor"]
    J -->|Não| I
    
    K --> L{"Sucesso?"}
    L -->|Sim| M["✅ Retornar resultado"]
    L -->|Não| I
    
    I --> N["❌ Erro final"]
```

## 7. Fluxo de Configuração de Ambiente

```mermaid
graph TD
    A["Iniciar aplicação"] --> B["Ler variáveis de ambiente"]
    B --> C{"Arquivo .env.local\nexiste?"}
    
    C -->|Sim| D["Carregar variáveis locais"]
    C -->|Não| E["Usar variáveis do sistema"]
    
    D --> F{"Qual provedor\nconfigured?"}
    E --> F
    
    F -->|OpenAI| G["OPENAI_API_KEY"]
    F -->|Anthropic| H["ANTHROPIC_API_KEY"]
    F -->|Google| I["GOOGLE_API_KEY"]
    F -->|Forge| J["BUILT_IN_FORGE_API_KEY"]
    
    G --> K["Validar chave"]
    H --> K
    I --> K
    J --> K
    
    K --> L{"Chave\nválida?"}
    L -->|Sim| M["✅ Provedor pronto"]
    L -->|Não| N["⚠️ Aviso: Chave inválida"]
    
    M --> O["Inicializar cliente"]
    N --> P["Usar fallback"]
    
    O --> Q["Aplicação pronta"]
    P --> Q
```

## 8. Fluxo de Decisão de Provedor (Prioridade)

```mermaid
graph TD
    A["Requisição de análise"] --> B["Verificar OPENAI_API_KEY"]
    B -->|Definida| C["✅ Usar OpenAI"]
    B -->|Não| D["Verificar ANTHROPIC_API_KEY"]
    
    D -->|Definida| E["✅ Usar Anthropic"]
    D -->|Não| F["Verificar GOOGLE_API_KEY"]
    
    F -->|Definida| G["✅ Usar Google"]
    F -->|Não| H["Verificar BUILT_IN_FORGE_API_KEY"]
    
    H -->|Definida| I["✅ Usar Forge"]
    H -->|Não| J["❌ Nenhum provedor disponível"]
    
    C --> K["Processar imagem"]
    E --> K
    G --> K
    I --> K
    J --> L["Retornar erro"]
    
    K --> M["Retornar resultado"]
```

## 9. Fluxo Completo de Análise de Imagem

```mermaid
graph TD
    A["🎥 Usuário captura imagem"] --> B["Converter para Base64"]
    B --> C["Enviar para servidor"]
    C --> D["Receber prompt do usuário"]
    D --> E["Validar entrada"]
    
    E --> F{"Entrada\nválida?"}
    F -->|Não| G["Retornar erro de validação"]
    F -->|Sim| H["Selecionar provedor LLM"]
    
    H --> I["Preparar payload"]
    I --> J["Chamar API do provedor"]
    
    J --> K{"Resposta\nrecebida?"}
    K -->|Não| L["Tentar fallback"]
    K -->|Sim| M["Processar resposta"]
    
    L --> N{"Fallback\nsucesso?"}
    N -->|Sim| M
    N -->|Não| O["Retornar erro"]
    
    M --> P["Determinar modo de saída"]
    P --> Q{"Modo\ntexto?"}
    
    Q -->|Sim| R["Formatar resposta em texto"]
    Q -->|Não| S["Gerar overlay visual"]
    
    R --> T["Salvar no histórico"]
    S --> T
    
    T --> U["Retornar ao cliente"]
    O --> U
    G --> U
    
    U --> V["✅ Análise concluída"]
```

## 10. Arquitetura de Camadas

```mermaid
graph TB
    subgraph "Frontend (React Native)"
        A["📱 Tela de Câmera"]
        B["📋 Tela de Resultado"]
        C["⚙️ Configurações"]
    end
    
    subgraph "Cliente tRPC"
        D["analyzeImage"]
        E["getHistory"]
    end
    
    subgraph "Servidor (Node.js)"
        F["Router tRPC"]
        G["Validação"]
    end
    
    subgraph "Camada de LLM"
        H["invokeLLM"]
        I["Seletor de Provedor"]
    end
    
    subgraph "Provedores de LLM"
        J["OpenAI"]
        K["Anthropic"]
        L["Google"]
        M["Forge"]
    end
    
    subgraph "Armazenamento"
        N["AsyncStorage Local"]
        O["Histórico de Análises"]
    end
    
    A --> D
    B --> E
    C --> D
    
    D --> F
    E --> F
    
    F --> G
    G --> H
    
    H --> I
    I --> J
    I --> K
    I --> L
    I --> M
    
    F --> N
    F --> O
```

## Legenda

- 📱 = Interface do usuário
- 🔑 = Chave de API
- ✅ = Sucesso
- ❌ = Erro
- ⚠️ = Aviso
- 🎥 = Câmera
- 📋 = Dados
- ⚙️ = Configurações

## Notas Importantes

1. **Prioridade de Provedores**: OpenAI > Anthropic > Google > Forge
2. **Fallback Automático**: Se um provedor falhar, tenta o próximo
3. **Variáveis de Ambiente**: Defina apenas UM provedor por vez
4. **Tratamento de Erros**: Cada provedor tem seu próprio tratamento
5. **Cache**: Respostas podem ser cacheadas para melhor performance
