---
name: fatura
description: >
  Processa fatura de cartão de crédito em lote e lança no FIN App via
  fin_fatura_transacoes (NÃO via despesa avulsa, regra de negócio do FIN).
  Aceita 5 formatos (preferência: OFX > CSV > CNAB 240 > texto colado > PDF).
  Identifica o cartão automaticamente, detecta período REAL da fatura
  (importante: pode variar do ciclo cadastrado), trata estorno (reversal_of_id)
  e parcelamento sem duplicar. Garante idempotência. No fim, oferece pagar
  a fatura via fin_pagar_fatura.
argument-hint: "[caminho-do-arquivo OU 'colado']"
allowed-tools: Read Write Edit Glob Grep Bash
---

## Quando usar

- Pessoa quer processar **uma fatura de cartão de crédito inteira**
- Frases tipo "processa essa fatura", "tá aí o OFX da fatura do [cartão]", "fatura do mês"
- Conciliação retroativa de faturas

## Quando NÃO usar

- **Extrato de conta corrente** → use `/financeiro:extrato`
- Lançamento de **uma única despesa** no cartão → use `/financeiro:lancar`
- Pagar fatura sem lançar transações → chame `fin_pagar_fatura` direto
- MCP do FIN não tá instalado → `/financeiro:instalar-fin-mcp`
- Plugin nunca rodou → `/financeiro:onboarding` primeiro

## Pré-requisitos

1. MCP do FIN responde
2. `~/.fin-plugin/config.json` existe → leia `financeiro_path`
3. Os 4 arquivos em `Financeiro/` existem
4. **Leu `fin://docs/guia` nessa sessão** (CRÍTICO pra essa skill — fatura tem várias regras de negócio do FIN)
5. **Pra fatura histórica (anterior à criação do cartão no FIN), confira a "Data da Fatura Inicial" do cartão no app.** Se a primeira transação da fatura é anterior a essa data, o app não a soma corretamente (apesar da API contar). Quando relevante, oriente: "se essa fatura é anterior à criação do cartão no FIN, abre o app e ajusta a 'Data da Fatura Inicial' pra uma data anterior à primeira transação antes de eu lançar." Detalhe completo no caso especial "Primeira fatura histórica de um cartão novo no FIN".

## Diferenças críticas vs `extrato`

| Aspecto | Extrato (conta) | Fatura (cartão) |
|---|---|---|
| Tool de lançamento | `fin_criar_despesa`, `fin_criar_receita`, `fin_criar_transferencia` | **`fin_fatura_transacoes`** (uma chamada com lista) |
| Período | Datas do extrato (mês civil normalmente) | **Período REAL da fatura** (pode variar do ciclo cadastrado) |
| Estorno | Raro | **Comum** — tratamento especial via `reversal_of_id` |
| Parcelamento | Não se aplica | **Comum** — FIN gera parcelas automaticamente, não duplicar |
| Pagamento | Não se aplica | Depois de lançar, oferece `fin_pagar_fatura` |
| Conta destino | Conta corrente/poupança | Cartão de crédito (também é conta no FIN) |

## Fluxo principal

### Passo 1 — Receber a fatura

3 formas (mesmo do extrato):

**Forma A: Caminho de arquivo em `$ARGUMENTS`**
**Forma B: Pessoa colou o conteúdo na conversa**
**Forma C: Pessoa anexou um PDF**

### Passo 2 — Identificar o cartão

Tenta identificar de qual cartão é a fatura:

1. **Pelo nome do cartão no conteúdo** (ex: nome da bandeira/emissor que aparece no header da fatura)
2. **Pelo BIN/IIN** (primeiros 6 dígitos do número do cartão se aparecer)
3. **Pelo final do cartão** (últimos 4 dígitos)
4. **Pelo nome do emissor**

Cruza com `Contas e Cartões.md` pra achar o cartão correspondente.

**Se não conseguir identificar:**
> Não consegui identificar de qual cartão é essa fatura. Qual dos teus cartões? [lista cartões de crédito de Contas e Cartões.md]

**Se identificar mais de um possível:**
> Pode ser [cartão A] ou [cartão B]. Qual?

### Passo 3 — Detectar período REAL da fatura

**REGRA CRÍTICA:** o período da fatura pode ser diferente do dia de fechamento cadastrado por causa de fim de semana, feriado, ciclo do banco.

Procura no conteúdo da fatura:
- "Período: 12/02/2026 a 11/03/2026" (string explícita)
- "Compras de DD/MM a DD/MM"
- Datas das transações (primeira e última)

O **período real** = (data da primeira transação, data da última transação) ou as datas explícitas do cabeçalho da fatura, o que for mais confiável.

**⚠️ Detectar arquivo histórico vs arquivo-da-fatura.** Vários bancos exportam o CSV/OFX com o histórico inteiro do cartão (várias faturas misturadas), não só a fatura corrente. Se o range de datas do arquivo for maior que ~45 dias, **NÃO assuma que é tudo a mesma fatura**. Mostra pra pessoa o range encontrado e os blocos prováveis (linhas pré-data X, linhas pós-data X) e pergunta:

> Esse arquivo cobre **N faturas** (de DD/MM a DD/MM). Vou processar só a fatura **[período real detectado]**, e ignorar as outras linhas. Confirma?

Linhas fora do período da fatura corrente devem ser **descartadas com motivo explícito** ("já em fatura paga", "fatura futura", "linha de pagamento — ver Casos especiais").

**Compara com o cartão em `Contas e Cartões.md`:**

```
Cartão: [nome]
Fechamento cadastrado: dia X
Período REAL detectado: DD/MM/YYYY a DD/MM/YYYY
→ Variação: fechamento real foi dia X+N (esperado dia X). Diferença +N dias.
```

Se variou ≤2 dias, **anota em `Contas e Cartões.md`** e segue:
- Coluna "fechamento observado" → atualiza pro dia mais recente
- Seção "Histórico de variação" → adiciona linha com data e motivo (se inferível: "fim de semana", "feriado", etc.)

**Avisa a pessoa:**
> Detectei: fatura do **[cartão]**, período real **DD/MM/YYYY a DD/MM/YYYY** (fechamento observado dia X, cadastrado é dia Y, diferença de N dias). Atualizei tua memória.

**Se variou >2 dias OU se nome do cartão na fatura é diferente do cadastrado:** **PAUSA, não segue.** Pergunta:

> O cartão **[nome no cadastro]** tá com fechamento dia [X], vencimento dia [Y] na minha memória. Mas essa fatura tá com fechamento dia [X+N] / vencimento dia [Y+N] / nome comercial "[nome real]". Isso pode ser:
> 1. O cartão mudou de plano/regra recentemente → atualizo cadastro com `fin_editar_conta`?
> 2. Essa fatura é de outro cartão que tu tem → me diz qual?
> 3. O cadastro tava errado desde o início → corrijo?
>
> Sem isso, posso lançar no cartão errado.

Não prossegue sem resposta. Esse cuidado existe porque banco brasileiro às vezes muda dia de vencimento na troca de plano sem o cliente perceber, e a memória do plugin pode ter sido cadastrada errada no `onboarding`.

### Passo 4 — Identificar a data de vencimento

Procura na fatura:
- "Vencimento: DD/MM/YYYY"
- "Pagar até DD/MM/YYYY"

Anota pra usar depois no `fin_pagar_fatura`.

**Se o vencimento detectado diverge >2 dias do `due_day` cadastrado** → ver Passo 3, mesmo fluxo de pausar e perguntar.

### Passo 5 — Parse das transações da fatura

Mesma lógica do `extrato` pros 5 formatos (OFX, CSV, CNAB 240, texto, PDF). Extrai pra cada transação:

- Data
- Descrição
- Valor (despesas geralmente positivas em fatura, mas o sinal depende do formato — atenção)
- Categoria/MCC se disponível (alguns formatos dão isso)
- **Parcelamento detectado** (ex: "PARC 03/12", "1/10")
- **Estorno detectado** (ex: "ESTORNO", valor negativo numa fatura, "DEVOLUÇÃO")

### Passo 6 — Tratamento de estornos

**REGRA CRÍTICA do FIN:** estorno em cartão NÃO é receita. No banco vira despesa vinculada à original (coluna `reversal_of_id`), mas **a única forma de criar é via `fin_criar_estorno`** — `fin_criar_despesa` não aceita mais `reversal_of_id` no body (retorna 400).

Pra cada transação que parece estorno (descrição contém "ESTORNO" / "DEVOLUÇÃO" / "REEMBOLSO" OU valor negativo numa fatura):

1. **Achar a despesa original** no FIN:
   - `fin_buscar_transacoes` filtrando por mesmo cartão, valor próximo (positivo do mesmo módulo), descrição parecida, dentro de uma janela de tempo (tipicamente até 3 meses antes)
2. **Se achou exatamente uma:** chama **`fin_criar_estorno`** (tool atômica): passa `original_transaction_id` + `amount_cents`. Valida ownership + 7 invariants, herda account/category/subcategory da original automaticamente, deriva `reversal_kind` ('full' ou 'partial') baseado no amount. Sem precisar passar descrição (vira "Estorno: <original desc>") nem conta.
3. **Se achou várias possíveis:** mostra pra pessoa: "Achei [N] despesas que podem ser a original desse estorno. Qual é? [lista]"
4. **Se não achou nenhuma:** avisa: "Não achei a despesa original desse estorno (R$X em [estabelecimento]). Vou lançar como despesa normal (sem vínculo com original). Tu pode deletar depois e recriar com `fin_criar_estorno` se achar a original."

**Nunca lança estorno como receita.**

### Passo 7 — Tratamento de parcelamento

**TL;DR:**
- **1/N (parcela 1)** → `fin_criar_despesa` com `installments: N`, `amount_cents = valor_parcela * N`. FIN cria as outras N-1 sozinho.
- **X/N onde 1 já tá no FIN** → pula, FIN já gerou.
- **X/N onde 1 NÃO tá no FIN (compra antiga)** → `fin_criar_despesa` com `installments: N` + `current_installment: X` + `original_purchase_date: <data real da compra>`. FIN coloca cada parcela na fatura certa.

**REGRA CRÍTICA do FIN:** parcelamento gera múltiplas transações automaticamente no FIN. Você não duplica.

Pra cada transação que parece parcelada:

1. **Detectar parcela**: descrição contém "PARC X/Y", "1/10", "(1/12)", etc.
2. **Identificar parcela 1 vs parcela N**:
   - **Parcela 1 (1/N)**: lança via `fin_criar_despesa` com `installments: N` e `amount_cents = valor_da_parcela * N` (valor TOTAL da compra). O FIN cria as N parcelas automaticamente nas faturas seguintes. **NÃO passar `current_installment`** — deixar o default (1).
   - **Parcela N (N > 1, com histórico da parcela 1 já no FIN)**: já foi gerada pelo FIN quando você lançou a parcela 1. **NÃO LANCE.** Pula.
   - **Parcela X (X > 1, sem histórico anterior no FIN — compra começou antes do FIN ser usado)**: **caminho preferido:** passa `installments: N`, `current_installment: X` e **`original_purchase_date`** com a data REAL da compra original. O backend caminha pelos ciclos do cartão (usando closing_day/due_day) e coloca cada parcela (X, X+1, ..., N) na fatura correta automaticamente. Não precisa pensar em `tx_date` nem `invoice_cycle_end` — só informar a data da compra histórica.
3. **Por que `original_purchase_date` é melhor que os workarounds anteriores:** antes o caller tinha que escolher entre (a) passar `tx_date` alinhado com o mês atual (anti-intuitivo) ou (b) lançar N-X+1 despesas avulsas numeradas manualmente. Ambos davam erro fácil. Agora é uma chamada só com os 3 campos + `original_purchase_date` e o backend coloca as parcelas nas faturas certas.
4. **Como saber se é parcela 1 ou N?**
   - Se for "1/12" no nome → parcela 1 (lançar sem `current_installment`)
   - Se for "5/12" no nome → buscar no FIN se a parcela 1 (ou qualquer anterior) já existe:
     - Se existe → FIN já gerou 5/12, pular.
     - Se não existe → compra começou antes do FIN. Use `original_purchase_date` + `current_installment` como no item 2 acima.
   - Se for "N/N" (última parcela) e sem histórico no FIN → lançar como **despesa única à vista** com o valor da parcela. Passar `invoice_cycle_end` = data de fechamento da fatura atual se a `tx_date` original cai num ciclo antigo.
   - Se a descrição não diz qual parcela é → busca no FIN. Se já tem, é continuação. Se não tem, assume à vista.

**Mostra na tabela de revisão:**
```
=== PARCELAMENTOS DETECTADOS ===
| Descrição               | Parcela | Ação                                  |
|-------------------------|---------|---------------------------------------|
| NOTEBOOK PARC 01/12     | 1/12    | Lançar como nova compra parcelada     |
| NOTEBOOK PARC 02/12     | 2/12    | Pular (FIN gerou automaticamente)     |
| TV PARC 05/10           | 5/10    | Pular (parcela 1 foi em fatura ant.)  |
```

### Passo 8 — Idempotência

**Antes de lançar**, busca no FIN as transações já existentes da fatura desse cartão pro período via `fin_fatura_transacoes(card_name, reference_month)`.

**Algoritmo de matching (CSV/extrato vs FIN), em 3 passadas:**

1. **Match exato:** `(tx_date, abs(amount_cents), tipo)` — onde tipo = `'refund'` se valor < 0 senão `'charge'`. Pra cada match, marca ambos lados como conciliados.
2. **Match com tolerância de data ±5 dias:** mesmo valor + tipo, data até 5 dias de diferença. Cobre o gap entre data da compra (CSV) e data de lançamento na fatura (FIN), e o caso do FIN-tx-lançada-no-dia-da-compra vs banco-cobrando-D+1.
3. **Match de parcela histórica (sem restrição de data):** mesmo valor + tipo. Cobre o caso de parcela X de compra antiga, onde o CSV mostra a data original de meses atrás e o FIN tem a data da fatura corrente.

Depois das 3 passadas:
- **CSV sem match** = pra lançar (novo)
- **FIN sem match** = sobra (revisar manualmente; geralmente é refund/parcela já tratada, ou erro)

Se restar sobra do FIN não-óbvia, **mostra pra pessoa** antes de seguir, não ignora.

Pra OFX use FITID quando disponível (mais confiável que data+valor).

**Sanity check obrigatório no fim do passo 12:** depois de lançar, chama `fin_fatura_cartao` de novo e confere se `total_cents` bate com o valor que a pessoa informou no início (ou que tava no cabeçalho da fatura). Se NÃO bater, mostra a diferença e pergunta antes de declarar sucesso.

### Passo 9 — Categorização automática

Lê `Estabelecimentos.md`. Aplica regras conhecidas. Marca como "a categorizar" o que não tiver regra.

### Passo 10 — Tabela de revisão

Mostra UMA tabela com TUDO:

```
=== FATURA: [cartão] — Período real DD/MM a DD/MM/YYYY — Vencimento DD/MM ===

=== JÁ EXISTEM NO FIN (vão ser ignoradas) ===
| Data       | Valor    | Descrição          | Categoria     |
... [N linhas]

=== PARCELAS JÁ GERADAS PELO FIN (vão ser ignoradas) ===
| Data       | Valor   | Descrição               | Parcela | Compra original    |
... [P linhas]

=== ESTORNOS (com original encontrada) ===
| Data       | Valor    | Descrição        | Despesa original revertida   |
... [E linhas]

=== CATEGORIZADAS AUTOMATICAMENTE ===
| Data       | Valor    | Descrição          | Categoria sugerida   | Regra |
... [M linhas]

=== A CATEGORIZAR (preciso de tu) ===
| #  | Data       | Valor    | Descrição          | Categoria? |
... [K linhas]

=== SUSPEITOS DE DUPLICATA ===
... [J linhas]

=== RESUMO ===
- Cartão: [nome real]
- Período: DD/MM a DD/MM/YYYY
- Vencimento: DD/MM/YYYY
- Total de linhas na fatura: N
- Já existiam: M
- Parcelas FIN-gerenciadas: P (puladas)
- Estornos: E
- Categorizadas: K
- A categorizar: Q
- Suspeitos: S
- Total a lançar: T
```

### Passo 11 — Coletar input

Mesmo do extrato:
1. Categorias dos "a categorizar"
2. Confirmação dos suspeitos (lançar/pular)
3. **Confirmação de estornos sem original** (lançar como despesa negativa? confirmar manualmente?)
4. Confirmação final

### Passo 12 — Lançar via `fin_fatura_transacoes`

**REGRA CRÍTICA do FIN:** fatura inteira é lançada via `fin_fatura_transacoes` em **uma chamada só** com a lista de transações, **NÃO** linha por linha via `fin_criar_despesa`.

Confira a description da tool `fin_fatura_transacoes` no MCP antes de chamar pra ver o formato exato esperado.

Estrutura provável (confira na description):
```
fin_fatura_transacoes(
  cartao_id: <id>,
  periodo_inicio: "YYYY-MM-DD",
  periodo_fim: "YYYY-MM-DD",
  vencimento: "YYYY-MM-DD",
  transacoes: [
    {
      data: "...",
      valor: ...,
      descricao: "...",
      categoria_id: ...,
      subcategoria_id: ...,
      parcelas: N (se aplicável),
      reversal_of_id: ... (se for estorno)
    },
    ...
  ]
)
```

Se a chamada falhar, anota e mostra erro claro.

### Passo 13 — Atualizar memória

#### `Estabelecimentos.md`
Estabelecimentos novos categorizados pela pessoa → linhas novas.

#### `Contas e Cartões.md`
Atualização de "fechamento observado" se variou (já feito no Passo 3).

#### `Status Conciliação.md`
```
| [cartão] | Fatura DD/MM-DD/MM/YYYY | Conciliado | YYYY-MM-DD | [FITIDs ou hashes] |
```

#### `Preferências.md`
Decisões não-óbvias se houver.

### Passo 14 — Oferecer pagar a fatura

**Depois de lançar com sucesso**, pergunta:

> Fatura lançada. Vencimento: DD/MM/YYYY. Total: R$ X. Quer que eu marque como paga agora? (vou usar `fin_pagar_fatura` — você precisa me dizer de qual conta saiu o pagamento)

Se sim:
- Pergunta a conta de débito
- Chama `fin_pagar_fatura` com cartão + conta + data
- Atualiza `Status Conciliação.md` se aplicável

Se não:
- Apenas avisa: "Beleza, fatura lançada mas não paga. Quando quiser pagar, me diz."

### Passo 15 — Resumo final

```
✓ Fatura processada.
- Cartão: [nome real]
- Período real: DD/MM a DD/MM/YYYY (variou ±N dias do ciclo cadastrado, atualizei)
- Vencimento: DD/MM/YYYY
- Total: R$ X
- Lançadas: T transações
- Estornos tratados: E (com reversal_of_id)
- Parcelas auto-geradas pelo FIN puladas: P
- Aprendi K estabelecimentos novos.
- Pago: ✓ via [conta] / SIM (ou: NÃO, lembro depois)
```

## Casos especiais

### Cobrança recorrente de assinatura (Spotify, Netflix, iCloud, etc.)

Aparece toda fatura, mesmo valor, mesma descrição. Aprende como regra **muito rápido** (1-2 ocorrências já é suficiente porque o padrão é óbvio).

### Anuidade do cartão

"ANUIDADE" / "ANUID DIFERENC" → categoriza em "Taxas, Juros & Impostos > Anuidade/Taxas cartão". Aprende como regra global pro cartão.

### IOF / Compra internacional

"IOF" / "REPASSE IOF" → "Taxas, Juros & Impostos > IOF". Compras em USD/EUR aparecem com símbolo da moeda — registra valor em BRL conforme a fatura mostra.

### Saque com cartão de crédito

"SAQUE CARTAO" / "SAQUE FATURA". Isso é um **adiantamento**, gera juros. Categoriza em "Taxas, Juros & Impostos > Juros saque cartão". Avisa a pessoa: "Detectei saque no cartão, isso gera juros. Tu sabia?"

### Pagamento parcial de fatura anterior aparecendo como crédito

Aliases conhecidos (cada banco usa um label diferente — match case-insensitive na descrição):
- "PAGAMENTO RECEBIDO" / "PAGAMENTO COM SALDO" / "PGTO ANTERIOR" / "PAG FATURA"
- "PAYMENT" / "CRED SALDO" / "SALDO A FAVOR"
- "DEB AUT FATURA" (débito automático da fatura)

Isso é o pagamento da fatura passada, NÃO é um estorno nem uma transação nova. **Pula**, não lança nada. Esse pagamento já foi registrado quando a fatura anterior foi paga via `fin_pagar_fatura`. Marca como descartado com motivo "linha de pagamento".

### Merchants ambíguos por design (Apple/Google Play/PayPal/MercadoPago/etc)

Algumas plataformas funcionam como **guarda-chuva**: a mesma string aparece pra dezenas de assinaturas/cobranças diferentes do mesmo usuário. Exemplos:
- `APPLE.COM/BILL` / `APPLECOMBILL` → pode ser Splitwise, YouTube Premium, iCloud, HBO Max, App Store, qualquer assinatura via Apple ID.
- `GOOGLE *<merchant>` / `GOOGLE PLAY` → assinaturas Android, jogos, YouTube.
- `PAYPAL *<merchant>` → varia por sub-string depois do asterisco.
- `MP *<merchant>` / `MERCADOPAGO` → cobrança via MercadoPago, varia pelo sub-merchant.

**NÃO categorizar automático nesses merchants**, mesmo com 3+ ocorrências. Sempre perguntar pra pessoa, ou inferir pelo valor (ex: "Apple R$ 99,90 — Splitwise? YouTube Premium? Outra?").

Em `Estabelecimentos.md` esses devem ficar marcados como **"ambíguo permanente"** (não "aguardando 3ª ocorrência"). Se o template de Estabelecimentos.md ainda não tem esse marcador, criar uma seção "Ambíguos permanentes (sempre perguntar)" separada da seção de "candidatos aguardando regra".

### Refund pendente / esperado

Cenário: a pessoa fez uma compra, pediu reembolso pro merchant, mas o crédito ainda não veio. A despesa precisa ser lançada **normalmente** na fatura corrente (porque vai cobrar normal), mas vale marcar pra acompanhamento.

Convenção:
- Lança a despesa com `tag: "refund-pendente"` (ou `refund-esperado-<valor>` se quiser registrar o valor esperado).
- No `Status Conciliação.md`, mantém uma seção **"Refunds pendentes"** com (data, descrição, valor, transaction_id, fatura onde foi cobrada).
- Quando o estorno chegar (próxima fatura ou na mesma), use `fin_criar_estorno(original_transaction_id, amount_cents)` apontando pra essa tx. Remove a entrada do "Refunds pendentes" e atualiza pra "Reembolsado em [data]".

A skill deve **proativamente perguntar** "Tem algum refund pendente esperado?" só se a pessoa mencionar reembolso, devolução, "pedi pra eles devolverem", etc. Não fica perguntando em toda fatura.

### Compra com pontos (sem valor monetário)

Algumas faturas mostram "RESGATE PONTOS R$0,00" — pula, não tem valor.

### Fatura com mais de uma moeda

Compras internacionais aparecem com USD/EUR + BRL convertido. Lança o valor em BRL (que é o que vai ser cobrado). Anota a moeda original na descrição se for útil.

### Compra estornada parcialmente

"ESTORNO PARCIAL R$50 de R$200" — busca a original de R$200 (`fin_buscar_transacoes`) e chama `fin_criar_estorno` com `original_transaction_id` + `amount_cents: 5000`. A original NÃO é apagada.

### Fatura muito grande (100+ linhas)

Processa em batches de 50 linhas pra revisão (mesma lógica do extrato).

## Erros comuns que você deve evitar

1. **Lançar fatura via `fin_criar_despesa`** → SEMPRE usa `fin_fatura_transacoes`
2. **Lançar estorno como receita** → SEMPRE via `fin_criar_estorno` (que grava como despesa vinculada via `reversal_of_id` no banco)
3. **Re-lançar parcelas que o FIN já gerou** → checa se já existe antes
4. **Confundir mês de vencimento com mês das compras** → fatura que vence em março contém compras de fevereiro/início de março
5. **Usar dia de fechamento cadastrado em vez do real** → sempre detecta o real do conteúdo da fatura
6. **Não atualizar fechamento observado em Contas e Cartões.md** → toda variação vira aprendizado
7. **Confundir pagamento de fatura anterior com transação nova** → "PAGAMENTO RECEBIDO" / "PAGAMENTO COM SALDO" / "PAYMENT" pulam
8. **Lançar saque de cartão como despesa normal** → categoria especial de juros
9. **Esquecer de oferecer pagar a fatura no fim** → sempre pergunta
10. **Pular o aviso sobre `fin://docs/guia`** → fatura é onde mais tem regra de negócio do FIN, lê a doc
11. **Tratar arquivo histórico do banco como se fosse só a fatura corrente** → vários bancos exportam histórico cumulativo; sempre detecta range de datas e isola a fatura alvo
12. **Categorizar APPLE.COM/BILL, GOOGLE *, PAYPAL *, MP * automaticamente** → merchants guarda-chuva, sempre perguntar
13. **Ignorar divergência grande (>2 dias) entre vencimento/fechamento detectado e cadastrado** → pausa e pergunta, pode ser cartão errado ou cadastro velho
14. **Não validar total da fatura no FIN contra o total informado pela pessoa depois de lançar** → sanity check obrigatório, evita erro silencioso passar

## Tom

PT-BR informal, direto. Sem travessão (—). Mostra trabalho ("identificando cartão...", "detectando período real...", "buscando despesas originais dos estornos..."), tabelas claras, confirmações curtas.
