# Relatório: viabilidade de extrair métricas locais de sessões da Devin CLI

> Objetivo: avaliar como estender o dashboard front-end-only (HTML/CSS/TS) que hoje lê
> transcrições `.jsonl` do Claude Code para também suportar dados de sessão da **Devin CLI**,
> mantendo a mesma filosofia — sem backend, sem chamar nenhum modelo, tudo processado no
> browser a partir de um arquivo local.

---

## 1. O que existe localmente, em resumo

A Devin CLI grava dois artefatos locais possíveis, com naturezas bem diferentes:

| | `sessions.db` | Transcript ATIF (`--export`) |
|---|---|---|
| Formato | SQLite (modo WAL) | JSON (um documento por sessão) |
| Quando existe | **Sempre**, para toda sessão | Só se a sessão foi iniciada com a flag `--export` |
| Escopo | Todas as sessões do usuário (multi-sessão) | Uma sessão por arquivo |
| Granularidade | Metadados de sessão + árvore de mensagens (`message_nodes`) | Metadados de sessão + **métricas por step** (tokens, ACU, timing) |
| Unidade de custo | ACU (possivelmente pré-agregado em metadados da sessão) | ACU por step (`committed_acu_cost`) |
| Documentação oficial do schema | Nenhuma (schema não publicado) | Parcial (formato citado, sem especificação pública completa) |
| Caminho (Linux/macOS) | `~/.local/share/devin/cli/sessions.db` | `~/.local/share/devin/cli/transcripts/<id>.json` (default) ou caminho custom |
| Caminho (Windows) | Não confirmado oficialmente (inferido: `%APPDATA%\devin\cli\sessions.db`) | Idem, inferido |

Isso é uma inversão em relação ao Claude Code, onde existe **uma única fonte** (`.jsonl`),
sempre completa e sempre presente. Na Devin CLI, a fonte "sempre presente" (`sessions.db`) é a
menos granular, e a mais granular (ATIF) é opt-in.

Detalhes completos de schema, campos e armadilhas estão nas skills criadas neste projeto:
`skills/devin-sessions-db-format/SKILL.md` e `skills/devin-atif-format/SKILL.md`.

---

## 2. Opções viáveis de extração programática, local, front-only

### Opção A — Ler o transcript ATIF (JSON) diretamente no browser ⭐ recomendada como caminho principal

**Como funciona:** o usuário roda `devin --export sessao.json -- <prompt>` (ou já tem uma
sessão exportada), e solta o `.json` no dashboard, exatamente como já faz hoje com o `.jsonl`
do Claude Code.

**Viabilidade:** total. É um arquivo JSON de texto puro — `JSON.parse` nativo no browser, sem
nenhuma biblioteca adicional. É o caminho de menor esforço de engenharia e o que mais se parece
com o pipeline que já existe (`File → text → parse → métricas`).

**Limitações:**
- Exige que o usuário lembre de usar `--export` **no início** da sessão — não é retroativo.
- Só cobre uma sessão por arquivo (sem visão agregada/multi-sessão).
- O formato não tem especificação pública completa; campos podem faltar ou mudar entre versões.

**Quando usar:** é o caminho ideal para quem já adota o hábito de exportar, e o mais simples de
entregar primeiro (MVP), pelos mesmos motivos que o parser do `.jsonl` do Claude foi o ponto de
partida daquele dashboard.

### Opção B — Ler `sessions.db` (SQLite) diretamente no browser

**Como funciona:** o usuário seleciona o arquivo `sessions.db` (via seletor de arquivo do
próprio browser), o dashboard carrega os bytes em memória e precisa interpretar o formato
binário do SQLite para extrair as tabelas relevantes.

A forma mais comum de resolver isso é compilar o motor SQLite para rodar no browser — mas isso
é, por definição, uma **dependência nova** no projeto, e a restrição do ambiente corporativo
(seção 0) bloqueia isso **por padrão**. Por isso esta opção se desdobra em duas variantes,
nenhuma das quais é "plug and play" hoje:

**B1 — motor SQLite de terceiros, mediante aprovação explícita.** Tecnicamente é a melhor
relação esforço/resultado (solução madura, roda inteiramente no navegador sem chamadas de
rede depois de incorporada ao projeto). **Só deve ser considerada se o usuário aprovar
explicitamente a adição dessa dependência como exceção documentada.** Sem essa aprovação, esta
variante está fora de escopo.

**B2 — Parser SQLite escrito à mão, zero dependências.** O formato de arquivo do SQLite
(cabeçalho, páginas, B-tree, formato de registro) é uma especificação pública e estável — é
possível ler as tabelas `sessions` e `message_nodes` sem nenhuma dependência externa,
decodificando os bytes do arquivo diretamente. Isso é **viável em teoria**, mas é um projeto de
engenharia relevante por si só (não é "mais um parser de texto" como o ATIF) e deve ser tratado
como tal: escopo próprio, estimativa própria, e sujeito a validação cuidadosa antes de confiar
no resultado. Não é recomendado como parte do MVP.

**Vantagens da Opção B (em qualquer variante), quando viabilizada:**
- **Sempre disponível** — não depende do usuário ter usado `--export`.
- **Multi-sessão de graça** — o mesmo arquivo tem todas as sessões do usuário, permitindo
  agregados que hoje são impossíveis no dashboard do Claude sem carregar dezenas de arquivos
  (ex.: "quais projetos mais consomem ACU", "sessões por semana").

**Limitações adicionais (além do problema de dependência):**
- Schema não-oficial: nomes de tabela/coluna podem variar entre versões da CLI. O parser
  precisa inspecionar a estrutura em runtime e degradar graciosamente.
- Arquivo é binário e cresce — para sessões muito longas ou históricos extensos, o custo de
  carregar tudo em memória no browser é maior que um `.jsonl`/`.json` de texto.
- O dado pode estar levemente desatualizado se houver uma sessão WAL não consolidada no momento
  em que o usuário copiou o arquivo. Mitigação: orientar o usuário a fechar a Devin CLI antes
  de exportar o arquivo para análise.

**Quando usar:** só depois de uma decisão explícita do usuário sobre qual variante (B1 ou B2)
seguir — nenhuma das duas deve ser assumida como "o próximo passo natural" sem essa conversa.

### Opção C — Combinar A + B (arquitetura de referência para o futuro, não para agora)

Detectar o tipo de arquivo solto pelo usuário (assinatura de bytes no início do arquivo binário
vs. JSON válido com `steps[]`) e rotear para o parser correto. Quando ambos estão disponíveis
para a mesma sessão (mesmo identificador de sessão), usar o ATIF para granularidade por step e
o `sessions.db` para os metadados agregados e para a navegação multi-sessão.

Essa é a arquitetura desejável a longo prazo, mas **depende inteiramente de a Opção B ter sido
viabilizada primeiro** (aprovação de dependência ou parser próprio) — não é algo a planejar em
detalhe antes dessa decisão.

### Opção D — Script auxiliar de pré-exportação (fora do escopo "front-only", mencionar como alternativa)

Um pequeno script rodado localmente pelo usuário (fora do dashboard) para consolidar
`sessions.db` num `.json` "achatado". Isso também tem uma restrição relevante: só pode usar
capacidades nativas do runtime, sem instalar bibliotecas externas de terceiros — a mesma regra
de aprovação de dependências se aplica aqui também. Além disso, **essa opção quebra a premissa
de "front-end only"** do dashboard e não é a recomendação primária; vale documentar como plano
B caso a Opção B se mostre inviável na prática (nem B1 aprovada, nem B2 com esforço
justificável).

### O que **não** é viável / não é recomendado

- **Depender de uma taxa ACU→USD embutida no dashboard.** Não existe taxa pública nem padrão;
  qualquer "US$" mostrado sem o usuário informar a taxa seria um número inventado. O dashboard
  deve mostrar ACU como unidade primária e oferecer um campo opcional "taxa ACU→USD (definida
  por você)" para quem quiser ver a conversão — nunca embutir um valor default.
- **Fazer scraping da sessão via aplicação web/cloud do produto.** Sai do escopo "dados
  locais" e do modelo "front-end only sem autenticação"; é uma superfície de manutenção e de
  risco (mudanças de interface/autenticação) desnecessária quando os dados já existem
  localmente — além de normalmente exigir chamadas de rede que o ambiente corporativo também
  não permite.
- **Assumir caminho fixo do `sessions.db` no Windows.** Não confirmado oficialmente — pedir ao
  usuário para selecionar o arquivo é mais robusto que tentar adivinhar um caminho absoluto.
- **Consultar qualquer referência externa durante a implementação.** Essa consulta já foi feita
  (uma única vez) para produzir este relatório e as skills associadas; repeti-la no ambiente
  corporativo não é uma opção — ver seção 0.

---

## 3. Comparação de esforço de implementação

| | Opção A (ATIF) | Opção B1 (`sessions.db` + motor de terceiros) | Opção B2 (`sessions.db`, parser próprio) |
|---|---|---|---|
| Parser novo | Pequeno (`JSON.parse` + tipos) | Médio (integrar o motor, queries, introspecção de schema) | Grande (decodificar formato binário SQLite do zero) |
| Dependência nova no bundle | **Nenhuma** | Uma dependência de terceiros — **requer aprovação explícita** | **Nenhuma** |
| Compatível com ambiente corporativo hoje, sem decisão adicional | **Sim** | **Não** — bloqueado até aprovação | Sim, mas alto custo de engenharia |
| Cobertura (usuários que conseguem usar sem preparo prévio) | Baixa — exige ter usado `--export` | Alta — funciona com qualquer instalação padrão | Alta — funciona com qualquer instalação padrão |
| Granularidade de métricas | Alta (por step, com tokens e ACU) | Média (por sessão; por-mensagem depende de decifrar `message_nodes`) | Igual à B1, mesmo trabalho de decifrar `message_nodes` |
| Multi-sessão | Não | Sim | Sim |
| Risco de quebrar por mudança de schema | Baixo (JSON tolerante a campo ausente) | Médio-alto (schema não documentado, migra entre versões) | Médio-alto (mesmo risco de schema + risco de bug no parser binário) |

**Recomendação de sequência, já considerando as restrições do ambiente corporativo:**

1. **Implementar a Opção A primeiro.** É a única opção que não depende de nenhuma decisão
   externa (dependência ou orçamento de engenharia) — MVP rápido, baixo risco, zero
   dependências, e já entrega valor real (métricas por step de uma sessão exportada).
2. **Não iniciar a Opção B sem antes decidir entre B1 e B2 com o usuário.** Essa é uma decisão
   de produto/arquitetura (aceitar uma dependência nova vs. investir em um parser próprio), não
   uma escolha técnica que o agente deva fazer sozinho.
3. Só depois dessa decisão, tratar `sessions.db` como parte do escopo, com sua própria
   validação contra um arquivo real (ver seção 5).

---

## 4. Recorte de MVP sugerido (mesma filosofia do MVP do dashboard do Claude)

Seguindo o mesmo princípio já aplicado (`mvp-metricas-dashboard.md` do projeto Claude): o MVP
fica inteiramente em métricas **Direta + Calculada**, sem taxa de câmbio, sem heurística de
texto livre (o formato de `chat_message.content` não está confirmado), sem histórico
multi-arquivo na v1.

| Bloco | Métricas | Fonte |
|---|---|---|
| Cabeçalho | Título, modelo, diretório de trabalho, duração de parede | `sessions.db.sessions` ou ATIF `agent.model_name` |
| Uso | ACU total (soma de steps ou total pré-agregado), tokens totais por tipo | ATIF, se disponível |
| Conversa | Nº de turnos do usuário vs. do agente, nº de tool calls, histograma por `function_name` | ATIF |
| Qualidade do dado | Aviso se não houver ATIF (métricas por-step indisponíveis), aviso se coluna de custo não existir na versão do `sessions.db` do usuário | ambos |

Ficam fora do MVP: custo em USD (fonte externa, sem taxa padrão), qualquer heurística sobre
texto de mensagens (schema de conteúdo não confirmado), métricas estruturais de fork/revert
(colunas de parentesco não confirmadas), e **toda a visão multi-sessão / `sessions.db`**, que
neste ambiente depende de uma decisão explícita ainda não tomada (Opção B1 vs. B2, seção 2) —
não é apenas "adiada por prioridade", é **bloqueada por política** até essa decisão.

---

## 5. Antes de implementar em produção

Nenhum dos passos abaixo envolve consultar referências externas — são validações feitas contra
arquivos reais que o usuário fornece, dentro do próprio ambiente corporativo.

1. **Obter um `sessions.db` e um transcript ATIF reais** (rodando a Devin CLI localmente ou
   pedindo uma amostra ao usuário) e validar cada nome de tabela/coluna/campo citado nas skills
   `devin-sessions-db-format` e `devin-atif-format` — hoje eles são hipótese, não fato.
2. Inspecionar o schema real de cada tabela relevante (usando apenas capacidades nativas do
   runtime, sem instalar nada, para essa inspeção pontual — não como parte do dashboard
   front-end) e ajustar os tipos TS de acordo com o que for encontrado, incluindo as colunas de
   parentesco de `message_nodes` que não puderam ser confirmadas nesta pesquisa.
3. Confirmar experimentalmente o comportamento de `--export` sem caminho (onde exatamente cai
   o arquivo default) e se `/export` dentro de uma sessão já iniciada tem algum efeito.
4. **Antes de escrever qualquer código para a Opção B, levar a decisão B1 vs. B2 ao usuário**
   explicitamente (aprovar uma dependência nova como exceção, ou investir em parser próprio) —
   não decidir isso unilateralmente durante a implementação.
5. Só depois disso, tratar o suporte à Devin CLI como "suportado" no dashboard — até lá, tratar
   como **beta/experimental** na UI, sinalizando isso ao usuário.