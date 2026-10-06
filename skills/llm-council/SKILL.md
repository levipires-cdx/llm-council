---
name: llm-council
description: >
  Conselho de LLMs em 3 estágios (estilo Karpathy): vários subagentes Claude com modelos e
  lentes diferentes respondem isolados, revisam e ranqueiam as respostas anônimas uns dos outros,
  e o presidente (você) sintetiza a resposta final. Use para /llm-council:llm-council, "conselho", "council",
  "pergunta pro conselho", ou decisões difíceis de arquitetura, design, estratégia e trade-offs
  onde uma segunda (e terceira) opinião vale o custo. `--brief <task>` gera o .md do brief para o
  usuário preencher antes de convocar. Não use para perguntas factuais simples.
---

# LLM Council

Você é o **presidente** do conselho. Você não opina no estágio 1 nem no 2: você organiza,
anonimiza, agrega e sintetiza. A pergunta é o argumento da skill (ou a última pergunta do usuário).

Custo: 8 chamadas de subagente (4 no estágio 1, 4 no estágio 2). Com `--quick` no argumento, pule
o estágio 2 (4 chamadas).

## Modos de uso

| Argumento | O que faz |
|---|---|
| `<pergunta>` | Conselho completo (estágios 0–3) |
| `--quick <pergunta>` | Pula o estágio 2 |
| `--brief <task>` | **Não convoca o conselho.** Gera o scaffolding do brief para o usuário preencher (seção abaixo) e para |
| `<caminho.md>` | Usa o arquivo como brief (normalmente um gerado por `--brief` e já preenchido). Combina com `--quick` |

## Modo `--brief` — scaffolding do brief

Objetivo: o usuário preenche o contexto que só ele sabe, e o conselho recebe um brief completo.

1. Classifique a task num destes tipos e use as seções específicas dele:
   - **Aplicação nova / MVP:** problema, usuários e jobs-to-be-done, escopo do MVP, fora de escopo,
     requisitos não funcionais (volume, latência, disponibilidade), integrações, dados sensíveis.
   - **Decisão de arquitetura ou stack:** decisão a tomar, opções já consideradas, critérios de
     escolha com peso, o que é caro de reverter, prazo da decisão.
   - **Feature em sistema existente:** repo e caminhos envolvidos, comportamento atual, comportamento
     desejado, contratos públicos afetados (API, schema, eventos), compatibilidade e rollback.
   - **Revisão de plano ou código:** caminho do artefato, o que já foi decidido e não está em
     discussão, onde o usuário tem mais dúvida.
   - **Estratégia de investigação (bug, incidente, performance):** sintoma, desde quando,
     reprodução, o que já foi descartado, evidências (logs, métricas, caminhos).
   - Outro: monte seções equivalentes. Não force um tipo que não cabe.
2. Toda task leva também as seções comuns: **Pergunta ao conselho** (uma frase, fechada,
   respondível), **Contexto**, **Restrições** (equipe, stack obrigatória, infra, prazo, custo,
   compliance), **Já decidido** (fora de discussão), **O que conta como boa resposta**,
   **Arquivos e links** (caminhos absolutos).
3. Formato do arquivo:
   - Título `# Brief do conselho: <task>` e uma linha `Status: rascunho`.
   - Cada seção com um comentário `<!-- ... -->` curto dizendo o que escrever e um exemplo
     concreto da task.
   - Campo a preencher: `TODO:`. Campo opcional: `(opcional)` no título da seção.
   - Pré-preencha só o que a conversa ou o repo já deixam óbvio, marcado com
     `<!-- presidente: confira -->`. Não pesquise a fundo para pré-preencher.
4. Salve em `docs/council/<AAAA-MM-DD>-<slug>.md` dentro do repo da task. Sem repo claro, salve
   em `docs/council/` do diretório de trabalho atual. Não sobrescreva arquivo existente.
5. Responda só com o caminho do arquivo, quantos `TODO:` ficaram e o comando para rodar depois:
   `/llm-council:llm-council <caminho>` (ou com `--quick`).

Ao receber um `<caminho.md>`: leia o arquivo. Se ainda tiver `TODO:` em seção que não é opcional,
liste as pendências e pergunte se o usuário quer completar ou convocar assim mesmo. O arquivo vira
o brief do estágio 0: remova os comentários `<!-- -->` e as seções opcionais vazias antes de enviar.
Depois do estágio 3, anexe ao fim do arquivo uma seção `## Veredito do conselho (<data>)` com a
saída final e troque `Status:` para `decidido`. Esse veredito vai em prosa normal, mesmo que os
membros (que respondem em caveman ultra) e a sessão estejam em caveman: o arquivo é documento
para outras pessoas.

## Membros

| Id | `subagent_type` | Model / effort | Lente |
|---|---|---|---|
| M1 | `llm-council:council-pragmatico` | `claude-opus-5-5` / high | Engenheiro pragmático |
| M2 | `llm-council:council-cetico` | `claude-sonnet-5-5` / high | Cético / red team |
| M3 | `llm-council:council-visionario` | `claude-opus-5-5` / xhigh | Visão ampla |
| M4 | `llm-council:council-operador` | `claude-haiku-4-5-20251001` / medium | Operador |

Cada membro é definido em `agents/<nome>.md` deste plugin: modelo, effort, lente e ferramentas
bloqueadas (só leitura) moram lá. **Nunca passe `model` na chamada `Agent`**: ele sobrescreve o modelo do arquivo.
Para trocar versão ou effort, edite o arquivo do membro.

## Estágio 0 — Enquadrar

Se o argumento for um `<caminho.md>`, o brief já existe (ver modo `--brief`): pule para o item 3.

1. Reescreva a pergunta em um **brief** autocontido: pergunta, objetivo, restrições conhecidas,
   caminhos absolutos de arquivos relevantes (se houver código envolvido) e o que conta como boa
   resposta. Subagentes não veem esta conversa: o brief precisa bastar sozinho.
2. Não pesquise a fundo você mesmo. Os membros leem o que precisarem.
3. Se a pergunta for ambígua a ponto de os membros responderem perguntas diferentes, pergunte ao
   usuário antes de convocar.

## Estágio 1 — Respostas independentes

Dispare os 4 membros **numa única mensagem** (4 chamadas `Agent` em paralelo,
`run_in_background: false`, `subagent_type` do membro, sem `model`). Prompt igual para todos:

```
Estágio 1 do conselho. Outros membros responderão a mesma pergunta de forma independente;
depois as respostas serão comparadas anonimamente.

<brief>

Formato:
## Resposta
<sua recomendação e o porquê, até ~400 palavras>
## Premissas
<premissas que, se falsas, mudam a resposta>
## Riscos
<principais riscos da sua recomendação>
## Confiança
<baixa | média | alta> — <uma linha de justificativa>
```

Se um membro falhar, siga com os demais (mínimo 3). Avise no resultado final.

## Estágio 2 — Revisão cruzada anônima

1. Embaralhe as respostas e rotule como `Resposta A`, `B`, `C`, `D`. Guarde o mapeamento
   (rótulo → membro) só para você. Remova das respostas qualquer pista de identidade (nome de
   modelo, menção à lente).
2. Dispare 4 revisores numa única mensagem, um por membro (mesmo `subagent_type` de M1..M4,
   sem `model`). Prompt:

```
Estágio 2 do conselho. Avalie as respostas anônimas para a pergunta abaixo.

<brief>

<Resposta A>
...
<Resposta D>

Avalie cada resposta em correção, profundidade, praticidade e riscos ignorados. Seja específico:
aponte erros factuais e premissas frágeis. Pode verificar afirmações lendo código/docs (só leitura).

Formato:
## Avaliação
### Resposta A
<pontos fortes, fracos, erros>
... (todas)
## Pontos que todas erraram ou ignoraram
<lista, ou "nenhum">
## FINAL RANKING
1. Resposta X
2. Resposta Y
3. Resposta Z
4. Resposta W
```

3. Agregue o ranking: para cada resposta, a média da posição dada pelos revisores, **excluindo
   o voto do revisor sobre a própria resposta** (você sabe o mapeamento). Menor média = melhor.
   Se um revisor não seguir o formato de `FINAL RANKING`, descarte só o voto dele.

## Estágio 3 — Síntese do presidente

Com todas as respostas, avaliações e o ranking agregado, escreva a resposta final:

- Não é votação nem colagem. Pegue o melhor raciocínio, corrija o que os revisores provaram errado,
  incorpore os pontos que todos ignoraram.
- Quando houver divergência real, decida e explique por quê; registre a posição vencida.
- Verifique você mesmo qualquer afirmação factual decisiva antes de apoiar a recomendação nela.

## Saída ao usuário

```
## Veredito
<resposta final sintetizada, direta>

## Consenso e divergência
- Consenso: <onde os membros concordaram>
- Divergência: <ponto> — <quem defendeu o quê> — <por que o presidente decidiu assim>
- Pontos cegos apontados na revisão: <lista>

## Ranking do conselho
| Pos | Resposta | Membro | Média | Confiança declarada |
|---|---|---|---|---|

## Ressalvas
<membros que falharam, premissas abertas, o que verificar antes de agir>
```

Não despeje as respostas brutas dos membros; ofereça mostrá-las se o usuário quiser.
