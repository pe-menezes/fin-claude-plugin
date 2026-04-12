# Changelog

Todas as mudanças relevantes deste plugin ficam documentadas neste arquivo.

O formato é baseado em [Keep a Changelog](https://keepachangelog.com/),
e o projeto segue [Semantic Versioning](https://semver.org/).

## [Unreleased]

## [0.2.0] - TBD

### Adicionado
- **Auto-instalação do MCP do FIN App** (zero edição manual de config):
  - `.mcp.json` na raiz do plugin declarando `fin-app` via `npx -y fin-app-mcp`, com env vars `FIN_BASE_URL` (fixa) e `FIN_API_KEY` (referenciando `${user_config.fin_api_key}`)
  - `userConfig.fin_api_key` no `plugin.json` marcado como `sensitive: true` (chave guardada no keychain do sistema, não no histórico da conversa)
  - Resultado: instalação em **3 passos** (`marketplace add` → `plugin install` → colar API key quando perguntado)
  - Nada de editar `claude_desktop_config.json`. Nada de `claude mcp add`. Tudo automático.
- **Suporte a USD / câmbio (compatível com FIN MCP v2.2.0+):**
  - Agente principal: roteamento de 5 padrões novos (vender/comprar dólar, ajustar saldo USD, gastar em USD, ajustar saldo BRL), regras de negócio do USD, leitura obrigatória da seção 10 do `fin://docs/guia` ao operar USD pela primeira vez na sessão
  - `skills/lancar`: novo "Passo 4.5 — Tratamento especial: USD, câmbio e ajuste de saldo" com 3 casos detalhados (Câmbio, Ajuste manual USD/BRL, Gasto em USD via ajuste de saldo). Tabela de exemplos ampliada com 5 frases novas. Notação USD adicionada
  - `skills/onboarding`: Bloco 1 ganha pergunta sobre conta em dólar e captura `currency`. Aviso sobre limitações do v0 USD. Modo C reconhece contas USD e exclui transações de "Câmbio" do agrupamento de estabelecimentos
  - 6 erros novos no checklist do agente principal e da skill `lancar`
  - Spec do plugin (no vault) atualizada com seção 8.3 sobre suporte USD

### Mudado
- `skills/instalar-fin-mcp` virou **fallback manual**: só roda se a auto-instalação do `.mcp.json` falhar, ou se a pessoa quiser revogar/atualizar a chave. Ganhou seção explicando o método novo no início.
- `agents/financeiro.md` atualizado: na verificação inicial do MCP, espera que o método automático tenha funcionado e só dispara fallback se detectar problema.
- README do repo: nova seção "Instalação em 3 passos" + "Atualizar a chave depois" + "Se a instalação automática falhar".

### Bumps
- `plugin.json`: 0.1.0 → 0.2.0
- `marketplace.json`: 0.1.0 → 0.2.0

## [0.1.0] - TBD

### Added
- Plugin scaffold inicial
- `.claude-plugin/plugin.json` e `marketplace.json`
- README e LICENSE
- Estrutura de pastas: `agents/`, `skills/`, `templates/`
