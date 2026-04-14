---
name: extrato
description: >
  Processa extrato bancário em lote e lança no FIN App. Aceita 5 formatos
  (preferência: OFX > CSV > CNAB 240 > texto colado > PDF). Garante
  idempotência (FITID pra OFX, hash pros outros). Categoriza automaticamente
  via Estabelecimentos.md, mostra tabela pra revisão, lança em batch e
  atualiza memória + Status Conciliação.md. Use quando a pessoa colar/anexar
  o extrato de uma conta corrente, poupança ou conta digital.
argument-hint: "[caminho-do-arquivo OU 'colado']"
allowed-tools: Read Write Edit Glob Grep Bash
---

## Quando usar

- Pessoa quer processar **um extrato bancário inteiro** (várias transações de uma vez)
- Frases tipo "processa esse extrato", "vou colar o extrato do mês", "tá aí o OFX da conta"
- Conciliação retroativa de meses sem lançamento

## Quando NÃO usar

- Lançamento de **uma única transação** → use `/financeiro:lancar`
- **Fatura de cartão de crédito** → use `/financeiro:fatura` (fluxo diferente, lança via `fin_fatura_transacoes`)
- MCP do FIN não tá instalado → `/financeiro:instalar-fin-mcp`
- Plugin nunca rodou nessa máquina → `/financeiro:onboarding` primeiro

## Pré-requisitos

1. MCP do FIN responde
2. `~/.fin-plugin/config.json` existe → leia `financeiro_path`
3. Os 4 arquivos em `Financeiro/` existem
4. Leu `fin://docs/guia` nessa sessão

## Fluxo principal

### Passo 1 — Receber o extrato

3 formas de receber:

**Forma A: Caminho de arquivo em `$ARGUMENTS`**
- Lê o arquivo do disco
- Detecta o formato pela extensão (`.ofx`, `.csv`, `.txt`, `.pdf`) e/ou pelo conteúdo (primeiras linhas)

**Forma B: Pessoa colou o conteúdo na conversa**
- Detecta o formato pelo conteúdo:
  - Começa com `OFXHEADER:` ou `<OFX>` → OFX
  - Tem cabeçalho CSV (vírgulas + nomes de coluna) → CSV
  - Linhas posicionais de 240 caracteres → CNAB 240
  - Texto não estruturado → texto colado

**Forma C: Pessoa anexou um PDF**
- Tenta parse via leitura do PDF
- Avisa que parse de PDF é frágil (ver Tratamento de PDF abaixo)

Pergunta a conta de destino (se não estiver óbvio do arquivo):

> Esse extrato é de qual conta? (lista de contas existentes em `Contas e Cartões.md`)

### Passo 2 — Parse por formato

#### OFX (preferido)

Parse XML padrão. Pra cada `<STMTTRN>`, extrai:
- `FITID` (ID único da transação no banco — **chave de idempotência**)
- `DTPOSTED` → data
- `TRNAMT` → valor (negativo = débito, positivo = crédito)
- `MEMO` → descrição
- `NAME` → nome do estabelecimento (se houver)
- `TRNTYPE` → tipo (DEBIT, CREDIT, XFER, etc.)

OFX é o melhor formato porque o `FITID` é um ID único do banco — você nunca duplica.

#### CSV

Parse com detecção de cabeçalho. Tenta identificar colunas:
- Data (formatos comuns: `dd/mm/yyyy`, `yyyy-mm-dd`, `dd-mm-yyyy`)
- Descrição/histórico
- Valor (atenção a separador decimal vírgula vs ponto)
- Tipo (D/C, débito/crédito, ou só sinal do valor)
- Saldo (opcional, ignora pra lançamento)

Se as colunas não tiverem nomes claros, **mostra as primeiras 3 linhas pra pessoa e pergunta qual coluna é qual**:

> Não consegui detectar as colunas automaticamente. Olha as primeiras linhas:
> ```
> [conteúdo]
> ```
> Qual coluna é a data? E o valor? E a descrição?

#### CNAB 240

Padrão Febraban posicional. Parse byte por byte conforme spec do banco. **Avisa que CNAB pode variar entre bancos** e pede pra confirmar se algo parecer estranho.

#### Texto colado

Best-effort. Tenta achar padrões:
- Linhas com data + valor + descrição
- Linhas em branco como separadores
- Cabeçalhos/rodapés ignorados

Mostra o que parseou pra confirmação antes de seguir.

#### PDF

**Avisa primeiro:**

> Detectei PDF. Parse de PDF é frágil porque cada banco tem layout diferente. Vou tentar, mas é importante tu revisar com atenção depois. Se tu tiver outro formato (OFX, CSV, CNAB), prefere mandar.

Tenta extração de texto do PDF (Bash com `pdftotext` ou similar se disponível). Aplica heurísticas pra achar linhas de transação. Mostra resultado pra pessoa.

### Passo 3 — Normalização

Depois do parse, normaliza tudo num formato interno:

```
[
  {
    chave_idempotencia: "FITID:abc123" ou "hash:xyz",
    data: "2026-03-15",
    valor: -45.00,
    descricao_original: "MERCADO XYZ LTDA",
    descricao_normalizada: "mercado xyz",
    tipo: "debito",
    conta: "Conta Caixa"
  },
  ...
]
```

**Geração de hash (formatos não-OFX):**
```
hash = sha1(data + "|" + valor + "|" + descricao_normalizada + "|" + conta)
```

### Passo 4 — Categorização automática via Estabelecimentos.md

Lê `Estabelecimentos.md`. Pra cada transação:

1. Tenta match exato pelo nome no extrato
2. Se não achar, tenta match parcial (substring) pela descrição normalizada
3. Se não achar, tenta match por palavra-chave conhecida
4. Se ainda não achar, marca como **"a categorizar"**

Resultado:
- **N transações categorizadas automaticamente** (com regra aplicada)
- **M transações a categorizar** (precisam input da pessoa)

### Passo 5 — Idempotência (anti-duplicata)

**ANTES de lançar qualquer coisa**, busca no FIN o que já existe no período:

```
fin_buscar_transacoes(
  conta_id: <id da conta>,
  data_inicio: <menor data do extrato>,
  data_fim: <maior data do extrato>
)
```

Pra cada transação retornada do FIN, calcula a chave de idempotência (mesmo método: FITID se tiver, hash caso contrário).

Compara com a lista do extrato e **separa em 3 grupos:**

| Grupo | Definição |
|---|---|
| **Já existe** | Chave bate com algo já no FIN |
| **Vai lançar** | Chave nova, vai ser criada |
| **Suspeito** | Chave não bate exato mas tem transação muito próxima (mesmo dia, valor igual, descrição parecida) |

**Suspeitos vão pra revisão manual.** Você não decide sozinho.

### Passo 6 — Mostrar tabela de revisão

Mostra UMA tabela com TUDO, organizada:

```
=== JÁ EXISTEM NO FIN (vão ser ignoradas) ===
| Data       | Valor    | Descrição          | Categoria         |
|------------|----------|--------------------|--------------------|
| 2026-03-01 | -45,00   | MERCADO XYZ        | Alimentação > Mer |
... [N linhas]

=== CATEGORIZADAS AUTOMATICAMENTE (vou lançar com a regra) ===
| Data       | Valor    | Descrição          | Categoria sugerida          | Regra aplicada |
|------------|----------|--------------------|------------------------------|----------------|
| 2026-03-15 | -50,00   | UBER *TRIP         | Transporte > App             | uber → ...     |
... [M linhas]

=== A CATEGORIZAR (preciso de tu) ===
| #  | Data       | Valor    | Descrição              | Categoria? |
|----|------------|----------|------------------------|------------|
| 1  | 2026-03-08 | -120,00  | LOJA NOVA SA           | ?          |
| 2  | 2026-03-12 | -89,00   | RESTAURANTE Y          | ?          |
... [K linhas]

=== SUSPEITOS DE DUPLICATA (revisar) ===
| #  | Data       | Valor    | Descrição        | Possível duplicata no FIN  |
|----|------------|----------|------------------|-----------------------------|
| 1  | 2026-03-10 | -75,00   | POSTO X          | "Posto X" R$75 em 2026-03-10|
... [J linhas]

=== RESUMO ===
- N já existem (vão ser puladas)
- M categorizadas (vou lançar direto se confirmar)
- K precisam de categoria
- J suspeitos pra ti decidir
- Total a lançar (max): M + K + J
```

### Passo 7 — Coletar input da pessoa

Pergunta na ordem:

#### Pra "A categorizar":
> Pra cada uma das K transações novas, me diz a categoria. Pode responder em bloco, tipo:
> 1. Lazer > Bar
> 2. Alimentação > Restaurante
> ...

Se ela disser "Loja Nova SA é tudo Pessoal > Roupas", aprende.

#### Pra "Suspeitos":
> Esses J parecem duplicatas. Pra cada um: lança mesmo (1) ou pula (0)?

#### Confirmação final:
> Vou lançar [total final] transações no FIN, conta [X]. Confirma?

### Passo 8 — Lançar em batch

**A partir de v2.3.4**, use `fin_criar_transacoes_batch` pra lançar até 100 rows de uma vez (mix de expense/income/transfer). Bate em 1 chamada o que antes eram N chamadas sequenciais.

```
{
  "transactions": [
    { "type": "expense", "amount_cents": 5000, "description": "Mercado XYZ", "account_name": "Caixa", "category_name": "Alimentação", "subcategory_name": "Mercado" },
    { "type": "income", "amount_cents": 500000, "description": "Salário", "account_name": "Caixa" },
    { "type": "transfer", "amount_cents": 20000, "description": "Saque 24h", "account_from": "Caixa", "account_to": "Dinheiro" },
    ...
  ]
}
```

Resposta: `{ total, created: [...], failed: [...] }`. **Partial success by design** — rows que falham NÃO revertem as anteriores. Mostra progresso:

> Lançando 47 transações em batch... ✓ 45 criadas, 2 falhas.

Pra cada row:
- `type: "expense"` → `fin_criar_despesa` semantics.
- `type: "income"` → `fin_criar_receita` semantics.
- `type: "transfer"` → `fin_criar_transferencia` semantics.
- Inclui na description a chave de idempotência (FITID ou hash) pra futura referência.

Se o batch tiver mais de 100 rows, quebra em múltiplas chamadas.

**Se alguma row falhar**, mostra a lista no fim: *"Lançadas 45 de 47. Falhas: [{ index: 12, error: 'categoria não encontrada' }, ...]"*. Pergunta como tratar.

### Passo 9 — Atualizar memória

#### `Estabelecimentos.md`
Pra cada estabelecimento NOVO categorizado pela pessoa, adiciona linha:
```
| LOJA NOVA SA | Loja Nova | Pessoal | Roupas | 2026-04-12 |
```

#### `Status Conciliação.md`
Adiciona linha pra esse processamento:
```
| Conta Caixa | 2026-03-01 a 2026-03-31 | Conciliado | 2026-04-12 | [lista de FITIDs ou hashes] |
```

#### `Preferências.md`
Se a pessoa deu uma regra explícita ("loja nova SA é sempre roupa"), adiciona em "Categorização padrão" ou "Decisões não-óbvias" conforme o caso.

### Passo 10 — Resumo final

```
✓ Extrato processado.
- Conta: Caixa
- Período: 2026-03-01 a 2026-03-31
- Total de linhas no arquivo: 64
- Já existiam no FIN: 17 (puladas)
- Lançadas agora: 47
- Falhas: 0
- Aprendi 8 estabelecimentos novos.
- Status Conciliação atualizado.
```

## Reconciliação de saldo pós-import (passo final obrigatório)

Depois de importar todas as transações do extrato em lote, **sempre** reconcilia o saldo da conta no FIN com o saldo real do app bancário. Isso é o que fecha o lote — se pular, o saldo fica à deriva e toda sessão futura vai ter que lidar com número errado.

**Fluxo simplificado (v2.3.4):**
1. Pergunta pra pessoa: *"Qual é o saldo atual do app do [banco] agora? (print da tela inicial ajuda)"*. Se o extrato tinha linha de "saldo final", usa como referência mas **confirma com a pessoa** porque o OFX pode ter data de corte diferente de "agora".
2. Chama `fin_saldos` pra pegar o saldo exibido atual. Se já bater, nada a fazer — avisa *"✓ Saldo já bate, R$X,XX"* e fim.
3. Se não bater, **primeiro investiga**: pergunta *"tá dando R$X a mais/menos — falta lançar alguma coisa desde o último movimento do extrato, ou tem duplicata?"*.
4. Se a pessoa confirmar que o saldo inicial do período estava errado no FIN (típico em primeira importação), chama `fin_ajustar_saldo_conta` passando o **saldo desejado** direto em `amount_cents`. A tool retorna `balance_cents_calculado` — confere se bateu. Se bateu, segue. Se não bateu, investiga (tem transação faltando/sobrando que não é resolvida por ajuste de `initial_balance` sozinho).

**O que mudou de antes:** não precisa mais calcular `initial_balance_novo` manualmente nem chamar `fin_listar_contas` pra pegar `initial_balance_atual`. A tool faz a conta e devolve o saldo exibido pós-ajuste. Se `balance_cents_calculado` != desejado, é sinal de problema de dados — investiga em vez de insistir.

**Atualizar `Status Conciliação.md`:** registra que o saldo foi reconciliado em DD/MM/AAAA e em qual valor, pra sessões futuras saberem o ponto de partida.

## Classificação automática de linhas (v2.3.4)

**A partir de v2.3.4**, em vez de fazer heurística client-side pra saque/pagamento de fatura/transferência cross-conta, use `fin_classificar_linha_extrato`. A tool recebe `{description, amount_cents, account_id, trntype?, fitid?}` e retorna `{type, confidence, reason, details}` onde type ∈ `expense|income|transfer|payment_invoice|atm_withdrawal`.

**Fluxo recomendado:** passa cada linha pelo classifier antes de decidir qual tool de lançamento usar. A resposta te diz o tipo e (quando possível) sugere a conta contraparte (conta Dinheiro pra ATM, cartão pra PAG FATURA, outra conta pra XFER).

```
// Exemplo
classify({
  description: "SAQUE 24H BANCO X",
  amount_cents: -20000,
  account_id: "uuid-c6",
  trntype: "ATM"
})
→ { type: "atm_withdrawal", confidence: "high",
    details: { suggested_cash_account_id: "uuid-dinheiro" } }
// → cria transferência C6 → Dinheiro via fin_criar_transferencia
```

O classifier tem blocklist de palavras genéricas (PIX, TED, DOC, etc) pra não dar falso positivo. Se a resposta vier com `confidence: "low"`, pergunta pra pessoa antes de lançar.

As seções abaixo ainda valem como referência semântica do que cada tipo significa, mas a **detecção** deve usar o classifier.

## Casos especiais

### Extrato com saldo inicial/final

Algumas linhas do extrato são "saldo anterior" e "saldo final". **Ignora como transação** (não vira `fin_criar_despesa`/`fin_criar_receita`), mas **guarda os valores** — o "saldo final" vai servir de referência inicial pro passo de **Reconciliação de saldo pós-import** acima.

### Extrato com tarifas e impostos

Tarifa bancária, IOF, anuidade são despesas normais. Categoriza em "Taxas, Juros & Impostos". Se a pessoa tiver subcategorias específicas (Tarifa bancária, IOF), aplica.

### Rendimento de poupança / juros

São receitas. Categoriza em "Rendimento" ou similar. Aprende como regra (todo banco X tem padrão de descrição "RENDIMENTO POUPANÇA", "JUROS S/SALDO", etc.) e da próxima vez aplica direto.

### Transferências entre contas próprias

Se o extrato mostra "TED" / "PIX" / "TRANSFERÊNCIA" pra outra conta da pessoa, é uma `fin_criar_transferencia`. Identifica a conta destino pelo `Contas e Cartões.md`. Se não conseguir identificar, pergunta.

**Atenção dupla contagem:** se a pessoa processar o extrato da conta A E da conta B no mesmo período, a transferência pode aparecer dos 2 lados. Lança UMA vez só, não duas. Use o hash pra detectar duplicata cross-conta.

### Extrato com débito automático de cartão de crédito

Linha tipo "PAGAMENTO FATURA C6". Isso NÃO é despesa de R$X. É um **pagamento de fatura**. Use `fin_pagar_fatura` em vez de `fin_criar_despesa`. Identifica o cartão no `Contas e Cartões.md` pelo nome.

### Extrato com saque ATM

Linha tipo "SAQUE 24H BANCO X". É uma **transferência** da conta pra "Dinheiro" (mesma regra de saque manual). Lança via `fin_criar_transferencia`.

Se a pessoa não tem conta "Dinheiro" no FIN, oferece criar uma vez (igual em `/financeiro:lancar`).

### Extrato muito grande (200+ linhas)

Se for muito grande, processa em batches de 50:
- Mostra os primeiros 50 pra revisão
- Pessoa confirma
- Lança esses 50
- Repete pra próximos 50
- Etc.

Não tenta colocar 300 linhas numa tabela só.

### Falha de parse

Se o parser falhar (formato corrompido, encoding estranho, layout não reconhecido):
1. Mostra o que conseguiu parsear (mesmo que parcial)
2. Mostra trechos do arquivo que falharam
3. Pede pra pessoa: "consegue tentar exportar em outro formato?" (sugere OFX como ideal)
4. Não lança nada se a confiança no parse for baixa

## Erros comuns que você deve evitar

1. **Lançar duplicata** → sempre checa idempotência ANTES de lançar
2. **Lançar pagamento de fatura como despesa** → é `fin_pagar_fatura`, não despesa
3. **Lançar saque como despesa** → é `fin_criar_transferencia` banco → Dinheiro
4. **Lançar transferência entre contas próprias 2x** → quando processar 2 extratos do mesmo período
5. **Confiar cegamente em parse de PDF** → sempre revisar, sempre avisar a pessoa
6. **Aprender estabelecimento sem confirmação** → precisa a pessoa categorizar antes de virar regra
7. **Aplicar regra silenciosamente em lote** → mostra na tabela qual regra foi aplicada em quais linhas
8. **Lançar tudo sem mostrar a tabela** → revisão é obrigatória
9. **Ignorar suspeitos sem perguntar** → pessoa decide se é duplicata ou não
10. **Esquecer de atualizar `Status Conciliação.md`** → todo lote processado vira linha lá

## Tom

PT-BR informal, direto. Sem travessão (—). Mostra trabalho ("parseando...", "buscando duplicatas no FIN..."), tabelas claras, confirmações curtas.
