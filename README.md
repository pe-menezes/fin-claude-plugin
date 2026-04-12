# FIN Claude Plugin

Plugin do Claude (Code + Desktop) pra gestão financeira pessoal agnóstica, usando o [FIN App](https://fin-app-wine.vercel.app) como backend via MCP.

## O que faz

- **Onboarding guiado:** entrevista a pessoa em blocos (contas, cartões, perfil de renda, estilo de vida) e cria contas, cartões, categorias e subcategorias no FIN automaticamente.
- **Lançamento avulso:** entende instruções em linguagem natural ("lança 45 no mercado, débito Nubank") e lança no FIN.
- **Lançamento em lote:** processa extrato bancário ou fatura de cartão (OFX, CSV, CNAB 240, texto, PDF) e envia tudo de uma vez, com revisão.
- **Conciliação:** compara o que tá no FIN com o que tá no extrato e resolve divergências.
- **Aprende com o uso:** memoriza estabelecimentos, padrões de categorização, regras de split, fechamento real de cartão, em arquivos `.md` que a pessoa pode editar.

## Como funciona

O plugin escreve tudo o que aprende em arquivos markdown padrão dentro de uma pasta `Financeiro/` que **a pessoa escolhe onde fica**:

- Dentro de um vault Obsidian (se a pessoa usar)
- Numa pasta normal do filesystem (default: `~/Financeiro/`)
- Qualquer caminho custom

Os arquivos são markdown puro (sem wikilinks, sem tags de Obsidian), então funcionam em qualquer editor.

## Slash commands

- `/financeiro:onboarding` — setup do FIN do zero ou via documento
- `/financeiro:lancar` — transação avulsa
- `/financeiro:extrato` — processar extrato bancário em lote
- `/financeiro:fatura` — processar fatura de cartão em lote
- `/financeiro:conciliar` — conciliação
- `/financeiro:instalar-fin-mcp` — pré-onboarding (instalar e configurar o MCP do FIN, se necessário)

## Instalação

### Pré-requisitos

1. Conta no [FIN App](https://fin-app-wine.vercel.app)
2. Claude Code ou Claude Desktop instalado
3. Node.js 18+ (pro `npx -y fin-app-mcp`)

### Adicionar o marketplace e instalar

```
/plugin marketplace add github.com/pe-menezes/fin-claude-plugin
/plugin install financeiro@fin-claude-plugin-marketplace
```

Na primeira execução, o plugin vai:
1. Perguntar onde guardar a pasta `Financeiro/`
2. Verificar se o MCP do FIN tá instalado (e ajudar a instalar se não)
3. Oferecer rodar o onboarding

## Documentação

Spec completa do plugin: ver `docs/spec.md` (em breve) ou o repo do projeto.

## License

MIT
