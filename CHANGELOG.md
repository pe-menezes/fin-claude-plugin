# Changelog

Todas as mudanças relevantes deste plugin ficam documentadas neste arquivo.

O formato é baseado em [Keep a Changelog](https://keepachangelog.com/),
e o projeto segue [Semantic Versioning](https://semver.org/).

## [Unreleased]

### Adicionado
- **Suporte a USD / câmbio (compatível com FIN MCP v2.2.0+):**
  - Agente principal: roteamento de 5 padrões novos (vender/comprar dólar, ajustar saldo USD, gastar em USD, ajustar saldo BRL), regras de negócio do USD, leitura obrigatória da seção 10 do `fin://docs/guia` ao operar USD pela primeira vez na sessão
  - `skills/lancar`: novo "Passo 4.5 — Tratamento especial: USD, câmbio e ajuste de saldo" com 3 casos detalhados (Câmbio, Ajuste manual USD/BRL, Gasto em USD via ajuste de saldo). Tabela de exemplos ampliada com 5 frases novas. Notação USD adicionada
  - `skills/onboarding`: Bloco 1 ganha pergunta sobre conta em dólar e captura `currency`. Aviso sobre limitações do v0 USD. Modo C reconhece contas USD e exclui transações de "Câmbio" do agrupamento de estabelecimentos
  - 6 erros novos no checklist do agente principal e da skill `lancar`
  - Spec do plugin (no vault) atualizada com seção 8.3 sobre suporte USD

### Em desenvolvimento
- Estrutura inicial do plugin (manifests, esqueleto de skills, agente, templates)
- Spec v1 finalizada

## [0.1.0] - TBD

### Added
- Plugin scaffold inicial
- `.claude-plugin/plugin.json` e `marketplace.json`
- README e LICENSE
- Estrutura de pastas: `agents/`, `skills/`, `templates/`
