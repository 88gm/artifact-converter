# Regra: invariantes ao analisar dados locais de sessão da Devin CLI

## Contexto

Projeto: extensão de um dashboard front-end-only (HTML/CSS/TS) — hoje funcional apenas para
transcrições `.jsonl` do Claude Code — para também ler dados locais de sessão da **Devin CLI**
(`sessions.db` e/ou transcript ATIF exportado via `--export`).

Base de conhecimento completa: skills `devin-sessions-db-format`, `devin-atif-format` e
`devin-session-metrics`.

## Invariantes obrigatórias

Toda vez que o agente for ler, parsear, tipar ou calcular algo a partir de dados locais da
Devin CLI, estas regras valem **sem exceção**:

1. **Duas fontes independentes, não intercambiáveis.** `sessions.db` sempre existe (toda
   sessão); o transcript ATIF só existe se a sessão foi rodada com `--export`. Nunca assuma que
   um usuário terá ATIF disponível — o fluxo precisa funcionar só com `sessions.db`.

2. **`sessions.db` é sempre aberto read-only.** Use `mode=ro` (ou equivalente da lib usada) e
   nunca escreva no arquivo. O banco está em modo WAL e pode estar aberto pela própria Devin
   CLI ao mesmo tempo; considerar os arquivos irmãos `-wal`/`-shm` ao copiar o banco para fora
   do ambiente do usuário.

3. **Schema do `sessions.db` não é oficialmente documentado.** Nomes de tabela/coluna vieram de
   engenharia reversa de terceiros, não de inspeção direta de um arquivo real. Antes de
   depender de uma coluna, rodar `PRAGMA table_info(<tabela>)` e degradar graciosamente
   (esconder a métrica, não quebrar o parser) se a coluna não existir.

4. **`message_nodes` é uma árvore, não uma lista.** Comandos `/fork` e `/revert` criam
   ramificações — análogo ao `uuid`/`parentUuid` do Claude Code. As colunas exatas de
   parentesco não estão confirmadas: nunca hardcode um nome de coluna de "parent" sem
   confirmá-lo no arquivo real do usuário primeiro.

5. **ACU não é USD.** O `committed_acu_cost` (ATIF) e qualquer `total_acu_cost` (`sessions.db`)
   estão em Agent Compute Units, uma unidade de uso interna da própria Devin CLI. **Não existe
   taxa de conversão ACU→USD local nem padrão.** Nunca exiba um valor em dólar calculado a
   partir de ACU sem que o usuário tenha fornecido a taxa explicitamente (nem hardcode uma taxa
   "típica" — isso seria inventar um número que parece factual).

6. **`--export` não é automático.** A flag precisa estar presente desde o início da sessão
   (`devin --export [PATH] -- prompt`) para o ATIF existir. Não assuma que rodar `/export`
   dentro de uma sessão já em andamento retroativamente gera o arquivo — o comportamento
   observado é de exibição de informação, não de ativação.

7. **Caminho do `sessions.db` no Windows não está confirmado.** Só o caminho do arquivo de
   configuração de usuário é conhecido com confiança. Para localizar `sessions.db`/
   `transcripts/` num app front-end, peça ao usuário para selecionar o arquivo manualmente em
   vez de tentar adivinhar/hardcodar um caminho absoluto — isso também cobre o caso de o
   usuário rodar a Devin CLI dentro do WSL, onde o caminho seria o POSIX padrão dentro do
   filesystem da distro.

8. **`schema_version` do ATIF não tem validação confirmada.** Não trave o parser por causa
   dela; trate campos como opcionais com defaults seguros, igual à postura já adotada para o
   campo `version` do `.jsonl` do Claude Code.

9. **Esta base de conhecimento não foi validada contra um arquivo real.** Foi compilada numa
   pesquisa preparatória única, feita fora deste ambiente, não a partir de inspeção direta de
   um `sessions.db`/ATIF real (a Devin CLI não estava instalada na máquina onde essa pesquisa
   foi feita). Ao implementar qualquer parser de produção, a primeira tarefa é sempre confirmar
   nomes de tabela/coluna/campo contra um arquivo real fornecido pelo usuário — nunca declarar
   "suportado" sem essa validação.

10. **Nunca inventar um número quando o dado está ausente.** Se uma coluna/campo esperado não
    existir na versão do usuário, mostrar "indisponível" na UI — nunca default para `0` (que
    parece um valor real) ou omitir silenciosamente a métrica sem aviso.

11. **Ambiente corporativo: sem consultas externas em tempo de implementação.** Este projeto
    roda num ambiente sem acesso liberado a referências externas para esse tipo de tarefa. As
    skills `devin-sessions-db-format`, `devin-atif-format` e `devin-session-metrics` já contêm
    toda a pesquisa necessária (feita uma única vez, fora deste ambiente, antes de virar
    documentação congelada). **Nunca** usar ferramentas de busca/rede para "confirmar" algo
    sobre a Devin CLI durante o trabalho normal neste projeto. Se a skill não cobrir um caso,
    sinalizar a lacuna ao usuário — não tentar buscar a informação por conta própria.

12. **Ambiente corporativo: nenhuma dependência nova sem aprovação explícita.** Não adicionar
    pacotes, bibliotecas compiladas ou qualquer dependência externa ao projeto sem o usuário
    aprovar explicitamente, caso a caso. Isso invalida por padrão qualquer caminho de
    implementação que exija instalar algo novo — ver `relatorio.md` §2 (Opção B) para o impacto
    direto disso na leitura de `sessions.db`. Na dúvida, prefira a solução com zero
    dependências, mesmo que exija mais código próprio, e deixe claro para o usuário quando uma
    métrica só é viável adicionando uma dependência.

## Aplicação

- Vale para todo código TypeScript de parsing/métrica que leia `sessions.db` e/ou transcripts
  ATIF neste projeto.
- Vale para respostas do agente que descrevam ou projetem esses cálculos.
- Em caso de conflito, a ordem de precedência é: instruções do usuário → estas invariantes →
  conteúdo das skills `devin-sessions-db-format` / `devin-atif-format` / `devin-session-metrics`.
