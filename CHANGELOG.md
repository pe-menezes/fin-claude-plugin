# Changelog

Todas as mudanças relevantes deste plugin ficam documentadas neste arquivo.

O formato é baseado em [Keep a Changelog](https://keepachangelog.com/),
e o projeto segue [Semantic Versioning](https://semver.org/).

## [Unreleased]

### Ajustes estruturais nas skills (2026-04-18)

Após investigação de falso positivo de divergência de saldo Itaú de R$12k (que era timing de Pix fim de semana com crédito D+1 útil + bug de cutoff em `CURRENT_DATE` no backend do FIN, já fixado em commit `66577e7`), incorporei três aprendizados nas skills pra não repetir o debate:

- **`extrato`** — seção "Reconciliação de saldo pós-import": agora diferencia explicitamente "saldo atual do app" vs "saldo do dia do extrato", cobre convenção D+1 de Pix em fim de semana, e define tolerância de ~R$5/mês pra rendimento de aplicação automática como ruído aceitável (não trava conciliação).
- **`conciliar`** — Modo 3 (conferência de saldo): mesma distinção saldo atual vs saldo do dia, mesma tolerância de rendimento. Evita falso positivo quando existe Pix fim de semana pendente com `tx_date` futura.
- **`lancar`** — nova seção "Pix em fim de semana / feriado (regra D+1 útil)": ao lançar Pix em sábado/domingo/feriado, default passa a ser data do próximo dia útil (pra bater com extrato quando conciliar). Confirmação explicita a regra aplicada.

## [0.4.0] - 2026-04-13

Minor release que sincroniza o plugin com o loop grande do FIN backend v2.3.4 — 20+ bugs/limitações/DX fechados no `fin-app-mcp` (de 33 pra 39 tools). Nenhuma quebra de instalação; é mudança **semântica** em como as skills conversam com a pessoa (principalmente USD e estornos).

Requer `fin-app-mcp@2.3.4` ou maior — o plugin usa `npx -y fin-app-mcp` então a atualização vem automática na próxima sessão. Se você instalou o plugin antes da v0.4.0, basta reiniciar o Claude Code pra pegar o MCP novo.

### Adicionado

- **6 tools novas no MCP** acessíveis via plugin (a skill `lancar`, `fatura`, `extrato`, `conciliar` e o agente `financeiro` foram atualizados pra conhecer):
  - `fin_patrimonio` — patrimônio consolidado multi-moeda (cash + checking) convertido pra BRL (ou USD) via cotação cached (1h). Responde *"quanto eu tenho hoje em reais, contando o dólar?"*.
  - `fin_criar_estorno` — estorno atômico. Passa `original_transaction_id` + `amount_cents` (parcial permitido), herda account/category/subcategory da transação original automaticamente. Substitui o fluxo manual de `fin_buscar_transacoes` + `fin_criar_despesa` com `reversal_of_id`.
  - `fin_classificar_linha_extrato` — classifier OFX/CSV. Recebe `{description, amount_cents, account_id, trntype?}` e retorna `{type: expense|income|transfer|payment_invoice|atm_withdrawal, details}`. Substitui heurística client-side pra saque ATM, pagamento de fatura, transfer cross-conta. Tem blocklist de palavras genéricas (PIX, TED, DOC) pra evitar falso positivo.
  - `fin_criar_transacoes_batch` — até 100 rows mix de expense/income/transfer em uma chamada. Partial success (linhas que falham não revertem as anteriores). A skill `extrato` usa pra lançamento em lote.
  - `fin_deletar_bill` — hard-delete de bill recorrente. Só funciona quando nenhuma ocorrência foi paga (preserva histórico). Alternativa ao soft-delete via `is_active: false`.
  - `fin_editar_occurrence` — edita uma ocorrência específica (`amount_cents`, `due_date`, `notes`) sem mexer no template mãe. Útil pra ajuste pontual de multa/reajuste num mês só.
- **Tags em transações** (`tags: string[]` em `fin_criar_despesa`, `fin_criar_receita`). Cortes ortogonais que complementam categoria/subcategoria (ex: `trabalho`, `reembolsável`, `viagem-NY`, `projeto-X`). A skill `lancar` captura no fluxo de confirmação.
- **Bills temporárias** (`end_date`, `max_occurrences` em `fin_criar_bill` / `fin_editar_bill`). Pra IPTU 10x, financiamentos, matrículas parceladas. Sem isso, bill gera ocorrência fantasma forever. A skill `onboarding` foi atualizada (Bloco 4, item 10) pra capturar isso no fluxo.
- **Bills com notas/tags/review_after** — contexto livre nas bills (antes ficava em `.md` separado no vault, frágil).
- **Pagar bill atrasado via link mode** — `fin_pagar_bill` aceita `transaction_id` pra amarrar a bill a uma transação já importada do extrato, sem criar duplicata. Resolve o caso "importou OFX primeiro, depois criou bill". A skill `extrato` menciona.
- **Multa/juros em `fin_pagar_bill`** — `fee_cents` + `fee_category_name`. Cria 2ª tx em "Taxas, Juros & Impostos > Multa" automaticamente.
- **Catch-up de mês passado** — `is_catch_up: true` + `catch_up_reference_month` em `fin_pagar_bill`. Permite pagar um mês atrasado junto com o corrente sem tocar na ocorrência corrente.
- **Filtro por status em `fin_bills_do_mes`** — `status: unpaid` / `pending,overdue` / etc. Resposta também traz `summary` agregado por status (`total_pending_cents`, `count_paid`, ...). A skill `conciliar` menciona.
- **Ocorrências pagas retornam dados da transação** — `account_name`, `category_name`, `tx_description`, `invoice_cycle_end` vêm resolvidos via JOIN. Antes vinha `null`.
- **`fin_editar_conta` aceita `currency`** — só funciona se a conta ainda não tem transações vinculadas (retorna 409 senão). Permite corrigir moeda de conta criada errada.
- **`fin_patrimonio` como roteamento direto** — o agente `financeiro` reconhece *"quanto eu tenho no total em reais"* / *"meu patrimônio"* e chama direto, sem passar por skill.

### Mudado

- **(SEMÂNTICO BREAKING) USD foi redesenhado.** A versão anterior deste plugin (0.3.0) dizia que *"contas em dólar são saldo manual. Não dá pra lançar despesa categorizada em USD ainda"* — correto pra v2.2/v2.3 do backend, mas **obsoleto a partir de v2.3.4**. Agora `fin_criar_despesa` e `fin_criar_receita` aceitam conta USD:
  - **Modo (a) — pessoa informa o BRL exato que saiu:** caller passa `amount_cents` (BRL) + `original_amount_cents` (USD) + `original_currency: "USD"`. Backend grava BRL como fonte da verdade pra relatórios e mantém o valor nativo em USD como metadata. Use quando a pessoa sabe o valor real que saiu da conta (Wise mostra, extrato do banco mostra).
  - **Modo (b) — pessoa informa a cotação:** caller passa `original_amount_cents` + `original_currency: "USD"` + `exchange_rate_cents_per_unit` (ex: `51023` = R$ 5,1023/USD). Backend calcula o BRL.
  - **Passar só `amount_cents` numa conta USD retorna 422.** A skill `lancar` pergunta ao usuário qual modo usar: *"gastou US$X na [Conta USD] — sabe quanto saiu em reais da tua conta, ou prefere estimar com uma cotação?"*.
  - **Por que cotação manual** (e não automática via `fin_cambio` interno): cotação varia por operação real (Wise spot ≠ casa de câmbio ≠ banco). Auto-converter na escrita distorce histórico. Exceção: `fin_patrimonio` converte dinamicamente **só na leitura** pra responder uma pergunta de balance, não de transação.
  - **`skills/lancar` Caso B e Caso C reescritos.** O workaround antigo (gasto em USD via `fin_ajustar_saldo_conta` sem categoria) foi removido — não era recomendação legada, era limitação de backend que saiu. Quem usava o workaround vai receber o fluxo novo na próxima sessão.
  - **`skills/onboarding` Bloco 1** reescrito — pergunta sobre conta USD e aviso de como operar (os 4 caminhos: despesa categorizada, câmbio, ajuste de saldo, patrimônio).
- **`fin_ajustar_saldo_conta` retorna saldo calculado.** A partir de v2.3.4, a tool devolve `balance_cents_calculado` e `delta_liquido_cents` no payload. Não precisa mais do roundtrip defensivo via `fin_saldos` pra validar. Passa o saldo desejado direto e confere no retorno. A armadilha do "sobrescreve `initial_balance`, não saldo exibido" continua existindo, mas agora a tool mostra o resultado pra você. As skills `lancar` e `extrato` foram simplificadas (a fórmula `initial_balance_novo = initial_balance_atual + (saldo_desejado − saldo_exibido_atual)` deixou de ser parte do fluxo normal — só é discutida se o retorno não bater).
- **Parcelamento retroativo via `original_purchase_date`.** A partir de v2.3.4, `fin_criar_despesa` aceita `original_purchase_date: YYYY-MM-DD` como caminho intuitivo pra compras já em andamento. Ex: *"Ray-Ban comprado em dezembro em 10x, tô na parcela 4"* → passa `installments: 10, current_installment: 4, original_purchase_date: "2025-12-23"` e o backend coloca a parcela 4 na fatura corrente do cartão + as próximas nas faturas futuras, tudo baseado no cutoff/closing_day. As skills `lancar` e `fatura` foram atualizadas — o workaround antigo de "lançar despesas avulsas numeradas manualmente" foi rebaixado a fallback raro.
- **`skills/extrato` usa `fin_classificar_linha_extrato`** em vez de heurística client-side pra detectar saque ATM, pagamento de fatura, transferência cross-conta.
- **`skills/extrato` usa `fin_criar_transacoes_batch`** em vez de loop sequencial pra lançamento em massa pós-parse.
- **`skills/lancar` e `skills/fatura` usam `fin_criar_estorno`** em vez de busca heurística + `fin_criar_despesa` com `reversal_of_id` manual.
- **`agents/financeiro.md`** ganhou 6 novas regras de negócio (bills v2.3.4, estorno atômico, classifier, batch, tags, parcelamento retroativo) e atualizou a regra USD e a de ajuste de saldo. Também ganhou 4 linhas novas no roteador de intenção (patrimônio, estorno, batch).

### Removido

- **Limitação "v0 USD"** da documentação do plugin. Não existe mais — USD é first-class a partir de v2.3.4.
- **Workaround "gasto em USD via ajuste de saldo sem categoria"** de `skills/lancar` e `skills/onboarding`. Substituído pelo fluxo nativo via `fin_criar_despesa`.

### Bumps

- `plugin.json`: 0.3.0 → 0.4.0
- `marketplace.json`: 0.3.0 → 0.4.0
- **Requer** `fin-app-mcp@2.3.4` ou maior (vem automático via `npx -y fin-app-mcp`).

## [0.3.0] - 2026-04-12

Minor release — primeira release em que o plugin é **efetivamente instalável e funcional** no Claude Code v2.1+ (desktop e CLI). Consolida as correções que antes existiam em branches de trabalho 0.2.1 e 0.2.2 e traz melhorias de skills aprendidas em uso real com a usuária piloto.

### Corrigido
- **Plugin instalava mas ficava com erro `Missing required user configuration value: fin_api_key`**. Causa: no `userConfig.fin_api_key` do `plugin.json`, faltava o campo `required: true`. Sem ele, o Claude Code v2.1+ considerava a config opcional e **nunca disparava o prompt de input da API key** durante a instalação. O `.mcp.json` então referenciava `${user_config.fin_api_key}` como string vazia e o MCP server do FIN não conseguia subir. Adicionado `required: true` — agora o Claude Code pede a API key no install como esperado.
- **Slash commands não apareciam no autocomplete** (`/financeiro:` não listava nada). Causa: o frontmatter das 6 skills declarava `allowed-tools` com separador por vírgula (`Read, Write, Edit, Glob, Grep, Bash`), formato não suportado pelo parser do Claude Code. A doc oficial exige **separador por espaço** (`Read Write Edit Glob Grep Bash`) ou lista YAML. Skills com `allowed-tools` inválido eram silenciosamente descartadas do registro de slash commands. Corrigido em todas as 6 skills.
- **`userConfig.fin_api_key`** no `plugin.json` faltava os campos `type: "string"` e `title: "FIN API Key"`, que são esperados pelo schema de userConfig do Claude Code. Adicionados.

### Mudado
- **`skills/lancar`**: adicionada seção sobre a armadilha do `fin_ajustar_saldo_conta` sobrescrever `initial_balance` em vez do saldo exibido. Fórmula correta: `initial_balance_novo = initial_balance_atual + (saldo_desejado − saldo_exibido_atual)`. Caso B e Caso C reescritos com exemplo real.
- **`skills/fatura`**: clarificações sobre (a) Data da Fatura Inicial do cartão, (b) diferença parcela 1 vs N, (c) uso de `current_installment` a partir da v2.3.2 do `fin-app-mcp` com regra dos dois cenários (alinhado vs desalinhado).
- **`skills/extrato`**: novo passo final obrigatório "Reconciliação de saldo pós-import", explicando como calcular `initial_balance` retroativamente pra fechar o saldo com o extrato.
- **`skills/onboarding`**: nova convenção "memória operacional mora no vault da pessoa" (todo o contexto do FIN vive em `Financeiro/` do vault, não em `~/.claude/projects/*/memory/`). Explica estrutura recomendada de `Sessões/`.

### Bumps
- `plugin.json`: 0.2.0 → 0.3.0
- `marketplace.json`: 0.2.0 → 0.3.0

## [0.2.0] - TBD

### Adicionado
- **Auto-instalação do MCP do FIN App** (zero edição manual de config):
  - `.mcp.json` na raiz do plugin declarando `fin-app` via `npx -y fin-app-mcp`, com env vars `FIN_BASE_URL` (fixa) e `FIN_API_KEY` (referenciando `${user_config.fin_api_key}`)
  - `userConfig.fin_api_key` no `plugin.json` marcado como `sensitive: true` (chave guardada no keychain do sistema, não no histórico da conversa)
  - Resultado: instalação em **3 passos** (`marketplace add` → `plugin install` → colar API key quando perguntado)
  - Nada de editar `claude_desktop_config.json`. Nada de `claude mcp add`. Tudo automático.
- **Suporte a USD / câmbio (compatível com FIN MCP v2.2.0+):**
  - Agente principal: roteamento de 5 padrões novos (vender/comprar dólar, ajustar saldo USD, gastar em USD, ajustar saldo BRL), regras de negócio do USD, leitura obrigatória da seção 10 do `fin://docs/guia` ao operar USD pela primeira vez na sessão
  - `skills/lancar`: novo "Passo 4.5 — Tratamento especial: USD, câmbio e ajuste de saldo" com 3 casos detalhados (Câmbio, Ajuste manual USD/BRL, Gasto em USD via ajuste de saldo). Tabela de exemplos ampliada com 5 frases novas. Notação USD adicionada
  - `skills/onboarding`: Bloco 1 ganha pergunta sobre conta em dólar e captura `currency`. Aviso sobre limitações do v0 USD. Modo C reconhece contas USD e exclui transações de "Câmbio" do agrupamento de estabelecimentos
  - 6 erros novos no checklist do agente principal e da skill `lancar`
  - Spec do plugin (no vault) atualizada com seção 8.3 sobre suporte USD

### Mudado
- `skills/instalar-fin-mcp` virou **fallback manual**: só roda se a auto-instalação do `.mcp.json` falhar, ou se a pessoa quiser revogar/atualizar a chave. Ganhou seção explicando o método novo no início.
- `agents/financeiro.md` atualizado: na verificação inicial do MCP, espera que o método automático tenha funcionado e só dispara fallback se detectar problema.
- README do repo: nova seção "Instalação em 3 passos" + "Atualizar a chave depois" + "Se a instalação automática falhar".

### Bumps
- `plugin.json`: 0.1.0 → 0.2.0
- `marketplace.json`: 0.1.0 → 0.2.0

## [0.1.0] - TBD

### Added
- Plugin scaffold inicial
- `.claude-plugin/plugin.json` e `marketplace.json`
- README e LICENSE
- Estrutura de pastas: `agents/`, `skills/`, `templates/`
