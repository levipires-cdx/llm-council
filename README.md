# llm-council

Plugin do Claude Code que convoca um **conselho de LLMs** para decisões difíceis (arquitetura, stack,
estratégia, trade-offs), no formato de 3 estágios proposto por Andrej Karpathy:

1. **Respostas independentes:** 4 subagentes Claude, cada um com um modelo e uma lente diferentes,
   respondem a mesma pergunta sem ver uns aos outros.
2. **Revisão cruzada anônima:** as respostas viram `Resposta A..D` e cada membro avalia e ranqueia
   todas. O ranking agregado descarta o voto de cada membro na própria resposta.
3. **Síntese:** a sessão principal (o presidente) escreve o veredito final com consenso, divergências
   e pontos cegos.

Os membros são só leitura: leem código e docs, mas não editam arquivos nem rodam comandos.

## Membros

| Agente | Modelo | Effort | Lente |
|---|---|---|---|
| `council-pragmatico` | `claude-opus-5-5` | high | Engenheiro pragmático |
| `council-cetico` | `claude-sonnet-5-5` | high | Cético / red team |
| `council-visionario` | `claude-opus-5-5` | xhigh | Visão ampla |
| `council-operador` | `claude-haiku-4-5-20251001` | medium | Operador / manutenção |

Para trocar modelo ou effort, edite o arquivo do membro em `agents/`.

## Requisitos

- Claude Code com suporte a plugins e ao campo `effort` em agentes.
- **Skill [`caveman`](https://github.com/JuliusBrussee/caveman)** instalada no nível do usuário.
  Os membros pré-carregam essa skill e respondem em estilo caveman ultra, o que reduz os tokens
  trocados entre os estágios. Instale seguindo o README do repositório dela (por exemplo,
  `npx skills add JuliusBrussee/caveman`). Sem ela, os membros não recebem as regras de estilo.

## Instalação

```bash
claude plugin marketplace add levipires-cdx/llm-council
claude plugin install llm-council@llm-council
```

Ou, dentro de uma sessão: `/plugin marketplace add levipires-cdx/llm-council` e depois
`/plugin install llm-council@llm-council`.

## Uso

| Comando | O que faz |
|---|---|
| `/llm-council:llm-council <pergunta>` | Conselho completo (8 chamadas de subagente) |
| `/llm-council:llm-council --quick <pergunta>` | Pula a revisão cruzada (4 chamadas) |
| `/llm-council:llm-council --brief <task>` | Não convoca o conselho: gera `docs/council/<data>-<slug>.md` com as seções a preencher para aquele tipo de task |
| `/llm-council:llm-council <caminho.md>` | Roda o conselho usando o brief preenchido e anexa o veredito ao arquivo |

Também dispara por linguagem natural: "pergunta pro conselho", "leva isso ao conselho".

### Fluxo recomendado

1. `--brief <task>` para gerar o brief.
2. Preencha os `TODO:` com o contexto que só você sabe.
3. `/llm-council:llm-council docs/council/<arquivo>.md`.
4. O veredito fica registrado no próprio arquivo, que passa a servir de registro da decisão.

## Custo

Uma rodada completa faz 8 chamadas de subagente, duas delas em opus com effort `xhigh`. Use o conselho
para decisões que valem esse custo e `--quick` para as menores. Não use para perguntas factuais.

## Licença

MIT
