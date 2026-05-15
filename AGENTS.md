# Builders Hub

Este repositorio e o hub open-source de skills de IA da V4. Funciona como base de trabalho pessoal + biblioteca compartilhada de skills.

## Como funciona

- `.claude/skills/` e `.agents/skills/` — skills disponiveis (espelhadas pra funcionar no Claude Code e Anti-Gravity)
- `squads/` e `bases/` — Knowledge Bases pessoais do usuario (gitignored, ficam locais)
  - **Padrao obrigatorio:** `squads/{squad}/clientes/{cliente}/`. Toda KB de cliente vive dentro de um squad. Cliente solto, fora de squad, NAO existe — `/novo-cliente` recusa criar.
  - `squads/{squad}/` — cada squad tem `README.md` com membros, `CLAUDE.md` com contexto e `docs/` com acordos do time. Crie squad com `/novo-squad` antes do primeiro cliente.
  - `squads/{squad}/clientes/{cliente}/` — cada cliente tem `calls/` (transcripts brutos), `checkins/` (pautas, ensaios e reviews), `docs/`, `campanhas/`, `mission-control/` (estado vivo), `links.md` (recursos recorrentes — NotebookLM, Drive, site, etc) e `CLAUDE.md`/`AGENTS.md` proprios.
  - `bases/{projeto}/` — KBs de qualquer outra area (docs, dados, referencias) que nao sao cliente.
- Cada KB pode ter um CLAUDE.md/AGENTS.md proprio (gerado por `/contexto`). Leia ele primeiro quando trabalhar naquele contexto.
- `REGISTRY.md` — catalogo auto-gerado de todas as skills compartilhadas, agrupado por papel

## Skills de setup/fluxo (base)

- `/onboarding` — Guia a primeira configuracao. Valida git/gh 100% e depois instala dependencias. Roda sempre que algo do setup quebrar.
- `/sync-hub` — Puxa as skills compartilhadas mais recentes do repo remoto.
- `/compartilhar-skill` — Empacota uma skill local e abre PR pro hub publico.
- `/criador-de-skills` — Cria skill nova com prefixo de papel obrigatorio.
- `/contexto` — Le tudo numa KB, gera CLAUDE.md/AGENTS.md e atualiza Mission Control quando for cliente.
- `/novo-squad` — Cria pasta de squad com README de membros (rode antes do primeiro cliente).
- `/novo-cliente` · `/novo-projeto` — Cria pasta de KB com estrutura padrao. `/novo-cliente` agora pede o squad e coleta links uteis (NotebookLM, Drive, site, outros) que ficam em `links.md`.
- `/geral-brainstormar-sobre-minha-funcao` — Descobre onde IA agrega mais valor no dia a dia.
- `/geral-sabatina` — Stress-test de planos.
- `/geral-frontend-design` — Gera interfaces frontend de alta qualidade (pra skills que produzem UI).

## Skills compartilhadas (hub)

Toda skill compartilhada pelo time segue o padrao `{prefixo}-{nome}`. Dois tipos de prefixo:

- **Papeis** (skills que entregam trabalho final, agrupadas por quem usa): `geral-*` · `gt-*` · `designer-*` · `copy-*` · `account-*` · `coord-*`
- **Fontes** (skills que puxam dados de integracoes externas, reutilizaveis por outras): `v4mos-*` · `google-*` · `ga4-*` · `meta-*` · `hubspot-*` · `kommo-*` · `shopify-*` · `tray-*`

- Dica: pra ver so as skills do seu papel, digita `/gt`, `/designer`, `/account`, `/copy`, `/coord` ou `/geral` no Claude Code — o autocomplete filtra pelo prefixo. `geral-*` sao skills que qualquer papel usa.

Consulte [REGISTRY.md](./REGISTRY.md) pra ver tudo que o time ja compartilhou. Pra contribuir veja [CONTRIBUTING.md](./CONTRIBUTING.md).

## Regras

- Sempre responda em portugues brasileiro.
- Quando o usuario pedir pra trabalhar com um cliente, entre em `squads/{squad}/clientes/{cliente}/` (caminho obrigatorio — cliente sempre dentro de squad). Pra projeto/area, use `bases/{projeto}/`.
- Nao invente dados. Se nao tem a informacao na KB, diga que nao tem.
- Quando o usuario fizer algo complexo, processual ou que ficou bom, sugira: "Isso ficou bom. Quer transformar em skill pra reutilizar? Roda /criador-de-skills. Quando estiver redonda, roda /compartilhar-skill pra o time usar tambem".
- **Duplo-write obrigatorio**: toda skill criada/editada deve existir identica em `.claude/skills/{nome}/` E `.agents/skills/{nome}/`. `/criador-de-skills` faz isso automaticamente; se voce editar manualmente, espelhe nos dois. `/sync-hub` tambem re-espelha apos pull.
- **Prefixo obrigatorio** em skills contribuidas: `{prefixo}-{nome}`. Prefixo pode ser de papel (geral/gt/designer/copy/account/coord) ou de fonte (v4mos/google/ga4/meta/hubspot/kommo/shopify/tray). Skills de base (onboarding, contexto, sync-hub, criador-de-skills, compartilhar-skill, novo-squad, novo-cliente, novo-projeto) sao excecao e ficam sem prefixo.
- Nunca commitar arquivos de `squads/` ou `bases/` — sao pessoais, ficam no `.gitignore` (so os templates `_template-*` sobem).
- Nunca editar `REGISTRY.md` a mao — e auto-gerado pelo script `scripts/build-registry.py` e pela GitHub Action.
- Se o fluxo git/gh quebrar em qualquer skill (sync, compartilhar, push), oriente rodar `/onboarding` de novo — os checks de setup sao a primeira coisa que ele faz.

## Perfil do Usuario

- **Nome:** Henrique Solon
- **Funcao:** Coordenador de PE&G na V4
- **Tempo na funcao:** Quase 4 anos; antes atuou como Cientista do Marketing na V4
- **Carteira e estrutura:** Atende cerca de 20 a 25 clientes por mes de assessoria, alem de clientes one-time, principalmente de Estruturacao Estrategica; lidera um squad com 5 pessoas, composto por 2 duplas de Gestor de Projetos + Gestor de Trafego e 1 Designer
- **Focos principais:** Retencao como frente principal; monetizacao como frente secundaria, com apoio do Account Manager / Gestor de Projetos
- **Contexto da carteira:** Maior parte da base em clientes Tiny e Small, o que aumenta a dificuldade de controlar churn e expandir receita
- **Ferramentas e fontes de contexto:** WhatsApp, eKyte, Google Calendar, Google Sheets, Health Score, Sprint Growth, planilha de Gestao do Squad, contas de Meta Ads e Google Ads, CRM dos clientes quando houver acesso
- **Workflows principais:** Resumo dos grupos de WhatsApp; auditoria de execucao no eKyte; radar de risco relacional e de performance dos clientes; preparacao da pauta/ata do Comite de Operacoes; preparacao de 1:1 com colaboradores
- **Principais dores:** Informacao descentralizada; excesso de reunioes; baixa confiabilidade do eKyte como fonte principal de execucao; dificuldade de acompanhar resultados e sinais de risco de forma preditiva; pouco tempo para preparar com qualidade Comites e 1:1
- **Prioridades IA:** Criar um resumo recorrente dos grupos de WhatsApp; montar um preparador de pauta/ata do Comite de Operacoes; estruturar um briefing melhor de 1:1; evoluir depois para auditoria de eKyte e radar semanal de risco da carteira
