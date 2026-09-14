---
name: devin-atif-format
description: >-
  Base de conhecimento sobre o formato ATIF (transcript JSON exportado pela Devin CLI via
  `devin --export`), incluindo onde o arquivo fica no disco, quando ele existe, sua estrutura
  de campos e as armadilhas de schema não-documentado. Use ao ler, parsear, tipar ou validar
  um arquivo `.json` exportado de uma sessão da Devin CLI.
---

# Formato ATIF (Agent Transcript Interchange Format) da Devin CLI

## 0. Nível de confiança desta skill

Esta base **não** foi compilada a partir da inspeção direta de um arquivo ATIF real (a Devin
CLI não estava instalada na máquina onde a pesquisa preparatória foi feita). Foi compilada a
partir de uma pesquisa preparatória única, feita fora deste ambiente, cruzando documentação
pública do produto com relatos de terceiros que já fazem parsing desse formato.

Trate os nomes de campo abaixo como **hipótese forte, não como verdade absoluta**. Antes de
depender de um campo em código de produção, valide contra um arquivo real do usuário (ver
regra 9 em `devin-session-invariants`).

**Ambiente corporativo — leia antes de usar esta skill.** Esta pesquisa foi feita uma única
vez, fora do ambiente corporativo, justamente porque esse ambiente não permite consultas a
URLs externas nem instalação de novas dependências. Este documento é a referência **congelada**
resultante dessa pesquisa. Ao trabalhar neste projeto: **não** chame ferramentas de rede
(WebFetch/WebSearch) para "confirmar" ou "atualizar" algo sobre a Devin CLI — se faltar
informação, sinalize a lacuna ao usuário. O parser desta skill usa apenas `JSON.parse` nativo:
**nenhuma dependência nova é necessária nem deve ser adicionada** para ler ATIF.

## 1. O que é e quando existe

- **ATIF** = formato de transcript que a Devin CLI grava ao usar a flag `--export`. Em versões
  mais recentes da CLI, os exports passaram a incluir mais detalhes por step, incluindo
  telemetria e métricas de tempo.
- **Não é gravado por padrão.** Ao contrário do `.jsonl` do Claude Code (sempre gravado), o
  ATIF só existe se a sessão foi iniciada com `--export [PATH]`. Sem essa flag, o único
  registro local da sessão é o `sessions.db` (ver skill `devin-sessions-db-format`).
- **Um arquivo por sessão.** É reescrito/estendido a cada turno ("export conversation to a
  file after each turn"), não é append-only linha a linha como o `.jsonl` — é um único
  documento JSON.
- **Local no disco:**
  - Se `--export PATH` foi passado com caminho explícito → grava exatamente ali
    (ex.: `devin --export out.json -- fix tests` → `out.json` no diretório atual).
  - Se `--export` foi passado **sem** caminho → usa um caminho default, provavelmente
    `~/.local/share/devin/cli/transcripts/<session_id>.json` no Linux/macOS (equivalente em
    `%APPDATA%\devin\cli\transcripts\` no Windows — **não confirmado**, inferido do padrão de
    outras configurações da CLI. Ver invariante de validação em `devin-session-invariants`).
- **Dentro da sessão interativa**, o slash command `/export` mostra informação sobre o export
  ativo (não altera o comportamento).

## 2. Estrutura de campos (nível documento)

```jsonc
{
  "session_id": "…",
  "schema_version": "ATIF-v1.4",   // não validado pelo parser oficial nem por terceiros
  "agent": {
    "model_name": "…"
  },
  "steps": [ /* ver §3 */ ]
}
```

- Não há confirmação de que `schema_version` seja de fato validado por quem consome o arquivo
  — o requisito mínimo observado é apenas um objeto parseável com `steps[]`. Um parser robusto
  deve seguir a mesma filosofia: não travar por causa de campos desconhecidos ou de uma versão
  de schema diferente.

## 3. Estrutura de um `step`

```jsonc
{
  "step_id": "…",
  "metadata": {
    "is_user_input": false,           // true = turno do usuário, costuma ser pulado nas métricas
    "created_at": "2026-05-01T12:00:00Z",
    "generation_model": "…",
    "request_id": "…",                // chave de dedup, análoga ao requestId do Claude
    "committed_acu_cost": 0.42,        // custo do STEP em ACU (Agent Compute Unit) — não é USD
    "metrics": {
      "input_tokens": 1234,
      "output_tokens": 567,
      "cache_creation_tokens": 0,
      "cache_read_tokens": 0
    }
  },
  "tool_calls": [
    { "function_name": "bash" /* … */ }
  ]
}
```

Pontos de atenção:

1. **`committed_acu_cost` é por step, não cumulativo.** Para o total da sessão, é preciso
   somar `Σ metadata.committed_acu_cost` de todos os steps — não existe (nesta versão
   documentada) um campo de total já pronto no nível raiz do ATIF, ao contrário do
   `cost-state.totalCostUSD` do Claude. (O `sessions.db` pode ter um total pré-agregado em
   `sessions.metadata.total_acu_cost` — ver skill `devin-sessions-db-format`; prefira essa
   fonte quando disponível, é o equivalente ao `cost-state`.)
2. **ACU ≠ USD.** Não existe taxa de conversão ACU→USD embutida localmente. Qualquer "custo em
   dólar" exibido exige uma taxa fornecida externamente pelo usuário (contrato/conta), nunca
   deve ser um valor chumbado no código. Ver `devin-session-invariants`.
3. **`is_user_input`** distingue turnos do usuário dos turnos do agente — equivalente a
   diferenciar `type:"user"` de `type:"assistant"` no `.jsonl` do Claude. Ao contar "turnos do
   agente" ou somar tokens de geração, filtre `is_user_input === false`.
4. **`request_id`** é a chave de deduplicação, análoga ao `requestId` do Claude — steps de
   streaming/retry do mesmo turno podem compartilhar o mesmo `request_id`.
5. **`tool_calls[].function_name`** é a única estrutura de chamada de ferramenta com confiança
   razoável; argumentos/resultados da ferramenta não têm schema confirmado nesta pesquisa —
   trate como `unknown` e acesse defensivamente.

## 4. Tipos TypeScript de referência (best-effort)

```ts
interface AtifStepMetrics {
  input_tokens?: number;
  output_tokens?: number;
  cache_creation_tokens?: number;
  cache_read_tokens?: number;
}

interface AtifStepMetadata {
  is_user_input?: boolean;
  created_at?: string; // ISO-8601, não confirmado o timezone
  generation_model?: string;
  request_id?: string;
  committed_acu_cost?: number; // ACU, não USD
  metrics?: AtifStepMetrics;
}

interface AtifToolCall {
  function_name?: string;
  [k: string]: unknown; // args/result não documentados
}

interface AtifStep {
  step_id?: string;
  metadata?: AtifStepMetadata;
  tool_calls?: AtifToolCall[];
  [k: string]: unknown;
}

interface AtifTranscript {
  session_id?: string;
  schema_version?: string; // não validar contra uma lista fixa
  agent?: { model_name?: string };
  steps: AtifStep[];
  [k: string]: unknown;
}
```

## 5. Esqueleto de parser (browser, front-end only)

```ts
export function parseAtif(raw: string): AtifTranscript | null {
  try {
    const doc = JSON.parse(raw);
    if (!doc || !Array.isArray(doc.steps)) return null; // único requisito real
    return doc as AtifTranscript;
  } catch {
    return null; // arquivo corrompido/incompleto (export pode ter sido interrompido)
  }
}

export function dedupeSteps(steps: AtifStep[]): AtifStep[] {
  const byReq = new Map<string, AtifStep>();
  let noReqId = 0;
  for (const s of steps) {
    const id = s.metadata?.request_id ?? `__no_request_id_${noReqId++}`;
    byReq.set(id, s); // mantém a última ocorrência, mesma lógica do requestId do Claude
  }
  return [...byReq.values()];
}

export function agentSteps(steps: AtifStep[]): AtifStep[] {
  return steps.filter((s) => s.metadata?.is_user_input !== true);
}
```

## 6. Como usar esta skill

1. Só tente ler ATIF se o usuário confirmar que rodou a sessão com `--export` (ou soltar um
   arquivo `.json` — sniff de conteúdo: objeto com `steps` array).
2. Nunca trave o parser por `schema_version` desconhecido ou campo ausente.
3. Deduplicar por `request_id` antes de somar tokens/ACU, mesma lógica de `requestId` do
   Claude Code.
4. Some `metadata.committed_acu_cost` por step para o total — não existe total pronto no ATIF.
5. Nunca exiba "USD" a partir de ACU sem uma taxa fornecida explicitamente pelo usuário.
6. Para o catálogo completo de métricas extraíveis, ver a skill `devin-session-metrics`.
7. Para a fonte alternativa/complementar (sempre presente, multi-sessão), ver
   `devin-sessions-db-format`.
