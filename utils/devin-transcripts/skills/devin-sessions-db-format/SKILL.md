---
name: devin-sessions-db-format
description: >-
  Base de conhecimento sobre o `sessions.db` (SQLite) da Devin CLI — o registro local que
  sempre existe para toda sessão, independente de flags de export. Cobre localização por SO,
  tabelas conhecidas, colunas, modo WAL e como abrir o arquivo com segurança (read-only)
  enquanto a CLI pode estar rodando. Use ao ler, consultar ou tipar dados vindos desse arquivo.
---

# `sessions.db` — armazenamento local sempre-ativo da Devin CLI

## 0. Nível de confiança desta skill

**Schema não-oficial e não-documentado publicamente.** Tudo abaixo vem de uma pesquisa
preparatória feita fora deste ambiente, cruzando informações espalhadas sobre o formato — não
de inspeção direta de um `sessions.db` real (a Devin CLI não estava instalada na máquina onde
essa pesquisa foi feita). Nomes de coluna e tabela **podem estar incompletos, desatualizados ou
variar entre versões da CLI**. Antes de depender de uma coluna em produção:

```sql
PRAGMA table_info(sessions);
PRAGMA table_info(message_nodes);
```

e ajuste o parser ao que a versão real do usuário realmente tem. Ver `devin-session-invariants`
regra 9.

**Ambiente corporativo — leia antes de usar esta skill.** Esta pesquisa foi feita uma única vez,
fora do ambiente corporativo, porque esse ambiente não permite consultas a referências externas
nem instalação de novas dependências. Este documento é a referência **congelada** resultante
dessa pesquisa — não repita a consulta externa; se faltar informação, sinalize a lacuna ao
usuário. **Importante:** ao contrário do ATIF (JSON puro), ler `sessions.db` (binário SQLite)
sem nenhuma dependência nova é significativamente mais difícil — ver §4 abaixo e `relatorio.md`
§2 para o impacto real disso na escolha de arquitetura.

## 1. O que é e por que é a fonte "sempre disponível"

- Ao contrário do ATIF (`--export`, opt-in), o `sessions.db` é escrito **para toda sessão**,
  automaticamente — é o equivalente funcional do diretório `~/.claude/projects/**/*.jsonl` do
  Claude Code: a fonte que sempre existe, sem o usuário precisar lembrar de nenhuma flag.
- É um banco **SQLite em modo WAL** (write-ahead log). Isso implica:
  - podem existir arquivos irmãos `sessions.db-wal` e `sessions.db-shm` com dados ainda não
    consolidados no arquivo principal;
  - se for copiar o banco para análise offline, copie os três arquivos juntos (ou force um
    checkpoint com `PRAGMA wal_checkpoint(TRUNCATE);` antes de copiar só o `.db`);
  - **sempre abra em modo somente-leitura** (`file:sessions.db?mode=ro`), nunca escreva —
    a CLI pode estar com o arquivo aberto ao mesmo tempo.
- Há indicação de que o schema evolui entre versões e de que migrações são versionadas — uma
  CLI mais antiga pode não conseguir abrir corretamente um banco escrito por uma versão mais
  nova. Trate isso como confirmação adicional de que o schema não deve ser tratado como
  estável entre versões.

## 2. Localização por sistema operacional

| SO | Caminho |
|---|---|
| Linux / macOS (POSIX) | `$XDG_DATA_HOME/devin/cli/sessions.db`, default `~/.local/share/devin/cli/sessions.db` |
| Windows | **Não confirmado para `sessions.db`.** O único caminho Windows conhecido com confiança é o do arquivo de configuração de usuário, em `%APPDATA%\devin\`. Por analogia com a mesma convenção de diretórios, o candidato mais provável para os dados de sessão é `%APPDATA%\devin\cli\sessions.db`, mas **valide no disco do usuário antes de hardcodar** — inclusive a possibilidade de o usuário rodar a Devin CLI dentro do WSL, caso em que o caminho é o POSIX acima dentro do filesystem da distro. |
| Layout anterior a uma migração de diretórios conhecida | `~/.config/<nome-antigo>/`, `~/.local/share/<nome-antigo>/`, `~/.cache/<nome-antigo>/` — versões antigas da CLI usavam um nome de diretório diferente, migrado depois para `devin/`. Um usuário com instalação antiga pode ter os dois layouts coexistindo até atualizar a CLI. |

A árvore de dados pode agrupar múltiplos "harnesses" (`cli`, e possivelmente outros nomes em
versões futuras/diferentes) como `<data_dir>/devin/<harness>/sessions.db` — ao procurar o
arquivo, não assuma que `cli` é a única pasta possível.

## 3. Tabelas conhecidas

### 3.1. `sessions`

Uma linha por sessão (equivalente a um arquivo `.jsonl` inteiro no mundo Claude).

| Coluna | Significado |
|---|---|
| `id` | Chave primária da sessão. Também é a chave de junção com o `session_id` de um arquivo ATIF exportado dessa sessão. |
| `title` | Título da sessão (equivalente ao `ai-title.aiTitle` do Claude) |
| `model` | Modelo usado (fallback quando não há granularidade por turno) |
| `working_directory` | Diretório de trabalho da sessão — usado para derivar nome/projeto |
| `created_at` | Timestamp de criação |
| `last_activity_at` | Timestamp da última atividade — preferir a este sobre `created_at` para "quando foi essa sessão" |
| `hidden` | Flag booleana — sessões ocultas devem ser filtradas por padrão em listagens |
| `metadata` | Possível coluna adicional contendo `total_acu_cost` (possivelmente JSON) — **presença e formato não confirmados de forma independente**, tratar como opcional |

### 3.2. `message_nodes`

Conteúdo da conversa. Assim como o `.jsonl` do Claude Code via `uuid`/`parentUuid`, há indicação
de que a conversa da Devin CLI **não é uma sequência linear**: comandos como `/fork [step]` e
`/revert <step>` criam ramificações e reescritas de histórico, e o modelo de dados
provavelmente reflete isso (descrito como uma árvore, não uma lista).

| Coluna | Significado |
|---|---|
| `chat_message` | JSON com `{ role, content, tool_calls }` — o payload real da mensagem |
| (coluna de ligação com `sessions`, nome exato não confirmado) | provável `session_id` ou equivalente — confirme com `PRAGMA table_info` |
| (colunas de parentesco/ordem, nomes exatos não confirmados) | necessárias para reconstruir a árvore e identificar o "caminho real" pós-`/fork`/`/revert`, análogo ao `mainPath` do Claude — **não hardcode nomes de coluna aqui sem inspecionar o arquivo real** |

`chat_message.role` conhecidos: `user`, `assistant`, `tool` (possivelmente com saída truncada
em algum limite de caracteres — não confirmado como limite da própria Devin), `system`
(tipicamente ignorado por ser boilerplate).

Há indicação de que metadados de uso também podem aparecer dentro do próprio `chat_message`
JSON (`metadata.metrics`, com os mesmos 4 contadores de token do ATIF) — ou seja, os tokens por
turno podem estar disponíveis tanto via `sessions.db.message_nodes.chat_message` quanto via
ATIF, dependendo da versão da CLI. Trate como redundância bem-vinda, não como garantia dupla.

### 3.3. `tool_call_state` e `prompt_history`

Existência provável, **schema interno não documentado**. Presumivelmente guardam estado de
execução de tool calls em andamento e histórico de prompts enviados. Não construa lógica de
produto em cima delas sem antes rodar `PRAGMA table_info` e inspecionar linhas reais.

## 4. Padrão de acesso — atenção à restrição de "sem dependências novas"

O exemplo abaixo usa apenas uma API de SQLite **nativa do runtime**, sem instalar nenhum
pacote, e serve para scripts internos rodando fora do browser, onde essa API já esteja
disponível:

```ts
// Sempre read-only. Sempre WAL-aware. Nunca escrever.
const db = openReadOnly(`file:${sessionsDbPath}?mode=ro`);

// Descobrir o schema real antes de assumir colunas:
const sessionCols = db.query(`PRAGMA table_info(sessions)`);
const nodeCols = db.query(`PRAGMA table_info(message_nodes)`);

// Sessões visíveis, mais recentes primeiro:
const sessions = db.query(
  `SELECT * FROM sessions WHERE hidden = 0 ORDER BY last_activity_at DESC`
);
```

**Qualquer biblioteca de SQLite que não seja nativa do runtime é um pacote externo — não usar
sem aprovação explícita do usuário** (regra 12 de `devin-session-invariants`). Confirme sempre
se a API nativa realmente está disponível na versão do runtime em uso antes de assumir que
existe no ambiente de execução.

**No dashboard front-end only (sem backend), a situação é mais restritiva.** Ler um arquivo
SQLite binário inteiramente no browser sem nenhuma dependência nova **não tem solução pronta e
leve** — a forma mais comum de resolver isso no mercado é justamente incorporar uma dependência
nova, e por isso está sujeita à mesma restrição de aprovação. As alternativas realistas são:

1. **Pedir aprovação explícita ao usuário** para incorporar uma biblioteca de terceiros como
   exceção documentada — só depois disso tratar como parte da arquitetura padrão.
2. **Escrever um leitor mínimo do formato de arquivo SQLite à mão**, sem dependências —
   tecnicamente viável (o formato de página/B-tree do SQLite é uma especificação pública e
   estável), mas é esforço de engenharia real, não um parser trivial como o do ATIF/JSON.
3. **Não suportar `sessions.db` no dashboard nesta fase** e depender só do ATIF (`--export`),
   que não exige nenhuma dependência nova (`JSON.parse` nativo já resolve).

Ver `relatorio.md` §2 para a análise completa de custo/benefício dessas três alternativas.

## 5. Como usar esta skill

1. `sessions.db` é a fonte **conceitualmente** primária/sempre-disponível, mas dado o
   ambiente corporativo (sem novas dependências), **o ATIF (`devin-atif-format`) é o caminho
   praticamente viável para o dashboard nesta fase** — ver §4. Trate o suporte a `sessions.db`
   como incremento condicionado a uma decisão explícita (aprovar dependência ou escrever
   parser próprio).
2. Sempre `PRAGMA table_info` antes de assumir uma coluna existe — o schema muda entre versões
   e nem todo campo citado aqui foi confirmado de forma independente.
3. Sempre abrir read-only; nunca escrever no arquivo do usuário.
4. Considerar os arquivos `-wal`/`-shm` ao copiar o banco para fora do ambiente do usuário.
5. `message_nodes` é uma árvore, não uma lista — não assuma ordem cronológica linear sem
   confirmar as colunas de parentesco/ordem reais.
6. Para o catálogo de métricas e como combiná-las com o ATIF, ver `devin-session-metrics`.
7. Nunca adicionar uma biblioteca de SQLite de terceiros ao projeto sem aprovação explícita do
   usuário (regra 12 de `devin-session-invariants`).
