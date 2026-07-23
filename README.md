# comparative-map-technical-offers
# comparative-map-technical-offers

Agente de IA (multi-etapas) para montar **mapas comparativos técnicos** de propostas de fornecedores contra um documento de referência (edital, folha de dados, especificação ou RFI).

O agente não elege vencedor. Ele congela os requisitos, extrai o que cada proposta oferece, classifica cada linha dentro de um conjunto fechado de vereditos, e sinaliza para decisão humana tudo que depende de julgamento ou contexto que os documentos não carregam. A conclusão do parecer é sempre do engenheiro.

Feito para rodar em plataforma de agentes com um único prompt de sistema (testado em Langdock com modelo da família Sonnet).

---

## O problema que resolve

Comparar propostas técnicas manualmente é lento e sujeito a um erro específico: o mapa vira ranking de quem esqueceu menos coisa na proposta, porque a proposta mais barata costuma ser a que menos entendeu o escopo. A qualidade de um mapa comparativo depende de **equalizar tecnicamente antes de comparar** — medir cada proposta contra o mesmo requisito, não uma contra a outra.

Este agente automatiza a extração e a classificação, mantendo o julgamento de aceitação com o humano.

---

## Princípio central de desenho

**A fronteira entre extração e julgamento é dupla, e cada lado fica num lugar diferente.**

- O agente **extrai** o que a referência exige e o que a proposta oferece — isso é mecânico: localizar e reportar, com citação de evidência.
- O agente **classifica** dentro de um enum fechado de vereditos — a única "inteligência" aplicada é reconhecer equivalência trivial de nomenclatura.
- O agente **nunca conclui** qual proposta vence, nem rejeita uma proposta sozinho. Rejeição e escolha são decisão humana.

Corolário que atravessa todo o prompt: **LLM lida com ambiguidade e leitura; aritmética e decisão de aceitação ficam fora do LLM** — em código determinístico ou em gate humano. O agente não soma vereditos nem calcula "quem tem mais OKs".

---

## Arquitetura em 4 etapas

O agente opera como um único prompt em etapas sequenciais, acumulando o texto extraído no contexto ao longo do ciclo. Um bloco de `[ESTADO]` no topo de cada resposta trava o avanço de etapa.

| Etapa | O que faz | Produz (fica no contexto) |
|---|---|---|
| 1 | Coleta a referência e **congela** a lista de requisitos | régua fixa de requisitos |
| 2 | Extrai cada proposta **isolada**, contra a régua congelada | um extrato por fornecedor |
| 3 | Classifica cada par (requisito, fornecedor) no enum fechado | vereditos |
| 4 | Consolida o mapa e lista as flags | mapa comparativo final |

### Por que agente único em etapas, e não subagentes

O desenho original usava subagentes — um extrator isolado por proposta, para que a leitura de uma não contaminasse o julgamento das outras. Isso falhou na prática por uma restrição de plataforma: **arquivos não trafegam para subagentes**. O subagente era invocado, mas o documento não chegava junto.

A solução foi inverter: um único agente que tem os documentos em contexto e trabalha sobre o texto, em etapas. **Nada precisa ser "passado" entre agentes porque tudo vive no mesmo contexto.**

Troca aceita conscientemente: perde-se o isolamento estrutural (o agente tem todas as propostas na memória ao mesmo tempo). Compensação por instrução: a Etapa 2 força medir cada proposta contra a régua congelada, nunca contra outra proposta, e fechar o extrato de uma antes de abrir a próxima. Instrução explícita substitui a barreira física.

---

## O enum de vereditos (conjunto fechado)

Nenhum valor fora desta lista é válido.

| Código | Significado | Quando |
|---|---|---|
| OK | Conforme especificado | ofertado == requerido literal, ou equivalência trivial de grafia |
| NM | Não mencionado | proposta não trouxe o item |
| A | Alternativa aceitável | difere do requerido mas é variação trivial de nomenclatura — usar com parcimônia |
| NA | Não aplicável | requisito condicional que não incide |
| FLAG | Precisa de decisão humana | divergência de substância, dado a comparar, ou dependência de contexto externo |

**Não existe atribuição automática de "não aceitável".** Rejeitar é decisão humana → marca FLAG.

### Regra de equivalência trivial (a única inteligência aplicada)

OK apenas para diferença de **grafia** do mesmo item, não de substância. Diferença que muda a substância (norma diferente, material diferente, dimensão diferente) vira FLAG. Na dúvida entre A e FLAG, escolhe FLAG — falso FLAG custa uma revisão humana; falso OK custa um erro que passa despercebido.

---

## Lições aprendidas em teste (documentadas para não repetir)

Estas são as correções que vieram de rodadas reais. Cada uma virou regra no prompt.

**1. "Agrupar" vira "resumir" — o modo de falha mais grave.**
O modelo lê "agrupe de forma comparável" como licença para colapsar blocos densos numa linha só. Sintoma: uma tabela de ~80 itens de funcionalidade vira 2 linhas; 6 estados de workflow nomeados viram "estados conforme referência". Isso destrói a granularidade que a classificação precisa — um "OK" contra um requisito empacotado é um falso-OK, porque o fornecedor pode dizer "atende plenamente" sem cobrir o item específico que sumiu no resumo.
→ Regra: **um requisito = uma linha que pode receber veredito sozinha.** Tabela de N itens vira N linhas. Teste de completude antes de congelar.

**2. Gap estrutural precisa de marcação própria.**
Quando a referência traz uma tabela de funcionalidades com colunas de obrigatoriedade e disponibilidade (ex.: "Obrigatório? / Já disponível no produto?"), um item marcado obrigatório-mas-não-nativo é um gap: a referência pede algo que o produto não tem de fábrica. Se colapsado num "conforme a seção X", o gap desaparece.
→ Regra: cada funcionalidade vira uma linha; item obrigatório-e-não-nativo é marcado `[GAP]`; na classificação, só vira OK se o fornecedor **descreve como entrega** o recurso não-nativo — afirmação genérica vira FLAG.

**3. "Same as [outro site]" é armadilha textual.**
Em casos multi-site, a referência frequentemente diz "igual ao site anterior" mas diverge nos detalhes — obrigatoriedade de campos, níveis de hierarquia, volume. Ler o "same" ao pé da letra colapsa os sites e apaga divergências reais.
→ Regra: divergência por site vira itens **separados**. Divergência de obrigatoriedade (obrigatório num site, opcional no outro) é divergência de substância, não detalhe a resumir.

**4. "Coberto com premissa" não é OK.**
Proposta que responde "sim, atendemos, desde que o cliente forneça infraestrutura / credenciais / acesso" não atende incondicionalmente — atende condicionado, e a condição transfere risco. Marcar isso como OK esconde o risco.
→ Regra: resposta condicionada, "responsabilidade do cliente" e "a ser validado" viram FLAG com a premissa explícita.

**5. Separar escopo técnico de comercial.**
Misturar "fornecedor não cotou o preço aqui" com "fornecedor divergiu tecnicamente" no mesmo balde de FLAG confunde a leitura — as ações são diferentes (cobrar anexo comercial vs avaliar aderência).
→ Regra: cada requisito recebe `natureza: tecnica|comercial`; flags saem agrupadas por natureza.

---

## Estrutura do repositório

```
prompts/          → o prompt de sistema do agente (a versão de produção)
examples/         → casos SINTÉTICOS que ilustram o comportamento (sem dado real)
reference-docs/   → vazia no repo; onde ficam docs reais ao rodar (barrada pelo .gitignore)
```

## Como usar

1. Cole o prompt de `prompts/` no campo de instruções do agente na plataforma.
2. Etapa 1: forneça o documento de referência (cole o texto — ver nota sobre arquivos abaixo). O agente extrai e congela os requisitos.
3. Etapa 2: forneça as propostas, identificando cada fornecedor. O agente extrai uma por vez.
4. Sinalize o fim da coleta. O agente classifica (Etapa 3) e consolida o mapa (Etapa 4).

**Nota sobre arquivos:** como a plataforma pode não trafegar binário para o agente de forma confiável, o fluxo mais robusto é converter o documento em texto antes (colar o texto, ou extrair via OCR) e alimentar o agente com texto — nunca depender de anexo de PDF/DOCX. O agente trabalha sobre texto.

## Escopo e limites

- O agente **não decide** qual proposta vence. Entrega o mapa e as flags; a conclusão é humana.
- O agente **não faz aritmética de pontuação** — se houver score ponderado, ele roda fora, em planilha ou código, sobre os vereditos.
- A régua de requisitos é **congelada** na Etapa 1 e não muda depois. Uma proposta não redefine o requisito; é medida contra ele.
