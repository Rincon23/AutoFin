<h1 align="center"> AutoFin </h1>

<p align="center">
  <img src="./img/AutoFin-imgre.png" width="500">
</p>



Assistente pessoal de finanças via Telegram, construído com N8N. Registra despesas por mensagem de voz, gerencia saldo e gera relatórios visuais — tudo integrado ao Google Sheets.

---

## 😫 O Problema

Registrar gastos no dia a dia exige **digitar manualmente** cada detalhe (descrição, valor, data, categoria) em uma planilha ou aplicativo. No celular, isso é ainda mais cansativo e cheio de atritos:

- Abrir a planilha, encontrar a célula certa e preencher linha por linha
- Corrigir erros de digitação e formatação de valores
- Perder tempo que poderia ser gasto com o que realmente importa

**Consequência:** a maioria das pessoas desiste do controle financeiro em poucos dias, e os gastos voltam a passar despercebidos.

## 💡 A Solução

Um **bot no Telegram** que entende **áudio** e **texto**.  
Você fala algo como:

> “Gastei 25 reais no almoço hoje”

E ele faz todo o trabalho:

- 🎙️ **Transcreve** o áudio usando IA
- 🧠 **Interpreta** automaticamente: descrição, valor, categoria e data (inclusive expressões como “ontem”)
- 📊 **Lança a despesa** na planilha do Google Sheets
- 💰 **Atualiza o saldo** e envia uma confirmação

Tudo sem digitar uma única letra. Além de registrar gastos, o bot oferece comandos para consultar saldo, gerar gráficos e até cancelar o último lançamento – **totalmente por voz ou toque no menu**.

## ✨ Funcionalidades

* 🎤 Recebe áudios via Telegram
* 🧠 Transcreve e interpreta usando IA (Google Gemini)
* 💰 Extrai automaticamente:

  * Descrição
  * Categoria
  * Data
  * Valor
* 📊 Preenche os dados diretamente no Google Sheets
* ✅ Retorna confirmação no Telegram

---

## 🛠️ Tecnologias

| Ferramenta | Uso |
|---|---|
| [N8N](https://n8n.io) | Orquestração do fluxo de automação |
| [Telegram Bot API](https://core.telegram.org/bots/api) | Interface de comunicação com o usuário |
| [Google Gemini](https://deepmind.google/technologies/gemini/) | Transcrição e interpretação de áudios |
| [Google Sheets](https://sheets.google.com) | Armazenamento de gastos, saldo e estado do chat |

---

## 🗂️ Estrutura do Google Sheets

O projeto utiliza uma planilha com três abas principais:

### `Gastos`
Registra cada despesa com as colunas:

| Id | Descrição | Data | Valor | Dia | Categoria | Este mês |
|---|---|---|---|---|---|---|

### `Saldo`
Armazena o saldo atual do usuário (linha 2, coluna `Saldo atual`).

### `Estado do chat`
Controla o estado da conversa para fluxos de múltiplos passos:

| Estado | Descrição |
|---|---|
| `padrao` | Estado inicial / sem ação pendente |
| `aguardando_atualizacao_saldo` | Aguardando novo valor para substituir o saldo |
| `aguardando_adicionar_saldo` | Aguardando valor a somar ao saldo |

---




# 🏗️ Arquitetura Completa do Workflow n8n

```mermaid
flowchart TD

    A[📱 Telegram Trigger] --> B{🔍 É Áudio?}

    B -- Sim --> C[🎤 Telegram Get File]
    C --> D[🤖 Gemini - Analyze Audio]
    D -- "✅ Sucesso (Main)" --> F[⚙️ Code - Parse JSON]
    D -- "❌ Erro (Error)" --> G[❌ Serviço indisponível]

    F --> I{🔀 Tratamento de Erros}

    B -- Não --> H[📨 Mensagem de Texto]

    I -- "Cancelar" --> J1[📋 Buscar Linhas]
    J1 --> J2[🔢 Get Número da Última Linha]
    J2 --> J3[💰 Get Saldo Atual]
    J3 --> J4[📋 Get Valor da Despesa]
    J4 --> J5[💾 Update Saldo]
    J5 --> J6[🗑️ Delete Última Linha]
    J6 --> J7[💬 Mensagem de Cancelamento]

    I -- "Descrição vazia / Valor = 0" --> K[❌ Não compreendido]

    I -- "Dados Válidos" --> L[📝 Adicionar Gasto]
    L --> M[Append/Update Row - Gastos]
    M --> N[Update Row - Saldo]
    N --> O[💬 Mensagem de Confirmação do Gasto]

    H --> O1[📊 Get Estado1]
    O1 --> P{📌 Aguardando dados?}

    P -- "Sim (estado ≠ padrão)" --> Q{🔀 Verificar Estado do Chat}
    P -- "Não (estado = padrão)" --> U{🔀 Opções Gerais}

    %% Bloco de retorno para Verificar Estado do Chat
    subgraph Retorno_VEC
        R1[💬 Menu1]
        R1 --> R1a[🔁 Update Estado → padrao]
    end

    %% Bloco de retorno para Opções Gerais
    subgraph Retorno_OG
        R2[💬 Menu1]
        R2 --> R2a[🔁 Update Estado → padrao]
    end

    Q -- "Cancelar" --> R1
    Q -- "aguardando_atualizacao_saldo" --> S{🔢 É Número? >= 0}
    S -- Sim --> S1[💾 Update Saldo2]
    S1 --> S2[🔁 Update Estado → padrao]
    S2 --> S3[💬 Saldo Atualizado]
    S -- Não --> S4[❌ Número Inválido - Reenvia]

    Q -- "aguardando_adicionar_saldo" --> T{🔢 É Número? >= 0}
    T -- Sim --> T1[💰 Get Saldo4]
    T1 --> T2[➕ Update Saldo3]
    T2 --> T3[🔁 Update Estado → padrao]
    T3 --> T4[💰 Get Saldo5]
    T4 --> T5[💬 Saldo Atualizado com Adição]
    T -- Não --> T6[❌ Número Inválido - Reenvia]

    U -- "Saldo" --> V[📋 Mostrar Opções Saldo]
    U -- "Consultar Saldo" --> W[💰 Get Saldo3]
    W --> W1[💬 Enviar Saldo]
    U -- "Atualizar Saldo" --> X[💬 Solicitar Novo Saldo]
    X --> X1[🔁 Estado → aguardando_atualizacao_saldo]
    U -- "Adicionar Saldo" --> Y[💬 Solicitar Valor]
    Y --> Y1[🔁 Estado → aguardando_adicionar_saldo]
    U -- "Relatórios" --> Z[📋 Opções Relatórios]
    U -- "Gastos Diário" --> AA[📥 Download Gráfico Diário]
    AA --> AA1[📤 Enviar Gráfico Diário]
    U -- "Gastos por Categoria" --> AB[📥 Download Gráfico Categoria]
    AB --> AB1[📤 Enviar Gráfico Categoria]
    U -- "Outro (texto livre)" --> R2
```






## ⚙️ Configuração

### Pré-requisitos

- Instância do N8N (Sendo utilizado via domínio "HTTPS")
- Bot do Telegram criado via [@BotFather](https://t.me/BotFather)
- Conta Google com acesso ao Google Sheets
- Chave de API do Google Gemini 


### Passos

1. **Importe o fluxo** no N8N usando o arquivo JSON do projeto
2. **Configure as credenciais:**
   - `telegramApi` — token do seu bot
   - `googleSheetsOAuth2Api` — OAuth2 da conta Google
   - `googlePalmApi` — chave da API do Gemini
3. **Crie a planilha** no Google Sheets conforme planilha versionada.
4. **Ative o webhook** do Telegram Trigger no N8N
5. **Publique os gráficos** da planilha como imagem (Arquivo → Publicar na web → Gráfico → Imagem) e atualize as URLs nos nós `Download Grafico Diario1` e `Download Grafico Categoria1`

---

## 💬 Como Usar

| Ação | Como fazer |
|---|---|
| Registrar gasto | Envie um áudio: *"Gastei 45 reais no mercado hoje"* |
| Cancelar último gasto | Envie um áudio: *"Cancelar"* |
| Ver saldo | Toque em **Saldo → Consultar Saldo** |
| Atualizar saldo | Toque em **Saldo → Atualizar Saldo** e envie o valor |
| Adicionar ao saldo | Toque em **Saldo → Adicionar Saldo** e envie o valor |
| Ver relatório | Toque em **Relatórios → Gastos Diário** ou **Gastos por Categoria** |

### Categorias reconhecidas pelo bot
`Alimentação` · `Transporte` · `Saúde` · `Moradia` · `Lazer` · `Educação` · `Outros`

---

## 📋 Tratamento de Erros

| Situação | Comportamento |
|---|---|
| Áudio sem descrição | Mensagem pedindo para repetir com descrição e valor |
| Áudio sem valor | Mensagem pedindo para repetir com descrição e valor |
| Gemini indisponível | Mensagem informando que o serviço está fora do ar |
| Valor inválido no saldo | Mensagem solicitando número maior ou igual a 0 |

---

# 👨‍💻 Autores 👨‍💻

[Gabriel Mello](https://github.com/GabrielMello159)

[Enzo Rincon](https://github.com/Rincon23)

[Luis Torres](https://github.com/LuisTorresGit)

[Vinicius Tobias](https://github.com/vinitobias672003)

[Julia Moraes](https://github.com/julinhafmm)

[Ulisses Fernandes](https://github.com/Ulilos)

---
