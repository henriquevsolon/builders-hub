---
name: geral-sync-call-transcripts
description: Puxa transcripts do Gemini Notes (Google Meet) do Drive da unidade e organiza automaticamente — calls com cliente vao pras pastas dos clientes (local em `calls/` + copia no Drive do cliente + entrada em `mission-control/historico-checkins.md`); reunioes internas do squad (Sprint Planning, Sprint Review, Comite de Operacoes, Daily, Retro) vao pras pastas do squad; 1:1s e materiais marcados como privados sao explicitamente IGNORADOS. Use sempre que o usuario disser "sincronizar transcripts", "puxar anotacoes do Gemini", "atualizar calls no Drive", "trazer transcricao da call", "organizar transcripts da semana", "rodar /geral-sync-call-transcripts", ou apos uma rodada de calls (com cliente OU internas). Tambem use quando o usuario terminou uma reuniao e quer trazer o transcript pra KB sem trabalho manual. Cobre fuzzy matching de cliente, varios padroes de nomenclatura, transcripts grandes (>token limit), duplicatas, copia entre Drives, state-file de idempotencia, paginacao e roteamento entre cliente/squad/privado.
area: geral
author: solonhv-hub
version: 2.0.0
---

# /geral-sync-call-transcripts — Sincroniza transcripts de calls (Gemini Notes → KB de cliente OU squad)

Voce vai puxar **automaticamente** os transcripts do Gemini Notes que estao no Drive da unidade (compartilhados via invites do Google Calendar que incluem o email da unidade como participante), classificar cada um por tipo (cliente / squad / privado / nao identificado) e organizar nos lugares certos.

## Arquitetura hibrida: local + Drive espelhado + git centralizando referencias

Esta skill foi desenhada pra rodar **tanto local (no Mac do coordenador) quanto remoto (via `/schedule` cloud)** sem perder consistencia. O modelo e dual-write:

| Camada | Papel | Acesso local | Acesso cloud (CCR) |
|---|---|---|---|
| **Local** (gitignored) | Workflow do dia-a-dia do coordenador | ✅ leitura/escrita | ❌ nao tem |
| **Drive da unidade** | Fonte de verdade compartilhada + espelho automatico | ✅ via MCP | ✅ via MCP |
| **Git (publico)** | Skills + `STATE-BACKUP-LOG.md` (so referencias, sem dados privados) | ✅ tudo | ✅ tudo (via clone) |

**Regras invariantes:**
1. **Ler do Drive primeiro** (state, historico, indices). Se nao existir no Drive, fallback pra local. Isso garante consistencia quando cloud e local rodam em janelas diferentes.
2. **Dual-write em cada operacao:** escreve simultaneamente em local (se rodando local) **E** Drive. Roda remoto = so Drive.
3. **Append-only no Drive:** cada entrada de historico vira **arquivo separado** (ex: `historico-reunioes/2026-05-15-sprint-review-iza-julio.md`). State file e versionado por timestamp (`sync-call-state-{ts}.json`). Drive nao tem `update_file` MCP — append-only e a forma robusta.
4. **Centralizacao no git:** cada operacao adiciona uma linha em `STATE-BACKUP-LOG.md` na raiz do hub (NAO gitignored) com timestamp, tipo, e file IDs do Drive. Isso da rastreabilidade no git sem expor conteudo sensivel.

**Drive folders criticos** (memorizar):
- `Builders Hub - State` (ID `1IKY3yRo84luz3Sn5p1HefXsC8n95T7C3`) — state files versionados, hub index
- Cada cliente: `historico-checkins/` (subpasta no Drive do cliente)
- Cada squad: `historico-reunioes/` e `auditorias/` (subpastas no Drive do squad)

## Roteamento por tipo de call

| Tipo | Sinais | Destino local | Destino Drive |
|---|---|---|---|
| **Privado** | Title contem `1:1`, `1on1`, `1 a 1`, `One on One`, `Feedback -` | **SKIP** (privacidade — fora do escopo da skill) | nao toca |
| **Squad-level** | Title contem `Sprint Planning`, `Sprint Review`, `Comite de Operacoes`, `Comitê de Operações`, `Daily`, `Retrospectiva`, `Retro `, `Reuniao do Squad` | `squads/{squad}/calls/` + entrada em `squads/{squad}/historico-reunioes.md` | Copia em `{Drive squad}/Reunioes Internas/` |
| **Cliente** | Title bate (fuzzy >= 0.85) com um cliente em `squads/*/clientes/*/` | `squads/{squad}/clientes/{cliente}/calls/` + entrada em `mission-control/historico-checkins.md` | Copia na pasta-raiz do Drive do cliente |
| **Nao identificado** | Nenhuma das condicoes acima | Lista no relatorio final pra revisao manual | nao toca |

## Pre-requisitos (validar antes de rodar)

1. **Drive MCP autenticado** com a conta de e-mail da unidade
2. **Builders Hub** com pelo menos um cliente OU squad em `squads/`
3. **Cliente:** `links.md` aponta pra Drive folder em formato `https://drive.google.com/drive/folders/{FOLDER_ID}`
4. **Squad:** `squads/{squad}/links.md` aponta pra Drive folder (a skill cria a estrutura se faltar — veja Passo 3)
5. **Regra organizacional:** invites de call DE CLIENTE OU REUNIAO DE SQUAD incluem o email da unidade. **1:1s NAO devem incluir** (preserva privacidade).

Se faltar Drive MCP, pare e oriente: "Drive MCP nao autenticado. Roda `/mcp` no Claude Code e seleciona 'claude.ai Google Drive'."

## Fluxo

### Passo 1 — Janela de tempo e state file

Pergunte (ou aceite via argumento):
> "Qual janela de tempo? (Enter = ultimas 24h | digite `48h`, `7d`, ou data ISO `2026-05-14T00:00:00Z`)"

Converta pra RFC 3339 UTC.

**Carregue o state file — Drive primeiro, local como fallback:**

1. Procure no Drive (pasta `Builders Hub - State`, ID `1IKY3yRo84luz3Sn5p1HefXsC8n95T7C3`) por arquivos com prefixo `sync-call-state-`. Pegue o de **modifiedTime mais recente** — esse e o state atual.
   ```
   search_files: parentId = '1IKY3yRo84luz3Sn5p1HefXsC8n95T7C3' and title contains 'sync-call-state-' and mimeType = 'application/vnd.google-apps.document'
   ```
2. Se achou, leia o conteudo via `read_file_content` (e um Google Doc com JSON dentro do corpo).
3. Se nao achou no Drive **e** estamos rodando local com `.sync-call-state.json` existindo, use o local (pra migracao de v1.x → v2.0).
4. Se nao achou em nenhum lugar, comece um state vazio.

Esquema do state file (v2.0):
```json
{
  "version": "1.2",
  "processed_file_ids": ["file_id_1", "file_id_2", ...],
  "last_run": "2026-05-15T17:00:00Z",
  "telemetry": {
    "patterns_seen": {
      "Check-in | {Cliente}": 12,
      "[CHECK-IN] {CLIENTE}": 7,
      "Sprint Review {Dupla}": 3,
      "WBR {Dupla}": 1
    },
    "first_seen": {
      "WBR": "2026-05-14"
    },
    "last_seen": {
      "Check-in | {Cliente}": "2026-05-15",
      "WBR {Dupla}": "2026-05-14"
    },
    "confirmed_types": ["WBR"],
    "private_intentional_ids": []
  }
}
```

Esse arquivo e gitignored. Ele e a fonte de verdade pra:
- **Idempotencia** — `processed_file_ids` evita reprocessar
- **Telemetria** — `patterns_seen` / `first_seen` / `last_seen` alimentam a `/coord-auditoria-rotinas-squad` na auditoria semanal. Cada padrao normalizado e registrado com contagem e datas.
- **Calibracao** — `confirmed_types` lista tipos novos que o coordenador validou. `private_intentional_ids` lista IDs de invites sem email da unidade que foram aprovados como "privado mesmo" — pra nao re-alertar.

### Passo 2 — Listar candidatos (com paginacao)

Use `mcp__claude_ai_Google_Drive__search_files` em paralelo:

```
# busca 1 (com acento)
title contains 'Anotações do Gemini' and modifiedTime > '{janela}'

# busca 2 (sem acento — caso o Gemini tenha gerado sem)
title contains 'Anotacoes do Gemini' and modifiedTime > '{janela}'
```

**Paginacao:** se a resposta tiver `nextPageToken`, repita a chamada com `pageToken: {token}` ate o token sair vazio. Junte todos os arquivos.

Junte os resultados das duas buscas, deduplicate por `id`. Use `excludeContentSnippets: true`.

Filtre por mimeType:
- `application/vnd.google-apps.document` (Gemini Notes novo — Google Doc nativo)
- `application/vnd.openxmlformats-officedocument.wordprocessingml.document` (.docx antigo)

### Passo 3 — Mapear destinos (clientes + squads)

Em paralelo, indexe:

**3a. Clientes locais.** Pra cada `squads/*/clientes/*/`:
- Leia `links.md`
- Regex pra Drive folder ID: `drive\.google\.com/drive/folders/([A-Za-z0-9_-]+)`
- Slug local = `basename` da pasta
- Squad pai = nome da pasta acima de `clientes/`
- Adicione ao indice: `{slug → {path_local, drive_folder_id, squad}}`

**3b. Squads locais.** Pra cada `squads/*/` (ignorando `_template-*`):
- Verifique se existe `links.md` no nivel do squad
- **Se nao existir:** pause a rodada, pergunte:
  > "Squad `{nome}` nao tem `links.md` configurado. Qual o Drive folder ID dele? (Cole o link da pasta no Drive, formato `https://drive.google.com/drive/folders/...`, ou Enter pra pular esse squad nesta rodada)"
  
  Se o usuario der o link: extraia o folder_id, crie `squads/{nome}/links.md` + `squads/{nome}/calls/` + `squads/{nome}/historico-reunioes.md` com cabecalho. Use o template em `references/squad-links-template.md` se quiser.

- Adicione ao indice: `{squad_slug → {path_local, drive_folder_id}}`
- **Subpasta no Drive do squad:** garanta que existe `Reunioes Internas` dentro do Drive do squad. Use `search_files` com `parentId={squad_folder_id} and title='Reunioes Internas' and mimeType='application/vnd.google-apps.folder'`. Se nao existir, crie via `mcp__claude_ai_Google_Drive__create_file` com `mimeType='application/vnd.google-apps.folder'`, `parentId={squad_folder_id}`, `title='Reunioes Internas'`. Guarde o folder_id da subpasta no indice.

### Passo 4 — Classificar e rotear cada arquivo

Pra cada arquivo do Passo 2:

**4a. Idempotencia (duas camadas).**
- Se `file.id` esta em `state.processed_file_ids` → SKIP (camada do state file).
- Se `file.parentId` esta no indice de Drive folders conhecidos (clientes do Passo 3a OU squads do Passo 3b OU suas subpastas `Reunioes Internas`) → considera "ja organizado externamente", **adiciona ao state** e SKIP. Essa e a primeira camada — funciona mesmo na primeira rodada com state vazio (e o que torna a skill segura pra rodar em hubs ja populados manualmente).

**4b. Detectar padroes privados.** Regex case-insensitive no titulo:
```
\b(1:1|1on1|1 a 1|one on one|feedback -)
```
Se bater → marca como **PRIVADO**, adiciona ao state e ao relatorio, continua proximo arquivo. Nunca tenta copiar/baixar.

**4c. Detectar padroes squad-level.** Regex case-insensitive:
```
\b(sprint planning|sprint review|comit[eê] de opera[cç][oõ]es|daily|retrospectiva|retro\b|reuni[aã]o do squad|\bwbr\b|weekly business review|alinhamento squad|sync squad)
```
Se bater → tente identificar o squad:
- Extraia palavras-chave do titulo (depois remover o padrao squad-level)
- Fuzzy match contra slugs de squads no indice (Passo 3b)
- Se so existe 1 squad no hub OU match >= 0.7 com um squad especifico → roteia pra ele
- Se 2+ squads e match ambiguo → pergunta ao usuario qual squad

Se squad identificado → vai pra **Passo 5 (squad)**.
Se nao identificado → adiciona ao relatorio como "squad ambiguo" e segue.

**4d. Detectar cliente (fuzzy).** Aplique extracao de cliente do titulo:

Padroes conhecidos (tente em ordem):
1. `{Cliente} + V4 Company - YYYY_MM_DD ...` → cliente = parte antes de ` + V4 Company`
2. `{NomeReuniao} | {Cliente} - YYYY/MM/DD ...` → cliente = parte depois do `|`
3. `Check-In {Cliente} - YYYY_MM_DD ...` → cliente = parte depois de `Check-In `
4. `Sprint Growth {Cliente} - ...` ou `Sprint Growth | {Cliente} - ...` → cliente = parte depois
5. `[YYYY/MM/DD] {Tipo} - {Cliente}` → cliente = parte depois do hifen
6. Fallback: texto entre o inicio e o primeiro ` - YYYY`

Normalize → slug (lowercase + sem acentos + hifens). Compare via `difflib.SequenceMatcher` (Python) contra slugs de clientes no indice (Passo 3a).

| Similaridade | Acao |
|---|---|
| >= 0.85 | Aceita auto → vai pra Passo 5 (cliente) |
| 0.70-0.84 | Pede confirmacao no chat → se OK, Passo 5 (cliente). Se nao, marca como "nao identificado" |
| < 0.70 | Marca como "nao identificado" |

### Passo 5 — Processar arquivo (cliente OU squad)

Pra um arquivo aceito (cliente ou squad), o fluxo e identico, so muda os destinos.

**5a. Verificar duplicata no Drive destino**
```
search_files: parentId = '{drive_destino_id}' and title = '{titulo_original}'
```
Se ja existir, pular `copy_file` (mas siga pra etapas locais).

**5b. Copiar pro Drive destino** (somente se nao duplicado)
```
copy_file:
  fileId: {id_original}
  parentId: {drive_destino_id}
  title: {titulo_original}
```
**Quirk:** Google Docs nativos retornam `fileSize: 1` mesmo com conteudo completo. Se desconfiar, valide via `read_file_content`.

**5c. Baixar conteudo pra arquivo local**

Use `read_file_content(fileId=id_original)`.

**Transcripts grandes (>~80KB):** a tool retorna erro com path do arquivo de tool-result salvo no disco. Quando isso acontecer:
```bash
jq -r '.fileContent' /caminho/do/tool-result.txt > /tmp/transcript-raw.txt
```

Antes de salvar, **verifique se o arquivo ja existe** no destino local:
- Cliente: `squads/{squad}/clientes/{cliente}/calls/{YYYY-MM-DD}-{slug-da-call}.md`
- Squad: `squads/{squad}/calls/{YYYY-MM-DD}-{slug-da-call}.md`

Se existir, pular esta etapa. Senao, salve com cabecalho:

```markdown
# Call {YYYY-MM-DD} - {Tipo da call inferido}

**Tipo:** {Check-In | Planejamento | Sprint Growth | Vendas | Kickoff | Sprint Planning | Sprint Review | Comite de Operacoes | Daily | Retrospectiva | Ad hoc}
**Escopo:** {cliente: nome | squad: nome}
**Duracao:** {extrair do transcript se aparecer}
**Origem:** Drive — Anotacoes do Gemini, file ID `{id_original}`
**Coletado via:** Drive MCP em {timestamp ISO}
**Owner do arquivo:** {file.owner}

---

{conteudo bruto do transcript}
```

**Slug da call** = `{tipo-kebab}` (ex: `planejamento-estrategico`, `sprint-review`, `comite-operacoes`).

**5d. Atualizar indice de historico (dual-write: local + Drive append-only)**

**Drive (sempre que rodar — local ou cloud):**
- **Cliente:** garante que existe subpasta `historico-checkins/` dentro da pasta do cliente no Drive (use `search_files` + `create_file` mimeType folder se faltar). Cria arquivo `{YYYY-MM-DD}-{slug}.md` nessa subpasta com a entrada nova.
- **Squad:** garante subpasta `historico-reunioes/` dentro da pasta do squad no Drive. Cria `{YYYY-MM-DD}-{slug}.md` la.
- Conteudo do arquivo: o mesmo bloco markdown que voce escreveria localmente (template abaixo). Cada arquivo = uma entrada. **Append-only via novo arquivo, sem update.**

**Local (so se rodando local):**
- **Cliente** → append em `squads/{squad}/clientes/{cliente}/mission-control/historico-checkins.md` em ordem cronologica:
```markdown
## {YYYY-MM-DD} - {Tipo}
**Modo:** ND
**Resumo (1 linha):** {extrair primeira frase da secao "Resumo" do Gemini Notes}
**Transcript:** [calls/{YYYY-MM-DD}-{slug}.md](../calls/{YYYY-MM-DD}-{slug}.md)
**Transcript no Drive do cliente:** copiado em {ISO}, file ID `{id_copia}`
**Pontos criticos:**
- {3-5 bullets da secao "Proximas etapas" do Gemini Notes}
```

**Squad** → adicione em `squads/{squad}/historico-reunioes.md`:
```markdown
## {YYYY-MM-DD} - {Tipo}
**Resumo (1 linha):** {primeira frase do Resumo}
**Transcript:** [calls/{YYYY-MM-DD}-{slug}.md](calls/{YYYY-MM-DD}-{slug}.md)
**Transcript no Drive do squad:** copiado em {ISO}, file ID `{id_copia}`
**Acoes principais:**
- {3-5 bullets das Proximas etapas}
```

**5e. Marcar como processado + atualizar telemetria**

- Adicione `file.id` em `state.processed_file_ids`.
- **Telemetria** (importante pra `/coord-auditoria-rotinas-squad`):
  - Normalize o titulo do arquivo pra "pattern" (substitua o nome do cliente/dupla/data por placeholders). Ex: `Check-in | Armazem Santa Filomena` → `Check-in | {Cliente}`; `Sprint Review - Iza/Julio` → `Sprint Review - {Dupla}`.
  - Incremente `state.telemetry.patterns_seen[pattern]` (cria se nao existir, soma 1).
  - Se e a primeira ocorrencia desse pattern: registre data atual em `state.telemetry.first_seen[pattern]`.
  - Sempre atualize `state.telemetry.last_seen[pattern]` com a data atual.
- Salve o state file.

### Passo 6 — Salvar state file final (dual-write versionado)

Atualize `last_run` no state com o timestamp atual. Salve em **ambos**:

**Drive (sempre):**
- Crie um NOVO arquivo em `Builders Hub - State` (ID `1IKY3yRo84luz3Sn5p1HefXsC8n95T7C3`) com titulo `sync-call-state-{YYYY-MM-DD-HH-MM}.json` (formato Google Doc nativo com JSON dentro do corpo)
- Use `create_file` com `contentMimeType: 'application/json'` e `disableConversionToGoogleType: false` pra ficar como Google Doc legivel (o Drive le e converte) — alternativa: `contentMimeType: 'text/plain'` com extensao `.json` no titulo. Teste qual da melhor.
- **NAO delete arquivos antigos** — o historico de states e seu audit trail. A "versao atual" e sempre o `modifiedTime` mais recente.

**Local (so se rodando local):**
- Sobrescreva `.sync-call-state.json` na raiz do hub com o conteudo atual.

### Passo 7 — Centralizar referencia no git (`STATE-BACKUP-LOG.md`)

Adicione UMA LINHA no final de `STATE-BACKUP-LOG.md` (arquivo NAO-gitignored na raiz do hub):

```
| 2026-05-15T19:30:00Z | /geral-sync-call-transcripts | state | sync-call-state-2026-05-15-19-30.json | 1ABC...xyz | processados: 2; clientes: 1; squad: 1 |
```

Formato da linha (delimitado por `|`):
- timestamp ISO
- skill que rodou
- tipo de artefato (`state`, `historico-cliente`, `historico-squad`, `auditoria`)
- titulo do arquivo no Drive
- file ID
- resumo curto (texto livre, sem dados sensiveis)

Se `STATE-BACKUP-LOG.md` nao existir, cria com cabecalho. Esse arquivo e a fonte de rastreabilidade no git — qualquer dev olhando o repo consegue auditar quando rodou cada operacao e onde esta o backup.

**Se rodando cloud (sem acesso a write local):** mesmo assim, escreva o log usando git via commit automatico apos a operacao. Pra simplificar: cloud monta o commit message com o conteudo da linha e usa `gh` ou similar pra abrir PR no repo com a linha nova. Ou (mais simples ainda): a cloud sobe a linha como arquivo separado em `Builders Hub - State/log/{ts}.txt` no Drive — e o coordenador, ao rodar `/sync-hub` local, agrega esses logs no `STATE-BACKUP-LOG.md` local e commita. Documentado abaixo na secao "Reconciliacao cloud → git".

### Passo 8 — Relatorio final

```
✅ Sincronizacao concluida ({janela}, desde {janela_ISO})

Candidatos no Drive: {total}
  → {pulado_state}: ja processado em rodadas anteriores (state file)
  → {pulado_organizado}: ja em pasta destino no Drive

Processados nesta rodada: {novos}
  • Clientes: {n_clientes} ({lista})
  • Squad-level: {n_squad} ({tipos: "Sprint Planning", "Comite", etc.})

Privados (skipados por design): {n_privados}
  • {nome_arquivo} — motivo: "1:1" / "Feedback" / ...

Nao identificados (revisar manual): {n_nao_id}
  • "{titulo}" — fuzzy {%}, sem cliente ou squad correspondente

Proximos passos sugeridos:
  • Pra cada call de cliente processada: considere rodar `/account-checkin-review` no cliente correspondente
  • Pra calls de squad: revisar o `historico-reunioes.md` do squad antes do proximo comite
```

## Regras

- **NUNCA** processe titulos com padrao privado (1:1, Feedback). Privacidade e nao-negociavel.
- **NUNCA** mova o arquivo original do Drive do GP — sempre `copy`.
- **SEMPRE** idempotente. State file + verificacao de duplicata garantem isso em camadas.
- **NAO** apague conteudo existente. Se um arquivo local ja existe, pula em vez de sobrescrever.
- **CALLS internas sem squad** ou **calls com cliente que nao bate** caem em "nao identificado" — isso e o comportamento correto. Nunca force matching.
- **Privacidade dos transcripts:** nao mostre o conteudo bruto no chat — so resumo + acoes principais.

## Tratamento de erros

### Drive MCP nao autenticado
`list_recent_files` falha com erro de auth → pare, oriente `/mcp` → selecionar Google Drive.

### Cliente sem Drive folder em links.md
Pule esse cliente. Liste no relatorio final: "X clientes sem Drive folder configurado — edite `links.md` ou rode `/novo-cliente` pra incluir".

### Squad sem links.md
Pergunte o folder no Passo 3b. Se usuario pular, processa so clientes nesta rodada.

### `read_file_content` falha por tamanho
Use jq direto no arquivo de tool-result. Header + conteudo bruto.

### `copy_file` falha (permissao)
Reporte com nome do destino e file ID. Sugira ao usuario garantir que a conta da unidade tem edit access na pasta destino.

### Conflito de matching
Pergunte qual escolher. Mostre opcoes com path completo.

### State file corrompido
Se `.sync-call-state.json` falhar de parsear, **NAO delete**. Renomeie pra `.sync-call-state.corrupted-{timestamp}.json`, crie novo vazio, avise o usuario que a rodada vai re-checar tudo (duplicatas de Drive/local previnem reprocessamento).

## Padroes privados a detectar (regex em titulo, case-insensitive)

```
\b(1:1|1on1|1 a 1|one[ -]on[ -]one|feedback\s*-|individual)
```

## Padroes squad-level a detectar

```
\b(sprint\s+planning|sprint\s+review|comit[eê]\s+de\s+opera[cç][oõ]es|daily|retrospectiva|retro\b|reuni[aã]o\s+do\s+squad|\bwbr\b|weekly\s+business\s+review|alinhamento\s+squad|sync\s+squad)
```

## Padroes de extracao de cliente (em ordem de tentativa)

```python
patterns = [
  r'^(.+?)\s*\+\s*V4\s*Company\s*-\s*\d{4}',        # "{Cliente} + V4 Company - YYYY..."
  r'\|\s*(.+?)\s*-\s*\d{4}',                         # "...| {Cliente} - YYYY..."
  r'^Check-In\s+(.+?)\s*-\s*\d{4}',                  # "Check-In {Cliente} - YYYY..."
  r'^Sprint\s+Growth\s*\|?\s*(.+?)\s*-\s*\d{4}',     # "Sprint Growth {Cliente} - ..." ou com |
  r'^\[\d{4}/\d{2}/\d{2}\]\s+\w+\s*-\s*(.+?)\s*-',   # "[YYYY/MM/DD] Tipo - {Cliente} - ..."
  r'^(.+?)\s*-\s*\d{4}',                              # Fallback
]
```

## Exemplo completo de uso (multi-tipo)

**Input:**
> "Sincroniza tudo dos ultimos 7 dias"

**Skill encontra 6 arquivos no Drive:**
1. `Planejamento Estrategico | Armazem Santa Filomena - 2026/05/15...` → CLIENTE armazem-santa-filomena
2. `Check-In Labteste - 2026_05_14...` → CLIENTE labteste (se existir no hub)
3. `Sprint Planning Squad Phoenix - 2026_05_13...` → SQUAD phoenix
4. `Comite de Operacoes - 2026_05_12...` → SQUAD (1 squad existente → phoenix auto)
5. `1:1 Henrique e Igor - 2026_05_11...` → PRIVADO (skip)
6. `Sprint Review - Nicolas e Anderson - 2026_05_10...` → SQUAD phoenix (1 squad → auto)

**Output:**
```
✅ Sincronizacao concluida (7d, desde 2026-05-08T17:00:00Z)

Candidatos no Drive: 6
Processados nesta rodada: 4
  • Clientes: 2 (armazem-santa-filomena, labteste)
  • Squad-level: 2 (Sprint Planning, Comite — squad-phoenix; Sprint Review — squad-phoenix)

Privados (skipados por design): 1
  • "1:1 Henrique e Igor - 2026_05_11..." — motivo: "1:1"

Nao identificados: 0
```

## Nomenclatura padrao sugerida (pra alinhar com o squad)

Pra reduzir casos de "nao identificado" e simplificar inferencia de tipo, sugira ao squad padronizar invites do Calendar assim:

**Calls com cliente (recorrentes):**
- `Check-in | {Cliente}` — check-ins recorrentes (semanais/quinzenais)
- `Kickoff | {Cliente}` ou `GC + Kickoff | {Cliente}` — kickoff inicial
- `Planejamento Estrategico | {Cliente}` — apresentacao de planejamento
- `Encontro {N} do E.E. | {Cliente}` — encontros estrategicos numerados (E.E. = Estruturacao Estrategica)
- `Sprint Growth | {Cliente}` — sprint growth por cliente

**Calls com cliente (pontuais):**
- `Alinhamento - {Cliente}` ou `{Topico} - {Cliente}` — calls ad hoc

**Reunioes do squad (internas, sem cliente):**
- `Sprint Planning {Dupla}` — ex: "Sprint Planning Igor/Anderson"
- `Sprint Review {Dupla}` — ex: "Sprint Review Iza/Julio"
- `WBR {Dupla}` — Weekly Business Review entre Coordenador + Dupla
- `Comite de Operacoes` — reuniao geral do squad
- `Daily {Dupla}` ou `Daily Squad` — dailies

**Calls privadas (NAO incluir email da unidade):**
- `1:1 - {Nome}` — entre coordenador e investidor

**Por que sem data no titulo:** o Gemini Notes ja adiciona timestamp automatico. Manter o titulo limpo facilita pattern matching e tambem deixa o invite mais legivel na agenda.

**O que a skill faz mesmo sem padronizacao total:** o fuzzy matching e regex tolerantes cobrem variacoes ("Filomema" vs "Filomena", caps lock, `[CHECK-IN]` com colchetes, underscore vs slash na data). A padronizacao acima e otimizacao — nao bloqueador.

## Observacoes

- **Padroes novos:** se aparecer titulo que nao cai em nenhum regex, adicione no Passo 4 e bump version pra 1.x. Padronizacao do invite com a unidade reduz esses casos.
- **Sprint Reviews por dupla:** Sprint Review e sempre por dupla GP+GT (nao tem versao consolidada do squad). Pra agregar o que aconteceu em todas as duplas na semana, use a skill complementar `/coord-resumo-semanal-squad` (a criar) que le todos os Sprint Reviews da semana e gera um digest pro Coordenador.
- **Skill complementar:** `/account-checkin-review` processa o transcript ja salvo (extrai combinados, atualiza apostas vivas, refina personas). Use as duas em sequencia pra calls de cliente.
- **Como rodar agendado:** combine com `/schedule` (skill nativa) pra disparar 1x/dia (ex: 19h fim do dia util).
- **Multi-squad:** se voce esta numa unidade com varios squads, o squad-name precisa aparecer no titulo da reuniao squad-level pra desambiguar. Padronize: "Sprint Planning Squad {Nome}" ou similar.
- **1:1s privados:** se voce precisa de uma skill que processe 1:1s tambem, faca uma skill SEPARADA com Drive autenticado em conta pessoal (nao da unidade). Nunca misture o fluxo.
