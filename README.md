# artifact-converter

Agente que converte artefatos de agentes de IA — hooks, skills, rules, servidores MCP, subagents e
plugins — entre os três padrões de harness suportados neste repositório: **Claude Code**, **GitHub
Copilot CLI** e **DevIn CLI**.

O agente existe em três variantes idênticas em funcionalidade, uma por harness, para que possa ser
invocado a partir de qualquer um deles:

| Harness | Localização |
|---|---|
| Claude Code | `.claude/agents/artifact-converter.md` |
| GitHub Copilot CLI | `.github/agents/artifact-converter.agent.md` |
| DevIn CLI | `.devin/agents/artifact-converter.md` |

## O que ele faz

Você fornece um artefato existente (caminho de arquivo, conteúdo colado, ou descrição) em **um** dos
três formatos, e o agente gera as versões correspondentes para os **outros dois**, preservando a
funcionalidade original. Exemplo: uma skill do Claude Code vira uma skill do Copilot CLI + uma skill
do DevIn CLI.

Tipos de artefato suportados:

- **Hook** — configuração de evento (matcher/tool, comando disparado)
- **Skill** — diretório com instruções + metadados de trigger
- **Rule** — arquivo de instruções para o agente (`CLAUDE.md`, `AGENTS.md`, etc.)
- **Servidor MCP** — entrada de configuração `mcpServers`
- **Subagent** — markdown com frontmatter + system prompt
- **Plugin** — manifesto de plugin

## Como ele decide o que fazer

1. **Identifica o tipo de artefato e o harness de origem** lendo o arquivo de entrada — frontmatter,
   chaves JSON, localização no filesystem. Se não conseguir determinar os dois com confiança, pergunta
   antes de prosseguir em vez de adivinhar o schema.

2. **Consulta a matriz de compatibilidade** (`compatibility-matrix/` na raiz do repositório) antes de
   gerar qualquer saída. Essa matriz é a fonte de verdade sobre o que é reutilizável sem tradução, o
   que precisa de reescrita, e como cada campo mapeia entre harnesses. O agente nunca inventa um
   mapeamento que não esteja documentado ali.

3. **Evita duplicação desnecessária**: quando o harness de destino já lê o artefato original nativamente
   (passthrough — por exemplo, Rules funcionam sem alteração em qualquer um dos três harnesses), o
   agente não cria um arquivo redundante por padrão. Em vez disso, explica que nenhuma tradução é
   necessária e resolve apenas o ponto de atrito específico, se houver algum (ex.: um `matcher` de hook
   que usa nomes de tool que não existem no harness de destino).

4. **Nunca sobrescreve o artefato original.** A conversão é sempre aditiva: o arquivo de entrada
   permanece intacto e continua sendo lido pelo harness de origem.

5. **Escreve as saídas em `outputs/`**, espelhando ali a localização nativa que cada arquivo teria no
   harness de destino (ex.: uma skill destinada ao Copilot vai para
   `outputs/.github/skills/<nome>/SKILL.md`). Isso deixa claro que os arquivos gerados são uma prévia —
   cabe a você copiá-los para o lugar real depois de revisar.

6. **Sinaliza incompatibilidades em vez de escondê-las.** Quando um campo, evento ou comportamento não
   tem equivalente confirmado no harness de destino, o agente avisa explicitamente no texto de resposta
   (nunca em comentário escondido no arquivo gerado): o que não converte, por que (citando a divergência
   documentada na matriz), e qual alternativa a matriz sugere, quando existe uma.

7. **Valida a saída antes de reportar como concluído**: relê cada arquivo gerado contra o schema exigido
   pelo harness de destino (campos obrigatórios, extensão de arquivo correta, localização correta dentro
   de `outputs/`). Se houver um passo de validação ao vivo recomendado pela matriz, o agente menciona
   esse próximo passo em vez de fingir tê-lo executado.

## O que você recebe ao final de uma conversão

- Os arquivos convertidos, prontos em `outputs/` (ou uma explicação do porquê nenhum arquivo novo é
  necessário, no caso de passthrough).
- Um resumo do que foi mapeado 1:1, o que exigiu uma decisão de tradução, e o que não é totalmente
  equivalente entre os harnesses — com a alternativa sugerida para cada incompatibilidade.

## Como usar

1. Invoque o agente `artifact-converter` a partir do harness que você já está usando (Claude Code,
   Copilot CLI ou DevIn CLI).
2. Aponte para o artefato existente — caminho de arquivo, conteúdo colado, ou descrição de onde ele está.
3. Diga qual harness é a origem, se não for óbvio pela localização do arquivo.
4. Revise a saída em `outputs/` e o resumo de compatibilidade antes de copiar os arquivos para a
   localização nativa real e validá-los no harness de destino (ex.: `/skills reload` no Copilot, `/agent`
   no Claude Code).

Se o repositório onde os arquivos convertidos devem viver não for este mesmo projeto, o agente pergunta
onde escrevê-los antes de criar qualquer coisa.

## Fonte de verdade

Todo o comportamento de mapeamento de campos vem de `compatibility-matrix/`:

- [`resume.md`](compatibility-matrix/resume.md) — tabela-resumo de compatibilidade por artefato
- [`rules.md`](compatibility-matrix/rules.md)
- [`skills.md`](compatibility-matrix/skills.md)
- [`hooks.md`](compatibility-matrix/hooks.md)
- [`subagents.md`](compatibility-matrix/subagents.md)
- [`plugins-mcp.md`](compatibility-matrix/plugins-mcp.md)
</content>
