---
name: conciliar
description: >
  Conciliação de uma conta ou cartão em um período: compara o que tá lançado
  no FIN com o que tá num extrato/fatura externo, mostra o que bate, o que
  só está no FIN, o que só está no extrato, suspeitos de duplicata e
  divergências de valor. Pessoa decide o que fazer com cada divergência
  (lançar o que falta, deletar o que sobra, marcar como conferido).
  Atualiza Status Conciliação.md no fim. Use quando a pessoa pedir
  "concilia conta X de mês Y" ou "tá batendo o saldo?".
argument-hint: "[conta/cartão] [período]"
allowed-tools: Read Write Edit Glob Grep Bash
---

## Quando usar

- Pessoa pediu "concilia conta X de março", "tá batendo o saldo?", "confere se não tá faltando nada"
- Depois de processar um extrato/fatura grande, pra dar uma segunda passada de conferência
- Pra fechar oficialmente um período de uma conta/cartão (marcar como "conferido até dia X")

## Quando NÃO usar

- Lançar transações novas (use `/financeiro:lancar`, `/financeiro:extrato`, `/financeiro:fatura`)
- Conciliação automática DURANTE o processamento de extrato/fatura — isso já acontece nas skills `extrato` e `fatura` via idempotência. Conciliar é um passo SEPARADO de revisão.
- MCP do FIN não tá instalado → `/financeiro:instalar-fin-mcp`

## Pré-requisitos

1. MCP do FIN responde
2. `~/.fin-plugin/config.json` existe → leia `financeiro_path`
3. Os 4 arquivos em `Financeiro/` existem
4. Leu `fin://docs/guia` nessa sessão

## Modos de uso

### Modo 1 — Conciliação contra arquivo externo (extrato/fatura)

A pessoa tem um extrato ou fatura e quer comparar com o que tá no FIN. Esse é o modo mais completo.

### Modo 2 — Conciliação só no FIN (sem arquivo externo)

A pessoa quer só revisar o que tá no FIN num período, sem ter um extrato pra comparar. Útil pra "tá tudo certo no Nubank esse mês?". Mostra todas as transações do período pra revisão visual.

### Modo 3 — Conferência de saldo

A pessoa fala "tá batendo o saldo da conta X?". Você compara:
- Saldo final calculado pelo FIN (`fin_saldos` + transações do período)
- Saldo final que a pessoa diz que tá no app do banco

Se bater, ✓. Se não bater, oferece rodar conciliação completa pra achar a diferença.

## Fluxo principal — Modo 1 (com arquivo externo)

### Passo 1 — Identificar conta/cartão e período

Pega `$ARGUMENTS` ou pergunta:
> Qual conta ou cartão? E qual período?

Detecta:
- **Conta:** match em `Contas e Cartões.md`
- **Período:** YYYY-MM-DD a YYYY-MM-DD. Aceita "março", "mês passado", "última fatura", etc., e converte.

### Passo 2 — Receber o arquivo externo

Pessoa anexa/cola o extrato ou fatura no formato disponível (OFX, CSV, CNAB, texto, PDF — mesma ordem de preferência das outras skills).

Se for fatura de cartão, faz o parse de período REAL do conteúdo (pode divergir do período pedido pela pessoa em ±2 dias por causa de ciclo).

### Passo 3 — Parse do externo

Mesma lógica das skills `extrato` e `fatura`. Resultado: lista normalizada de transações com chave de idempotência (FITID ou hash).

### Passo 4 — Buscar transações do FIN no período

```
fin_buscar_transacoes(
  conta_id: <id>,
  data_inicio: <início>,
  data_fim: <fim>
)
```

Resultado: lista de transações do FIN. Pra cada uma, calcula a chave de idempotência (FITID se for OFX-importada, hash caso contrário).

### Passo 5 — Comparar e classificar

Cruza as duas listas. Cada transação cai em **uma de 5 categorias**:

| Categoria | Definição |
|---|---|
| **Bate certinho** | Mesma chave em ambos, mesmo valor, mesma data, mesma descrição |
| **Bate com leve diferença** | Mesma chave OU descrição muito próxima + valor igual + data dentro de ±2 dias |
| **Só no FIN** | Existe no FIN mas não tem correspondente no externo |
| **Só no externo** | Existe no externo mas não tem correspondente no FIN |
| **Suspeito de duplicata no FIN** | Múltiplas transações no FIN matchando a mesma do externo |

### Passo 6 — Mostrar relatório de conciliação

Tabela única organizada por categoria:

```
=== CONCILIAÇÃO: Conta C6 — Período 13/02 a 14/03/2026 ===

✓ BATEM CERTINHO (47)
| Data       | Valor    | Descrição (FIN)        | Status |
|------------|----------|-------------------------|--------|
... (lista resumida ou só contagem se muitas)

⚠ BATEM COM LEVE DIFERENÇA (3)
| Data FIN   | Data Ext   | Valor    | Descrição (FIN)  | Descrição (Ext) | Diferença    |
... (mostra o que diverge)

❌ SÓ NO FIN — não tem no extrato (5)
| Data       | Valor    | Descrição              | Categoria         | Ação? |
|------------|----------|------------------------|-------------------|-------|
... (a pessoa decide: deletar do FIN, marcar como OK mesmo assim, ou investigar)

❌ SÓ NO EXTRATO — não tem no FIN (8)
| Data       | Valor    | Descrição              | Ação? |
|------------|----------|------------------------|-------|
... (a pessoa decide: lançar no FIN, ignorar, ou investigar)

🚨 SUSPEITO DE DUPLICATA NO FIN (1)
| Data       | Valor   | Descrição     | Transações no FIN matchando |
... (a pessoa decide qual deletar)

=== RESUMO ===
- 47 ✓ batem
- 3 ⚠ leve diferença
- 5 ❌ só no FIN
- 8 ❌ só no extrato
- 1 🚨 suspeito de duplicata
- Saldo do período: FIN=R$X / Externo=R$Y / Diferença=R$Z
```

### Passo 7 — Resolver divergências

Pra cada categoria de divergência, pergunta como agir.

#### Pra "Só no FIN" (5 transações)
Possíveis razões:
- Lançamento manual que não apareceu no extrato (ex: cartão de outra pessoa, transferência interna não exportada)
- Lançamento errado (categoria/valor/data divergiu)
- Duplicata real

> Pra cada uma das 5 transações que estão só no FIN:
> 1. **Manter** (foi lançada manual, ok não estar no extrato)
> 2. **Deletar do FIN** (foi erro)
> 3. **Investigar** (deixar marcada e voltar depois)

A pessoa responde número por número ou em bloco.

Se "deletar", confirma com `fin_deletar_transacao`.

#### Pra "Só no extrato" (8 transações)
Possíveis razões:
- Faltou lançar
- Foi conscientemente ignorado (ex: depósito que não conta)

> Pra cada uma das 8 transações que estão só no extrato:
> 1. **Lançar agora no FIN**
> 2. **Ignorar** (não vai ser lançada)
> 3. **Investigar**

Se "lançar agora", chama as tools de criação correspondentes (`fin_criar_despesa` / `fin_criar_receita` / `fin_criar_transferencia` / `fin_fatura_transacoes` se for cartão), aplicando categorização via `Estabelecimentos.md`.

#### Pra "Leve diferença" (3 transações)
Mostra o que diverge (data, valor, descrição) e pergunta:
> A linha do FIN tá certa, ou a do extrato tá certa?

Se a do extrato tá certa: oferece editar via `fin_editar_transacao`.
Se a do FIN tá certa: marca como OK.

#### Pra "Suspeito de duplicata"
> Tem 2 transações no FIN matchando a mesma do extrato. São duplicatas? Qual deletar?

A pessoa escolhe.

### Passo 8 — Atualizar `Status Conciliação.md`

Quando a pessoa terminar de resolver tudo (ou pelo menos confirmar o que fica como pendência), atualiza:

```markdown
## Conciliações
| Conta/Cartão | Período | Status | Data conciliação | FITIDs/hashes lançados |
|--------------|---------|--------|------------------|------------------------|
| C6 | 2026-02-13 a 2026-03-14 | Conciliado | 2026-04-12 | (FITIDs ou hashes) |
```

Se sobraram pendências não resolvidas:
```markdown
## Pendências
- C6 13/02-14/03/2026: 1 transação só no FIN (R$45 dia 25/02) marcada pra investigar
- C6 13/02-14/03/2026: 2 transações só no extrato (R$120 e R$30) marcadas pra investigar
```

### Passo 9 — Resumo final

```
✓ Conciliação concluída.
- Conta: C6
- Período: 13/02 a 14/03/2026
- 47 transações conferidas (batem)
- 3 leves diferenças resolvidas
- 5 só no FIN: 4 mantidas, 1 deletada
- 8 só no extrato: 7 lançadas, 1 ignorada
- 1 duplicata: removida
- Status: Conciliado
- Pendências: nenhuma (ou: lista)
```

## Fluxo — Modo 2 (sem arquivo externo)

Mais simples. Pessoa só quer revisar o que tá no FIN num período.

### Passo 1 — Identificar conta + período

### Passo 2 — Buscar transações do FIN

```
fin_buscar_transacoes(conta_id, data_inicio, data_fim)
```

### Passo 3 — Mostrar tabela

Tabela paginada (50 por vez se necessário). Pra cada linha:
- Data, valor, descrição, categoria, conta

### Passo 4 — Detectar anomalias automaticamente

- **Valores fora do padrão:** transações 3x maiores que a média do estabelecimento
- **Estabelecimentos novos** (não estão em `Estabelecimentos.md`)
- **Possíveis duplicatas** (mesma data + valor + descrição)
- **Categorização inconsistente** (mesmo estabelecimento com categorias diferentes)

Mostra essas como "atenção" sem forçar ação.

### Passo 5 — Pessoa decide

> Vi 3 coisas que merecem atenção:
> 1. R$1.200 em "Mercado" (média costuma ser R$300)
> 2. "Loja Z" apareceu pela 1ª vez
> 3. "Posto X" foi categorizado em "Manutenção" mas costuma ser em "Combustível"
>
> Quer revisar alguma? Ou tá tudo certo?

### Passo 6 — Marcar como conferido

> Tudo conferido? Vou marcar essa conta como conciliada até [data].

Atualiza `Status Conciliação.md`.

## Fluxo — Modo 3 (conferência de saldo)

### Passo 1 — Pegar saldo do FIN
```
fin_saldos
```

### Passo 2 — Pessoa diz o saldo do app

> Quanto tá teu saldo no app do banco agora?

### Passo 3 — Comparar

- Bate: ✓ "Tá batendo. R$X."
- Não bate: "Diferença de R$Y. Quer rodar conciliação completa pra achar a divergência?"

Se sim, vai pro Modo 1 ou 2 conforme a pessoa tenha extrato ou não.

## Casos especiais

### Conciliação de cartão de crédito (fatura)

Cartão é diferente de conta corrente. A "conciliação" de cartão é feita por **fatura** (período fechado), não por mês civil.

- Usa o **período real da fatura** detectado em `Contas e Cartões.md`
- Compara contra a fatura real (PDF, OFX, etc. exportada do app do cartão)
- Inclui pagamentos de fatura como item separado (não confunde com transações)

### Conciliação retroativa de meses sem lançamento

Pessoa diz "quero conciliar agosto, setembro, outubro". Você processa um período de cada vez. No fim, mostra um sumário consolidado.

### Conciliação encontra transferência cross-conta não-pareada

Tipo: extrato da conta A mostra "TRANSFERENCIA PARA CONTA B R$500", mas no FIN essa transferência não existe.

- Cria como `fin_criar_transferencia` da A pra B
- Avisa: "Vou criar essa transferência cross-conta. Quando tu rodar conciliação da conta B, ela já vai estar lá."

### Conciliação detecta diferença de saldo inicial

Tipo: o saldo final do extrato + lançamentos do mês não bate com o saldo inicial do extrato seguinte. Isso indica que o FIN tem saldo inicial errado pra essa conta.

- Avisa a pessoa: "O saldo inicial dessa conta no FIN parece estar errado por R$X. Quer que eu lance um ajuste de R$X?"
- Se sim, cria via `fin_criar_receita` ou `fin_criar_despesa` em categoria "Ajuste Financeiro"

## Erros comuns que você deve evitar

1. **Deletar transação do FIN sem confirmação** → SEMPRE pergunta
2. **Lançar transação que era duplicata** → SEMPRE checa antes
3. **Confundir período de fatura com mês civil** → cartão usa período real da fatura
4. **Esquecer de atualizar `Status Conciliação.md`** → toda conciliação concluída vira linha lá
5. **Sumir com pendências sem registrar** → sempre anota o que ficou pra resolver
6. **Marcar como conciliado sem a pessoa revisar** → ela decide quando virar "conferido"
7. **Não tratar transferência cross-conta** → verifica se a outra ponta existe
8. **Sobrescrever `Status Conciliação.md` em vez de adicionar linha** → é histórico, não substitui

## Tom

PT-BR informal, direto. Sem travessão (—). Tabelas claras, divide divergências por categoria, pergunta uma por vez quando precisa decidir, mostra resumo no fim.
