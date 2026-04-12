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
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

## Quando usar

- Pessoa quer processar **uma fatura de cartão de crédito inteira**
- Frases tipo "processa essa fatura", "tá aí o OFX da fatura do C6", "fatura do mês do Nubank"
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

1. **Pelo nome do cartão no conteúdo** (ex: "Nubank Mastercard", "C6 Black", "Sicredi Woop")
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

**Compara com o cartão em `Contas e Cartões.md`:**

```
Cartão: C6
Fechamento cadastrado: dia 12
Período REAL detectado: 13/02/2026 a 14/03/2026
→ Variação: fechamento real foi dia 14 (esperado dia 12). Diferença +2 dias.
```

Se variou, **anota em `Contas e Cartões.md`**:
- Coluna "fechamento observado" → atualiza pro dia mais recente
- Seção "Histórico de variação" → adiciona linha com data e motivo (se inferível: "fim de semana", "feriado", etc.)

**Avisa a pessoa:**
> Detectei: fatura do **C6**, período real **13/02/2026 a 14/03/2026** (fechamento observado dia 14, cadastrado é dia 12, diferença de 2 dias). Atualizei tua memória.

### Passo 4 — Identificar a data de vencimento

Procura na fatura:
- "Vencimento: DD/MM/YYYY"
- "Pagar até DD/MM/YYYY"

Anota pra usar depois no `fin_pagar_fatura`.

### Passo 5 — Parse das transações da fatura

Mesma lógica do `extrato` pros 5 formatos (OFX, CSV, CNAB 240, texto, PDF). Extrai pra cada transação:

- Data
- Descrição
- Valor (despesas geralmente positivas em fatura, mas o sinal depende do formato — atenção)
- Categoria/MCC se disponível (alguns formatos dão isso)
- **Parcelamento detectado** (ex: "PARC 03/12", "1/10")
- **Estorno detectado** (ex: "ESTORNO", valor negativo numa fatura, "DEVOLUÇÃO")

### Passo 6 — Tratamento de estornos

**REGRA CRÍTICA do FIN:** estorno em cartão NÃO é receita. É **uma despesa com `reversal_of_id` apontando pra transação original**.

Pra cada transação que parece estorno (descrição contém "ESTORNO" / "DEVOLUÇÃO" / "REEMBOLSO" OU valor negativo numa fatura):

1. **Achar a despesa original** no FIN:
   - `fin_buscar_transacoes` filtrando por mesmo cartão, valor próximo (positivo do mesmo módulo), descrição parecida, dentro de uma janela de tempo (tipicamente até 3 meses antes)
2. **Se achou exatamente uma:** vai usar como `reversal_of_id`
3. **Se achou várias possíveis:** mostra pra pessoa: "Achei [N] despesas que podem ser a original desse estorno. Qual é? [lista]"
4. **Se não achou nenhuma:** avisa: "Não achei a despesa original desse estorno (R$X em [estabelecimento]). Vou lançar como despesa negativa sem `reversal_of_id`. Tu pode editar depois se quiser."

**Nunca lança estorno como receita.**

### Passo 7 — Tratamento de parcelamento

**REGRA CRÍTICA do FIN:** parcelamento gera múltiplas transações automaticamente no FIN. Você não duplica.

Pra cada transação que parece parcelada:

1. **Detectar parcela**: descrição contém "PARC X/Y", "1/10", "(1/12)", etc.
2. **Identificar parcela 1 vs parcela N**:
   - **Parcela 1**: a primeira aparição. Esta sim deve ser lançada via `fin_fatura_transacoes` com o campo de parcelamento (N parcelas, valor total, etc.). O FIN cria as outras N-1 automaticamente nas faturas seguintes.
   - **Parcela N (N > 1)**: já foi gerada pelo FIN quando você lançou a parcela 1. **NÃO LANCE.** Pula.
3. **Como saber se é parcela 1 ou N?**
   - Se for "1/12" no nome → parcela 1 (lançar)
   - Se for "5/12" no nome → parcela 5 (não lançar, já existe)
   - Se a descrição não diz qual parcela é → busca no FIN se já tem essa parcela cadastrada (`fin_buscar_transacoes` por descrição+cartão+valor parcial). Se já tem, é continuação. Se não tem, é parcela 1 ou compra à vista.

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

**Antes de lançar**, busca no FIN as transações já existentes da fatura desse cartão pro período:

```
fin_fatura_cartao(cartao_id: <id>, periodo: <período real detectado>)
```

OU

```
fin_fatura_transacoes (consulta) — confira a description da tool
```

Calcula chave de idempotência pra cada (FITID se OFX, hash caso contrário). Compara e separa em **3 grupos** (mesmo do extrato):
- Já existe (pula)
- Vai lançar (novo)
- Suspeito (revisar)

### Passo 9 — Categorização automática

Lê `Estabelecimentos.md`. Aplica regras conhecidas. Marca como "a categorizar" o que não tiver regra.

### Passo 10 — Tabela de revisão

Mostra UMA tabela com TUDO:

```
=== FATURA: C6 — Período real 13/02 a 14/03/2026 — Vencimento 20/03 ===

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
- Cartão: C6
- Período: 13/02 a 14/03/2026
- Vencimento: 20/03/2026
- Total de linhas na fatura: 87
- Já existiam: 12
- Parcelas FIN-gerenciadas: 8 (puladas)
- Estornos: 2
- Categorizadas: 47
- A categorizar: 18
- Suspeitos: 0
- Total a lançar: 65
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
  periodo_inicio: "2026-02-13",
  periodo_fim: "2026-03-14",
  vencimento: "2026-03-20",
  transacoes: [
    {
      data: "...",
      valor: ...,
      descricao: "...",
      categoria_id: ...,
      subcategoria_id: ...,
      parcelas: 12 (se aplicável),
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
| C6 | Fatura 13/02-14/03/2026 | Conciliado | 2026-04-12 | [FITIDs ou hashes] |
```

#### `Preferências.md`
Decisões não-óbvias se houver.

### Passo 14 — Oferecer pagar a fatura

**Depois de lançar com sucesso**, pergunta:

> Fatura lançada. Vencimento: 20/03/2026. Total: R$ 4.521,32. Quer que eu marque como paga agora? (vou usar `fin_pagar_fatura` — você precisa me dizer de qual conta saiu o pagamento)

Se sim:
- Pergunta a conta de débito
- Chama `fin_pagar_fatura` com cartão + conta + data
- Atualiza `Status Conciliação.md` se aplicável

Se não:
- Apenas avisa: "Beleza, fatura lançada mas não paga. Quando quiser pagar, me diz."

### Passo 15 — Resumo final

```
✓ Fatura processada.
- Cartão: C6
- Período real: 13/02 a 14/03/2026 (variou +2 dias do ciclo cadastrado, atualizei)
- Vencimento: 20/03/2026
- Total: R$ 4.521,32
- Lançadas: 65 transações
- Estornos tratados: 2 (com reversal_of_id)
- Parcelas auto-geradas pelo FIN puladas: 8
- Aprendi 14 estabelecimentos novos.
- Pago: ✓ via Conta C6 / SIM (ou: NÃO, lembro depois)
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

"PAGAMENTO RECEBIDO" / "PGTO ANTERIOR" — isso é o pagamento da fatura passada, NÃO é um estorno nem uma transação. **Pula**, não lança nada. Esse pagamento já foi registrado quando a fatura anterior foi paga via `fin_pagar_fatura`.

### Compra com pontos (sem valor monetário)

Algumas faturas mostram "RESGATE PONTOS R$0,00" — pula, não tem valor.

### Fatura com mais de uma moeda

Compras internacionais aparecem com USD/EUR + BRL convertido. Lança o valor em BRL (que é o que vai ser cobrado). Anota a moeda original na descrição se for útil.

### Compra estornada parcialmente

"ESTORNO PARCIAL R$50 de R$200" — lança o estorno de R$50 com `reversal_of_id` apontando pra original de R$200. A original NÃO é apagada.

### Fatura muito grande (100+ linhas)

Processa em batches de 50 linhas pra revisão (mesma lógica do extrato).

## Erros comuns que você deve evitar

1. **Lançar fatura via `fin_criar_despesa`** → SEMPRE usa `fin_fatura_transacoes`
2. **Lançar estorno como receita** → SEMPRE despesa com `reversal_of_id`
3. **Re-lançar parcelas que o FIN já gerou** → checa se já existe antes
4. **Confundir mês de vencimento com mês das compras** → fatura que vence em março contém compras de fevereiro/início de março
5. **Usar dia de fechamento cadastrado em vez do real** → sempre detecta o real do conteúdo da fatura
6. **Não atualizar fechamento observado em Contas e Cartões.md** → toda variação vira aprendizado
7. **Confundir pagamento de fatura anterior com transação nova** → "PAGAMENTO RECEBIDO" pula
8. **Lançar saque de cartão como despesa normal** → categoria especial de juros
9. **Esquecer de oferecer pagar a fatura no fim** → sempre pergunta
10. **Pular o aviso sobre `fin://docs/guia`** → fatura é onde mais tem regra de negócio do FIN, lê a doc

## Tom

PT-BR informal, direto. Sem travessão (—). Mostra trabalho ("identificando cartão...", "detectando período real...", "buscando despesas originais dos estornos..."), tabelas claras, confirmações curtas.
