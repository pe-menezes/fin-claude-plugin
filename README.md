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

1. **Conta no [FIN App](https://fin-app-wine.vercel.app)** (cria em 1 minuto)
2. **API key do FIN** gerada em https://fin-app-wine.vercel.app/settings/api-keys (formato `fin_live_xxx...`)
3. **Claude Code** ou **Claude Desktop** instalado
4. **Node.js 18+** (necessário pro `npx -y fin-app-mcp` que o plugin instala automaticamente)

### Instalação em 3 passos

A partir da v0.2.0, o plugin **instala o MCP do FIN automaticamente** via `.mcp.json` declarado no próprio plugin. Tu não precisa editar `claude_desktop_config.json` nem rodar `claude mcp add`. Só copia, cola, fornece a chave.

**Passo 1 — Adicionar o marketplace:**
```
/plugin marketplace add github.com/pe-menezes/fin-claude-plugin
```

**Passo 2 — Instalar o plugin:**
```
/plugin install financeiro@fin-claude-plugin-marketplace
```

**Passo 3 — Colar a API key quando perguntado.**

No momento da instalação, o Claude vai pedir o valor de `fin_api_key` (configurado como `sensitive`, então fica guardado no keychain do sistema, não no histórico da conversa). Cola a chave no formato `fin_live_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx` e confirma.

**Pronto.** O MCP `fin-app` é registrado automaticamente. Reinicia o Claude se ele pedir, e dá `/financeiro:onboarding`.

### Primeira execução

Na primeira vez, o plugin vai:
1. Perguntar **onde guardar a pasta `Financeiro/`** (vault Obsidian, pasta normal, ou caminho custom)
2. **Verificar se o MCP do FIN tá funcionando** (chamada `fin_listar_contas`)
3. **Detectar o estado do FIN** e disparar o modo certo do onboarding:
   - FIN vazio → Modo A (questionário guiado em 6 blocos)
   - FIN com dados → Modo C (aprendizado retroativo do que já existe)
   - Tu tem documento de setup → Modo B (importação)

### Atualizar a chave depois (revogou, perdeu, vai mudar)

Reinstala o plugin pra triggar o prompt de userConfig de novo:
```
/plugin reinstall financeiro@fin-claude-plugin-marketplace
```

Ou desinstala e reinstala:
```
/plugin uninstall financeiro
/plugin install financeiro@fin-claude-plugin-marketplace
```

### Se a instalação automática falhar

Em caso de bug (formato `.mcp.json` não suportado por alguma versão do Claude, ou prompt de `userConfig` não apareceu), tem a skill de fallback:
```
/financeiro:instalar-fin-mcp
```

Ela guia passo a passo a configuração manual via `claude mcp add` (Code) ou edição do `claude_desktop_config.json` (Desktop).

## Documentação

Spec completa do plugin: ver `docs/spec.md` (em breve) ou o repo do projeto.

## License

MIT
