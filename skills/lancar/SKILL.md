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

#### Regras de negócio críticas (do FIN v2.3.4)

1. **`fin_criar_despesa` ACEITA contas USD** (a partir de v2.3.4). `amount` sempre é gravado em BRL (fonte da verdade pra relatórios), mas a tool aceita `original_amount_cents` + `original_currency: "USD"` pra preservar o valor nativo. Dois modos:
   - **Modo (a) — pessoa informa o BRL exato:** passa `amount_cents` (BRL que saiu) + `original_amount_cents` (US$) + `original_currency: "USD"`. Use quando a pessoa sabe o valor real que saiu da conta (Wise mostra, extrato do banco mostra).
   - **Modo (b) — pessoa só sabe a cotação:** passa `original_amount_cents` + `original_currency: "USD"` + `exchange_rate_cents_per_unit` (cotação BRL por 1 USD em 1/10000, ex: 51023 = R$5,1023/USD). Backend calcula o BRL. Use quando a pessoa só tem a cotação estimada.
   - **NUNCA** passa só `amount_cents` numa conta USD. Retorna 422.
   - O que perguntar pra pessoa: *"Gastou US$X na [Conta USD] — sabe quanto saiu em reais da tua conta, ou prefere estimar com uma cotação?"*
2. **Câmbio é uma operação atômica** via `fin_cambio`. Cria 2 transações vinculadas (uma despesa na origem, uma receita no destino), ambas com categoria "Câmbio" e um `exchange_pair_id` compartilhado. Se uma falhar, a outra é desfeita.
3. **O FIN nunca usa cotação automática ao gravar.** Nem em `fin_cambio`, nem em `fin_criar_despesa` USD. A pessoa informa valores manualmente porque a taxa real varia (Wise spot ≠ casa de câmbio ≠ banco). Exceção: `fin_patrimonio` converte USD→BRL dinamicamente **só na leitura** pra responder "quanto eu tenho hoje em reais" — essa é uma pergunta de balance, não de transação.
4. **`fin_ajustar_saldo_conta` retorna `balance_cents_calculado` direto** (a partir de v2.3.4). Antes a gente tinha que re-chamar `fin_saldos` depois pra validar. Agora a tool calcula o saldo exibido pós-ajuste e devolve junto com `delta_liquido_cents`. **Ainda é armadilha** que a tool sobrescreve `initial_balance`, não o saldo exibido — mas agora você não precisa calcular nada manualmente: passa o saldo desejado e confere no retorno se `balance_cents_calculado` bateu com o esperado. Se não bateu, investiga. Ver **Caso B/C** abaixo.
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

**O que mudou (v2.3.4):** `fin_ajustar_saldo_conta` agora retorna `balance_cents_calculado` e `delta_liquido_cents` diretamente na resposta. Você não precisa mais fazer o roundtrip manual calculando `initial_balance_novo`. Só passa o saldo que a pessoa quer ver e confere no retorno.

**Armadilha conhecida:** a tool sobrescreve `initial_balance`, não o saldo exibido. Mas agora o backend calcula o saldo pós-ajuste pra você e devolve. Se `balance_cents_calculado` (no retorno) não bater com o desejado, investiga — tem transação faltando/sobrando.

**Padrões de fala:**
- "Tô com $487 na carteira agora" → ajuste pra valor exibido final = $487
- "Agora tenho $1200 na Wise" → ajuste pra $1200
- "Achei mais $50, total tá em $537" → ajuste pra $537
- "Ajusta o Caixa pra 1234,56" → ajuste pra R$1234,56
- "O saldo do Itaú tá errado, tá em R$2000 e devia ser R$2150" → ajuste pra R$2150

**Fluxo simplificado (vale pra BRL e USD):**
1. Identifica a conta.
2. Confirma com a pessoa em uma linha:
   > Ajustar saldo: Wise USD → $487. Confirma?
3. Chama `fin_ajustar_saldo_conta` passando o **saldo desejado** em centavos:
   ```
   { "account_name": "Wise USD", "amount_cents": 48700 }
   ```
4. **Valida pelo retorno da tool**: confere que `balance_cents_calculado` bate com o desejado.
   - **Se bateu:** avisa o saldo novo e segue.
   - **Se não bateu:** a tool aceitou mas o saldo exibido ficou diferente porque a conta tem transações que fazem o cálculo `initial_balance + Σ(tx)` não dar no valor que a pessoa quer. Não tenta de novo — explica pra pessoa e investiga antes (provavelmente tem transação faltando ou sobrando no FIN).

**Se precisar de valor "delta" (ex: "achei mais $50" sem dizer total):**
1. Lê `fin_saldos` pra pegar o saldo exibido atual.
2. Calcula `saldo_desejado = atual + 50`.
3. Segue fluxo acima.

**Gasto em USD:** NÃO é mais caso de ajuste de saldo. A partir de v2.3.4, use `fin_criar_despesa` direto com `original_amount_cents` + `original_currency: "USD"`. Ver regra de negócio #1 acima.

#### Caso C — Ajustar saldo BRL pra correção

Mesmo fluxo do Caso B — `fin_ajustar_saldo_conta` funciona pra BRL igualzinho.

> "Ajusta o saldo do Caixa pra R$1234,56"

1. Confirma: *Ajustar saldo: Caixa → R$1234,56. Confirma?*
2. Chama `fin_ajustar_saldo_conta` com `amount_cents: 123456`.
3. Confere `balance_cents_calculado` no retorno.

**Atenção:** ajuste de saldo **não cria transação**, então não aparece em relatórios mensais como movimentação. É um ajuste contábil. Se a pessoa quiser que apareça como receita/despesa categorizada, oriente a usar `fin_criar_receita`/`fin_criar_despesa`.

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
| Despesa USD (categorizada) | `fin_criar_despesa` com `original_amount_cents` + `original_currency: "USD"` |
| Receita BRL ou USD | `fin_criar_receita` (mesma lógica multi-moeda) |
| Transferência (incluindo saque) | `fin_criar_transferencia` |
| Câmbio (vender ou comprar dólar) | `fin_cambio` |
| Ajuste de saldo (BRL ou USD) | `fin_ajustar_saldo_conta` |
| Estorno de cartão (já sabe o ID da original) | `fin_criar_estorno` |
| "Quanto eu tenho no total em reais?" | `fin_patrimonio` |

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

- O FIN tem suporte nativo a parcelamento. Passa `installments: N` em `fin_criar_despesa`.
- **NÃO crie N transações manualmente.** O FIN gera as parcelas automaticamente.
- Confirmação especial:
  > Despesa R$1200 em 12x de R$100 / Notebook / Tecnologia > Eletrônicos / C6 crédito. Primeira parcela cai na fatura que fecha em [data]. Confirma?

**Parcelamento retroativo (compra antiga já em andamento):**

> "Minhas Ray-Bans, comprei em dezembro 10x, tô na 4ª parcela"

A partir de v2.3.4, o caminho intuitivo é passar `original_purchase_date` + `installments` + `current_installment`:

```
{
  "description": "Ray-Ban",
  "amount_cents": 137000,
  "installments": 10,
  "current_installment": 4,
  "original_purchase_date": "2025-12-23",
  "account_name": "Caixa Cartão",
  "category_name": "Pessoal",
  "subcategory_name": "Acessórios"
}
```

O backend lê o cutoff do cartão e calcula em qual fatura a parcela 4 cai, 5, 6, ..., 10. Sem precisar pensar em `tx_date` nem em `invoice_cycle_end`. **Esse é o caminho preferido.**

O workaround antigo (lançar cada parcela avulsa numerada manualmente com `invoice_cycle_end` forçado) ainda funciona mas fica como fallback — use só se `original_purchase_date` não conseguir resolver por algum motivo específico.

### Estorno

Se a pessoa disser **"foi estornado"** ou "veio estorno de X":

- **Estorno NÃO é receita.** É uma despesa vinculada à transação original via `reversal_of_id`.
- **v2.3.4 simplificou:** se você já sabe o UUID da original (achou via `fin_buscar_transacoes` ou `fin_fatura_transacoes`), usa `fin_criar_estorno` — tool atômica que herda account/category/subcategory da original automaticamente. Só precisa passar `original_transaction_id` + `amount_cents`.
- Fluxo:
  1. Busca a original com `fin_buscar_transacoes` (valor + estabelecimento próximos, janela de 3 meses).
  2. Mostra: *"Achei a despesa original (R$100, Lojas X, dia Y). Vou criar o estorno apontando pra ela. Confirma?"*
  3. Chama `fin_criar_estorno({original_transaction_id: "...", amount_cents: 10000})`. Pode ser parcial.
  4. Se não achar a original, pergunta detalhes antes.

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

### Pix em fim de semana / feriado (regra D+1 útil)

Pix feito em sábado, domingo ou feriado é creditado no extrato bancário com **data D+1 útil** (convenção bancária brasileira). Exemplo: Pix feito sábado 18/04 sai pro destinatário com data de crédito segunda 20/04.

**Regra pra lançamento:**

- Se a pessoa falar *"acabei de fazer um Pix"* num sábado/domingo/feriado → **lança com a data do próximo dia útil**, não a data de hoje. Isso garante que bate com o extrato do banco quando conciliar.
- Confirmação na linha deve deixar explícita: *"...data 20/04 (próximo dia útil, padrão Pix fim de semana). Confirma?"*
- Se a pessoa quiser lançar com a data real em que fez (sábado), ela tem que pedir explicitamente — e alertar que vai divergir do extrato.

Isso vale pra **entradas** também: se alguém fez Pix pro Pedro no sábado, a entrada vai aparecer no extrato com data de segunda.

Regra não vale pra TED, débito automático, boleto — esses têm suas próprias convenções e normalmente o banco já mostra a data certa pra pessoa copiar.

### Valor ambíguo

Se a pessoa não disser o valor claramente, pergunta. Não chuta.

### Estabelecimento ambíguo

Se a pessoa disser só "lança um almoço" sem nome, pergunta:
> Onde foi o almoço? (nome do lugar pra eu poder lembrar das próximas)

Se ela disser "qualquer lugar, sei lá", aceita "Almoço" como descrição genérica e categoriza em Alimentação > Restaurante (sem aprender estabelecimento).

## Erros comuns que você deve evitar

1. **Lançar saque como despesa** → saque é transferência banco → Dinheiro
2. **Lançar estorno como receita** → estorno é despesa. Use `fin_criar_estorno` se já sabe o UUID da original (herda tudo automaticamente); senão `fin_criar_despesa` com `reversal_of_id` manual.
3. **Criar parcelas manualmente** → o FIN cria automático via `installments`. Pra compra antiga em andamento (parcela X/N com X > 1), o caminho preferido v2.3.4 é passar `original_purchase_date` junto — o backend coloca cada parcela na fatura certa usando o cutoff do cartão. Só cai em workaround manual se `original_purchase_date` falhar por algum motivo específico.
4. **Aplicar regra aprendida sem mostrar** → sempre diz "apliquei tua regra: X → Y"
5. **Atualizar regra existente sem confirmar** → se a pessoa categorizou diferente dessa vez, pergunta se é exceção ou nova regra
6. **Aprender estabelecimento depois de uma única ocorrência sem confirmação** → sempre confirma a categoria antes de gravar
7. **Não confirmar antes de criar** → toda mutação tem confirmação em uma linha
8. **Encher linguiça** → resposta é curta. "✓ Lançado. [resumo]" e fim.
9. **Pedir confirmação com 5 linhas de texto** → uma linha basta
10. **Esquecer de atualizar `Estabelecimentos.md`** → toda transação com estabelecimento novo gera linha nova
11. **Passar só `amount_cents` numa conta USD** → `fin_criar_despesa` exige `original_amount_cents` + `original_currency: "USD"` quando a conta é USD. Modo (a): + BRL exato. Modo (b): + `exchange_rate_cents_per_unit`. Ver Regra #1 do Passo 4.5.
12. **Calcular `initial_balance_novo` manualmente antes de `fin_ajustar_saldo_conta`** → não precisa mais a partir de v2.3.4. Passa o saldo desejado direto, valida `balance_cents_calculado` no retorno. Se não bater, investiga (tem tx faltando/sobrando).
13. **Inventar cotação no câmbio** → o FIN não usa cotação automática. Se a pessoa só falou uma das duas quantias, pergunta a outra
14. **Lançar câmbio como 2 transferências separadas** → câmbio é `fin_cambio` (atômico, 1 chamada, 2 transações vinculadas com `exchange_pair_id`)
15. **Tentar câmbio com cartão de crédito** → só funciona entre 2 contas cash de moedas diferentes
16. **Buscar transação original do estorno manualmente e chamar `fin_criar_despesa` com `reversal_of_id`** → usa `fin_criar_estorno` (atômico, herda account/category/subcategory).

## Tom

PT-BR informal, direto. Sem travessão (—). Resposta curta, ação rápida.
