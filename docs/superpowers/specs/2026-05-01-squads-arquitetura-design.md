# Spec — Camada de Squad em `squads/`

**Data:** 2026-05-01
**Status:** Implementado

> **Nota de update (2026-05-01):** durante a execucao, a pasta wrapper `clientes/` foi renomeada para `squads/` pra deixar a hierarquia explicita. Onde este spec menciona `clientes/{squad}/...`, leia como `squads/{squad}/...`. O caminho final padronizado e: `squads/{squad}/clientes/{cliente}/`. Cliente solto fora de squad nao existe.

## Contexto

Hoje o builders-hub tem clientes diretos em `clientes/{cliente}/`. Cada cliente e uma KB pessoal (gitignored) com `calls/`, `docs/`, `campanhas/`, `CLAUDE.md` e `.env`. Skills relevantes: `/novo-cliente` cria a pasta a partir de `clientes/_template/`; `/contexto` gera o `CLAUDE.md` da KB.

Precisamos agrupar clientes por squad — um squad e um time fixo de pessoas (gestor, account, trafego, criativo, CS, etc.) que atende varios clientes. Um cliente pertence a um squad so.

## Objetivo

Adicionar uma camada de squad entre `clientes/` e o cliente. Criar uma skill `/novo-squad` para criar a pasta do squad com README de membros. Atualizar `/novo-cliente` e `/contexto` para o novo layout. Migrar o cliente existente (`lopes-campanelli-sociedade-de-advogados`) para um `squad-exemplo`.

`bases/` nao sofre mudanca.

## Estrutura de pastas final

```
clientes/
├── _template-cliente/                  ← renomeado de _template/, vai pro repo
├── _template-squad/                    ← novo, vai pro repo
└── {squad}/                            ← gitignored
    ├── CLAUDE.md                       ← contexto do squad (lido em cascata pelo Claude Code)
    ├── README.md                       ← quem e quem (formato livre, leitura humana)
    ├── docs/                           ← docs gerais do squad
    └── clientes/
        └── {cliente}/                  ← estrutura padrao de cliente (igual hoje)
            ├── CLAUDE.md
            ├── .env
            ├── .env.example
            ├── calls/
            ├── docs/
            └── campanhas/
```

Decisoes:

- **Cliente solto e proibido.** Todo cliente vive dentro de um squad. `/novo-cliente` recusa criar se nao houver squad e manda rodar `/novo-squad` antes.
- **Roles dos membros sao livres.** A skill nao impoe lista fixa de cargos.
- **Dois arquivos no nivel do squad: `CLAUDE.md` + `README.md`.** O `CLAUDE.md` da contexto pra IA em cascata; o `README.md` lista membros pra leitura humana. O CLAUDE.md aponta pro README.
- **Sem `.env` no nivel do squad.** Credenciais ficam no `.env` do cliente (igual hoje).
- **Edicao de membros depois e manual.** Editar o `README.md` direto. Skill dedicada de edicao fica fora de escopo (YAGNI).

## Componentes

### 1. Templates no repo

**`clientes/_template-squad/`** (novo, commitado):

```
_template-squad/
├── CLAUDE.md
├── README.md
├── docs/.gitkeep
└── clientes/.gitkeep
```

Conteudo do `CLAUDE.md` no template:

```markdown
# Squad {NOME}

Os membros do squad estao listados em `README.md`.
Os clientes deste squad ficam em `./clientes/`.
```

Conteudo do `README.md` no template:

```markdown
# Squad {NOME}

## Membros

- (preenchido por `/novo-squad`)
```

**`clientes/_template-cliente/`** (renomeado de `_template/`):

Mesmo conteudo de hoje. So muda o nome da pasta. Tem `.env.example`, `CLAUDE.md` placeholder, e subpastas `calls/`, `docs/`, `campanhas/` (com `.gitkeep`).

### 2. Skill nova: `/novo-squad`

Sem prefixo (skill de base, igual a `/novo-cliente` e `/novo-projeto`). Duplo-write em `.claude/skills/novo-squad/` e `.agents/skills/novo-squad/`.

Fluxo:

1. **Pergunta nome do squad.** Ex.: "Squad Performance" → pasta `squad-performance` (lowercase + hifens).
2. **Cria a estrutura** copiando `clientes/_template-squad/` para `clientes/{nome-formatado}/`. Substitui o placeholder `{NOME}` no `CLAUDE.md` e no `README.md` pelo nome original digitado pelo usuario (com capitalizacao preservada — ex: "Squad Performance"), nao pela versao em hifens.
3. **Coleta membros em loop:**
   > "Adiciona um membro do squad: nome + funcao (ex: 'Joao Silva — Gestor de Trafego'). Aperta Enter sem digitar nada quando terminar."
   
   Aceita N entradas. Para no primeiro Enter vazio.
4. **Substitui** o placeholder `- (preenchido por /novo-squad)` no `README.md` pela lista coletada (um membro por bullet).
5. **Confirma** mostrando a arvore criada e dica:
   > "Squad criado. Roda `/novo-cliente` pra adicionar um cliente neste squad."

### 3. Skill atualizada: `/novo-cliente`

Mudancas no fluxo:

1. **Antes de pedir nome do cliente**, escaneia `clientes/*/` filtrando `_template-*`. Resultado e a lista de squads existentes.
   - Se vazia: para com mensagem "Voce ainda nao tem squad. Roda `/novo-squad` antes."
   - Se nao vazia: mostra lista numerada e pergunta "Em qual squad esse cliente entra?". Aceita numero ou nome.
2. **Cria a pasta** em `clientes/{squad}/clientes/{nome-formatado}/` copiando `_template-cliente/`.
3. **Resto identico ao fluxo de hoje**: NotebookLM, geracao do `CLAUDE.md` do cliente, copia do `.env.example` pra `.env`, mensagem final.

### 4. Skill atualizada: `/contexto`

Hoje le todos os arquivos da pasta corrente e gera `CLAUDE.md`. Precisa detectar o nivel:

- **Pasta corrente tem subpasta `clientes/`** (e nao tem `calls/docs/campanhas/`) → e um squad. Gera `CLAUDE.md` consolidando: `README.md` (membros) e arquivos em `docs/` do squad. So lista os nomes das pastas de clientes filhos — nao le os arquivos internos deles (cada cliente tem seu proprio `CLAUDE.md`, gerado separadamente).
- **Pasta corrente tem `calls/`, `docs/`, `campanhas/`** → e um cliente. Comportamento atual.
- **Outro caso** → erro: "Roda `/contexto` dentro de uma pasta de squad ou cliente."

### 5. CLAUDE.md raiz do builders-hub (atualizar)

Reescrever o trecho que descreve `clientes/` e `bases/`:

> - `clientes/{squad}/clientes/{cliente}/` — KBs de clientes, agrupados por squad. Crie squad com `/novo-squad` antes do primeiro cliente.
> - `bases/{projeto}/` — KBs de qualquer outra coisa (estudos, projetos pessoais).
> - Cada squad tem `README.md` com os membros e `CLAUDE.md` com contexto pro Claude.
> - Cada cliente tem o `CLAUDE.md` proprio gerado por `/contexto`.

Atualizar tambem a lista de "Skills de setup/fluxo" pra incluir `/novo-squad`.

### 6. `.gitignore` (atualizar)

Adicionar excecao pro novo template:

```
!/clientes/_template-cliente/
!/clientes/_template-squad/
```

Remover a linha antiga `!/clientes/_template/` (ja vai estar renomeado).

`bases/_template/` continua intocado.

### 7. Migracao do cliente existente

Como parte da execucao do plano:

1. Renomear `clientes/_template/` → `clientes/_template-cliente/` com `git mv`.
2. Rodar `/novo-squad` (manual ou simulado) pra criar `clientes/squad-exemplo/` — membros placeholder que o usuario preenche depois.
3. Mover `clientes/lopes-campanelli-sociedade-de-advogados/` → `clientes/squad-exemplo/clientes/lopes-campanelli-sociedade-de-advogados/` com `git mv`. (Como o conteudo do cliente e gitignored, na pratica e um `mv` mesmo — git nao rastreia esses arquivos.)

## Skills nao impactadas neste spec

Skills que recebem o cliente como argumento ou via prompt (ex.: `account-pesquisa-profunda-cliente`, `v4mos-dados-meta-ads`, `trafego-meta-diagnostico`) usam caminho dinamico. Nao precisam mudanca no spec. Se alguma quebrar por assumir caminho hardcoded, ajustamos como bug fix durante a execucao.

## Fora de escopo

- Squad em `bases/`. Decidido: `bases/` fica plano.
- Multi-squad por cliente. Decidido: cliente pertence a um squad so.
- Skill de edicao de membros do squad. Edicao manual no `README.md` resolve por agora.
- Cargos fixos / lista pre-definida de roles. Roles sao livres.

## Criterios de aceite

1. `/novo-squad` cria a pasta com `CLAUDE.md`, `README.md`, `docs/`, `clientes/` e preenche os membros coletados.
2. `/novo-cliente` recusa rodar se nao houver squad e cria o cliente em `clientes/{squad}/clientes/{cliente}/` quando houver.
3. `/contexto` funciona corretamente em pasta de squad e em pasta de cliente.
4. `clientes/_template/` foi renomeado pra `_template-cliente/` e o `.gitignore` reflete os dois templates novos.
5. `clientes/lopes-campanelli-sociedade-de-advogados/` foi movido pra `clientes/squad-exemplo/clientes/`.
6. CLAUDE.md raiz descreve a estrutura nova e cita `/novo-squad`.
7. Skills `/novo-squad`, `/novo-cliente`, `/contexto` existem identicas em `.claude/skills/` e `.agents/skills/`.
