# financials-agent

> Agentic RAG chatbot over company financials, starting with SEC filings.

Projeto de portfólio: um **chatbot no Telegram** com **Agentic RAG** sobre informações financeiras de empresas. A primeira fonte são os documentos públicos da **SEC** (10-K, 10-Q, 20-F, 6-K…); o nome deixa o escopo aberto para outras fontes no futuro (CVM, releases de RI, transcrições de *earnings calls*). O agente (Claude, via Messages API) decide sozinho quando usar cada tool: busca na web, busca semântica nos textos dos filings (RAG) e consulta a uma base numérica estruturada (fatos XBRL).

Este arquivo registra as decisões de arquitetura (com os motivos) e a ordem de execução do projeto.

---

## 1. Convenções

- Python 3.12. Toda a infraestrutura (Postgres/pgvector e os serviços) sobe com `docker compose`.
- **Segredos e dados pessoais** (chave da Anthropic, token do Telegram, senha do Postgres, email do User-Agent da SEC, IDs da whitelist) ficam em `.env`, que nunca vai para o git. Mantenha um `.env.example` versionado, sem valores reais.
- **Configurações** (modelo, effort, `max_tokens`, janela de conversa, `max_uses` do web search, lista de empresas da SEC, parâmetros de chunking, modelo de embedding) ficam em `config.toml`, **versionado**. Nada disso fica fixo no código.
- A configuração é carregada com `pydantic-settings`: primeiro o `config.toml`, depois o `.env` e as variáveis de ambiente por cima (útil para testar outro modelo ou configurar o Docker sem editar o arquivo).
- **Commits** seguem o [Conventional Commits](https://www.conventionalcommits.org), em inglês: `<tipo>(<escopo>): <descrição no imperativo>`. Tipos: `feat`, `fix`, `docs`, `refactor`, `test`, `chore`, `build`, `ci`, `perf`, `style`. Escopos: `chatbot`, `ingestion`, `shared`, `mcp`. Só a linha de assunto, sem corpo nem rodapé; uma mudança lógica por commit; `!` marca quebra de compatibilidade.

---

## 2. Arquitetura geral

```
                    ┌──────────────────────── Postgres + pgvector (Docker) ─────────────────────────┐
                    │ companies · filings · filing_documents · financial_facts · chunks             │
                    │ users · conversations · messages                                              │
                    └───────────────▲───────────────────────────────▲─────────────────▲────────────┘
                                    │ escreve                       │ lê              │ lê
 SEC EDGAR ──> [ingestion/] (agendado)                              │                 │
               discover → download → companyfacts → parse → chunk → embed            │
                                                                    │                 │
 Telegram ──> [chatbot/] whitelist → conversa → loop do agente ─> [shared/tools] ─────┘
                                     (Messages API + web_search)          ▲
                                                                          │ mesmas funções
 Claude Code (uso local)   ──MCP──> [mcp_server/] ─────────────────────────┘
```

### Monorepo
Um repositório só, com serviços separados:
```
financials-agent/
├── AGENTS.md
├── docker-compose.yml      # db (pgvector/pgvector) + chatbot + ingestion
├── config.toml             # configurações versionadas
├── .env.example            # modelo do .env (segredos e dados pessoais)
├── shared/                 # schema do banco, config, modelo de embedding, tools de dados
├── chatbot/                # bot do Telegram + loop do agente
├── ingestion/              # pipeline da SEC (agendado)
├── mcp_server/             # expõe shared/tools via MCP (Claude Code ou outro cliente MCP)
└── tests/
```
**Por que monorepo:** é um projeto solo de portfólio. Um link só mostra a arquitetura inteira, um `docker compose up` sobe tudo, o schema e as migrações do banco (o contrato entre os serviços) ficam num lugar só, e o **modelo de embedding é garantidamente o mesmo** na ingestão e na consulta.

---

## 3. LLM: Claude via Messages API

### Decisão
- **Messages API**, pelo SDK Python oficial (`anthropic`), autenticada com **chave de API** do Console (platform.claude.com), cobrada por token com créditos pré-pagos. Não existe cota mensal gratuita. Configure um limite de gasto no Console.
- **Por que não a assinatura nem o Agent SDK:** a documentação do Agent SDK diz que, sem aprovação prévia, não é permitido oferecer o login do claude.ai (assinatura) em produtos de terceiros, inclusive agentes feitos com o Agent SDK. Além disso, o Agent SDK é o Claude Code empacotado: as tools padrão dele (arquivos, terminal) não fazem sentido num bot, as sessões ficam em arquivos locais (não no nosso Postgres) e a migração depois seria em grande parte código descartável. O **Claude Code Channels** (plugin oficial de Telegram para uma sessão local do Claude Code, que usa a assinatura) serve para uso pessoal, não para construir um chatbot próprio. A assinatura do Claude fica para **desenvolver** o projeto com o Claude Code e para usar o servidor MCP (seção 7).
- **Modelo:** configurável no `config.toml`. Padrão de produção: `claude-opus-5-5`. Para desenvolvimento barato: `claude-haiku-5-5`. Use só IDs exatos, sem sufixo de data.
- **Thinking:** no Opus 5.5 ele está sempre ligado (adaptativo); `{"type": "disabled"}` dá erro 400. A profundidade se controla com `output_config={"effort": ...}`. O padrão do Opus 5.5 é `medium`; para chat, `low` costuma bastar. Deixe configurável.
- Não use prefill de mensagem do assistente (dá 400 nos modelos atuais) nem `tool_choice` forçado (`any`/`tool` dá 400 no Opus 5.5); use `auto`.
- **Recusas:** trate `stop_reason == "refusal"`. Para o Opus 5.5, avalie o parâmetro server-side `fallbacks` (beta `server-side-fallback-2026-07-01`, `fallbacks="default"`).
- **Prompt caching:** o histórico é reenviado a cada turno, então use `cache_control={"type": "ephemeral"}`. Mantenha o system prompt e a lista de tools estáveis (sem data ou hora no system prompt, tools em ordem fixa), porque qualquer byte diferente no prefixo invalida o cache.

### Tools
| Tipo | Exemplos | Quem executa |
|---|---|---|
| Tool de servidor | `web_search` (`{"type": "web_search_20260209", "name": "web_search", "max_uses": 5}`) | A Anthropic. Só se declara. A busca é cobrada à parte. |
| Tool com schema da Anthropic | `bash`, `text_editor`, `memory` | O nosso código. **Não usamos por enquanto.** |
| Tool própria | `search_filings` (RAG), `query_financial_facts` (SQL) | O nosso código (`shared/tools`) |

As tools nativas do Claude Code (Read, Bash, WebSearch…) **não existem** na Messages API.

### Loop do agente: manual (decidido)
O loop é escrito à mão (~40 linhas), sem o Tool Runner (`client.beta.messages.tool_runner`, que está em beta). Motivos: precisamos persistir cada mensagem nova, inclusive os `tool_result`, e o loop explícito demonstra domínio do ciclo.
```python
while True:
    response = client.messages.create(model=..., system=..., tools=..., messages=messages, ...)
    messages.append({"role": "assistant", "content": response.content})
    if response.stop_reason == "tool_use":       # o modelo pediu uma tool nossa
        results = [executar(b) for b in response.content if b.type == "tool_use"]
        messages.append({"role": "user", "content": results})   # TODOS os tool_result numa única mensagem
        continue
    if response.stop_reason == "pause_turn":     # tool de servidor (web search) pausou: reenviar para continuar
        continue
    break                                         # end_turn, max_tokens, refusal
```
- Ponha um limite de iterações (ex.: 10) para evitar loops infinitos.
- Uma tool que falhar volta como `tool_result` com `is_error: true`; nunca descarte o resultado.
- Ao serializar os blocos para o banco, use `block.to_dict()` (preserva assinaturas e campos exatamente como a API espera).
- Com web search, as respostas trazem `citations` nos blocos `text`. Liste as URLs únicas como "Fontes" no fim da mensagem do Telegram.

### Blocos de conteúdo
A resposta da API é uma lista de blocos tipados: `text` (com `citations` opcionais), `thinking` (texto vazio por padrão, mas com uma `signature` que precisa voltar intacta), `tool_use` (`id`, `name`, `input`), `tool_result` (enviado por nós: `tool_use_id`, `content`, `is_error`), `server_tool_use` e `web_search_tool_result` (busca feita pela Anthropic, com `encrypted_content`).

---

## 4. Canal: Telegram

- Só Telegram por enquanto. Interface web (Streamlit) e WhatsApp ficam para depois; WhatsApp só pela Cloud API oficial da Meta, nunca por bibliotecas não oficiais.
- Biblioteca: `python-telegram-bot` (assíncrona), em modo polling.
- **Whitelist determinística antes de qualquer chamada ao LLM:** usar o **ID numérico** do usuário no Telegram (o `@username` pode mudar). Quem não está na lista é **ignorado em silêncio** e só registrado no log.
- Um lock por usuário, para que mensagens seguidas não embaralhem o histórico. Processar updates de forma concorrente entre usuários diferentes.
- Respostas com mais de 4096 caracteres são divididas em várias mensagens.
- Indicador "digitando…" enquanto o agente trabalha.
- O **núcleo do chat é independente de canal**: o Telegram é só um adaptador fino. Outros canais devem poder chamar o mesmo serviço.

---

## 5. Conversas e histórico

### Definições
- **Conversa:** conjunto de mensagens com o mesmo contexto.
- **Turno:** uma pergunta do usuário e tudo o que acontece até a resposta final. Um turno pode gerar várias linhas no banco.
- **Mensagem:** uma linha na tabela `messages`.

### Regras (estratégia "A": inatividade + comando)
- Uma nova conversa começa quando passam **10 minutos desde a última resposta do bot** (configurável no `config.toml`; conta-se da resposta, não da pergunta, para que um processamento longo não "mate" a conversa).
- `/nova` força uma conversa nova.
- Responder (*reply*) a uma mensagem antiga do bot retoma a conversa daquela mensagem, mesmo fora da janela. Por isso guardamos o `telegram_message_id` das respostas.
- Dentro da conversa, cada nova mensagem é enviada com **todo o histórico da conversa atual** (a API não guarda estado).
- Evolução futura (estratégia "D"): ao abrir uma conversa nova, injetar um resumo da anterior.
- Descartado: classificar com LLM se a mensagem é continuação (latência e custo extras; quando erra, o bot "esquece" no meio do assunto).

### Persistência: guardar tudo (decidido)
Todas as mensagens vão para o banco, **inclusive `tool_use`/`tool_result`, thinking e resultados de busca**, com o `content` completo em JSON. Motivos: perguntas de seguimento precisam dos dados já trazidos pelas tools e das fontes; é a única forma de depurar se um erro veio do modelo ou da tool; e a avaliação com Ragas precisa dos contextos recuperados. A coluna `text` é preenchida só nas mensagens do usuário e nas respostas finais, para ter uma visão "limpa" da conversa:
```sql
SELECT role, text FROM messages WHERE conversation_id = 42 AND text IS NOT NULL;
```
Conversas expiradas **continuam no banco**; só deixam de ser enviadas ao modelo. Um turno só é gravado depois de completo, para o histórico nunca ficar com um `tool_use` sem o `tool_result` correspondente.

### Tabelas do chatbot
```
users          (id, channel, external_id, display_name, created_at)  UNIQUE(channel, external_id)
conversations  (id, user_id → users, started_at, last_activity_at, closed_at)
messages       (id, conversation_id → conversations, role, content JSONB, text,
                model, input_tokens, output_tokens, telegram_message_id, created_at)
```

---

## 6. Dados: SEC EDGAR

### Fonte
- **Lista de empresas** configurável. Para cada uma, buscar **10-K, 10-Q** (americanas), **20-F, 6-K** (estrangeiras) e considerar o **8-K** (o EX-99.1 traz o release de resultados das americanas). Incluir emendas (`/A`). O **40-F** (canadenses) é opcional.
- Endpoints (JSON oficiais, sem scraping de páginas):
  - `https://www.sec.gov/files/company_tickers.json`: ticker → CIK
  - `https://data.sec.gov/submissions/CIK##########.json`: filings da empresa (CIK com 10 dígitos e zeros à esquerda)
  - `https://www.sec.gov/Archives/edgar/data/{cik}/{accession}/`: documentos do filing
  - `https://data.sec.gov/api/xbrl/companyfacts/CIK##########.json`: **todos os fatos XBRL já processados**. Usar isto para a base numérica em vez de parsear os XMLs na mão.
- **Regras da SEC:** um `User-Agent` com nome e email de contato é obrigatório (o email vem do `.env`) e o limite é de **10 requisições por segundo**, com rate limiter no código.

### Peculiaridades
- 10-K e 10-Q são HTML com **inline XBRL** (tags `ix:`).
- O 20-F normalmente usa **IFRS** (taxonomia `ifrs-full`), e as americanas usam `us-gaap`. Daí a coluna `taxonomy`.
- O **6-K quase nunca tem XBRL financeiro**: o conteúdo útil está nos anexos (EX-99.1), às vezes em PDF. Serve para o RAG, não para a base numérica.
- **Não existe 10-Q do 4º trimestre:** o 4T = anual − 9 meses.
- **Ano fiscal ≠ ano civil** (o ano fiscal da Apple termina em setembro). Períodos como "3T26" se referem ao `fiscal_year`/`fiscal_period` do XBRL (`dei:DocumentFiscalPeriodFocus`).

### Pipeline de ingestão (`ingestion/`, agendado)
`discover` (novos filings da lista) → `download` (documentos para armazenamento local) → `companyfacts` (base numérica) → `parse` (HTML → Markdown) → `chunk` → `embed`.
- **Idempotente:** o accession number é único; só se processa o que ainda não existe.
- Uma coluna **`status`** (ex.: `discovered → downloaded → parsed → embedded`, mais `failed`) permite reprocessar só o que falhou.
- **`storage_uri`** em vez de "path": hoje um caminho local, no futuro S3/MinIO, sem mudar o schema.
- Agendamento: a definir (APScheduler, cron num container ou o agendador do sistema operacional).

### Tabelas de dados
```
companies        (cik PK, ticker, name, is_foreign)
filings          (accession_number PK, cik → companies, form, filing_date, report_date,
                  fiscal_year, fiscal_period, status)
filing_documents (id, accession_number → filings, filename, doc_type [10-K, EX-99.1…],
                  storage_uri, sha256, downloaded_at, parsed_at, status)
financial_facts  (id, cik, taxonomy, concept, unit, value, period_start, period_end,
                  fiscal_year, fiscal_period, form, accession_number, frame)
chunks           (id, filing_document_id → filing_documents, section, content, embedding VECTOR(n),
                  embedding_model, cik, fiscal_year, fiscal_period, form, tsv TSVECTOR)
```

---

## 7. RAG

### Extração (HTML → Markdown)
- Como todos os documentos vêm da mesma fonte, usamos um **script dedicado e determinístico**: **BeautifulSoup** seleciona o conteúdo e remove o que não interessa (scripts, estilos, cabeçalhos e rodapés repetidos, tags `ix:` ocultas), e **markdownify** converte o resto para Markdown, **preservando tabelas, títulos e listas**.
- **PDFs** (alguns anexos de 6-K): decidir quando aparecerem. A opção preferida é o **Docling** (parser de PDF com modelos pequenos de layout e tabela; roda em CPU). Alternativas: `pymupdf4llm`, `pdfplumber`, OCR, ou um LLM com visão para os casos extremos.

### Chunking
- Primeira fronteira: o **Item** do formulário (no 10-K: Item 1 Business, 1A Risk Factors, 7 MD&A, 7A, 8…; o 10-Q e o 20-F têm os seus). O item vira metadado (`section`).
- Dentro do item, quebrar por subtítulo e depois por tamanho (~500–1000 tokens, com 10–20% de sobreposição).
- Cada chunk carrega metadados para **filtrar antes da busca vetorial** (`WHERE cik = … AND fiscal_year = … AND fiscal_period = …`). Isso resolve a fraqueza dos embeddings com números e períodos: o vetor de "3T25" e o de "3T26" são quase iguais.

### Embeddings
- Os documentos estão em **inglês** e as perguntas virão em **português**, então o modelo de embedding precisa ser **multilíngue** (busca entre idiomas). Escolha do modelo a definir na etapa de RAG (ex.: família multilingual-e5 ou bge-m3, local via `sentence-transformers`, ou uma API).
- Gravar `embedding_model` em cada chunk. A ingestão e o chatbot usam o mesmo modelo, definido em `shared/`.

### Busca
1. **Vetorial:** pgvector, distância de cosseno (`<=>`), índice HNSW (`USING hnsw (embedding vector_cosine_ops)`).
2. **Palavra-chave:** full-text do Postgres (`tsvector` com a configuração `english`, índice GIN). Atenção: o `ts_rank` não usa IDF, ou seja, não é BM25. Para BM25 de verdade, avaliar a extensão ParadeDB `pg_search` ou a lib `rank_bm25`.
3. **Fusão por RRF:** `score = Σ 1/(60 + posição)` sobre as listas (posição começa em 1), opcionalmente com pesos.
4. **Re-ranking:** um cross-encoder multilíngue reordena os ~30 candidatos e ficam os ~5 melhores.

### Agentic RAG
O RAG é uma **tool** que o agente usa **quando julgar necessário**, e não um passo fixo em toda mensagem. Tools previstas em `shared/tools`:
- `search_filings(query, cik?, fiscal_year?, fiscal_period?, form?, section?)`: busca híbrida + re-ranking, devolvendo os trechos com a fonte (empresa, form, período, item)
- `query_financial_facts(...)`: consulta estruturada a `financial_facts`. **Números vêm do SQL, não do RAG.**

A `description` de cada tool é o que faz o modelo saber quando usá-la; escreva-a com cuidado e teste.

### Servidor MCP (`mcp_server/`)
Expõe as mesmas funções de `shared/tools` via MCP, para explorar os dados em conversas no **Claude Code** (ou em qualquer cliente MCP). O Claude Code inicia o processo sozinho quando ele está registrado (`claude mcp add` ou `.mcp.json`). O bot **não** passa pelo MCP: ele chama as funções diretamente.

---

## 8. Avaliação

- **Ragas** (open source; o custo é das chamadas ao LLM juiz). Métricas: faithfulness, answer relevancy, context precision/recall.
- Montar um conjunto de teste com perguntas e respostas de referência sobre os filings, incluindo perguntas que **devem** e que **não devem** acionar cada tool.
- Usar os resultados para **comparar versões** (ex.: busca só vetorial × híbrida × híbrida + re-ranking), não como nota absoluta. Documentar os números no README.

---

## 9. Roadmap (ordem das etapas)

| # | Etapa | Status |
|---|---|---|
| 1 | **Esqueleto do monorepo + `docker-compose` com Postgres/pgvector**: estrutura de pastas, config (`config.toml` + `.env`), schema e migrações (Alembic), Dockerfile do chatbot | ⏳ próximo |
| 2 | **Chatbot no Telegram**: Messages API + `web_search`, loop manual, whitelist, histórico completo no Postgres, janela de 10 min, `/nova`, retomada por *reply*, fontes no fim da resposta, testes | ⬜ |
| 3 | **Ingestão da SEC**: discover → download → companyfacts → parse (BeautifulSoup + markdownify) → chunk → embed, idempotente e com status; agendamento | ⬜ |
| 4 | **Tools de dados**: `query_financial_facts` (SQL) e `search_filings` (RAG vetorial com filtros de metadados), integradas ao bot e expostas no servidor MCP | ⬜ |
| 5 | **Qualidade da busca**: busca híbrida (full-text/BM25 + RRF) e re-ranking com cross-encoder | ⬜ |
| 6 | **Avaliação** com Ragas e conjunto de teste; resultados no README | ⬜ |
| 7 | **Extras**: resumo da conversa anterior (estratégia D), PDFs com Docling, interface Streamlit, WhatsApp, deploy | ⬜ |

**Próximo passo:** detalhar as etapas 1 e 2 (estrutura de pastas, tabelas, fluxo de uma mensagem do Telegram até a resposta).

---

## 10. Referências

- Agent SDK (autenticação): https://code.claude.com/docs/en/agent-sdk/overview
- Claude Code Channels: https://code.claude.com/docs/en/channels
- Tratamento de stop reasons: https://platform.claude.com/docs/en/build-with-claude/handling-stop-reasons
- Acesso aos dados da SEC (User-Agent, limites): https://www.sec.gov/os/accessing-edgar-data
- APIs EDGAR (submissions, companyfacts): https://www.sec.gov/search-filings/edgar-application-programming-interfaces
- pgvector: https://github.com/pgvector/pgvector
- Ragas: https://docs.ragas.io
