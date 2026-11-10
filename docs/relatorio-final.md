# Relatório final - Desempenho da API Claude

Este relatório consolida os parâmetros ajustados e a validação experimental do estudo que observa a API de mensagens da Anthropic (`POST /v1/messages`) como caixa-preta, medindo latência, TTFT e vazão efetiva de saída em três modelos da família Claude 5.5 sob requisições sequenciais com resposta curta.

## 1. Perguntas de pesquisa

1. Como o número de tokens de entrada ($n$) e o porte do modelo (`claude-haiku-5-5`, `claude-sonnet-5-5`, `claude-opus-5-5`) afetam a **latência total** e a **vazão efetiva de saída**?
2. Com que precisão o modelo analítico linear $L(n) = \alpha + \beta n + \frac{D}{1000}$, ajustado **apenas** nos prompts `curto` ($n = 458$) e `longo` ($n = 6.878$), prevê a latência do prompt `medio` ($n = 1.742$)?

## 2. Validação dos controles experimentais

Todas as verificações de integridade do desenho foram atendidas nas 45 chamadas (`3 modelos × 3 tamanhos × 5 repetições`):

| Controle | Valor observado |
| --- | --- |
| Total de chamadas medidas | 45 (15 por modelo, 5 por tamanho) |
| Status HTTP 200, sem erros | 45/45 (100%) |
| Tokens de entrada (curto / médio / longo) | 458 / 1.742 / 6.878 (contados == efetivos) |
| Tokens de saída | 7 em 100% das chamadas; `stop_reason = end_turn` |
| Criações / leituras de cache | 0 / 0 |
| Tempo pós-TTFT médio (`latency_s − ttft_s`) | 3,81 ms (DP = 1,00 ms) |
| Participação do TTFT na latência | 99,63% em média |

**Consequência metodológica central:** como a saída é rigorosamente constante (7 tokens) e o tempo pós-TTFT é de apenas ~3,8 ms, **mais de 99,5% da latência total corresponde ao TTFT** (rede, fila e *prefill* dos tokens de entrada). A vazão efetiva (`output_tokens / latency_s`) é, portanto, essencialmente o inverso da latência total e não mede a velocidade isolada de geração.

## 3. Estatísticas descritivas

Latência média e vazão efetiva média por modelo e tamanho (5 repetições cada):

| Modelo | Prompt | $n$ | Latência média (s) | DP (s) | Vazão média (tok/s) |
| --- | --- | ---: | ---: | ---: | ---: |
| haiku-5-5 | curto | 458 | 0,8208 | 0,1056 | 8,63 |
| haiku-5-5 | medio | 1.742 | 0,8813 | 0,1152 | 8,05 |
| haiku-5-5 | longo | 6.878 | 0,8585 | 0,0848 | 8,21 |
| sonnet-5-5 | curto | 458 | 0,9281 | 0,0114 | 7,54 |
| sonnet-5-5 | medio | 1.742 | 0,9407 | 0,0374 | 7,45 |
| sonnet-5-5 | longo | 6.878 | 1,2539 | 0,0382 | 5,59 |
| opus-5-5 | curto | 458 | 1,0369 | 0,0763 | 6,78 |
| opus-5-5 | medio | 1.742 | 1,1089 | 0,0463 | 6,32 |
| opus-5-5 | longo | 6.878 | 2,0358 | 0,2202 | 3,47 |

**Leitura:**

- **Haiku** é praticamente insensível a $n$: a latência varia apenas ~38 ms entre 458 e 6.878 tokens (0,821 s → 0,859 s) e a vazão permanece estável (8,63 → 8,21 tok/s). A variação aleatória de rede/fila (DP ≈ 0,08–0,12 s) domina qualquer efeito de $n$.
- **Sonnet** e **Opus** mostram forte crescimento do *prefill* com $n$: a latência sobe de 0,928 s → 1,254 s (Sonnet) e de 1,037 s → 2,036 s (Opus), reduzindo a vazão efetiva para 5,59 e 3,47 tok/s, respectivamente, no prompt longo.
- O DP elevado do Opus no prompt longo (0,2202 s) decorre de um pico isolado na 5ª repetição (2,417 s vs. ~1,94 s nas demais), analisado na Seção 6.

## 4. Modelo analítico ajustado

Ajuste OLS de $L(n) = \alpha + \beta n$ sobre as 10 repetições de treino (curto + longo) de cada modelo. $D_{\max}$ é o maior desvio absoluto de treino (em ms). $R^2$ refere-se ao conjunto de treino.

| Modelo | $\alpha$ (s) | $\beta$ (µs/token) | $D_{\max}$ (ms) | $R^2$ treino |
| --- | ---: | ---: | ---: | ---: |
| haiku-5-5 | 0,8181 | 5,88 | 175,10 | 0,046 |
| sonnet-5-5 | 0,9049 | 50,76 | 50,86 | 0,977 |
| opus-5-5 | 0,9656 | 155,60 | 381,35 | 0,920 |

**Interpretação dos parâmetros:**

- $\beta$ cresce ~26× do Haiku ao Opus (5,88 → 155,60 µs/token), quantificando o custo marginal de *prefill* por token de entrada e separando nitidamente as três classes de modelo.
- No **Haiku**, $R^2 = 0,046$: a inclinação $\beta n$ é tão pequena que o ruído de rede/fila ($D_{\max} = \pm175$ ms) domina o sinal. O modelo linear tem pouco poder explicativo aqui, não por má especificação, mas porque **não há dependência prática de $n$** nesse regime.
- No **Sonnet** ($R^2 = 0,977$) e no **Opus** ($R^2 = 0,920$), a reta captura bem a tendência: o *prefill* é o fator dominante da latência.

## 5. Validação no ponto de interpolação (`medio`, $n = 1.742$)

Previsão $\hat{L}(1.742) = \alpha + 1.742\,\beta$ confrontada com a média das 5 latências medidas:

| Modelo | Previsto (s) | Medido (s) | Erro (s) | Erro (%) | Repetições na faixa $\hat{L} \pm D_{\max}/1000$ |
| --- | ---: | ---: | ---: | ---: | :---: |
| haiku-5-5 | 0,8283 | 0,8813 | −0,0530 | −6,01% | 4/5 |
| sonnet-5-5 | 0,9933 | 0,9407 | +0,0525 | +5,58% | 3/5 |
| opus-5-5 | 1,2367 | 1,1089 | +0,1278 | +11,52% | 5/5 |

**Observações:**

- Os erros ficam entre −6% e +11,5%, considerando que o modelo foi ajustado **sem jamais ver** o ponto médio.
- Em Sonnet e Opus a reta **superestima** o ponto intermediário: o *prefill* entre 458 e 1.742 tokens aproveita melhor o paralelismo do hardware remoto do que na faixa estendida até 6.878 tokens. Isto é, a latência real cresce de forma **sublinear/côncava** em $n$, enquanto a reta ajustada nos extremos impõe inclinação única, viés esperado e coerente com a sublinearidade do *prefill* em GPU mencionada no plano.
- A cobertura da faixa $\pm D_{\max}/1000$ é ilustrativa, não probabilística. No Opus a faixa é larga (±381 ms) e cobre 5/5; no Sonnet é estreita (±51 ms) e cobre 3/5.

## 6. Ordem de execução e sensibilidade a outlier

A série temporal das 45 chamadas (espaçadas por 5 s, embaralhadas com `seed=42`) não revela deriva sistemática de rede/fila ao longo da coleta. O único ponto destoante é a 5ª repetição do prompt `longo` no Opus (2,417 s).

Reajuste do Opus excluindo esse pico:

| Cenário (opus-5-5) | $\alpha$ (s) | $\beta$ (µs/token) | $D_{\max}$ (ms) | Previsto médio (s) | Erro (%) |
| --- | ---: | ---: | ---: | ---: | ---: |
| Completo (10 repetições) | 0,9656 | 155,60 | 381,35 | 1,2367 | +11,52% |
| Sem outlier | 0,9724 | 140,75 | 107,75 | 1,2176 | +9,80% |

A remoção do outlier reduz $D_{\max}$ de 381 → 108 ms e melhora marginalmente a previsão (+11,52% → +9,80%), mas **não altera a conclusão qualitativa**: a reta continua superestimando o ponto médio. O viés é estrutural (sublinearidade), não causado por um único ponto.

## 7. Contabilização de tokens e custo estimado

Custo por $C \approx \frac{\sum (n_i\,p_\text{entrada} + o_i\,p_\text{saída})}{10^6}$, com tarifas de referência por milhão de tokens (entrada / saída): Haiku US$ 1 / US$ 5; Sonnet US$ 3 / US$ 15; Opus US$ 5 / US$ 25.

| Modelo | Chamadas | Tokens entrada | Tokens saída | Custo (US$) |
| --- | ---: | ---: | ---: | ---: |
| haiku-5-5 | 15 | 45.390 | 105 | 0,0459 |
| sonnet-5-5 | 15 | 45.390 | 105 | 0,1377 |
| opus-5-5 | 15 | 45.390 | 105 | 0,2296 |
| **TOTAL** | **45** | **136.170** | **315** | **0,4132** |

O custo total do estudo foi de **US$ 0,41**. O gasto é dominado pelos tokens de entrada (resposta constante de 7 tokens), escalando diretamente com a tarifa de entrada de cada classe.

## 8. Conclusões

1. **Efeito de $n$ e do porte do modelo.** A latência é governada quase integralmente pelo TTFT. O custo marginal por token de entrada ($\beta$) separa as três classes em escala de ~26×: Haiku é praticamente plano (5,88 µs/token), enquanto Sonnet (50,76) e Opus (155,60) têm latência fortemente crescente em $n$, com queda proporcional da vazão efetiva.
2. **Precisão do modelo analítico.** Ajustada apenas nos extremos, a reta prevê o ponto médio com erro de −6,01% (Haiku), +5,58% (Sonnet) e +11,52% (Opus). Em Sonnet e Opus há superestimação sistemática do ponto intermediário, indicando crescimento sublinear/côncavo do *prefill* em $n$, um viés estrutural de um modelo linear de dois pontos, não um artefato de outlier.
3. **Regimes distintos de variabilidade.** No Haiku a variação aleatória ($D_{\max} = \pm175$ ms) domina a inclinação ($R^2 = 0,046$); em Sonnet e Opus a inclinação domina o ruído ($R^2 = 0,98$ e $0,92$).

## 9. Limitações

- **5 repetições** por combinação limitam a estatística; DPs são apreciáveis, sobretudo no Haiku, onde $n$ quase não influi.
- Observação **caixa-preta**: rede, fila e carga remota estão embutidas na latência e não são separáveis.
- O modelo linear de dois pontos **não captura a sublinearidade** do *prefill*; a validação cobre **um único ponto de interpolação** por modelo e não extrapola para $n > 6.878$.
- As faixas $\pm D_{\max}/1000$ são ilustrativas: não representam uma distribuição real nem garantem limites para chamadas futuras.
- Tarifas e latências refletem o serviço no momento da coleta e podem variar.
