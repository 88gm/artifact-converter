# Plugins & MCP Servers

Status: **Plugins → Copilot ⚠️ precisa port / DevIn ✅ reutilizável. MCP → quase-cópia direta nos dois.**

## Plugins

### Origem no Claude Code

`.claude-plugin/plugin.json` como manifesto, mais diretórios opcionais `agents/`, `skills/`, config de
hooks, `.mcp.json`.

### Copilot CLI

**Estruturalmente próximo, mas sem passthrough documentado.** O layout de plugin do Copilot
(`plugin.json` na raiz, `agents/`, `skills/`, `hooks.json`, `.mcp.json`) espelha de perto as convenções
do Claude Code — mesmos diretórios de componente, ideia de manifesto similar. A diferença estrutural
confirmada: **localização do manifesto** — Claude Code aninha em `.claude-plugin/plugin.json`; Copilot
espera um `plugin.json` solto na raiz do plugin. Nada na documentação confirma que o Copilot lê
`.claude-plugin/` diretamente do jeito que lê `.claude/skills/` ou `CLAUDE.md` — tratar manifestos de
plugin como precisando de cópia explícita nativa.

Distribuição em 3 canais: marketplaces (`marketplace.json`), repositórios diretos, caminhos locais.
Instalação via `copilot plugin install` / `/plugin install`, ou declarativa em `enabledPlugins`
(`~/.copilot/settings.json` pessoal ou `.github/copilot/settings.json` repo).

#### Passos de tradução

1. Criar `plugin.json` na raiz do plugin (ao lado de, não substituindo, `.claude-plugin/plugin.json` se
   o objetivo é publicar nos dois ecossistemas).
2. `agents/`, `skills/`, hooks e config MCP internos ainda precisam da tradução por tipo de artefato dos
   outros arquivos deste diretório — skills geralmente não precisam de nada se já escaneiam caminho
   equivalente a `.claude/skills/`; hooks e subagents precisam de reescrita real.
3. Confirmar campos obrigatórios vs. opcionais do manifesto contra a "CLI plugin reference" viva antes
   de publicar — não adivinhar schema além do confirmado (`name` obrigatório; `version`, `description` e
   os diretórios de componente são os únicos campos que a doc consultada verifica).

### DevIn CLI

**Reutilizável — a superfície de compatibilidade mais generosa do DevIn.** DevIn reconhece três formatos
de manifesto de plugin lado a lado, sem tradução necessária para *instalar* qualquer um deles:

1. `.devin-plugin/plugin.json` — nativo do DevIn.
2. `.claude-plugin/plugin.json` — **plugins do Claude Code funcionam no DevIn como estão.**
3. `plugin.json` na raiz — spec aberta Agent Plugins 1.0.0.

DevIn também honra o `.mcp.json` na raiz de um plugin Claude e o campo `mcpServers` do manifesto
diretamente. Um plugin escrito para o sistema de plugins do Claude Code geralmente **não precisa de
tradução** para instalar e rodar sob DevIn — só sinalizar isso ao usuário, não fabricar uma cópia
paralela em `.devin-plugin/` a menos que ele queira explicitamente um manifesto DevIn-branded.

#### Passos de tradução (só se um manifesto DevIn-específico for realmente desejado)

1. Criar `.devin-plugin/plugin.json` ao lado de (não substituindo) `.claude-plugin/plugin.json`, se o
   objetivo é publicar nos dois ecossistemas com metadados distintos.
2. Skills/rules/hooks/agents/MCP internos geralmente não precisam duplicar — os dois formatos de
   manifesto apontam para as mesmas convenções de diretório; traduzir só onde um tipo de artefato
   genuinamente não é compatível (skills e subagents precisam de tradução; rules e hooks geralmente não).

## MCP Servers

### Origem no Claude Code

`.mcp.json`, servidores aninhados sob a chave `mcpServers`, entradas `{command,args,env}` (stdio) ou
`{url,...}` (remoto).

### Copilot CLI

**Quase-cópia direta.** Mesmo schema-base (`mcpServers`-keyed, `{command,args,env}` ou
`{url,headers,...}`). Diferenças práticas:
- Copilot quer um campo `type` explícito (`"local"`, `"stdio"`, `"http"`, `"sse"`) onde Claude Code
  infere stdio-vs-remoto pelas chaves presentes.
- Copilot suporta um `tools` allow-list opcional por servidor, sem equivalente documentado no Claude
  Code.

Localizações: pessoal `~/.copilot/mcp-config.json`; projeto primário `.mcp.json` (busca do diretório de
trabalho até a raiz do repo); projeto secundário `.github/mcp.json`. Precedência: arquivos mais
próximos do diretório de trabalho sobrescrevem os de cima; config de projeto sobrescreve pessoal.

#### Passos de tradução

1. Copiar cada entrada de `mcpServers` do `.mcp.json` como está — o mesmo `.mcp.json` funciona para os
   dois harnesses simultaneamente se os schemas permanecerem compatíveis; só bifurcar o arquivo se
   `type`/`tools` precisarem diferir por servidor.
2. Adicionar `"type"` explícito a toda entrada que faltar — `"local"` para quem tem `command`/`args`,
   `"http"` ou `"sse"` para quem tem `url` (checar o transporte real, não adivinhar `"http"` por padrão
   para um servidor `"sse"`).
3. Deixar segredos (tokens, chaves em `env`/`headers`) onde já estão; Copilot não documenta um arquivo
   pessoal gitignored separado — se o `.mcp.json` do projeto é commitado, manter segredos fora dele e
   usar `~/.copilot/mcp-config.json` (pessoal, não commitado) para credenciais.

### DevIn CLI

**Estruturalmente compatível.** Mesmo schema-base `mcpServers`-keyed. DevIn também honra o `.mcp.json`
de um plugin Claude diretamente quando o formato do plugin é `.claude-plugin/`. Para um `.mcp.json` de
projeto avulso (não-plugin), copiar as entradas de servidor para `.devin/mcp_config.json` é quase cópia
direta — checar se os nomes de campo batem com o schema DevIn (campos de servidor remoto como
`oauthClientId` podem não ter equivalente 1:1 no Claude Code; copiar o que existe, não inventar valores
para o que não existe).

Localizações: pessoal `~/.config/devin/mcp_config.json` (`%APPDATA%\devin\mcp_config.json` no Windows);
projeto compartilhado `.devin/mcp_config.json`; projeto local/gitignored `.devin/mcp_config.local.json`.
Versões antigas do DevIn guardavam servidores sob chave `mcpServers` no config principal — migram
automaticamente no startup, não portar manualmente.

#### Passos de tradução

1. Copiar cada entrada de servidor do objeto `mcpServers` do `.mcp.json` para `.devin/mcp_config.json`
   — **atenção**: o arquivo do DevIn *é* o objeto de servidores diretamente, não embrulhado numa chave
   `mcpServers` (checar a versão do DevIn, versões antigas embrulhavam).
2. Separar qualquer coisa com segredos (chaves de API, tokens em `env`) para
   `.devin/mcp_config.local.json`, que é gitignored — não commitar credenciais no arquivo compartilhado.
3. Manter o `.mcp.json` original — continua funcionando para o Claude Code e, no caso de plugin, o
   DevIn lê ele diretamente também.
</content>
