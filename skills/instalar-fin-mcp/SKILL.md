---
name: instalar-fin-mcp
description: >
  Instala e configura o MCP server do FIN App no Claude Code ou Claude Desktop.
  Use quando o plugin financeiro detectar que as tools fin_* não estão
  disponíveis, ou quando a pessoa pedir explicitamente pra configurar o FIN.
  Guia passo a passo: gerar API key no FIN App, montar comando de instalação
  ou editar config manualmente, reiniciar Claude e verificar.
argument-hint: "(sem argumentos)"
allowed-tools: Read, Write, Bash
---

## Quando usar

- O agente principal tentou chamar uma tool `fin_*` e ela não existe
- A pessoa falou "instala o FIN", "configura o MCP", "como conecto o FIN"
- Erro `Missing env: FIN_BASE_URL` ou `unauthorized` ao chamar tool do FIN

## Quando NÃO usar

- O MCP do FIN já tá instalado e funcionando (faça `fin_listar_contas` pra confirmar antes de rodar essa skill)
- A pessoa não tem conta no FIN App ainda — nesse caso, mande criar primeiro em https://fin-app-wine.vercel.app

## Pré-requisitos

Antes de tudo, confirme com a pessoa:

1. **Tem conta no FIN App?** Se não, peça pra criar em https://fin-app-wine.vercel.app e pause aqui.
2. **Tem Node.js 18+ instalado?** Rode `node --version` no terminal pra confirmar. Se não tiver, peça pra instalar antes.

## Fluxo

### Passo 1 — Gerar a API key no FIN App

Instrua a pessoa exatamente assim (não invente passos):

> Você precisa gerar uma API key no FIN App. Faça o seguinte:
>
> 1. Acesse https://fin-app-wine.vercel.app/settings/api-keys
> 2. Faça login se ainda não estiver logada
> 3. Clique em **+ Nova chave**
> 4. Dê um nome descritivo (ex: "Claude - {nome do computador}")
> 5. Clique em **Criar chave**
> 6. **Copie a chave imediatamente** (formato `fin_live_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`)
>
> ⚠️ A chave aparece **uma única vez**. Se perder, tem que revogar e gerar nova.
>
> Quando tiver a chave em mãos, me avisa.

**IMPORTANTE:** quando a pessoa colar a chave na conversa, **avise pra ela não fazer isso** ("nunca cola chave aqui, ela fica gravada no histórico"). Peça pra ela ter a chave por perto mas não colar — você só precisa que ela cole no terminal ou no settings.json. Se ela já colou, siga adiante mas reforce que vai precisar revogar essa chave depois e gerar outra.

### Passo 2 — Detectar plataforma

Pergunte ou detecte: a pessoa tá usando **Claude Code** (terminal) ou **Claude Desktop** (app)?

Pra detectar via shell: `which claude` retorna caminho se Code estiver instalado. No Desktop, a config fica em `~/Library/Application Support/Claude/claude_desktop_config.json` (macOS).

### Passo 3a — Instalação no Claude Code (recomendado)

Monte o comando exato e peça pra pessoa colar no terminal:

```bash
claude mcp add fin-app \
  -s user \
  -e FIN_BASE_URL=https://fin-app-wine.vercel.app \
  -e FIN_API_KEY=fin_live_SUA_CHAVE_AQUI \
  -- npx -y fin-app-mcp
```

Substitua `fin_live_SUA_CHAVE_AQUI` pela chave dela. **NÃO peça pra ela colar a chave aqui na conversa pra você montar o comando.** Mostre o template, peça pra ela substituir no terminal dela mesma.

A flag `-s user` instala globalmente (vale pra qualquer projeto).

### Passo 3b — Instalação no Claude Desktop

A pessoa precisa editar `claude_desktop_config.json` manualmente. Localização:

- **macOS:** `~/Library/Application Support/Claude/claude_desktop_config.json`
- **Linux:** `~/.config/Claude/claude_desktop_config.json`
- **Windows:** `%APPDATA%\Claude\claude_desktop_config.json`

Mostre o JSON pra adicionar:

```json
{
  "mcpServers": {
    "fin-app": {
      "command": "npx",
      "args": ["-y", "fin-app-mcp"],
      "env": {
        "FIN_BASE_URL": "https://fin-app-wine.vercel.app",
        "FIN_API_KEY": "fin_live_SUA_CHAVE_AQUI"
      }
    }
  }
}
```

**Avisos importantes:**
- Se já existirem outros servers em `mcpServers`, **adicione `fin-app` como mais uma entrada**, não substitua o objeto inteiro
- Substitua `fin_live_SUA_CHAVE_AQUI` pela chave real, no arquivo, fora da conversa
- Não cole o JSON com a chave aqui

### Passo 4 — Reiniciar o Claude

Peça pra pessoa fechar **completamente** o Claude (Code ou Desktop) e abrir de novo. A primeira invocação demora alguns segundos porque o `npx -y fin-app-mcp` baixa o pacote do npm na primeira vez.

### Passo 5 — Verificar instalação

**No Claude Code:** peça pra rodar no terminal:

```bash
claude mcp list
```

Procure por `fin-app` na lista. Deve aparecer como conectado.

**No Claude Desktop:** peça pra ela voltar pra conversa e pedir pro Claude fazer um teste simples. Você pode rodar `fin_listar_contas` direto. Se retornar lista (mesmo vazia), tá funcionando.

### Passo 6 — Teste rápido

Chame `fin_listar_contas`. Se retornar OK (lista vazia ou com contas), instalação concluída. Diga:

> Pronto, MCP do FIN tá instalado e funcionando. Quer rodar o onboarding agora? (`/financeiro:onboarding`)

## Troubleshooting

Quando der erro, identifique pelo sintoma e responda:

| Sintoma | Causa | Solução |
|---|---|---|
| `claude mcp list` não mostra `fin-app` | Config não foi salva, ou Claude Code não foi reiniciado | Verifique `~/.claude/settings.json` (ou `claude_desktop_config.json`), reinicie Claude completamente |
| `Missing env: FIN_BASE_URL` | Env var não foi setada na config | Reabra a config e confira que `env` tem `FIN_BASE_URL` E `FIN_API_KEY` |
| `unauthorized` | API key errada, expirada ou revogada | Vá em https://fin-app-wine.vercel.app/settings/api-keys, gere nova chave, atualize a config |
| `auth_lookup_failed` | Problema temporário no Supabase do FIN | Tente de novo em alguns segundos. Se persistir, verifique se o app tá no ar |
| `npx` muito lento na 1ª invocação | Normal — baixando pacote (~16KB + deps) | Aguarde. Próximas invocações são instantâneas. Se >30s consistentemente, `npm cache clean --force` |
| Tools `fin_*` ainda não aparecem após reiniciar | Cache do Claude, ou config em local errado | Confira que o JSON está válido (use `jq < ~/.claude/settings.json`), reinicie de novo |

## Como revogar uma chave (se vazar)

Se a pessoa colou a chave em algum lugar público (incluindo essa conversa), avise:

1. Acesse https://fin-app-wine.vercel.app/settings/api-keys
2. Encontre a chave na lista de "Ativas"
3. Clique em **Revogar** e confirme
4. A revogação é instantânea
5. Gere uma nova chave e atualize a config do Claude

## Segurança

- **NUNCA** peça pra pessoa colar a API key na conversa do Claude
- **NUNCA** grave a API key em arquivo da pasta `Financeiro/` ou em config do plugin
- A API key fica **só** no `settings.json` do Claude Code ou no `claude_desktop_config.json` do Desktop
- Esses arquivos não são versionados nem sincronizados (a menos que a pessoa configure intencionalmente)

## Atualizações futuras

O `npx -y fin-app-mcp` sempre baixa a última versão do pacote npm. Quando a Anthropic ou o time do FIN publicar uma nova versão, ela vai entrar automaticamente na próxima invocação. A pessoa não precisa fazer nada.

Pra forçar uma versão específica, ela pode trocar `fin-app-mcp` por `fin-app-mcp@2.1.0` (ou outra versão) nos `args` da config.

## Após instalação concluída

Diga pra pessoa o que fazer a seguir:

> MCP do FIN instalado. Próximo passo: configurar suas contas e cartões no FIN. Rode `/financeiro:onboarding` ou só me diga "vamos configurar minhas finanças".
