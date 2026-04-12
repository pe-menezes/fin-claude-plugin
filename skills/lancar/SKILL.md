---
name: lancar
description: >
  Lançamento avulso de transação no FIN App. Entende instruções em linguagem
  natural ("lança 45 no mercado, débito Nubank", "20 conto no pão, dinheiro",
  "saquei 200 no Itaú", "recebi 4mil do cliente X"), aplica regras aprendidas
  de Estabelecimentos.md, trata dinheiro vivo e saque corretamente, sempre
  confirma em uma linha antes de criar. Aprende e atualiza memória depois.
argument-hint: "[descrição da transação em linguagem natural]"
allowed-tools: Read, Write, Edit, Glob, Grep
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
| "lança 45 no mercado, débito Nubank" | despesa | 45 | mercado | Nubank (débito) |
| "20 conto no pão, dinheiro" | despesa | 20 | padaria/pão | Dinheiro |
| "120 no posto, crédito C6" | despesa | 120 | posto | C6 (crédito) |
| "recebi 4 mil do cliente X" | receita | 4000 | cliente X | (perguntar conta destino) |
| "transferi 500 do Nubank pro C6" | transferência | 500 | — | Nubank → C6 |
| "saquei 200 no Itaú" | **transferência** | 200 | — | Itaú → Dinheiro |
| "PIX 50 pra mãe" | despesa OU transferência | 50 | mãe | (perguntar conta origem) |

**Notação numérica BR:**
- "20 conto" / "20 mango" / "20 pila" = R$20
- "4 mil" / "4k" = R$4.000
- "500 reais" / "500" = R$500
- Vírgula é decimal: "45,50" = R$45,50

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
| Despesa | `fin_criar_despesa` |
| Receita | `fin_criar_receita` |
| Transferência (incluindo saque) | `fin_criar_transferencia` |

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
3. **Criar parcelas manualmente** → o FIN cria automático, use o campo de parcelamento
4. **Aplicar regra aprendida sem mostrar** → sempre diz "apliquei tua regra: X → Y"
5. **Atualizar regra existente sem confirmar** → se a pessoa categorizou diferente dessa vez, pergunta se é exceção ou nova regra
6. **Aprender estabelecimento depois de uma única ocorrência sem confirmação** → sempre confirma a categoria antes de gravar
7. **Não confirmar antes de criar** → toda mutação tem confirmação em uma linha
8. **Encher linguiça** → resposta é curta. "✓ Lançado. [resumo]" e fim.
9. **Pedir confirmação com 5 linhas de texto** → uma linha basta
10. **Esquecer de atualizar `Estabelecimentos.md`** → toda transação com estabelecimento novo gera linha nova

## Tom

PT-BR informal, direto. Sem travessão (—). Resposta curta, ação rápida.
