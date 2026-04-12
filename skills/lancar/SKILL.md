---
name: lancar
description: >
  Lançamento avulso de transação no FIN App. Entende instruções em linguagem
  natural ("lança 45 no mercado, débito Nubank", "20 conto no pão, dinheiro",
  "saquei 200 no Itaú", "recebi 4mil do cliente X", "vendi $100 e veio R$540",
  "tô com $487 na carteira"), aplica regras aprendidas de Estabelecimentos.md,
  trata dinheiro vivo, saque, câmbio USD/BRL e ajuste de saldo corretamente,
  sempre confirma em uma linha antes de criar. Aprende e atualiza memória.
argument-hint: "[descrição da transação em linguagem natural]"
allowed-tools: Read Write Edit Glob Grep
---

## Quando usar

- Pessoa quer lançar **uma única transação** rapidamente
- Frases tipo "lança X reais em Y", "gastei X no Z", "recebi X do W", "transferi X de A pra B"
- Saque de dinheiro vivo

## Quando NÃO usar

- **Várias transações de uma vez** (extrato, fatura, lista) → use `/financeiro:extrato` ou `/financeiro:fatura`
- Pessoa quer lançar fatura inteira do cartão → `/financeiro:fatura`
- MCP do FIN não tá instalado → `/financeiro:instalar-fin-mcp`
- Plugin nunca rodou nessa máquina → `/financeiro:onboarding` primeiro

## Pré-requisitos (verificar antes de cada chamada)

1. MCP do FIN responde (testa com `fin_listar_contas` se ainda não fez na sessão)
2. `~/.fin-plugin/config.json` existe → leia `financeiro_path`
3. Os 4 arquivos em `Financeiro/` existem (Preferências, Contas e Cartões, Estabelecimentos, Status Conciliação). Se não existem, dispara `/financeiro:onboarding`.
4. Leu `fin://docs/guia` nessa sessão (se não, leia agora)

## Fluxo principal

### Passo 1 — Parse da instrução

Pega `$ARGUMENTS` (ou a fala livre da pessoa) e extrai:

- **Tipo:** despesa, receita, transferência (+ caso especial: saque)
- **Valor:** sempre em reais (BRL)
- **Estabelecimento/origem/destino:** nome do lugar/pessoa/empresa
- **Forma de pagamento / conta:** débito, crédito, dinheiro, PIX, conta específica
- **Data:** hoje (default), ou data explícita se mencionada ("ontem", "dia 5", "10 de março")
- **Observação:** qualquer detalhe adicional

**Exemplos de parsing:**

| Frase | Tipo | Valor | Estabelecimento | Conta |
|---|---|---|---|---|
| "lança 45 no mercado, débito Nubank" | despesa | R$45 | mercado | Nubank (débito) |
| "20 conto no pão, dinheiro" | despesa | R$20 | padaria/pão | Dinheiro |
| "120 no posto, crédito C6" | despesa | R$120 | posto | C6 (crédito) |
| "recebi 4 mil do cliente X" | receita | R$4000 | cliente X | (perguntar conta destino) |
| "transferi 500 do Nubank pro C6" | transferência | R$500 | — | Nubank → C6 |
| "saquei 200 no Itaú" | **transferência** | R$200 | — | Itaú → Dinheiro |
| "PIX 50 pra mãe" | despesa OU transferência | R$50 | mãe | (perguntar conta origem) |
| "vendi $100 e veio R$540" | **câmbio** (vender USD) | $100 / R$540 | — | Conta USD → Conta BRL |
| "comprei $50 por R$280" | **câmbio** (comprar USD) | R$280 / $50 | — | Conta BRL → Conta USD |
| "tô com $487 na carteira" | **ajuste de saldo** | $487 (absoluto) | — | Conta USD |
| "gastei $20 no AliExpress" | **ajuste de saldo** (subtrai) | -$20 | AliExpress | Conta USD |
| "ajusta o Caixa pra 1234,56" | **ajuste de saldo** | R$1234,56 (absoluto) | — | Conta BRL |

**Notação numérica BR:**
- "20 conto" / "20 mango" / "20 pila" = R$20
- "4 mil" / "4k" = R$4.000
- "500 reais" / "500" = R$500
- Vírgula é decimal: "45,50" = R$45,50

**Notação USD:**
- "$100" / "100 dólares" / "100 dolar" / "100 USD" = US$100
- "$1.5k" / "1500 dólares" = US$1.500
- Quando a pessoa só fala "100" sem moeda, **assume BRL** (default)
- Quando a pessoa usa `$` no início, **assume USD**
- Em caso de dúvida, pergunta ("100 reais ou 100 dólares?")

### Passo 2 — Tratamento especial: dinheiro vivo

Se a pessoa mencionar **"dinheiro", "vivo", "espécie", "carteira", "papel", "cash"**, o plugin mapeia pra conta **"Dinheiro"** (ou nome equivalente em `Contas e Cartões.md`).

**Se a conta "Dinheiro" não existir ainda no FIN:**

> Vi que tu pagou em dinheiro mas tu não tem uma conta "Dinheiro" cadastrada no FIN. Quer que eu crie agora? (Dinheiro vivo é uma conta como qualquer outra: tem saldo, recebe lançamentos, e quando tu sacar do banco eu já registro como transferência banco → Dinheiro automaticamente.)

Se ela aceitar, cria via `fin_criar_conta` (tipo "Dinheiro" ou equivalente, saldo inicial 0 ou perguntado).

### Passo 3 — Tratamento especial: saque

**Saque NÃO é despesa.** Se a pessoa disser "saquei X no banco Y", isso é uma `fin_criar_transferencia` da conta bancária pra conta "Dinheiro".

Confirmação:

> Saque R$200 / Itaú → Dinheiro (transferência). Confirma?

Se ela disser "não, é despesa mesmo" (caso raro, tipo taxa de saque), aceita e lança como despesa.

### Passo 4 — Tratamento especial: PIX pra pessoa

PIX pra alguém pode ser **despesa** (se for pagamento) ou **transferência** (se for entre contas próprias) — depende do contexto.

- "PIX 50 pra padaria" → despesa
- "PIX 100 pra mãe" → despesa (categoria "Família & Amigos > Mesada" ou similar, conforme `Estabelecimentos.md`)
- "PIX 500 do Nubank pro Itaú" → transferência

Se ambíguo, pergunta:
> 50 pra mãe é despesa (vai gastar) ou transferência (entre tuas contas)?

### Passo 4.5 — Tratamento especial: USD, câmbio e ajuste de saldo

**Pré-requisito:** se a pessoa tem alguma conta USD no FIN, leia a **seção 10 do `fin://docs/guia`** uma vez por sessão antes de operar qualquer coisa em USD. Sem isso, você comete erros de modelo.

#### Regras de negócio críticas (do FIN)

1. **`fin_criar_despesa` e `fin_criar_receita` são bloqueados pra contas USD** (BRL only). Se você tentar, vai falhar com mensagem clara apontando pra tool certa. **Nunca tente lançar despesa/receita categorizada em USD.**
2. **Câmbio é uma operação atômica** via `fin_cambio`. Cria 2 transações vinculadas (uma despesa na origem, uma receita no destino), ambas com categoria "Câmbio" e um `exchange_pair_id` compartilhado. Se uma falhar, a outra é desfeita.
3. **O FIN não usa cotação automática.** A pessoa informa as **duas quantias manualmente** (USD e BRL), porque a taxa real varia (banco, spread, operação manual). Você nunca calcula cotação.
4. **`fin_ajustar_saldo_conta` sobrescreve o `initial_balance` da conta — NÃO o saldo exibido.** Essa é a armadilha mais cara da API. O FIN calcula `saldo_exibido = initial_balance + soma(receitas) − soma(despesas) − soma(transfers_out) + soma(transfers_in)`. Se você passar "o saldo atual que a pessoa quer ver", o FIN vai recalcular por cima e o valor vai ficar errado. **Regra correta** (vale pra BRL e USD): pra fazer o saldo exibido chegar num valor desejado, use `initial_balance_novo = initial_balance_atual + (saldo_desejado − saldo_exibido_atual)`. Na prática: lê `fin_saldos` pra descobrir `saldo_exibido_atual`, calcula a diferença pro desejado, soma essa diferença no initial_balance via a tool. Ver **Caso B/C** abaixo pra exemplos passo-a-passo. **Exceção:** conta recém-criada sem nenhuma transação ainda — aí `initial_balance` = saldo exibido e você pode passar o saldo desejado direto.
5. **Câmbio só funciona entre 2 contas cash de moedas diferentes** (uma BRL, outra USD). Cartão de crédito não é suportado.
6. **Cartão de crédito em USD não é suportado** no v0.

#### Caso A — Câmbio (vender ou comprar dólar)

**Padrões de fala:**
- "Vendi $100 e veio R$540" → vender USD (USD → BRL)
- "Comprei $50 por R$280" → comprar USD (BRL → USD)
- "Cambiei R$1000 em $185" → comprar USD
- "Troquei $200 e veio R$1080" → vender USD

**Fluxo:**
1. Identifica direção (vender = USD→BRL, comprar = BRL→USD)
2. Identifica as duas contas (`from_account_name`, `to_account_name`):
   - **Vender:** from = conta USD da pessoa (Wise, conta nos EUA, carteira USD), to = conta BRL
   - **Comprar:** from = conta BRL, to = conta USD
   - Se a pessoa não disser explicitamente qual conta, lê `Contas e Cartões.md` e pergunta se houver mais de uma opção
3. Captura **as duas quantias** (em centavos da moeda de cada conta):
   - `amount_from_cents` = quanto sai da origem
   - `amount_to_cents` = quanto entra no destino
   - Se a pessoa só falou uma das duas (ex: "vendi $100" sem dizer quanto recebeu), **pergunta a outra**: "Vendeu $100 — quanto veio em reais?"
4. Confirma em **uma linha**:
   > Câmbio: vender US$100 → R$540 / Wise USD → Itaú / hoje. Confirma?
5. Chama `fin_cambio` com os campos:
   ```
   {
     "from_account_name": "Wise USD",
     "to_account_name": "Itaú",
     "amount_from_cents": 10000,
     "amount_to_cents": 54000,
     "description": "Venda de dólar"
   }
   ```
6. Sucesso: avisa que criou as 2 transações vinculadas em "Câmbio".

**Avisos importantes:**
- **Nunca invente cotação.** Se a pessoa só sabe uma das duas quantias, pergunta a outra. Não calcula.
- **Confere a categoria "Câmbio"** existe (criada automaticamente no primeiro uso pelo FIN, mas tu pode ver via `fin_listar_categorias`)
- **Não funciona com cartão de crédito** — se a pessoa tentar câmbio envolvendo cartão, avisa e oferece operar conta cash equivalente

#### Caso B — Ajustar saldo manualmente (USD ou BRL)

**ATENÇÃO (armadilha):** `fin_ajustar_saldo_conta` sobrescreve o `initial_balance`, não o saldo exibido. Se a conta tem transações importadas, **não** passe o "saldo atual desejado" direto — o FIN recalcula por cima e o valor fica errado. Use a fórmula do fluxo abaixo.

**Padrões de fala:**
- "Tô com $487 na carteira agora" → ajuste pra valor exibido final = $487
- "Agora tenho $1200 na Wise" → ajuste pra $1200
- "Achei mais $50, total tá em $537" → ajuste pra $537
- "Achei mais $50 na carteira" (sem dizer total) → precisa ler saldo exibido atual, somar 50, ajustar pro resultado
- "Ajusta o Caixa pra 1234,56" → ajuste pra R$1234,56
- "O saldo do Itaú tá errado, tá em R$2000 e devia ser R$2150" → ajuste pra R$2150

**Fluxo (vale pra BRL e USD):**
1. Identifica a conta (lê `Contas e Cartões.md`).
2. Lê `fin_saldos` pra descobrir `saldo_exibido_atual` e `fin_listar_contas` pra descobrir `initial_balance_atual` (campo `initial_balance_cents`).
3. Calcula:
   ```
   diferenca = saldo_desejado − saldo_exibido_atual
   initial_balance_novo = initial_balance_atual + diferenca
   ```
4. Confirma em uma linha mostrando **antes e depois do saldo exibido**:
   > Ajustar saldo: Wise USD de $437 → $487 (somou $50). Confirma?
5. Chama `fin_ajustar_saldo_conta` passando **`initial_balance_novo`** como `amount_cents`:
   ```
   {
     "account_name": "Wise USD",
     "amount_cents": <initial_balance_novo_em_centavos>
   }
   ```
6. **Valida**: re-chama `fin_saldos` e confere que o saldo exibido agora bate com o desejado. Se não bater, tem transação faltando/sobrando — investiga antes de seguir.
7. Sucesso: avisa o novo saldo exibido.

**Exemplo real (Lou, C6 BRL, 2026-04-12):**
- Saldo exibido atual: −R$ 4.641,67 (−464167)
- Initial balance atual: R$ 3.673,79 (367379) — valor errado deixado por ajuste anterior
- Saldo desejado: R$ 3.673,79 (367379)
- diferenca = 367379 − (−464167) = 831546
- initial_balance_novo = 367379 + 831546 = 1198925 (R$ 11.989,25)
- Passa `amount_cents: 1198925` → saldo exibido vira R$ 3.673,79 ✅

**Caso especial dentro do ajuste de saldo: gasto em USD (sem categorização)**

> "Gastei $20 no AliExpress"

No v0 do FIN, **não dá pra lançar despesa categorizada em USD**. O jeito recomendado:
1. Segue o fluxo acima com `saldo_desejado = saldo_exibido_atual − 20 USD`.
2. **Avisa a pessoa explicitamente que a categorização ficou de fora** (limitação do v0):
   > Ajustei: Wise USD de $487 → $467 (gastou $20 no AliExpress). Não dá pra categorizar gasto em USD ainda (limitação do v0), mas o saldo tá certo. Quando tu fizer câmbio depois, a saída de dólar fica registrada na categoria Câmbio. Beleza?

#### Caso C — Ajustar saldo BRL pra correção

Mesma lógica do Caso B (fluxo completo lá: ler saldo exibido + initial_balance, calcular diferença, somar no initial_balance). Útil pra corrigir saldo após conferir extrato ou reconciliar depois de importar lote.

> "Ajusta o saldo do Caixa pra R$1234,56"

1. Lê `fin_saldos` e `fin_listar_contas` pra pegar `saldo_exibido_atual` e `initial_balance_atual`.
2. Calcula `initial_balance_novo = initial_balance_atual + (saldo_desejado − saldo_exibido_atual)`.
3. Confirma em uma linha (mostrando saldo exibido antes/depois, não o initial_balance):
   > Ajustar saldo: Caixa de R$1100 → R$1234,56. Confirma?
4. Chama `fin_ajustar_saldo_conta` com `amount_cents = initial_balance_novo`.
5. Re-checa com `fin_saldos` que o exibido ficou certo.

**Atenção 1:** ajuste de saldo BRL **não cria transação**, então não aparece em relatórios mensais como movimentação. É um ajuste contábil. Se a pessoa quiser que apareça como receita/despesa categorizada, oriente a usar `fin_criar_receita`/`fin_criar_despesa` em vez disso.

**Atenção 2:** se o saldo exibido não bater com o desejado depois do ajuste, **não tenta de novo com outro valor**. Isso significa que tem transação faltando/sobrando no FIN. Investiga o extrato original antes.

#### Quando pedir leitura prévia de saldo

Você precisa ler o saldo atual ANTES do ajuste em 2 situações:
1. **Delta:** "achei mais $50", "tirei $30", "somou X" — você precisa do saldo pra calcular o absoluto
2. **Confirmação visual:** sempre que for útil mostrar antes/depois pra pessoa confirmar (basicamente sempre)

Use `fin_saldos` ou `fin_listar_contas` (a tool retorna os saldos junto com as contas).

### Passo 5 — Aplicar regras de Estabelecimentos.md

Lê `Estabelecimentos.md`. Pra cada estabelecimento mencionado:

**Se tem regra cadastrada:**
- Aplica direto a categoria/subcategoria
- **Mostra explicitamente:** "apliquei tua regra: mercado → Alimentação > Mercado"

**Se NÃO tem regra cadastrada:**
- Pergunta a categoria: "Mercado vai em qual categoria? Ex: Alimentação > Mercado"
- Aceita a resposta e prepara pra aprender (vai gravar no fim)

**Se tem regra mas a pessoa quer mudar dessa vez:**
- Aceita a mudança pra essa transação específica
- Pergunta: "Isso é uma exceção dessa vez ou tu quer atualizar a regra pra todas as próximas?" — não atualiza a regra sem confirmação explícita

### Passo 6 — Validar conta/cartão

Lê `Contas e Cartões.md`. Pra cada conta/cartão mencionado:

**Se existe:** segue.

**Se NÃO existe:**
- Pergunta: "Não tenho [conta] cadastrado. Quer que eu crie agora?"
- Se sim, faz `fin_criar_conta` (com tipo, dados básicos), atualiza `Contas e Cartões.md`
- Se não, pede pra pessoa escolher uma conta existente da lista

### Passo 7 — Confirmação em uma linha

**Antes de qualquer mutação no FIN, sempre confirma em uma linha.**

Formato:

```
Despesa R$45 / Mercado / Alimentação > Mercado / Nubank débito / hoje. Confirma?
```

```
Receita R$4000 / Cliente X / Trabalho > Avulsos / Conta Caixa / hoje. Confirma?
```

```
Transferência R$500 / Nubank → C6 / hoje. Confirma?
```

```
Saque R$200 / Itaú → Dinheiro (transferência) / hoje. Confirma?
```

A pessoa responde sim/não/ajuste. Se ajuste, ajusta e re-confirma.

### Passo 8 — Executar no FIN

Conforme o tipo:

| Tipo | Tool |
|---|---|
| Despesa BRL | `fin_criar_despesa` |
| Receita BRL | `fin_criar_receita` |
| Transferência (incluindo saque) | `fin_criar_transferencia` |
| Câmbio (vender ou comprar dólar) | `fin_cambio` |
| Ajuste de saldo (BRL ou USD) | `fin_ajustar_saldo_conta` |
| Gasto em USD (sem categorização, v0) | `fin_ajustar_saldo_conta` (subtraindo do saldo) |

Confere se a tool executou OK. Se erro, mostra mensagem clara e tenta uma vez mais ou pergunta como proceder.

### Passo 9 — Atualizar memória `.md`

Depois do lançamento bem-sucedido:

**Se aprendeu um estabelecimento novo** (não tava em `Estabelecimentos.md`):
- Adiciona linha na tabela de `Estabelecimentos.md`:
  ```
  | Mercado XYZ | Mercado | Alimentação | Mercado | YYYY-MM-DD |
  ```

**Se a pessoa fez uma escolha não-óbvia** (categorizou algo de um jeito que diverge da regra anterior, ou explicitou "isso aqui sempre vai em Y porque..."):
- Adiciona em `Preferências.md > Decisões não-óbvias`

**Se a pessoa criou conta/cartão novo:**
- Adiciona em `Contas e Cartões.md`

### Passo 10 — Resposta final

Curta, direta:

> ✓ Lançado. R$45 em Mercado / Alimentação > Mercado / Nubank débito.
> Aprendi: "mercado xyz" → Alimentação > Mercado.

Sem floreio. Pessoa quer saber que deu certo e seguir.

## Casos especiais

### Bills (recorrentes)

Se a pessoa lançar algo que parece **gasto recorrente** (luz, água, internet, aluguel, condomínio, plano de celular), depois de lançar pergunta uma vez:

> Esse gasto é recorrente? Se for, posso transformar em "bill" no FIN pra ele aparecer todo mês automaticamente.

Se sim, cria via `fin_criar_bill`. Se não, segue com a despesa avulsa normal.

**Não pergunta toda vez.** Só na primeira ocorrência de cada estabelecimento. Depois que virou bill, lança automaticamente nas próximas.

### Parcelamento

Se a pessoa disser **"parcelado em N vezes"** ou "X parcelas":

- O FIN tem suporte nativo a parcelamento. Use o campo correto na tool de criação (confira na description de `fin_criar_despesa`).
- **NÃO crie N transações manualmente.** O FIN gera as parcelas automaticamente.
- Confirmação especial:
  > Despesa R$1200 em 12x de R$100 / Notebook / Tecnologia > Eletrônicos / C6 crédito. Primeira parcela cai na fatura que fecha em [data]. Confirma?

### Estorno

Se a pessoa disser **"foi estornado"** ou "veio estorno de X":

- **Estorno NÃO é receita.** É uma despesa com `reversal_of_id` apontando pra transação original.
- Você precisa achar a transação original primeiro: `fin_buscar_transacoes` filtrando por valor + estabelecimento próximos
- Mostra: "Achei a despesa original (R$100, Lojas X, dia Y). Vou criar o estorno apontando pra ela. Confirma?"
- Se não achar a original, pergunta detalhes pra encontrar antes de tentar lançar

### Múltiplas formas de pagamento na mesma transação

Tipo: "paguei 50 no PIX e 30 em dinheiro no almoço".
→ São 2 transações separadas. Lança uma de cada vez, mas confirma as duas juntas:

> Vou lançar 2 despesas:
> 1. R$50 / Restaurante / Alimentação > Restaurante / [conta PIX] / hoje
> 2. R$30 / Restaurante / Alimentação > Restaurante / Dinheiro / hoje
> Confirma as duas?

### Data não-padrão

Se a pessoa mencionar data ("ontem", "anteontem", "dia 5", "10 de março"), parse pra ISO (YYYY-MM-DD) e usa no `data` da transação. Pra "hoje" (default), usa data atual.

Pra "essa semana" / "mês passado" / coisas vagas → pergunta o dia exato.

### Valor ambíguo

Se a pessoa não disser o valor claramente, pergunta. Não chuta.

### Estabelecimento ambíguo

Se a pessoa disser só "lança um almoço" sem nome, pergunta:
> Onde foi o almoço? (nome do lugar pra eu poder lembrar das próximas)

Se ela disser "qualquer lugar, sei lá", aceita "Almoço" como descrição genérica e categoriza em Alimentação > Restaurante (sem aprender estabelecimento).

## Erros comuns que você deve evitar

1. **Lançar saque como despesa** → saque é transferência banco → Dinheiro
2. **Lançar estorno como receita** → estorno é despesa com `reversal_of_id`
3. **Criar parcelas manualmente** → o FIN cria automático via `installments`. **Exceção (parcela X/N de compra antiga, X > 1):** a partir da v2.3.2 dá pra usar `current_installment` **só se `tx_date` for a data da parcela atual** (não da compra original). Se `tx_date` é da compra original e os dias não alinham com o ciclo atual, o FIN rotula errado (Bug #12) — aí lança como **despesas avulsas numeradas manualmente** `"Nome (parcela X/N)"`, uma por mês, com `invoice_cycle_end` na primeira pra forçar a fatura atual. Ver `skills/fatura/SKILL.md` pra regra completa dos 2 cenários.
4. **Aplicar regra aprendida sem mostrar** → sempre diz "apliquei tua regra: X → Y"
5. **Atualizar regra existente sem confirmar** → se a pessoa categorizou diferente dessa vez, pergunta se é exceção ou nova regra
6. **Aprender estabelecimento depois de uma única ocorrência sem confirmação** → sempre confirma a categoria antes de gravar
7. **Não confirmar antes de criar** → toda mutação tem confirmação em uma linha
8. **Encher linguiça** → resposta é curta. "✓ Lançado. [resumo]" e fim.
9. **Pedir confirmação com 5 linhas de texto** → uma linha basta
10. **Esquecer de atualizar `Estabelecimentos.md`** → toda transação com estabelecimento novo gera linha nova
11. **Tentar `fin_criar_despesa` em conta USD** → bloqueado. Use `fin_ajustar_saldo_conta` pra subtrair do saldo (sem categorização, limitação v0)
12. **Passar "o saldo que a pessoa quer ver" direto pro `fin_ajustar_saldo_conta`** → a tool sobrescreve o `initial_balance`, não o saldo exibido. Se a conta tem transações, o FIN vai recalcular por cima e o valor fica errado. Use a fórmula `initial_balance_novo = initial_balance_atual + (saldo_desejado − saldo_exibido_atual)`. Ver **Caso B/C** nesta skill pra fluxo completo.
13. **Inventar cotação no câmbio** → o FIN não usa cotação automática. Se a pessoa só falou uma das duas quantias, pergunta a outra
14. **Lançar câmbio como 2 transferências separadas** → câmbio é `fin_cambio` (atômico, 1 chamada, 2 transações vinculadas com `exchange_pair_id`)
15. **Tentar câmbio com cartão de crédito** → só funciona entre 2 contas cash de moedas diferentes
16. **Esquecer de avisar limitação v0 ao gastar em USD** → sempre explica que categorização ficou de fora, pra pessoa não achar que é bug

## Tom

PT-BR informal, direto. Sem travessão (—). Resposta curta, ação rápida.
