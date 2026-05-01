---
name: novo-cliente
description: Cria uma nova pasta de cliente dentro de um squad com estrutura padrao, CLAUDE.md inicial e links.md. Pergunta squad, nome e links uteis (NotebookLM, Drive, site, outros). Use quando o usuario rodar /novo-cliente ou disser que quer adicionar um cliente novo.
---

Voce vai criar a pasta de um novo cliente DENTRO de um squad. Todo cliente vive em `clientes/{squad}/clientes/{cliente}/`.

## Processo

### Passo 1 — Escolher o squad

Liste os squads existentes em `clientes/` (ignore qualquer pasta `_template-*`):

```bash
ls clientes/ | grep -v '^_template-'
```

- **Se nao existir nenhum squad**: pare e diga:
  > "Voce ainda nao tem squad. Roda `/novo-squad` antes pra criar o squad e depois rode `/novo-cliente` de novo."
  
  NAO crie cliente fora de squad.

- **Se existirem squads**: liste numerada e pergunte:
  > "Em qual squad esse cliente entra? (digite o numero ou o nome)"
  
  Aceite numero ou nome. Guarde o nome formatado (com hifens) como `[squad]`.

### Passo 2 — Nome do cliente

Pergunte:
> "Qual o nome do cliente?"

Converta para lowercase-com-hifens (ex: "Academia Estação Saúde" → "academia-estacao-saude"). Guarde como `[cliente]` (formatado) e `[Nome do Cliente]` (original).

### Passo 3 — Criar a estrutura

```bash
cp -r clientes/_template-cliente "clientes/[squad]/clientes/[cliente]"
# Copia o .env.example pra .env (inicial vazio, o usuario preenche conforme for usando)
cp "clientes/[squad]/clientes/[cliente]/.env.example" "clientes/[squad]/clientes/[cliente]/.env"
```

O `.env` e gitignored por padrao (clientes/ inteiro e — so `_template-*/` sobe pro repo). Credenciais ficam locais.

### Passo 4 — Coletar links uteis

Pergunte uma de cada vez. Aperta Enter sem digitar pula a pergunta.

1. > "NotebookLM desse cliente? Cola o link (formato `https://notebooklm.google.com/notebook/XXXXX`) ou Enter pra pular."
2. > "Pasta no Google Drive? Cola o link ou Enter pra pular."
3. > "Site do cliente? Cola a URL ou Enter pra pular."
4. **Loop:** > "Mais algum link util? Formato: 'descricao — URL' (ex: 'Looker do cliente — https://...'). Enter sem digitar encerra."
   - Aceita N entradas. Para no primeiro Enter vazio.

Se NotebookLM tiver link, extraia o notebook ID (parte depois de `/notebook/`).

### Passo 5 — Escrever `links.md`

Atualize `clientes/[squad]/clientes/[cliente]/links.md` substituindo o template:

```markdown
# Links uteis

Recursos recorrentes deste cliente. Atualize sempre que aparecer link novo.

## Bases de conhecimento
- **NotebookLM:** [URL ou —]
  - **Notebook ID:** [ID ou —]

## Drives e armazenamento
- **Google Drive:** [URL ou —]

## Web
- **Site:** [URL ou —]

## Outros
- [descricao — URL]
- [descricao — URL]
```

Itens nao informados ficam com `—`. A secao "Outros" recebe os itens do loop (passo 4.4).

### Passo 6 — Escrever `CLAUDE.md` do cliente

Crie `clientes/[squad]/clientes/[cliente]/CLAUDE.md` com:

```markdown
# [Nome do Cliente]

## Recursos
Veja `links.md` na raiz desta pasta pra todos os links recorrentes (NotebookLM, Drive, site, dashboards, etc).
[Se tiver NotebookLM, adicione uma linha: "Use `notebooklm` CLI com o notebook ID `[ID]` pra consultar a base."]

## Contexto
Rode `/contexto` apos adicionar dados (calls, docs, campanhas) pra gerar o contexto completo do cliente.

## Quando trabalhar com este cliente
- Comece lendo `links.md` pra saber dos recursos disponiveis.
- Se o usuario compartilhar um link util durante a conversa (Drive novo, dashboard, sheet, conta de anuncios), pergunte se ele quer adicionar a `links.md`.
```

### Passo 7 — Confirmar

Mostre a estrutura criada:
```
clientes/[squad]/clientes/[cliente]/
├── CLAUDE.md
├── links.md         # links uteis (NotebookLM, Drive, site, outros)
├── .env            # suas credenciais (gitignored)
├── .env.example    # template das credenciais
├── calls/
├── docs/
└── campanhas/
```

Diga:
> "Cliente criado dentro do squad [squad]. Os links que voce passou ja estao em `links.md`. Jogue os dados nas pastas (calls, docs, campanhas) e rode `/contexto` quando tiver pronto. O `.env` ta vazio — preenche conforme as skills pedirem."
