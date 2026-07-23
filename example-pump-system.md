# Exemplo sintético — comparação de propostas de um sistema de bombeamento

Caso fictício, criado apenas para ilustrar o comportamento do agente. Nenhum dado real. Dois fornecedores: **Fornecedor A** (respostas genéricas, transfere responsabilidade) e **Fornecedor B** (respostas específicas, declara valores). O contraste entre os dois é o ponto didático — mostra por que "coberto com premissa" vira FLAG e não OK.

---

## Referência (fictícia) — o que é REQUERIDO

Trecho de uma especificação fictícia de um conjunto motobomba:

- Material da carcaça: aço inox AISI 316
- Vazão nominal: 50 m³/h
- Pressão de trabalho: informar
- Motor: informar potência e rendimento
- Proteção contra corrosão externa: se aplicável ao ambiente de instalação
- Distância mínima para manutenção: informar (depende do layout do site)
- Certificação de ruído < 80 dB(A): obrigatória
- Painel de controle remoto: obrigatório; **não é item de linha padrão do produto** (gap)

---

## Régua congelada (saída da Etapa 1)

Repare: cada requisito é uma linha verificável. "Motor" foi quebrado em potência e rendimento porque são dois dados verificáveis separados.

```json
{
  "objeto": "Conjunto motobomba (exemplo sintético)",
  "referencia": "spec-fictícia-bomba-rev0",
  "lista_requisitos": [
    { "id": "mat_carcaca", "descricao": "Material da carcaça", "valor_requerido": "Aço inox AISI 316", "acao_esperada": "Confirmar", "natureza": "tecnica", "site": null, "referencia_local": "seç. materiais" },
    { "id": "vazao", "descricao": "Vazão nominal", "valor_requerido": "50 m³/h", "acao_esperada": "Confirmar", "natureza": "tecnica", "site": null, "referencia_local": "seç. desempenho" },
    { "id": "pressao", "descricao": "Pressão de trabalho", "valor_requerido": "Informar", "acao_esperada": "Informar", "natureza": "tecnica", "site": null, "referencia_local": "seç. desempenho" },
    { "id": "motor_potencia", "descricao": "Potência do motor", "valor_requerido": "Informar", "acao_esperada": "Informar", "natureza": "tecnica", "site": null, "referencia_local": "seç. motor" },
    { "id": "motor_rendimento", "descricao": "Rendimento do motor", "valor_requerido": "Informar", "acao_esperada": "Informar", "natureza": "tecnica", "site": null, "referencia_local": "seç. motor" },
    { "id": "protecao_corrosao", "descricao": "Proteção contra corrosão externa", "valor_requerido": "Se aplicável ao ambiente", "acao_esperada": "Se_aplica", "natureza": "tecnica", "site": null, "referencia_local": "seç. acabamento" },
    { "id": "dist_manutencao", "descricao": "Distância mínima para manutenção", "valor_requerido": "Informar (depende do layout do site)", "acao_esperada": "Informar", "natureza": "tecnica", "site": null, "referencia_local": "seç. instalação" },
    { "id": "ruido", "descricao": "Certificação de ruído", "valor_requerido": "< 80 dB(A), obrigatória", "acao_esperada": "Confirmar", "natureza": "tecnica", "site": null, "referencia_local": "seç. normas" },
    { "id": "painel_remoto", "descricao": "Painel de controle remoto", "valor_requerido": "[GAP: obrigatório, não é item de linha padrão]", "acao_esperada": "Confirmar", "natureza": "tecnica", "site": null, "referencia_local": "seç. automação" }
  ]
}
```

---

## Vereditos (saída da Etapa 3) — comportamentos ilustrados

| requisito | Fornecedor A | Fornecedor B | por quê |
|---|---|---|---|
| mat_carcaca | OK | OK | ambos oferecem "AISI 316" (A) / "SS 316" (B → equivalência trivial de grafia) |
| vazao | OK | OK | ambos confirmam 50 m³/h |
| pressao | FLAG | FLAG | "Informar" com valor a comparar — critério de escolha não está no doc, é decisão humana |
| motor_potencia | FLAG | FLAG | idem — valores coletados, comparação é humana |
| protecao_corrosao | NA | NA | condicional que não incide neste ambiente |
| dist_manutencao | FLAG | FLAG | depende do layout do site — contexto que o doc não carrega |
| ruido | FLAG | OK | A responde "atendemos, desde que o cliente forneça o ambiente de teste" → condicionado, vira FLAG. B anexa certificado. |
| painel_remoto | FLAG | OK | GAP: A diz "sim, suportamos" sem descrever como entrega o item não-padrão → FLAG. B descreve a customização → OK. |

### Os três comportamentos que este exemplo prova

1. **Equivalência trivial** (`mat_carcaca`): "AISI 316" ≡ "SS 316" → OK. Grafia diferente, mesma substância.
2. **Coberto com premissa vira FLAG** (`ruido`, Fornecedor A): "atendemos desde que o cliente forneça X" não é atendimento incondicional. A condição transfere risco → FLAG, não OK.
3. **Gap sem mecanismo vira FLAG** (`painel_remoto`, Fornecedor A): contra um item marcado `[GAP]`, "sim, suportamos" genérico não basta — o produto não tem o item de fábrica, então o fornecedor precisa descrever *como* entrega. Sem o mecanismo → FLAG. Fornecedor B descreve → OK.

O agente não conclui qual fornecedor é melhor. Ele mostra que o Fornecedor A acumulou FLAGs em pontos que transferem risco ou não descrevem entrega, e que o Fornecedor B declarou valores concretos. A leitura desse padrão — e a decisão — é do engenheiro.
