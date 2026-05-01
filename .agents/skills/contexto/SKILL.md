---
name: contexto
description: Le todos os arquivos em uma KB (cliente, squad ou projeto) e gera CLAUDE.md e AGENTS.md com o contexto completo. Detecta o nivel automaticamente. Use quando o usuario rodar /contexto ou quiser que a IA "conheca" um cliente, squad ou projeto.
---

Voce vai analisar uma Knowledge Base e gerar os arquivos `CLAUDE.md` e `AGENTS.md` que funcionem como "memoria" pra qualquer trabalho futuro.

## Estrutura esperada

- `clientes/{squad}/` — pasta de squad (tem `README.md` com membros, `docs/`, e subpasta `clientes/`)
- `clientes/{squad}/clientes/{cliente}/` — pasta de cliente (tem `calls/`, `docs/`, `campanhas/`, `links.md`)
- `bases/{projeto}/` — pasta de projeto/area (tem `docs/`, `dados/`, `referencias/`)

## Processo

### Passo 1 — Identificar a base

Detecte o que rodar com base na pasta corrente do usuario:

- **Pasta corrente e cliente** (tem `calls/`, `docs/`, `campanhas/` E nao tem subpasta `clientes/`): use ela direto.
- **Pasta corrente e squad** (tem subpasta `clientes/` E `README.md`): use ela direto.
- **Pasta corrente e projeto** (`bases/{X}/`): use ela direto.
- **Caso contrario:** liste todas as KBs disponiveis e pergunte:
  - Squads: `clientes/*/` (ignorando `_template-*`)
  - Clientes: `clientes/*/clientes/*/`
  - Projetos: `bases/*/` (ignorando `_template`)

### Passo 2 — Detectar o tipo

- **CLIENTE** (operacao): tem `calls/`, `docs/`, `campanhas/`
- **SQUAD**: tem subpasta `clientes/` e `README.md` com membros
- **PROJETO/AREA** (generico): tem `docs/`, `dados/`, `referencias/`

### Passo 3 — Ler tudo

Leia TODOS os arquivos da pasta:

- **Cliente**: leia tudo em `calls/`, `docs/`, `campanhas/`, `links.md` e qualquer outro arquivo.
- **Squad**: leia `README.md` e tudo em `docs/`. NAO leia o conteudo dos clientes filhos — so liste os nomes das pastas.
- **Projeto**: leia tudo recursivamente.

Leia cada arquivo por completo. Nao pule nada.

### Passo 4 — Analisar e gerar

**Se for CLIENTE (operacao):**

Extraia: nome da empresa, segmento, produto/servico, publico-alvo, diferenciais, canais, investimento, metricas, contatos, combinados, pendencias, objetivos, teses, historico, proximos passos. Tambem inclua os links uteis encontrados em `links.md`.

Gere o `CLAUDE.md` e o `AGENTS.md` (mesmo conteudo) com:

```markdown
# [Nome da Empresa]

## Resumo
[2-3 frases: quem e, o que faz, momento atual]

## Recursos
Veja `links.md` na raiz desta pasta pra todos os links uteis.
[Liste aqui os principais inline: NotebookLM, Drive, site — pra ja entrarem no contexto cascateado.]

## Negocio
- **Segmento:** [X]
- **Produto/Servico:** [X]
- **Publico-alvo:** [X]
- **Diferenciais:** [X]

## Operacao
- **Canais ativos:** [X]
- **Investimento:** [X/mes]
- **Metricas atuais:** [CPC, CPL, ROAS, etc]
- **Problemas:** [X]
- **Oportunidades:** [X]

## Relacionamento
- **Contatos:** [nomes e funcoes]
- **Combinados:** [o que foi prometido/acordado]
- **Pendencias:** [entregas pendentes]

## Estrategia
- **Objetivos:** [X]
- **Teses atuais:** [X]
- **Historico:** [o que ja testaram]
- **Proximos passos:** [X]

## Notas Importantes
[Qualquer informacao critica que nao se encaixou acima]

## Quando trabalhar com este cliente
- Comece lendo `links.md` pra saber dos recursos disponiveis.
- Se o usuario compartilhar um link util durante a conversa, pergunte se quer adicionar a `links.md`.
```

**Se for SQUAD:**

Extraia do `README.md`: nome do squad, membros (nome + funcao). Extraia de `docs/`: acordos do squad, processos, links uteis. Liste os clientes filhos (so os nomes das pastas).

Gere o `CLAUDE.md` e o `AGENTS.md` (mesmo conteudo) com:

```markdown
# Squad [Nome]

## Membros
- [Nome — Funcao]
- ...

## Clientes
- [nome-formatado-da-pasta]
- ...

## Acordos e processos
[Sintese do que esta em docs/. Se vazio, "Nada documentado ainda — adicione em docs/."]

## Notas Importantes
[Qualquer info critica que nao se encaixa acima]
```

**Se for PROJETO/AREA (generico):**

Extraia: nome, objetivo, pessoas, responsabilidades, dados, metricas, processos, workflows, problemas, oportunidades, decisoes, pendencias.

Gere o `CLAUDE.md` e o `AGENTS.md` (mesmo conteudo) com:

```markdown
# [Nome do Projeto/Area]

## Resumo
[2-3 frases: o que e, qual o objetivo, momento atual]

## Contexto
- **Objetivo:** [X]
- **Pessoas envolvidas:** [nomes e papeis]
- **Status atual:** [X]

## Dados
- **Metricas principais:** [o que foi encontrado nos dados]
- **Fontes:** [de onde vem os dados]

## Processos
- **Workflows identificados:** [o que a area faz]
- **Ferramentas usadas:** [se mencionado]

## Situacao Atual
- **Problemas:** [X]
- **Oportunidades:** [X]
- **Decisoes tomadas:** [X]
- **Pendencias:** [X]

## Notas Importantes
[Qualquer informacao critica que nao se encaixou acima]
```

### Passo 5 — Apresentar ao usuario

Mostre um resumo do que encontrou e os arquivos gerados. Pergunte:
- "Tem algo que eu errei ou que falta?"
- "Quer adicionar alguma informacao que nao estava nos arquivos?"

Ajuste conforme o feedback.

### Passo 6 — Confirmar

Salve e diga:
> "Pronto. Agora toda vez que voce trabalhar nessa pasta, a IA vai ler esse contexto automaticamente. Se os dados mudarem, rode `/contexto` de novo pra atualizar."

## Regras

- NAO invente informacoes. Se nao encontrou algo, deixe como "[nao disponivel]".
- Se a KB estiver vazia ou quase vazia, avise e sugira quais dados adicionar.
- Priorize fatos sobre interpretacoes.
- Mantenha os arquivos concisos — maximo 150 linhas.
- Em pasta de squad, NUNCA leia o conteudo de pastas de clientes filhos. Cada cliente tem seu proprio CLAUDE.md.
