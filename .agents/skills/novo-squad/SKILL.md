---
name: novo-squad
description: Cria uma nova pasta de squad em squads/ com README de membros, links.md (incluindo Drive folder do squad), pasta de calls internas, historico de reunioes e estrutura padrao. Pergunta nome, membros e Drive folder. Use quando o usuario rodar /novo-squad ou disser que quer criar um squad novo.
---

Voce vai criar a pasta de um novo squad com a estrutura padrao, README listando os membros, CLAUDE.md/AGENTS.md iniciais, links.md com Drive folder, pasta `calls/` pra reunioes internas e `historico-reunioes.md`.

## Contexto

Squads sao times fixos de pessoas (gestor, account, trafego, criativo, CS, etc.) que atendem varios clientes. Um cliente pertence a um squad so. **Padrao obrigatorio:** `squads/{squad}/clientes/{cliente}/`. A estrutura final do squad e:

```
squads/{squad}/
├── CLAUDE.md          ← contexto do squad
├── AGENTS.md          ← espelho do contexto para outros agentes
├── README.md          ← quem e quem (formato livre)
├── links.md           ← Drive folder do squad + recursos recorrentes
├── historico-reunioes.md  ← indice de reunioes internas (atualizado por /geral-sync-call-transcripts)
├── calls/             ← transcripts brutos de reunioes internas
├── docs/              ← docs gerais do squad
└── clientes/          ← clientes do squad (criados via /novo-cliente)
```

## Processo

### Passo 1 — Nome do squad

Pergunte:
> "Qual o nome do squad?"

Guarde duas versoes:
- **Nome digitado** (capitalizacao original): ex: "Squad Performance"
- **Nome formatado** pra pasta: lowercase + hifens, sem acentos: ex: "squad-performance"

### Passo 2 — Criar a estrutura base

```bash
cp -r bases/_template/_template-squad "squads/[nome-formatado]"
```

Substitua `{NOME}` em `CLAUDE.md`, `AGENTS.md`, `README.md`, `links.md` e `historico-reunioes.md` da nova pasta pelo **nome digitado** (preservando capitalizacao).

### Passo 3 — Coletar membros

Em loop, pergunte:
> "Adiciona um membro do squad: nome + funcao (ex: 'Joao Silva — Gestor de Trafego'). Aperta Enter sem digitar nada quando terminar."

A cada resposta nao-vazia, guarde na lista. Enter vazio encerra.

Se a lista ficar vazia, siga em frente — o README fica com placeholder.

### Passo 4 — Coletar Drive folder do squad (recomendado)

Pergunte:
> "Cola o link da pasta do squad no Google Drive (formato `https://drive.google.com/drive/folders/...`). Esse Drive guarda materiais do squad (POPs, planilhas, atas) e e onde a skill `/geral-sync-call-transcripts` vai salvar transcripts de reunioes internas. Aperta Enter pra pular agora."

Se o usuario der o link:
- Extraia o `folder_id` via regex `drive\.google\.com/drive/folders/([A-Za-z0-9_-]+)`
- Substitua em `links.md`:
  - `**Google Drive (raiz do squad):** —` → `**Google Drive (raiz do squad):** {URL completa}`

Se pular: deixa o `—` no `links.md` mesmo — pode ser preenchido depois. Avise:
> "Ok, pulei. Quando voce tiver o link, edita `squads/{nome-formatado}/links.md` direto OU rode `/geral-sync-call-transcripts` que ele pergunta de novo."

### Passo 5 — Atualizar o README

Em `README.md`, substitua a linha:
```
- (preenchido por `/novo-squad`)
```

Pela lista coletada:
```
- Joao Silva — Gestor de Trafego
- Maria Souza — Account
- ...
```

Se nenhum membro foi coletado, deixa o placeholder ou substitui por `- (a preencher)`.

### Passo 6 — Confirmar

Mostre a estrutura criada:
```
squads/[nome-formatado]/
├── CLAUDE.md
├── AGENTS.md
├── README.md          ← com os membros
├── links.md           ← com Drive folder (se informado)
├── historico-reunioes.md
├── calls/             ← vazio, pronto pra receber transcripts
├── docs/
└── clientes/          ← vazio, pronto pra receber clientes
```

Diga:
> "Squad criado. Proximos passos:
> 1. Roda `/novo-cliente` pra adicionar clientes neste squad
> 2. Adicione o e-mail da unidade como participante nos invites das reunioes internas do squad — `/geral-sync-call-transcripts` vai puxar os transcripts automaticamente"

## Regras

- Nome formatado da pasta e SEMPRE lowercase + hifens. Sem espacos, sem acentos.
- Se ja existir pasta com o nome formatado, avise e pergunte se quer escolher outro nome.
- Squad e gitignored por padrao (so os templates `_template-*` sobem pro repo).
- Membros tem formato livre — aceite qualquer string.
- Drive folder e **opcional** mas recomendado — sem ele, `/geral-sync-call-transcripts` nao consegue arquivar reunioes internas do squad (vai pedir o link em runtime).
