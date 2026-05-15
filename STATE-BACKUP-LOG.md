# STATE-BACKUP-LOG

Registro de cada execucao das skills que escrevem estado/historico no Drive da unidade. Esse log e o **ponto de centralizacao no git** da arquitetura hibrida (local + Drive espelhado + git pra rastreabilidade).

**Nao contem dados sensiveis** — so referencias (file IDs do Drive, timestamps, contagens). Acesso aos arquivos referenciados continua protegido pelas permissoes do Drive.

**Append-only.** Cada execucao adiciona uma linha no final. Nao remova entradas — esse e o audit trail.

## Como funciona

- `/geral-sync-call-transcripts` adiciona linha apos cada rodada (state, historico, transcripts copiados).
- `/coord-auditoria-rotinas-squad` adiciona linha apos cada auditoria semanal gerada.
- Skills rodando local: escrevem direto aqui.
- Skills rodando cloud (via /schedule): escrevem em `Builders Hub - State/log/{ts}.txt` no Drive. Coordenador roda `/sync-hub` (ou similar) pra agregar esses logs no `STATE-BACKUP-LOG.md` local e commitar.

## Schema

```
| ISO_TIMESTAMP | SKILL | TIPO | TITULO_ARQUIVO_DRIVE | FILE_ID | RESUMO_CURTO |
```

- **TIPO** pode ser: `state`, `historico-cliente`, `historico-squad`, `transcript-copia`, `auditoria`
- **RESUMO_CURTO** e texto livre sem dados sensiveis (ex: "processados: 2; clientes: 1; squad: 1")

## Log

| Timestamp | Skill | Tipo | Titulo | File ID | Resumo |
|---|---|---|---|---|---|
| 2026-05-15T17:17:26Z | /geral-sync-call-transcripts | transcript-copia | Planejamento Estratégico \| Armazém Santa Filomena - 2026/05/15 10:22 | 1jpmCUjllTepFcsv-bE6_e0FH17JUjh-epi6mKxBDs34 | armazem-santa-filomena: 1 call de planejamento copiada do Drive do Igor pra pasta do cliente |
| 2026-05-15T18:18:05Z | /geral-sync-call-transcripts | transcript-copia | Sprint Review - Iza/Júlio - 2026/05/15 08:59 | 1pdVE0v3s4jWAF5hKhex9BQxqNKTtEdONYjoaLsGUPME | squad-phoenix: Sprint Review Iza/Julio copiada pra Reunioes Internas |
| 2026-05-15T18:32:33Z | /geral-sync-call-transcripts | transcript-copia | Sprint Review - Nícolas/Anderson - 2026/05/15 08:15 | 1pQuxYcK53D1ZxLJhPsddnyQAh_5nQRAn9Fd_NsadfZQ | squad-phoenix: Sprint Review Nicolas/Anderson copiada (ultimo dia do Nicolas) |
| 2026-05-15T19:50:33Z | (setup manual) | infraestrutura | Builders Hub - State (folder) | 1IKY3yRo84luz3Sn5p1HefXsC8n95T7C3 | Pasta dedicada criada no Drive da unidade pra state files versionados |
