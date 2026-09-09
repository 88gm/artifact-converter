# MVP do dashboard — quais métricas exibir e por quê

> Recorte de escopo para a **primeira versão** do SPA que lê um `.jsonl` de sessão do Claude
> Code. Base: skills `claude-jsonl-format` e `claude-session-metrics`.
> Legenda de categoria: **D** Direta · **C** Calculada · **E** Estrutural · **H** Heurística · **X** Fonte externa.

---

## Princípio do recorte

O MVP fica **inteiramente em D + C + E**: métricas que saem de um único arquivo, sem tabela de
preços, sem tokenizer, sem histórico, sem LLM. Isso garante:

1. **Zero dependência externa** — o app funciona 100% offline, só com o arquivo dropado.
2. **Nada de calibração** — nenhuma heurística com limiar discutível para a v1 (heurísticas
   entram na v2, atrás de sliders).
3. **Confiança total no número** — se alguém questionar, dá para apontar o campo exato do JSONL.

Ficam **de fora do MVP**: custo recomputado por turno (X), % de janela de contexto (X),
percentis vs. histórico (X), score de frustração (H), detecção de "travado" (H), TDD (H),
abandono (H), segmentação em tarefas (H).

---

## As 12 métricas do MVP

### Bloco A — "O que foi essa sessão?" (cabeçalho + tiles)

| # | Métrica | Cat | Origem | Por que está no MVP |
|---|---|---|---|---|
| 1 | **Duração de parede** (início → fim) | C | `max(timestamp) − min(timestamp)` | Primeira pergunta de qualquer um: quanto tempo isso levou. Uma subtração de timestamps ISO, à prova de erro. |
| 2 | **Custo total (US$)** | D | `cost-state.totalCostUSD` (última linha) | O CLI já calculou. É o número que mais interessa a quem paga a conta, e não exige tabela de preços porque já vem pronto. |
| 3 | **Modelos usados** | D | `distinct(assistant.message.model)` | Contexto essencial para ler o custo e a velocidade. Uma sessão que usou só Haiku vs. só Sonnet conta histórias diferentes. |
| 4 | **Turnos do assistente × prompts do usuário** | C | `dedupeAssistants().length` e `count(user == "human")` | Mede o "tamanho" da conversa e já força a implementação correta de dedup e classificação de linha `user` — a base de todo o resto. |
| 5 | **Linhas de código +/−** | D | `cost-state.totalLinesAdded / totalLinesRemoved` | Proxy direto de "quanto trabalho de código saiu daqui". Campo pronto, sem interpretação. |

### Bloco B — "Para onde foi o tempo e o esforço?"

| # | Métrica | Cat | Origem | Por que está no MVP |
|---|---|---|---|---|
| 6 | **Tempo ativo vs. ocioso** (barra dividida) | C | ativo = `Σ system[turn_duration].durationMs`; ocioso = parede − ativo | Distingue "sessão longa porque foi difícil" de "sessão longa porque fiquei almoçando". Ambos os lados vêm de campos medidos pelo harness. |
| 7 | **Duração por turno** (timeline de barras) | D | `system[subtype=turn_duration].durationMs` | É o gráfico central do dashboard. Mostra o ritmo da sessão e onde estão os turnos pesados. Valor cru, um por turno. |
| 8 | **Histograma de uso de ferramentas** | C | `Counter(tool_use.name)` no `mainPath` | Diz o estilo da sessão num relance (muito `Read` = exploração; muito `Bash` = execução/teste). Contagem simples de blocos `tool_use`. |
| 9 | **Razão leitura : escrita** | C | `(Read+Grep+Glob+ToolSearch) / (Edit+Write+NotebookEdit)` | Uma única métrica derivada da #8 que resume "explorou muito antes de mexer?". Barata e informativa. |

### Bloco C — "Deu tudo certo?"

| # | Métrica | Cat | Origem | Por que está no MVP |
|---|---|---|---|---|
| 10 | **Interrupções + permissões negadas** (contagem + marcadores na timeline) | D | `Σ assistant.isAbortedMidStream`, `Σ user.interruptedMessageId`, `Σ user.toolDenialKind` | Sinais de fricção **objetivos** (o usuário apertou stop / rejeitou uma ação) — não precisa de heurística. Alto valor diagnóstico. |
| 11 | **Erros de API** | D | `count(assistant.isApiErrorMessage)` | Overload / context-length aparecem marcados. Explica turnos estranhos e lentidão. Campo booleano explícito. |
| 12 | **Razão de linearidade** (houve rewind?) | E | `mainPath.length / totalNósDeConversa` (1.0 = sem rewind) + `nº de branch points` | O diferencial vs. `ccusage`. Revela retrabalho que uma leitura linear do arquivo esconderia. Só precisa do grafo `parentUuid`, que você já constrói para o `mainPath`. |

---

## Por que **não** cada um dos cortes

| Deixado de fora | Categoria | Motivo do corte no MVP |
|---|---|---|
| Custo por turno / por linha / por tarefa | X | Exige tabela de preços empacotada e mantida. O total (#2) já entrega o essencial. |
| % de ocupação da janela de contexto | X | Precisa do catálogo de janelas por modelo. Bom para a v2. |
| Percentil vs. histórico do usuário | X | Precisa de vários arquivos carregados — não existe na "primeira sessão dropada". |
| Score de frustração / turno de correção | H | Depende de dicionário de palavras e limiares em português/inglês. Fácil de errar e gerar falso positivo constrangedor. v2 com slider. |
| Detecção de "travado" (thrash) | H | Limiar arbitrário (nº de repetições, tamanho da janela). v2. |
| Sinal de TDD / higiene de git | H | Regex sobre comandos `Bash`; útil mas opinativo. v2. |
| Abandono | H | Definição frágil ("terminou logo após erro"). v2. |
| Curva de contexto / compactações | C | Determinística, mas exige um gráfico de área empilhada bem-feito — mais custo de UI do que valor no MVP. v1.1. |
| Árvore de subagentes | E | Só aparece em sessões com `Task`; a maioria não tem. Adiar até ter dados de teste com sidechain. |
| Tokens de raciocínio, thinking ratio | C/H | Interessante mas nicho. Entra junto com o painel de tokens detalhado. |

---

## Ordem de implementação sugerida

1. **Parser + fundação** (skill `claude-jsonl-format` §7): `parseTranscript`, índice `uuid`,
   `mainPath`, `dedupeAssistants`, `classifyUser`. Sem isso nenhuma métrica é confiável.
2. **Bloco A** (#1–5) — tiles estáticos, entrega valor imediato.
3. **Bloco B** (#6–9) — a timeline (#7) é o maior item de UI; #8 e #9 saem de graça do mesmo laço.
4. **Bloco C** (#10–12) — marcadores na timeline reaproveitam o componente do passo 3.
5. **Faixa de avisos de qualidade** (transversal): linhas ignoradas no parse,
   `cost-state.hasUnknownModelCost`, ausência de `cost-state`, `linearidade < 1`.

---

## Checklist de "pronto para v1"

- [ ] Todas as 12 métricas calculadas por funções puras, testáveis sem DOM.
- [ ] Dedup por `requestId` aplicado antes de qualquer contagem/soma (#2, #4).
- [ ] `input_tokens` nunca somado isolado (não há métrica de token no MVP, mas o parser já
      deve expor `inputReal` para a v2).
- [ ] Linhas `user` classificadas — #4 não conta `tool_result` nem `isMeta`.
- [ ] Cada tile/gráfico rotulado com a categoria (D/C/E).
- [ ] App abre e calcula tudo sem nenhuma requisição de rede.
- [ ] Testado com `.jsonl` de pelo menos 2 versões diferentes de CLI (campo `version`).
