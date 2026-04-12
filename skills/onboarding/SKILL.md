---
name: onboarding
description: >
  Onboarding completo do FIN App. Três modos: Modo A (questionário guiado em 6
  blocos, do zero) — Modo B (importação de documento de setup pronto) — Modo C
  (aprendizado retroativo, quando o FIN já tem contas/categorias/histórico e o
  plugin precisa só aprender o que já existe e popular a memória). Use quando o
  FIN da pessoa estiver vazio, quando ela tiver um documento pronto, ou quando
  ela já usa o FIN há tempo mas é a primeira vez rodando o plugin. Idempotente:
  rodar 2x não duplica nada.
argument-hint: "[caminho-do-documento-de-setup OU 'retroativo']"
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

## Quando usar

- **Modo A — FIN vazio:** `fin_listar_contas` retorna lista vazia. Pessoa pediu "configura meu FIN do zero", "vamos do zero", "instala o FIN pra mim"
- **Modo B — Documento pronto:** pessoa tem um documento de setup pronto e quer importar
- **Modo C — Aprendizado retroativo:** pessoa **já usa o FIN há tempo**, tem contas/cartões/categorias/histórico, mas é a **primeira vez rodando o plugin financeiro**. O plugin precisa aprender o que já existe e popular a memória `.md` em vez de criar do zero.

## Quando NÃO usar

- O MCP do FIN não tá instalado — rode `/financeiro:instalar-fin-mcp` antes
- A pasta `Financeiro/` da pessoa **já tem `Estabelecimentos.md` populado** com regras (ou seja, o plugin já aprendeu antes nessa máquina) — nesse caso, o trabalho de aprendizado já foi feito, não precisa rodar onboarding

## Pré-requisitos (verificar ANTES de começar)

1. **MCP do FIN instalado.** Tente `fin_listar_contas`. Se falhar, dispare `/financeiro:instalar-fin-mcp` e pause.
2. **Pasta `Financeiro/` definida.** Leia `~/.fin-plugin/config.json`. Se não existir, faça o setup de localização (ver agente principal, seção "Primeira execução"), depois volte.
3. **Ler `fin://docs/guia`** pra ter as regras de negócio do FIN frescas.
4. **Verificar se FIN já tá vazio.** Rode `fin_listar_contas` e `fin_listar_categorias`. Se já houver coisa, **avise a pessoa antes** e pergunte se ela quer continuar (modo idempotente: você só cria o que falta) ou abortar.

## Detecção de modo

Antes de perguntar qualquer coisa, **detecte automaticamente** rodando:

```
fin_listar_contas
fin_listar_categorias
```

Lógica de detecção:

1. **FIN vazio** (sem contas E sem categorias) → vai pro **Modo A** (do zero), a menos que `$ARGUMENTS` aponte pra um documento (aí Modo B)
2. **FIN populado** (contas + categorias + provavelmente histórico) → vai pro **Modo C** (aprendizado retroativo), a menos que a pessoa explicitamente peça outro modo
3. **`$ARGUMENTS` é caminho de arquivo `.md`/`.txt`** → vai pro **Modo B** (importação), independente do estado do FIN
4. **`$ARGUMENTS` contém "retroativo" / "aprender" / "já tenho"** → força **Modo C** mesmo se o FIN parecer vazio

Quando detectar Modo C automaticamente, **avise a pessoa** antes de seguir:

> Vi que tu já tem [N] contas, [M] cartões e [K] categorias no FIN. Vou rodar em **modo retroativo**: não vou criar nada, vou só ler o que já tem, analisar teu histórico e popular minha memória pra aprender teus padrões. Beleza? (isso é não-destrutivo, posso parar a qualquer momento)

Se a pessoa quiser outro modo mesmo com FIN populado (raro: ela quer redo do zero), respeita.

---

## Modo A — Questionário guiado (do zero)

Você vai entrevistar a pessoa em **6 blocos**. Avance bloco por bloco. **Não pule nada.** Confirme ao final de cada bloco antes de seguir.

Tom: parceiro direto, pergunta uma coisa de cada vez quando precisa de detalhe, agrupa quando dá pra agrupar. Não enche linguiça.

### Bloco 1 — Contas (bancárias, dinheiro vivo, USD)

Pergunte:

> **Bloco 1: Contas.** Em quais bancos tu tem conta? Pode listar todas, pode ser conta corrente, poupança, conta digital.

Pra cada banco que ela listar, capture:
- **Nome do banco** (exatamente como ela falou, não normaliza)
- **Tipo:** corrente, poupança, digital
- **Moeda:** BRL (default — não pergunta se for óbvio que é banco BR)
- **Saldo atual** (opcional, pode ser 0 — ela informa depois se quiser)

Depois, **pergunta sobre dinheiro vivo**:

> Tu costuma andar com dinheiro vivo? Carteira, cofre em casa, envelope?

Se sim, crie uma conta tipo "Dinheiro" (ou o nome que ela preferir, ex: "Carteira"). **Dinheiro vivo é uma conta como qualquer outra no FIN**, vai ter saldo, recebe lançamentos, participa de transferências.

Se ela tem múltiplos lugares de dinheiro vivo (carteira + cofre + envelope viagem), pergunte:

> Quer que eu junte tudo numa conta única "Dinheiro" ou prefere separar (Carteira, Cofre, etc.)?

Default: uma conta única.

**Pergunta sobre conta em dólar:**

> Tu tem alguma conta em **dólar**? Wise, conta nos EUA, carteira de USD, qualquer coisa que guarda valor em USD em vez de reais?

Se sim, pra cada conta USD capture:
- **Nome** (Wise USD, Conta EUA, Carteira USD, etc.)
- **Tipo:** cash (todas as contas USD são cash no v0 do FIN, não tem cartão de crédito USD)
- **Moeda:** USD
- **Saldo atual em USD** (opcional, pode ajustar depois via `fin_ajustar_saldo_conta`)

**Avise sobre as limitações do v0 USD:**

> Aviso: contas em dólar no FIN, no momento, são "saldo manual". Tu não consegue lançar despesa categorizada em USD ainda (limitação v0). O que dá pra fazer:
>
> 1. **Câmbio** USD ↔ BRL (vender ou comprar dólar) — cria 2 transações vinculadas, categoria "Câmbio"
> 2. **Ajustar saldo** da conta USD a qualquer momento (somar/subtrair dólar)
> 3. **Gastar em USD** (ex: AliExpress, viagem) entra como ajuste de saldo, sem categoria
>
> Quando tu fizer câmbio, eu registro nas duas pontas direitinho. Beleza?

**Confirme o bloco 1** antes de seguir:

> Anotei: [N contas BRL bancárias] + [conta Dinheiro, se aplicável] + [N contas USD, se aplicável]. Confirma ou ajusta?

### Bloco 2 — Cartões de crédito

> **Bloco 2: Cartões de crédito.** Tem cartão de crédito? De quais bancos ou lojas?

Pra cada cartão, capture:
- **Nome** (como ela fala)
- **Limite** (se ela souber)
- **Dia de vencimento** (dia do mês que a fatura vence)
- **Melhor dia de compra** (dia que ela pode comprar pra ganhar mais prazo)
- **Vinculado a qual conta?** (cartão de banco vinculado a conta corrente, OU cartão de loja independente sem conta associada)

**Cálculo automático do fechamento:**
> fechamento = melhor dia de compra - 1

Se a pessoa souber o dia de fechamento direto, aceite. Se ela não souber nem o melhor dia nem o fechamento, peça pra ela conferir no app do banco e voltar.

**Cartão de loja vs cartão de banco:**
- Cartão de banco digital (mesma instituição da conta corrente) → vinculado à conta correspondente
- Cartão de varejista (loja) → SEM conta vinculada

**Aviso sobre variação de fechamento (importante):**

> Esse dia de fechamento que tu me passou é referência. Na prática, pode variar 1 ou 2 dias por causa de fim de semana, feriado ou ciclo do banco. Quando eu processar uma fatura tua depois, eu uso o período real da fatura, não o dia fixo. Beleza?

**Confirme o bloco 2** antes de seguir.

### Bloco 3 — Perfil de renda

> **Bloco 3: Tua renda.** Tu é CLT, PJ, autônoma, ou um misto?

#### Se CLT
Sugira categorias de receita simples e confirme:
- Salário
- VA/VR
- Bônus
- PLR
- 13º

Pergunte se tem outras fontes (aluguel, freela ocasional, etc.).

#### Se PJ ou Autônomo
Fluxo aprofundado:

> Tu emite nota por uma empresa (CNPJ próprio) ou recebe direto como PF?

Se tem CNPJ:
> Qual o nome ou apelido dessa empresa? Vou usar como categoria mãe das tuas receitas. (ex: "Trabalho [nome]")

Se recebe como PF, a categoria mãe vira só "Trabalho" ou "Renda Profissional".

> Quais tipos de trabalho tu faz? Lista pra mim, cada tipo vai virar uma subcategoria.

**NÃO sugira tipos.** A pessoa que diz. Anote literal.

> Tem contrato fixo mensal com algum cliente?

Se sim, crie uma subcategoria especial pra esse contrato fixo (ex: "Fixo [nome do cliente]"), pra ela conseguir separar fixo de avulso na análise.

**Quando tem CNPJ PJ, avise:**
> Como tu é PJ, vou criar uma categoria de despesa "Taxas, Juros & Impostos" com subs "Impostos PJ (DAS)", "INSS", "Contador". Beleza?

#### Se Misto (CLT + freela)
Combina os dois fluxos: cria categorias CLT + categorias do freela com o mesmo aprofundamento.

#### Receitas extras (independem do perfil)
Pergunta no fim do bloco:
> Tem alguma renda extra? Aluguel, rendimento de investimento, reembolso, venda de coisa pessoal?

Cria categorias conforme o que ela disser. Sugere genéricas: Renda Fixa Pessoal, Rendimento, Reembolso, Pontuais, Ajuste Financeiro.

**Confirme o bloco 3.**

### Bloco 4 — Estilo de vida (perguntas rápidas)

Esse é o bloco mais conversacional. Pergunta uma coisa de cada vez, mas rápido. Cada resposta gera uma ou mais categorias.

> **Bloco 4: Estilo de vida.** Vou fazer umas perguntas rápidas pra montar tuas categorias de despesa.

1. **"Tem pet?"** → se sim, cria categoria **Pets** com subs (Ração, Veterinário, Banho/Tosa, Brinquedos, Outros)
2. **"Tem carro ou moto?"** → se sim, cria categoria **Transporte** com subs (Combustível, Estacionamento, Manutenção, Seguro, IPVA, Lavagem, App, Transporte Público, Outros). Se não tem carro/moto, cria Transporte com subs menores (App, Transporte Público, Outros)
3. **"Mora de aluguel ou imóvel próprio?"** → cria **Casa** com sub **Aluguel** OU **Financiamento** (uma das duas, não as duas). Sempre adiciona subs comuns: Contas fixas (luz, água, internet, condomínio), Limpeza, Manutenção, Móveis e Utensílios, Outros
4. **"Faz atividade física?"** → adiciona **Saúde > Atividade física**
5. **"Faz terapia?"** → adiciona **Saúde > Terapia** (separada de Consultas)
6. **"Tem profissão ou hobby que muda como tu enxerga consumo cultural?"** → pergunta importante. Streaming, cinema, teatro, livros, podcasts, museus podem ir pra **Lazer** OU pra **Educação & Desenvolvimento** dependendo da resposta. Profissões criativas, artísticas, de pesquisa ou ensino tendem a tratar consumo cultural como trabalho/estudo, e nesse caso vai pra Educação. **Default = Lazer.**
7. **"Tem rotina de estética ou cuidados regulares? Cabelo, manicure, drenagem, depilação?"** → cria subs específicas em **Pessoal** (achatadas se necessário, ver regra de achatamento abaixo)
8. **"Fuma, vapeia, ou tem algum vício controlado que tu quer rastrear?"** → cria **Pessoal > Tabaco/Vícios**
9. **"Tem algum gasto recorrente importante que não se encaixa no padrão?"** → captura o inesperado, cria categoria/sub conforme

### Achatamento de hierarquia (REGRA CRÍTICA)

**O FIN só suporta 2 níveis: categoria → subcategoria.** Se a pessoa quiser 3 níveis (ex: Estética > Cabelo > Corte/Coloração), você precisa achatar:

**Opção 1 — Achatar pra subs diretas com prefixo:**
- Estética Cabelo Corte
- Estética Cabelo Coloração
- Estética Manicure
- Estética Drenagem

**Opção 2 — Reduzir granularidade:**
- Estética Cabelo (sem detalhar mais)
- Estética Manicure
- Estética Drenagem

Explique o limite pra pessoa antes de criar:
> Por enquanto o FIN só tem 2 níveis (categoria + subcategoria). Pra essa rotina de cuidados, posso fazer assim: [opção 1 ou 2]. Qual prefere?

### Bloco 5 — Proposta de categorias completa

Com tudo dos blocos 3 e 4, você tem a base. Agora você consolida e mostra a **proposta completa** pra revisão.

Formato da proposta:

```markdown
## Proposta de categorias

### Despesas
- **Alimentação**: Mercado, Restaurante, Delivery, Lanche, Outros
- **Casa**: Aluguel, Contas fixas (luz, água, internet, condomínio), Limpeza, Manutenção, Móveis e Utensílios, Outros
- **Transporte**: Combustível, Estacionamento, Manutenção, Seguro, IPVA, Lavagem, App, Transporte Público
- **Saúde**: Atividade física, Terapia, Consultas, Exames, Procedimentos, Medicamentos, Higiene
- **Lazer & Social**: Bar/Rua, Eventos, Viagem
- **Educação & Desenvolvimento**: Cursos, Livros, Streaming, Cinema, Teatro
- **Pessoal**: Estética Cabelo, Estética Manicure, Roupas/Acessórios, Tabaco/Vícios
- **Família & Amigos**: Presentes, Mesada
- **Tecnologia & Assinaturas**: Assinaturas (Software/Cloud), IA, Eletrônicos
- **Pets**: Ração, Veterinário, Banho/Tosa
- **Taxas, Juros & Impostos**: Anuidade/Taxas cartão, IOF, Juros, Multa

### Receitas
- **Salário**: Fixo, VA/VR, Bônus, PLR, 13º
- **Pontuais**: Venda de item pessoal, Outros
- **Reembolso**: Amigos, Família
```

(Adapte às respostas reais — esse é só formato.)

Mostre e pergunte:

> Essa é minha proposta. Quer ajustar alguma coisa? Tirar, adicionar, renomear?

A pessoa pode rodar **quantas rodadas de ajuste precisar** até confirmar. Cada rodada, você atualiza a lista e mostra de novo.

**Captura de "decisões não-óbvias":** durante o ajuste, se a pessoa fizer uma escolha que não é o default (ex: "Streaming → Educação em vez de Lazer", "Higiene → Saúde em vez de Pessoal", "Plano de celular → Casa em vez de Tecnologia"), **anote essa decisão**. Vai pra `Preferências.md > Decisões não-óbvias` no final.

### Bloco 6 — Geração e execução

#### Passo 1: Gerar o documento de setup

Crie `Financeiro/Setup.md` com tudo consolidado:

```markdown
---
tipo: setup-financeiro
status: pronto-para-criar
gerado_em: {ISO timestamp}
---

# Setup FIN

## Contas

| Nome | Tipo | Saldo inicial |
|---|---|---|

## Cartões

| Nome | Limite | Fechamento | Vencimento | Melhor dia compra | Conta vinculada |
|---|---|---|---|---|---|

## Categorias de despesa

(lista categoria > subcategorias)

## Categorias de receita

(lista categoria > subcategorias)

## Decisões não-óbvias

(captura de escolhas que fugiram do default, com motivo se houver)
```

**Mostre o documento gerado** pra confirmação final. A pessoa lê, ajusta se quiser, confirma.

#### Passo 2: Executar no FIN

Antes de qualquer criação, **rode `fin_listar_contas` e `fin_listar_categorias`** pra ver o que já existe. **Pula o que já existe** (idempotência).

Ordem de criação:
1. **Contas** — `fin_criar_conta` pra cada conta nova (incluindo Dinheiro)
2. **Cartões** — também via `fin_criar_conta` (cartões são contas tipo crédito no FIN, ou via tool específica se houver — confira a description da tool)
3. **Categorias de despesa** — `fin_criar_categoria` pra cada
4. **Subcategorias de despesa** — `fin_criar_subcategoria` pra cada
5. **Categorias de receita** — `fin_criar_categoria`
6. **Subcategorias de receita** — `fin_criar_subcategoria`

**Mostre o progresso em tempo real:**
> Criando contas... [3/4 ✓] Caixa, Sicredi, Nubank
> Criando cartões... [2/6 ✓] C6, Nubank
> Criando categorias... [5/12 ✓] ...

#### Passo 3: Popular os arquivos `.md` da pasta `Financeiro/`

Depois que tudo foi criado no FIN, popula:

**`Financeiro/Contas e Cartões.md`** — lista todas as contas e cartões criados, com fechamento cadastrado, vencimento, conta vinculada.

**`Financeiro/Preferências.md`** — começa vazio mas com a seção **Decisões não-óbvias** já preenchida com as decisões capturadas no Bloco 5.

**`Financeiro/Estabelecimentos.md`** — começa vazio (vai populando conforme uso).

**`Financeiro/Status Conciliação.md`** — começa vazio (vai populando conforme conciliações).

#### Passo 4: Resumo final

> Pronto. Criado: X contas, Y cartões, Z categorias de despesa com W subs, K categorias de receita com L subs. Tudo no FIN. Tuas decisões não-óbvias ficaram salvas em Preferências.md.
>
> Próximos passos sugeridos:
> - Lançar uma despesa pra testar: `/financeiro:lancar` ou só me diz "lança X reais em Y"
> - Importar um extrato: `/financeiro:extrato`
> - Importar uma fatura: `/financeiro:fatura`

---

## Modo B — Importação de documento de setup pronto

A pessoa já tem um documento (porque fez o questionário antes, porque alguém montou pra ela, ou porque ela escreveu manualmente).

### Passo 1: Localizar e ler o documento

Se a pessoa passou o caminho como `$ARGUMENTS`, use. Se não, pergunte:

> Qual o caminho do documento de setup?

Leia o arquivo. Faça parse de:
- **Contas** (nome, tipo, saldo inicial se houver)
- **Cartões** (nome, limite, fechamento OU melhor dia, vencimento, conta vinculada se houver)
- **Categorias de despesa** (categoria → subcategorias)
- **Categorias de receita** (categoria → subcategorias)
- **Decisões não-óbvias** (se a seção existir)

### Passo 2: Validar

Cheque se tem dados suficientes pra criar tudo. Se faltar algo crítico:
- **Cartão sem dia de vencimento ou fechamento** → pergunta só esse dado
- **Categoria sem subcategorias** → cria a categoria mesmo assim, mas avisa
- **Conta sem tipo** → assume "corrente" se for nome de banco, ou pergunta

**Não pergunta o que já tem.** Só o que falta.

### Passo 3: Mostrar resumo

```
Vou criar no FIN:
- 4 contas: Caixa, Sicredi, Nubank, C6
- 6 cartões: C6, Nubank, Sicredi, Caixa, C&A (loja), Renner (loja)
- 11 categorias de despesa com 47 subs
- 6 categorias de receita com 19 subs

Confirma?
```

### Passo 4: Executar

Mesma lógica do Modo A, Passo 2 do Bloco 6:
1. Verifica idempotência (`fin_listar_contas`, `fin_listar_categorias`)
2. Cria só o que falta
3. Mostra progresso
4. Popula os `.md` da pasta `Financeiro/`

### Passo 5: Resumo final

Mesmo do Modo A.

---

## Modo C — Aprendizado retroativo (FIN já populado)

A pessoa **já usa o FIN há tempo**. Tem contas, cartões, categorias e histórico de transações. Esse é o caso mais comum de quem já é usuário do FIN antes de instalar o plugin.

**Princípio:** o Modo C é **não-destrutivo**. Você não cria, não edita, não deleta nada no FIN. Você só **lê**, analisa e popula os arquivos `.md` da pasta `Financeiro/` da pessoa.

### Fase 1 — Inventário (automático, sem perguntar)

Rode em sequência:

1. `fin_listar_contas` → captura todas as contas (e cartões, que no FIN são contas tipo crédito)
2. `fin_listar_categorias` → captura todas as categorias e subcategorias

**Identifica também as contas USD** (campo `currency` ou similar — confira na description da tool). Conta a quantidade.

Mostra um resumo curto:

> Encontrei: [N] contas bancárias BRL, [U] contas em USD (se houver), [M] cartões de crédito, [K] categorias com [W] subcategorias no total. Vou popular tua memória local com isso.

Se detectou contas USD, avisa que vai tratar separado:

> Detectei [U] conta(s) em dólar. Pra essas, no v0 do FIN só dá pra usar câmbio (`fin_cambio`) e ajuste de saldo (`fin_ajustar_saldo_conta`). Vou popular o histórico de câmbio na análise.

**Popule `Financeiro/Contas e Cartões.md`** automaticamente:
- Tabela de contas: nome, tipo, **moeda (BRL/USD)**, apelidos (vazio inicialmente, vai aprendendo)
- Tabela de cartões: nome, tipo, limite, dia fechamento (cadastrado), vencimento, conta vinculada (se houver)
- Coluna "fechamento observado" começa vazia (vai populando na Fase 5)

### Fase 2 — Janela de análise

Pergunte:

> Pra eu aprender teus padrões, vou analisar teu histórico de transações. **Quanto tempo de histórico tu quer que eu analise?** (default: últimos 6 meses)
>
> 1. Últimos 3 meses
> 2. Últimos 6 meses (default)
> 3. Últimos 12 meses
> 4. Todo o histórico (pode demorar mais)

Default = 6 meses. Aceita qualquer escolha.

### Fase 3 — Análise de histórico de transações

Puxa todas as transações do período via `fin_buscar_transacoes` ou `fin_todas_transacoes` (a tool certa depende da API — confira a description antes).

Para cada transação, capture:
- Data
- Descrição (nome do estabelecimento)
- Valor
- Categoria
- Subcategoria
- Conta/cartão usado

Mostra progresso:
> Analisando histórico... [243/512 transações processadas]

### Fase 4 — Detecção de padrões de estabelecimentos

Agrupa as transações por **descrição/estabelecimento** (normaliza: lowercase, remove caracteres especiais, agrupa variantes próximas tipo "UBER *TRIP" + "UBER *VIAGEM" + "UBER").

**Exclui transações da categoria "Câmbio"** desse agrupamento — câmbio não tem "estabelecimento" no sentido tradicional. Apenas conta as ocorrências pra reportar no resumo final ("Detectei N operações de câmbio no período").

Pra cada estabelecimento, analisa:
- **Quantas vezes apareceu** no período
- **Quais categorias/subcategorias** foram usadas
- **Categoria majoritária** (a mais usada)

Classifica em 3 grupos:

**Grupo 1 — Padrão consistente (3x ou mais, sempre mesma categoria/subcategoria)**
→ Vira regra automática. Vai direto pro `Estabelecimentos.md`.

**Grupo 2 — Padrão provável (2x na mesma categoria, OU 3x+ com leve variação)**
→ Vira candidato. Mostra pra pessoa decidir.

**Grupo 3 — Inconsistente (categorias diferentes em ocorrências diferentes)**
→ Marca como ambíguo. Não vira regra. Anota em `Preferências.md > Estabelecimentos ambíguos` pra eventual resolução manual.

### Fase 5 — Detecção de fechamento real dos cartões

Pra cada cartão de crédito que apareceu nas transações, busca as faturas via `fin_fatura_cartao` ou tool equivalente.

Pra cada fatura:
- Identifica a **data real de fechamento** (último dia do período da fatura)
- Compara com o **dia cadastrado** do cartão

Se houver variação (1-2 dias de diferença em algumas faturas), atualiza `Contas e Cartões.md`:
- Coluna "fechamento observado" recebe o dia mais frequente nas últimas faturas
- Seção "Histórico de variação" recebe nota tipo: "Cartão X: cadastrado dia 19, observado dia 19 em [3 das 5 faturas], dia 20 em [2 das 5 faturas]. Variação por fim de semana."

### Fase 6 — Revisão de estabelecimentos com a pessoa

Mostra uma **tabela única** com os estabelecimentos detectados, organizados por grupo:

```
=== Padrões automáticos (já vou gravar) ===
| Estabelecimento     | Categoria       | Sub        | # ocorrências |
|---------------------|-----------------|------------|---------------|
| iFood               | Alimentação     | Delivery   | 23            |
| Uber                | Transporte      | App        | 18            |
| Drogaria São Paulo  | Saúde           | Higiene    | 12            |
...

=== Candidatos (preciso tu confirmar) ===
| Estabelecimento     | Padrão sugerido           | Confirma? |
|---------------------|---------------------------|-----------|
| Mercado XYZ         | Alimentação > Mercado     | [s/n]     |
| Posto Shell         | Transporte > Combustível  | [s/n]     |
...

=== Ambíguos (não viraram regra) ===
| Estabelecimento     | Categorias usadas                        |
|---------------------|------------------------------------------|
| Amazon              | Casa (3x), Pessoal (2x), Tecnologia (1x) |
...
```

Pergunta:

> Aceita os padrões automáticos? E pros candidatos, quais tu confirma?

A pessoa pode:
- Aceitar tudo de uma vez ("aceita tudo")
- Confirmar candidatos um por um
- Rejeitar ou corrigir alguma regra automática (raro mas possível)

Atualiza `Estabelecimentos.md` com o que ela confirmou.

### Fase 7 — Detecção de "decisões importantes" (sem hardcode de defaults)

Esse é o passo mais delicado. **Você não tem default pra comparar**, então não pode dizer "Streaming foi pra Educação, isso é não-óbvio". O que você faz:

1. Pega as **15 categorizações mais frequentes e consistentes** do histórico (estabelecimento → categoria → sub que aparecem 5+ vezes sem variação)
2. Mostra essa lista pra pessoa
3. Pergunta:

> Olha essas 15 regras que tu aplica direto. Quais delas são **decisões importantes que tu quer que eu lembre como regra de vida** (e não só inferi do histórico)? Marca as que importam.
>
> Exemplo: se tu sempre lança Spotify em "Casa > Contas fixas" e isso é uma escolha intencional tua, marca. Se foi só por acaso, deixa pra lá.

A pessoa marca quais são "regras de vida". Essas vão pra `Preferências.md > Decisões não-óbvias` com o motivo (pergunta o motivo de cada uma se ela quiser explicar — opcional).

Esse approach é **agnóstico**: você não decidiu o que é "não-óbvio" baseado em heurística sua. A pessoa que disse.

### Fase 8 — Status de conciliação

Pergunta:

> Quais contas e cartões tu já considera **conciliados** (extrato/fatura conferida) até alguma data?

A pessoa lista. Você popula `Status Conciliação.md`:

```markdown
## Conciliações
| Conta/Cartão | Período                      | Status      | Data conciliação | FITIDs lançados |
|--------------|------------------------------|-------------|------------------|-----------------|
| Conta X      | 2026-01-01 a 2026-03-31      | Conciliado  | 2026-04-05       | (importado)     |
```

Se a pessoa não lembrar exatamente, marca como "histórico assumido" e segue.

### Fase 9 — Resumo e próximos passos

```
Modo C concluído. Aprendi:
- N contas e M cartões (em Contas e Cartões.md)
- X regras de estabelecimento automáticas (em Estabelecimentos.md)
- Y candidatos confirmados
- Z decisões importantes (em Preferências.md)
- Variação de fechamento detectada em [N] cartões
- Status de conciliação até [data] em [N] contas

A partir de agora, eu já sei como tu opera. Próximos lançamentos vão sair muito mais rápido.

Quer testar? Diz "lança X reais em Y" e vou aplicar o que aprendi.
```

### Princípios do Modo C

- **Não-destrutivo:** zero criação/edição/deleção no FIN. Só leitura.
- **Não inventa regra:** todo aprendizado vem de evidência no histórico (frequência mínima 3x pra automático, 2x pra candidato).
- **Sem hardcode de defaults:** não tem lista de "categorizações óbvias" embutida. A pessoa que diz o que é regra importante.
- **Mostra trabalho:** sempre exibe o que tá processando ("analisando 512 transações...", "detectando padrões...").
- **Respeita ambíguo:** se um estabelecimento foi categorizado de jeitos diferentes, não vira regra. Fica como "ambíguo" pra ela resolver depois.
- **Pode rodar de novo:** se a pessoa rodar o Modo C 2x, ele atualiza `Estabelecimentos.md` com novos padrões aprendidos (incremental, não substitui).

### Quando Modo C ainda precisa da pessoa intervir

Mesmo sendo automático, em alguns pontos pede confirmação:
1. **Janela de análise** (Fase 2)
2. **Candidatos a regra** (Fase 6)
3. **Marcar decisões importantes** (Fase 7)
4. **Status de conciliação** (Fase 8)

Tudo o resto é automático.

---

## Idempotência

**Rodar onboarding 2x não pode duplicar nada.** Mecanismo:

1. Antes de criar conta: `fin_listar_contas` → checa se já existe pelo nome
2. Antes de criar categoria: `fin_listar_categorias` → checa se já existe pelo nome
3. Antes de criar subcategoria: itera as subs da categoria e checa
4. Pula tudo que já existe
5. No fim, mostra resumo: "Criado X novo, Y já existia"

## Confirmação de credencial (cuidado)

Se você suspeitar que o MCP do FIN pode estar logado com credencial de outra pessoa (ex: você tá ajudando alguém num computador onde já tem outro setup), **avise antes de criar**:

> Antes de eu criar tudo, confirma: o FIN tá logado com a tua credencial mesmo, né? Se for, manda bala. Se não, gera uma chave nova no FIN App e atualiza a config primeiro.

Esse aviso vai como pendência no documento de setup também (seção `## Pendências`).

## Erros comuns que você deve evitar

1. **Criar conta/categoria sem checar se já existe** → sempre `fin_listar_*` antes
2. **Sugerir nomes de bancos** → pergunta aberta, deixa a pessoa dizer
3. **Marretar categorias da maioria** → cada pessoa tem as dela
4. **Esquecer dinheiro vivo** → sempre pergunta no Bloco 1
5. **Não capturar decisões não-óbvias** → toda escolha que foge do default vai pra `Preferências.md`
6. **Tentar criar 3 níveis de categoria** → o FIN só tem 2, sempre achata e explica
7. **Pular o aviso de variação de fechamento** → a pessoa precisa saber que fechamento varia
8. **Lançar transações no onboarding** → onboarding só cria estrutura (contas, cartões, categorias). Lançamentos vêm depois.
