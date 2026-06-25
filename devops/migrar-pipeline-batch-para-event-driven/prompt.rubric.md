---
prompt_avaliado: prompt.md
versao: 1.0.0
tipo: analitica
escala: 0-2 por critério (total 0–8, corte >= 6)
avaliador: llm-as-judge (claude-sonnet-4-6)
---

# Rubrica — Cadeia de Migração Batch para Event-Driven

Rubrica analítica aplicada sobre o output completo da cadeia de 3 elos. Cada critério avalia um elo distinto; C4 avalia a coerência entre eles. Âncoras calibradas contra o cenário Forge daily-ingest (Aegis platform).

## Regra de decisão

**PASSA quando:**

- Nenhum critério está com nota **0** ⚠️
- A soma dos 4 critérios é **>= 6 de 8**

Nota 0 em qualquer critério reprova o output inteiro, mesmo que a soma dos demais passe o corte (mecânica de veto).

---

## C1 — Diagnóstico Elo 1 (0–2)

Avalia se o Elo 1 entrega um diagnóstico completo e disciplinado: todas as 5 seções obrigatórias presentes, etapas classificadas pelo sistema tripartite (trivialmente convertível / reescrita stateful / inerentemente batch), handoff ao final, e nenhuma solução ou plano de migração proposto.

| Nota | Descrição |
|------|-----------|
| 2 | Todas as 5 seções presentes (mapa de acoplamentos, inventário de etapas, riscos, restrições invioláveis, lacunas de informação). Etapas classificadas pelo sistema tripartite. Termina com bloco Handoff com <= 6 bullets. Nenhuma solução ou plano de migração sugerido. |
| 1 | >= 1 seção ausente ou superficial; ou classificação tripartite ausente; ou solução implícita sugerida no diagnóstico. |
| 0 | Elo 1 ausente, ou fewer than 3 seções substantivas, ou propõe uma migração direta sem diagnóstico. |

**Âncora para o cenário Forge:**
- Nota 2: classifica deduplicação (Etapa 2) como "exige reescrita stateful" e agregação horária (Etapa 4) como "exige reescrita stateful (windowed)"; identifica leitura de offsets (Etapa 1) como inerentemente batch; marca enriquecimento (Etapa 3) e escrita BQ (Etapas 5, 6) como trivialmente convertíveis.
- Nota 0 clássica: "Recomendo migrar para Flink com streaming consumer" em vez de diagnosticar.

---

## C2 — Estratégia Elo 2 (0–2)

Avalia se o Elo 2 entrega uma sequência de fases incrementais, cada fase com: objetivo, divisão streaming vs batch, preservação dos dependentes, critério de avanço mensurável, gatilho e plano de rollback. Proibido big-bang.

| Nota | Descrição |
|------|-----------|
| 2 | >= 2 fases numeradas; cada fase tem objetivo, o que migra para streaming, o que permanece em batch, como dependentes continuam servidos, critério de avanço mensurável e rollback com gatilho. Cada fase é independentemente reversível. Termina com Handoff por fase. |
| 1 | Fases presentes mas >= 1 fase sem critério de avanço OU sem rollback OU sem descrição de como dependentes continuam servidos. |
| 0 | Propõe virada única (big-bang), ou menos de 2 fases, ou ausência de rollback em todas as fases. |

**Âncora para o cenário Forge:**
- Fase 1 natural: pipeline shadow para validar Etapas triviais sem tocar produção.
- Fase 2 natural: dual-write na agregação horária (streaming escreve em paralelo ao batch).
- Fase 3 natural: migração da dedup stateful e desativação do batch.
- Billing job (SLA de faturamento) deve ser explicitamente preservado em todas as fases.

---

## C3 — Plano Executável Elo 3 (0–2)

Avalia se o Elo 3 entrega um runbook executável para a FASE_ALVO com todas as 6 seções obrigatórias, passos numerados com validação individual, prova de equivalência de dados, rollback passo a passo e checklist final.

| Nota | Descrição |
|------|-----------|
| 2 | Todas as 6 seções presentes (pré-condições, passos de execução, validação de integridade de dados, procedimento de rollback, critério de saída, monitoramento). Passos numerados com ação + validação + tempo/risco. Usa `<<preencher: ...>>` para contexto desconhecido. Termina com Checklist de Ida/Volta. |
| 1 | >= 1 seção ausente; ou passos sem validação individual; ou validação de integridade de dados ausente; ou rollback não passo a passo. |
| 0 | Elo 3 ausente ou detalhando fase diferente da FASE_ALVO, ou sem rollback, ou sem critério de saída. |

**Âncora para o cenário Forge (Fase 2):**
- Pré-condição: Fase 1 concluída e shadow validado por 7 dias.
- Validação de integridade: comparação between batch e streaming para forge.hourly_aggregates (divergência < 0,01%).
- Ponto-de-não-retorno: migração do Billing job para consumir streaming como autoritativo.
- Checklist de Ida: streaming convergido por 14 dias, Billing staging validado.
- Checklist de Volta: desativar dual-write streaming, batch volta a ser único writer.

---

## C4 — Coerência da cadeia (0–2)

Avalia se cada elo usa as informações do elo anterior de forma consistente: Elo 2 referencia as classificações do Elo 1 (stateful, trivialmente convertível, restrições invioláveis); Elo 3 detalha exatamente a FASE_ALVO definida no Elo 2 sem inventar fases novas ou contradizer a estratégia.

| Nota | Descrição |
|------|-----------|
| 2 | Elo 2 menciona explicitamente as etapas classificadas no Elo 1 ao definir o que migra em cada fase. Elo 3 detalha exatamente a FASE_ALVO do Elo 2 sem contradizer os critérios de avanço ou rollback definidos. Restrições invioláveis do Elo 1 respeitadas em todos os elos. |
| 1 | Cadeia parcialmente coerente: >= 1 elo ignora ou contradiz uma informação relevante do elo anterior. |
| 0 | Elos desconectados: outputs poderiam ter sido gerados sem as entradas da cadeia; ou Elo 3 detalha fase diferente da FASE_ALVO. |

**Âncora para o cenário Forge:**
- Elo 2 deve referenciar que dedup (Etapa 2) é stateful e por isso vai para Fase 3, não para Fase 1.
- Elo 3 deve detalhar Fase 2 (dual-write na agregação) e não Fase 1 ou Fase 3.
- Schema do BigQuery e particionamento por data são restrições invioláveis do Elo 1 -- devem aparecer no plano de Elo 3.

---

## Calibração de referência — cenário Forge daily-ingest

| Critério | Nota | Justificativa |
|----------|------|---------------|
| C1 -- Diagnóstico Elo 1 | 2/2 | Snapshot rico o suficiente para classificação tripartite completa |
| C2 -- Estratégia Elo 2 | 2/2 | Diagnostico pre-crafted tem classificação e restrições claras para guiar fases |
| C3 -- Plano Elo 3 | 2/2 | Contexto operacional e estratégia pre-crafted permitem runbook concreto |
| C4 -- Coerência | 2/2 | Cadeia tem inputs encadeados explícitos que o modelo deve referenciar |
| **Total** | **8/8** | Gate em >= 6/8 -- tolerância <= 1 ponto/critério |
