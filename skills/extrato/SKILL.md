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

**Lança em sequência** (uma por vez se a tool não suportar batch). Mostra progresso:

> Lançando... [12/47 ✓]

Pra cada lançamento:
- Tipo: despesa (`fin_criar_despesa`) ou receita (`fin_criar_receita`)
- Transferências detectadas no extrato (TRNTYPE=XFER no OFX) → `fin_criar_transferencia`
- Inclui no campo `external_id` ou descrição a chave de idempotência (FITID ou hash) pra futura referência

Se algum lançamento falhar, anota e segue. No fim mostra: "Lançadas X de Y. Falhas: [lista]".

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

**Fluxo:**
1. Pergunta pra pessoa: "Qual é o saldo atual do app do [banco] agora? (print da tela inicial ajuda)". Se o extrato tinha linha de "saldo final", usa o valor dele como referência inicial mas **confirma com a pessoa** porque extrato OFX/CSV pode ter data de corte diferente de "agora".
2. Chama `fin_saldos` e `fin_listar_contas` pra pegar `saldo_exibido_atual` e `initial_balance_atual`.
3. Calcula a diferença: `diferenca = saldo_desejado − saldo_exibido_atual`.
4. Se `diferenca == 0` → saldo já bate, nada a fazer. Avisa "✓ Saldo bateu certinho, R$ X,XX".
5. Se `diferenca != 0`:
   - **Primeiro investiga**: a diferença pode indicar transação faltando ou sobrando. Pergunta: "tá dando R$ X a mais/menos — falta lançar alguma coisa desde o último movimento do extrato, ou tem alguma transação duplicada?".
   - Se a pessoa confirmar que o saldo inicial do período do extrato estava errado no FIN (caso típico: primeira importação de uma conta), aplica o ajuste retroativo no `initial_balance`:
     ```
     initial_balance_novo = initial_balance_atual + diferenca
     ```
     Chama `fin_ajustar_saldo_conta` com `amount_cents: initial_balance_novo`.
   - Re-chama `fin_saldos` pra confirmar que o exibido agora bate.

**Armadilha crítica:** `fin_ajustar_saldo_conta` **sobrescreve o `initial_balance`**, não o saldo exibido. Se passar o saldo desejado direto, o FIN recalcula por cima e o valor fica errado. **Sempre use a fórmula acima.** Ver `skills/lancar/SKILL.md` → Caso B/C pra fluxo detalhado.

**Atualizar `Status Conciliação.md`:** registra que o saldo foi reconciliado em DD/MM/AAAA e em qual valor, pra sessões futuras saberem o ponto de partida.

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
