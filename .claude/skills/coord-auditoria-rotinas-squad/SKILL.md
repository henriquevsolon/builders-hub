---
name: coord-auditoria-rotinas-squad
description: Auditoria semanal das rotinas internas e dos invites de cliente do squad. Cruza Google Calendar + telemetria da `/geral-sync-call-transcripts` + historico de reunioes pra detectar (1) invites fora do padrao de nomenclatura, (2) invites com cliente sem o email da unidade — vao gerar transcript perdido, (3) tipos novos de reuniao que apareceram, (4) rotinas que sumiram (WBR, Sprint Review, Comite). Apos analise automatica, PERGUNTA ao coordenador pra confirmar se houve/tera alteracoes esperadas. Gera relatorio em `squads/{squad}/auditorias/YYYY-MM-DD-auditoria-semanal.md` pra historico. Use sempre que o usuario disser "auditar rotinas do squad", "verificar nomenclatura dos invites", "checar drift do squad", "auditoria semanal", "rodar coord-auditoria-rotinas-squad", OU quando rodar via `/schedule` (cron sugerido: toda sexta as 19h). Tambem use antes do Comite de Operacoes pra ter dados frescos da semana.
area: coord
author: solonhv-hub
version: 2.0.0
---

# /coord-auditoria-rotinas-squad — Auditoria semanal das rotinas e invites do squad

Voce vai auditar a semana do squad olhando 3 fontes (Google Calendar + state file da sync + historico-reunioes.md) e cruzar pra detectar drift na nomenclatura, cobertura e ritual de reunioes. Apos analise, voce **pergunta** ao coordenador pra calibrar interpretacao, depois gera relatorio gravado pra historico.

## Arquitetura hibrida (v2.0)

Como a sync, essa skill roda **tanto local quanto cloud (via `/schedule`)** com dual-write:

- **Le state file no Drive primeiro** (pasta `Builders Hub - State`, ID `1IKY3yRo84luz3Sn5p1HefXsC8n95T7C3`) — file com prefixo `sync-call-state-` mais recente. Fallback local: `.sync-call-state.json` na raiz do hub.
- **Le historico no Drive** — subpastas `historico-reunioes/` (squad) e `historico-checkins/` (cliente) contem 1 arquivo por entrada.
- **Le Calendar** via `mcp__google-calendar__list-events` (Calendar MCP autenticado com `henrique.solon@v4company.com`).
- **Escreve o relatorio em ambos os lugares**: Drive (subpasta `auditorias/` dentro de SQUAD PHOENIX) **E** local (`squads/{squad}/auditorias/`, so se rodando local).
- **Registra no `STATE-BACKUP-LOG.md`** (raiz do hub, NAO gitignored) com referencia ao file ID do Drive — esse log e o ponto de centralizacao no git pra rastreabilidade.

## Pre-requisitos

1. **Google Calendar MCP autenticado** com conta do coordenador (testar via `mcp__google-calendar__manage-accounts action=list`)
2. **Squad com infraestrutura completa**:
   - `squads/{squad}/links.md` (Drive folder configurado)
   - `squads/{squad}/historico-reunioes.md` (mesmo que vazio)
   - `squads/{squad}/auditorias/` (pasta — criar se faltar)
3. **`.sync-call-state.json` na raiz do hub** (gerado pela `/geral-sync-call-transcripts`). Se nao existir, skill funciona em modo degradado (so com Calendar e historico) — avisa o coordenador.
4. **Squad com README.md** listando membros do squad (a skill usa pra mapear duplas e papeis).

## Fluxo

### Passo 1 — Definir janela e squad

- **Janela default:** ultimos 7 dias (segunda passada 00:00 ate sexta atual 23:59, no timezone America/Sao_Paulo)
- Pode aceitar argumento `janela=14d` ou `janela=2026-05-08..2026-05-15`
- Se houver mais de 1 squad no hub, pergunte qual auditar. Se houver so 1, usa direto.

Use `mcp__google-calendar__get-current-time` pra ancorar a data atual.

### Passo 2 — Coletar dados de 3 fontes (em paralelo)

**Fonte A — Google Calendar**
```
mcp__google-calendar__list-events:
  calendarId: 'primary'
  timeMin: {janela_start}
  timeMax: {janela_end}
  fields: ['id', 'summary', 'start', 'end', 'attendees', 'organizer', 'description', 'status']
```

Filtre por eventos onde:
- O organizador OU pelo menos 1 participante e membro do squad (cruzar com README.md do squad)
- Status != "cancelled"

Pra cada evento, extraia:
- titulo, data, organizador, lista de participantes
- **email da unidade presente nos participantes?** (sim/nao)
- categoria preliminar (squad-level / cliente / privado / desconhecido) com base nos regex da `/geral-sync-call-transcripts`

**Fonte B — `.sync-call-state.json` (telemetria)**
- Le `telemetry.patterns_seen`, `telemetry.first_seen`, `telemetry.last_seen` (campos da v1.2 da sync)
- Se a v1.2 ainda nao tiver rodado, esse bloco vira `[]` — segue em frente sem ele.

**Fonte C — `squads/{squad}/historico-reunioes.md`**
- Parsea entradas `## YYYY-MM-DD - {Tipo}` da janela
- Conta ocorrencias por tipo

### Passo 3 — Analisar em 4 dimensoes

**Dimensao 1: Nomenclatura — invites fora do padrao**

Pra cada evento do Calendar, classifique:
- ✅ **Padrao** — bate com regex de squad-level OU cliente conhecido OU privado
- ⚠️ **Variante** — bate parcialmente mas com formato nao-canonico (ex: `[CHECK-IN]` em vez de `Check-in |`)
- ❓ **Desconhecido** — nao bate com nada

Liste os ⚠️ e ❓ com sugestao de correcao quando obvia.

**Dimensao 2: Cobertura — invites de cliente sem email da unidade**

Pra cada invite cuja categoria seja **cliente** (do Calendar), verifique se o email da unidade esta na lista de participantes. **O email da unidade vem do `links.md` do squad** ou de variavel `UNIT_EMAIL` no `.env` da raiz do hub. Se nao estiver configurado, perguntar uma vez ao usuario (e gravar pra proximas rodadas).

Liste por **GP responsavel** (organizador do invite). Dimensao critica porque sem o email da unidade, o transcript do Gemini nao vai aparecer pra `/geral-sync-call-transcripts`.

**Dimensao 3: Tipos novos**

Cruze padroes de titulo desta semana contra `telemetry.first_seen` (state file). Se um padrao apareceu pela primeira vez na janela:
- Liste como 🆕 com contagem
- Sugira: "Adicionar na regex de squad-level?" / "Adicionar na regex de cliente?" / "Marcar como privado?"

**Dimensao 4: Drift — rotinas que sumiram**

Pra cada **rotina esperada** do squad, conte ocorrencias na janela e compare com baseline:

| Rotina | Esperado | Se falhar |
|---|---|---|
| **WBR** (por dupla) | 1x/semana por dupla | warning amarelo (semana 1) → vermelho (semana 2+) |
| **Sprint Review** (por dupla) | 1x/semana por dupla | warning amarelo |
| **Sprint Planning** (por dupla) | 1x/semana por dupla | warning amarelo |
| **Comite de Operacoes** | 1x/semana | warning vermelho se faltar |

A skill **nao define drift estaticamente** — apenas mostra o que viu vs o que esperava, e **pergunta ao coordenador no Passo 4** se a falta foi planejada (mudanca de cadencia) ou drift real.

### Passo 4 — Validacao interativa com o coordenador

Apos a analise automatica, **PERGUNTA** ao coordenador (formato corrido, nao questionario):

> "Antes de gerar o relatorio final, queria confirmar:
>
> 1. **{tipo novo X apareceu Y vezes}** — e mesmo tipo novo pra ficar (e adicionar na regex), ou pontual?
> 2. **{rotina Z nao rolou essa semana}** — foi planejado (ferias, feriado, agenda cheia) ou drift real?
> 3. **{invite W sem email da unidade}** — esse era pra ter o email ou e privado mesmo?
> 4. Algo no calendario da semana **que vem** que voce sabe que vai mudar (rotina pausando, nova rotina entrando)?"

Aguarda resposta. Cada resposta calibra:
- Tipos confirmados como novos → atualiza state file (anotacao pra propor edit na sync depois)
- Drift confirmado como drift → vai pro relatorio como ⚠️ acionavel
- Invites privados sem unidade → anotados como "intencional, nao alertar de novo" (registrar IDs no state file)
- Mudancas anunciadas → vao pra secao "Mudancas esperadas" do relatorio

### Passo 5 — Gerar relatorio (dual-write Drive + local)

**Drive (sempre):** garante subpasta `auditorias/` dentro da pasta do squad no Drive (use `search_files` + `create_file` mimeType folder). Cria arquivo `{YYYY-MM-DD}-auditoria-semanal.md` la com `create_file` (contentMimeType `text/markdown` ou `text/plain`).

**Local (so se rodando local):** salva em `squads/{squad}/auditorias/YYYY-MM-DD-auditoria-semanal.md`.

**Git (sempre):** adiciona linha em `STATE-BACKUP-LOG.md` na raiz do hub com a referencia:
```
| {ISO_ts} | /coord-auditoria-rotinas-squad | auditoria | {squad}-YYYY-MM-DD-auditoria-semanal.md | {file_id_drive} | {n_invites}/{n_padrao}/{n_warnings} |
```

Conteudo do relatorio (formato identico nos dois lugares):

```markdown
# Auditoria Rotinas — Squad {Nome} — {data}

## Janela
{start} a {end} (timezone {tz})

## Resumo executivo
- **Total de invites na janela:** {N}
- ✅ {n_padrao} no padrao
- ⚠️ {n_variante} variantes (corrigir)
- ❓ {n_desconhecido} desconhecidos
- 🚨 {n_sem_unidade} invites de cliente **sem email da unidade** (vao perder transcript)
- 🆕 {n_novos} tipos novos
- 👻 {n_drift} rotinas que sumiram

## Detalhes

### 1. Nomenclatura
**Variantes a corrigir:**
- "{titulo}" — {data} — sugestao: "{titulo_canonico}"

**Desconhecidos (revisar manualmente):**
- "{titulo}" — {data} — {organizador}

### 2. Cobertura (invites sem email da unidade)
Por GP:
- **{GP nome}:**
  - "{titulo}" — {data} — clima: {cliente identificado | nao identificado}

### 3. Tipos novos
- "{titulo padrao}" apareceu {N}x — confirmado: {SIM/NAO} — acao: {atualizar regex / monitorar}

### 4. Drift
- **{Rotina}:** {ocorrencias} de {esperado} esperadas — confirmado: {planejado / drift real}

## Validacao com o coordenador
{Transcripcao das respostas do coordenador no Passo 4}

## Mudancas esperadas (input do coordenador)
{Coisas que vao mudar na proxima semana — info pra alimentar o monitoramento futuro}

## Sugestoes acionaveis pro Comite de Operacoes
- {bullet point pra discutir com squad — ex: "Lembrar Igor e Iza de incluir email da unidade em todos invites com cliente"}
- {bullet point — ex: "Avaliar se rotina X deve voltar"}

## Anexos
- State file usado: `.sync-call-state.json` (versao {ver}, last_run: {ts})
- Eventos do Calendar lidos: {N}
- Membros do squad considerados: {lista}
```

### Passo 6 — Atualizar state file (se houver tipos novos confirmados)

Se o coordenador confirmou tipo novo "permanente":
- Adicione em `telemetry.confirmed_types` no `.sync-call-state.json`
- Sugira ao usuario rodar `/criador-de-skills edit geral-sync-call-transcripts` pra adicionar o padrao na regex da skill de sync (mostre o diff sugerido — nao aplique sem confirmacao)

### Passo 7 — Devolver o relatorio

Mostre ao usuario:
- Path do relatorio salvo
- Resumo executivo (5-6 bullets)
- Itens acionaveis principais (top 3)

Se tem itens criticos (🚨 invites sem unidade) — **mencione o GP responsavel** pra ele cobrar.

## Como agendar (`/schedule`)

Pra rodar automatico toda sexta as 19h (timezone do coordenador):

```
/schedule create
  name: auditoria-rotinas-{squad}-semanal
  cron: "0 19 * * 5"  # toda sexta 19h
  command: /coord-auditoria-rotinas-squad squad={squad}
```

Quando o `/schedule` disparar, a skill roda — porem o Passo 4 (validacao interativa) so funciona se voce abrir o Claude Code e ver as perguntas. Recomendacao: a skill grava as perguntas em `squads/{squad}/auditorias/{data}-pendente.md` e quando voce abrir o Claude Code de novo, a skill detecta as perguntas pendentes e te apresenta antes de gerar o relatorio final.

## Regras

- **Nao auto-corrija invites**. Auditoria so observa e sugere — nunca mexe no Calendar.
- **Privacidade:** se um invite for marcado privado pelo coordenador, registre IDs no state file e nunca mais alerte sobre ele.
- **Idempotente:** rodar 2x na mesma semana gera 2 relatorios separados (timestamp diferentes), nao sobrescreve.
- **Nao quebre se falta fonte:** modo degradado quando state file ou Calendar nao estao disponiveis — relatar o que conseguir e avisar o que faltou.
- **Email da unidade nao hardcoded:** sempre buscar via `links.md` do squad ou `.env` raiz. Se nao tiver, perguntar uma vez e gravar.
- **Foco em rotinas do squad:** auditoria nao deve "puxar" invites de outros squads — usar README.md do squad pra delimitar quem conta como membro.

## Skills complementares

- **`/geral-sync-call-transcripts`** — fornece a telemetria. Idealmente roda ANTES dessa auditoria (ex: cron quinta 23h, auditoria sexta 19h) pra state file estar fresco.
- **`/criador-de-skills`** — pra editar regex de `/geral-sync-call-transcripts` quando aparecer tipo novo permanente.

## Exemplo de relatorio (simulado)

```
✅ Auditoria semanal salva em squads/squad-phoenix/auditorias/2026-05-15-auditoria-semanal.md

Resumo:
  • 14 invites na semana
  • ✅ 9 no padrao
  • ⚠️ 3 variantes (Check-in com formatos diferentes)
  • 🚨 2 invites com cliente SEM email da unidade — Igor responsavel por 2
  • 🆕 1 tipo novo: "WBR" (3x esta semana) — confirmado, regex sera atualizada
  • 👻 Sprint Review Iza/Julio nao aconteceu — coordenador confirmou: agenda cheia, volta semana que vem

Top 3 acionaveis pro Comite:
  1. Igor: cobrar inclusao do email v4souzasantos@gmail.com nos invites com cliente
  2. Padronizar formato de Check-in pro time (mostrar template no Comite)
  3. Atualizar regex da /geral-sync-call-transcripts pra incluir 'WBR' (proposta pronta)
```
