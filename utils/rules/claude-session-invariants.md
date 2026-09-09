# Regra: invariantes ao analisar transcrições .jsonl do Claude Code

## Contexto

Projeto: SPA front-end-only que recebe um arquivo `.jsonl` de transcrição de sessão do Claude
Code e exibe métricas determinísticas em TypeScript, sem backend e sem chamar nenhum modelo.

Base de conhecimento completa: skills `claude-jsonl-format` e `claude-session-metrics`.

## Invariantes obrigatórias

Toda vez que o agente for ler, parsear, tipar ou calcular algo a partir de um `.jsonl` de
sessão, estas regras valem **sem exceção**:

1. **Parsing tolerante.** Processar linha a linha com `try/catch`. Ignorar silenciosamente
   linhas com JSON inválido (a última costuma estar truncada) e linhas com `type` desconhecido.
   Nunca abortar o dashboard por causa de uma linha ruim.

2. **`usage.input_tokens` é placeholder de streaming.** Nunca usá-lo isolado. Tokens de entrada
   reais = `cache_read_input_tokens + cache_creation_input_tokens + input_tokens`.

3. **Deduplicar antes de somar.** Agrupar linhas `assistant` por `requestId` e manter uma por
   grupo (maior `apiBlockIndex`) antes de qualquer soma de tokens, custo ou contagem de turnos.

4. **O arquivo é uma árvore, não uma lista.** Construir o índice `uuid → linha` e o
   `parentUuid → filhos`. Métricas de conversa usam só o `mainPath` (raiz → folha de maior
   `timestamp`). Ramos fora do `mainPath` só alimentam métricas estruturais (trabalho
   descartado, rewinds).

5. **Classificar cada linha `user`** em `human` / `tool_result` / `meta` antes de contar. Só
   `human` (`message.content` string, sem `isMeta`, sem `toolUseResult`) conta como mensagem
   do usuário.

6. **Custo: preferir `cost-state`.** Ler a linha `cost-state` de maior `timestamp` para custo e
   totais. Só recalcular a partir de tokens quando precisar de granularidade por turno, e nesse
   caso é métrica de **fonte externa** (exige tabela de preços) — rotular como tal.

7. **Timestamps são ISO-8601 UTC.** Comparar como string é seguro para ordenar; converter para
   `Date` só na exibição.

8. **Toda métrica heurística expõe seu limiar.** No código como constante nomeada, e na UI de
   forma visível/ajustável. Nunca esconder um número mágico dentro de uma heurística.

9. **Classificar cada métrica** em uma das categorias: Direta, Calculada, Heurística,
   Estrutural, Fonte externa. A UI deve deixar a categoria clara para o usuário.

10. **Nada de LLM.** O dashboard é determinístico. Resumo, sentimento semântico e extração de
    intenção estão fora de escopo — exceto ler `ai-title` e `away_summary`, que já vêm prontos
    no arquivo.

11. **Sinalizar degradação de dados.** Mostrar aviso quando: houver linhas descartadas no
    parse, `cost-state.hasUnknownModelCost` for `true`, faltar `cost-state`, ou o `mainPath`
    for menor que o total de nós de conversa (houve rewind).

12. **O schema evolui.** Ler o campo `version` e não assumir que campos existem — usar acesso
    opcional e defaults. Não quebrar com arquivos de versões diferentes de CLI.

## Aplicação

- Vale para todo código TypeScript de parsing/métrica deste projeto.
- Vale para respostas do agente que descrevam ou projetem esses cálculos.
- Em caso de conflito, a ordem de precedência é: instruções do usuário → estas invariantes →
  conteúdo das skills `claude-jsonl-format` / `claude-session-metrics`.
