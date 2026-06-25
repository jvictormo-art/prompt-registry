---
prompt_avaliado: prompt.md
versao: 1.0.0
tipo: analitica
escala: 0-2 por critério (total 0–8, corte >= 6)
avaliador: llm-as-judge (claude-sonnet-4-6)
---

# Rubrica — Análise Comparativa de Backpressure em Barramento de Eventos

Rubrica analítica aplicada sobre outputs do prompt de análise de backpressure. A galeria de cada critério traz âncoras calibradas contra o cenário Relay 2.5x (Aegis platform, pico de 200k msgs/s).

## Regra de decisão

**PASSA quando:**

- Nenhum critério está com nota **0** ⚠️
- A soma dos 4 critérios é **>= 6 de 8**

Nota 0 em qualquer critério reprova o output inteiro, mesmo que a soma dos demais passe o corte (mecânica de veto).

---

## C1 — Diagnóstico quantificado (0–2)

Avalia se a resposta usa os números do cenário para dimensionar o gargalo — fator de pico, excesso por consumidor, lag acumulado — e identifica qual SLA está imediatamente ameaçado.

| Nota | Descrição |
|------|-----------|
| 2 | Calcula o fator de pico (pico ÷ throughput sustentado), o excesso de taxa ao menos para o consumidor mais restrito, o lag acumulado estimado e a margem da retenção. Identifica explicitamente qual SLA é violado primeiro e quando. |
| 1 | Menciona a sobrecarga mas omite >= 1 cálculo-chave, ou identifica o SLA errado como o mais ameaçado. |
| 0 | Descreve o gargalo de forma vaga sem usar nenhum número da entrada. |

**Âncora para o cenário Relay 2.5x:**
- Fator de pico: 200k / 80k = 2.5x
- Excesso Sentinel: 200k - 60k = 140k msgs/s → lag cresce 140k msgs por segundo
- Lag acumulado em 90min: 140k × 5.400s = 756M msgs
- Retenção disponível: 200k × 6h × 3.600 = 4,32B msgs → 756M é ~17,5% da capacidade (seguro)
- SLA violado imediatamente: Sentinel (500ms), pois o lag começa a acumular no instante zero do pico

---

## C2 — Comparação estruturada (0–2)

Avalia se a resposta compara >= 3 estratégias em tabela markdown com todas as colunas preenchidas, e aplica corretamente as restrições inegociáveis para eliminar candidatos incompatíveis.

| Nota | Descrição |
|------|-----------|
| 2 | Tabela com >= 3 estratégias, todas as 5 colunas preenchidas com avaliações ancoradas no cenário. Restrições inegociáveis (zero perda para Sentinel e Forge) aplicadas corretamente para eliminar ou penalizar estratégias incompatíveis. Considera ao menos uma combinação de estratégias. |
| 1 | Tabela presente com >= 3 estratégias mas >= 1 célula vaga ou genérica, OU restrições inegociáveis não aplicadas corretamente na eliminação. |
| 0 | Menos de 3 estratégias comparadas, tabela ausente, ou recomendação dada sem comparação. |

**Âncora para o cenário Relay 2.5x:**
- Rate limiting no produtor descartando mensagens VIOLA zero-perda para Sentinel → deve ser marcado como incompatível ou condicionado a DLQ
- Restrição histórica (rate limiting causou rejeição silenciosa em 2025-11) deve aparecer como fator de complexidade ou risco
- Combinação candidata natural: escalonamento de Sentinel + DLQ para picos extremos

---

## C3 — Recomendação rastreável (0–2)

Avalia se a recomendação é derivável dos números e restrições do cenário — não de preferência genérica — e se o custo do caminho escolhido é explicitado.

| Nota | Descrição |
|------|-----------|
| 2 | Recomenda um caminho (ou combinação) ancorado nos números do cenário (pico 200k, SLA 500ms, zero-perda, orçamento +40%). Declara explicitamente o que o caminho escolhido custa em troca. Menciona >= 1 condição da entrada que, se mudasse, mudaria a recomendação. |
| 1 | Recomendação feita mas com ancoragem parcial (algum raciocínio genérico), OU sem declarar o custo/tradeoff, OU sem mencionar condições de mudança. |
| 0 | Recomendação genérica não rastreável ao cenário, ou viola uma restrição inegociável (ex.: recomenda rate limiting com descarte para Sentinel). |

**Âncora para o cenário Relay 2.5x:**
- Escalonamento de Sentinel (de 60k para 200k/s) é a única forma de zerar o lag sem perda
- Custo: +40% de infra é o limite; escalar Sentinel 3x pode exceder o orçamento sem otimização
- Condição de mudança: se zero-perda para Cerebro também fosse inegociável, DLQ seria obrigatório

---

## C4 — Honestidade epistêmica (0–2)

Avalia se a resposta declara explicitamente as premissas assumidas quando os dados de entrada não permitem um cálculo preciso.

| Nota | Descrição |
|------|-----------|
| 2 | Declara explicitamente >= 1 premissa assumida quando cálculo exigiu dado não fornecido, OU confirma que todos os cálculos foram derivados diretamente da entrada sem premissas adicionais. |
| 1 | Faz >= 1 premissa silenciosa em um cálculo sem declarar, mas não inventa dados. |
| 0 | Inventa números não deriváveis da entrada sem nenhum reconhecimento, ou omite completamente a seção de premissas/lacunas. |

**Âncora para o cenário Relay 2.5x:**
- Dado não fornecido: custo por réplica adicional dos consumidores (necessário para verificar se escalonamento 3x cabe no orçamento +40%)
- Dado não fornecido: se os consumidores compartilham o mesmo tópico ou tópicos separados (impacta a estratégia de particionamento)
- Premissa declarável: "assumindo custo linear por réplica para estimar aderência ao orçamento"

---

## Calibração de referência — cenário Relay 2.5x

| Critério | Nota | Justificativa |
|----------|------|---------------|
| C1 — Diagnóstico quantificado | 2/2 | Números suficientes na entrada para todas as contas |
| C2 — Comparação estruturada | 2/2 | 4 candidatas fornecidas + estrutura de tabela exigida pelo prompt |
| C3 — Recomendação rastreável | 2/2 | Restrições e números permitem âncora precisa |
| C4 — Honestidade epistêmica | 2/2 | Custo por réplica é lacuna real e declarável |
| **Total** | **8/8** | Gate em >= 6/8 -- tolerância <= 1 ponto/critério |
