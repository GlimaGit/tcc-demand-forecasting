# RESULTADOS.md — Resultados e Interpretação


## Estatísticas Descritivas da Série

| Estatística | Valor |
|-------------|-------|
| Período | 2013-01-01 a 2017-08-15 |
| N (observações) | 1.684 dias |
| Média diária (vendas) | 128.832,83 unidades |
| Mediana diária | 128.207,50 unidades |
| Desvio padrão | 63.121,65 unidades |
| Mínimo | 810,00 unidades |
| Máximo | 339.352,00 unidades |
| Zeros na série | 0 (série agregada sem zeros) |
| Feriados nacionais | 136 dias |
| Preço do petróleo (WTI) | 26,19 a 110,62 USD/barril |

---

## Testes Estatísticos

### Teste ADF — Série Original
- Estatística t: −2,5291
- p-valor: 0,1085
- Interpretação: **Não estacionária** (p > 0,05). A série apresenta tendência crescente que impede a rejeição da hipótese nula de raiz unitária.

### Teste ADF — Primeira Diferença
- Estatística t: −9,9637
- p-valor: < 0,0001
- Interpretação: **Estacionária** (p ≈ 0). Uma diferenciação é suficiente para eliminar a tendência. Confirma d=1 para o SARIMA.

### Decomposição Sazonal
- Tipo adotado: **Multiplicativa**
- Justificativa: A série cresce +223,6% entre 2013 e 2017; a amplitude sazonal em termos absolutos aumenta proporcionalmente ao nível, caracterizando natureza multiplicativa. O coeficiente de variação dos resíduos da decomposição multiplicativa (0,22) é substancialmente menor que o da aditiva.
- Tendência observada: crescimento de 59.829 (média 2013) para 193.624 (média 2017) = **+223,6%** no período
- **Sazonalidade semanal** (período=7 na série diária): pico no **domingo** (média: 177.977), vale na **quinta-feira** (média: 99.489), amplitude de **78,9%**
- **Sazonalidade anual** (período=12 na série mensal): índices sazonais multiplicativos:

| Mês | Índice | Mês | Índice |
|-----|--------|-----|--------|
| Jan | 1,130 | Jul | 1,004 |
| Fev | 0,906 | Ago | 0,874 |
| Mar | 1,073 | Set | 1,067 |
| Abr | 0,916 | Out | 1,017 |
| Mai | 0,902 | Nov | 1,045 |
| Jun | 0,936 | Dez | 1,129 |

Picos anuais em janeiro e dezembro (festas/Ano Novo); vale em agosto. Amplitude anual: ~29% entre pico e vale.

### Distribuição das Vendas Diárias
- Assimetria (skewness): 0,496 — leve assimetria positiva
- Curtose de excesso (excess kurtosis): −0,378 — distribuição levemente platicúrtica (caudas mais leves que a Normal; usando a convenção em que a Normal tem curtose de excesso = 0)

---

## Impacto do Terremoto de Abril/2016

- Data do evento: 16 de abril de 2016
- Impacto observado: **+28,8%** nas vendas de BEVERAGES nos 14 dias seguintes ao terremoto (média pós-14d: 209.328 vs. média 30 dias antes: 162.519)
- Interpretação: O aumento reflete duas fontes de demanda complementares: (1) **compra preventiva dos consumidores** — acumulação de estoques domésticos de bebidas diante de incerteza logística e de abastecimento; (2) **compras institucionais da rede Favorita para doação humanitária** — papel social assumido pela rede durante a crise. Ambas são comportamentos legítimos de demanda.
- Tratamento adotado: manter na série sem remoção ou interpolação. Incluído como evento especial no Prophet (`holidays` DataFrame). Para SARIMA e XGBoost, o evento está distante do período de teste (últimos 90 dias = maio-agosto 2017). Ver DECISOES.md.

---

## Métricas dos Modelos — Hold-out de 90 Dias

> Fórmula sMAPE: `(1/n) × Σ |y − ŷ| / ((|y| + |ŷ|) / 2) × 100` (Makridakis, 1993)
> Período de teste: 2017-05-18 a 2017-08-15 (90 dias)
> Versão v2 — correções: SARIMAX com exog=[onpromotion, oil_price]; XGBoost sem look-ahead bias

| Modelo | MAE | RMSE | sMAPE (%) |
|--------|-----|------|-----------|
| Naive | 36.748 | 53.083 | 18,58 |
| Sazonal Naive | 21.801 | 28.285 | 10,80 |
| SARIMAX (1,1,1)(1,1,1,7) | 35.336 | 43.419 | 20,76 |
| Prophet | 18.484 | 24.768 | 9,39 |
| **XGBoost** | **12.944** | **18.506** | **6,40** |

---

## Análise Residual — XGBoost (melhor modelo)

| Estatística | Valor |
|-------------|-------|
| Média dos resíduos (viés) | +4.451 unidades |
| Desvio padrão dos resíduos | 18.063 unidades |
| Mínimo / Máximo | −55.646 / +67.838 |
| Durbin-Watson | 1,5225 |

**Interpretação do Durbin-Watson:** O valor de 1,522 (intervalo esperado sem autocorrelação: 1,5–2,5 aproximadamente) indica leve autocorrelação positiva nos resíduos. O modelo não captura integralmente a estrutura temporal remanescente, possivelmente associada à variação na amplitude do ciclo semanal.

**Top 5 maiores erros absolutos:**

| Data | Real | Previsto | Erro | Dia da Semana |
|------|------|----------|------|---------------|
| 2017-06-11 | 311.184 | 243.346 | +67.838 | Domingo |
| 2017-08-13 | 202.354 | 258.000 | −55.646 | Domingo |
| 2017-06-04 | 339.352 | 284.273 | +55.079 | Domingo |
| 2017-08-12 | 182.318 | 232.792 | −50.474 | Sábado |
| 2017-05-21 | 280.849 | 234.681 | +46.168 | Domingo |

**Padrão identificado:** Os 5 maiores erros concentram-se em domingos e sábados — dias de pico da sazonalidade semanal. Os erros de subestimação (positivos) ocorrem em junho de 2017, quando as vendas reais atingiram seus maiores valores históricos; os erros de superestimação (negativos) ocorrem em agosto de 2017, sugerindo um declínio não antecipado nas vendas no final do período de teste. O modelo captura bem o padrão médio semanal, mas subperforma nos domingos de amplitude atípica.

---

## Ranking e Interpretação

### Melhor modelo por métrica

O XGBoost obteve o melhor desempenho nas três métricas avaliadas:

| Posição (por MAE) | Modelo | MAE | RMSE | sMAPE (%) |
|-------------------|--------|-----|------|-----------|
| 1º | XGBoost | 12.944 | 18.506 | 6,40 |
| 2º | Prophet | 18.484 | 24.768 | 9,39 |
| 3º | Sazonal Naive | 21.801 | 28.285 | 10,80 |
| 4º | SARIMAX | 35.336 | 43.419 | 20,76 |
| 5º | Naive | 36.748 | 53.083 | 18,58 |

**Nota:** Por sMAPE, o Naive (18,58%) supera o SARIMAX (20,76%), invertendo as posições 4º/5º nessa métrica.

O XGBoost superou o segundo colocado (Prophet) em **30,0% no MAE**, **25,3% no RMSE** e **31,9% no sMAPE**.

### Os baselines foram superados?

Dos três modelos principais (SARIMAX, Prophet e XGBoost), apenas Prophet e XGBoost superaram ambos os baselines em todas as métricas:

- **Prophet:** superou o Sazonal Naive em 15,2% (MAE), 12,5% (RMSE) e 13,1% (sMAPE). Superou o Naive em todas as métricas.
- **XGBoost:** superou o Sazonal Naive em 40,6% (MAE), 34,6% (RMSE) e 40,8% (sMAPE). Superou o Naive em todas as métricas.
- **SARIMAX:** ficou **abaixo** do Sazonal Naive em todas as métricas — MAE 62,1% pior, RMSE 53,5% pior, sMAPE 92,2% pior. Por sMAPE, também ficou abaixo do Naive (20,76% vs. 18,58%).

### SARIMAX abaixo dos baselines: interpretação

O desempenho do SARIMAX(1,1,1)(1,1,1,7) com `exog=[onpromotion, oil_price]` ficou aquém até mesmo do Sazonal Naive — e por sMAPE, abaixo do Naive. Este resultado, embora contraintuitivo, é explicável por uma combinação de fatores:

(1) **Natureza multiplicativa da série:** o modelo de espaço de estados do SARIMAX pressupõe variância homocedástica. A série BEVERAGES tem amplitude sazonal crescente com o nível (+223,6% de tendência), o que viola esse pressuposto e compromete a estimação dos parâmetros.

(2) **Sazonalidade limitada a s=7:** a especificação sazonal captura apenas o padrão semanal. A sazonalidade anual (amplitude ~29%, com pico jan/dez e vale ago) não é capturada, gerando erros sistemáticos ao longo do horizonte de 90 dias.

(3) **Efeito adverso dos regressores exógenos:** a inclusão de `oil_price` — cuja correlação com vendas é provavelmente um artefato temporal (ver DECISOES.md) — introduz ruído no modelo de estado-espaço. O coeficiente do oil_price estimado no treino não generaliza bem para o período de teste. Adicionalmente, `onpromotion`, apesar de correlacionado com vendas no treino (r=0,513), pode ter comportamento distinto nos 90 dias de teste, amplificando o erro.

(4) **Interação entre tendência e sazonalidade:** a dupla diferenciação (d=1, D=1) necessária para tratar tendência e sazonalidade semanal pode gerar instabilidade nas previsões de horizonte longo (90 dias), especialmente quando combinada com parâmetros MA e SAR de ordem 1.

Este resultado ilustra uma limitação conhecida dos modelos ARIMA/SARIMA: quando a série apresenta variância heterocedástica, padrões sazonais múltiplos e regressores com relações possivelmente espúrias, modelos não-lineares e baseados em árvores tendem a superar as abordagens estatísticas clássicas (Makridakis, Spiliotis e Assimakopoulos, 2018).

### Onde cada modelo acerta e erra

**Naive:** Replica o último valor observado para todo o horizonte. Em séries com forte sazonalidade semanal, converge rapidamente para a média (único valor constante), perdendo todos os ciclos. Erro sistemático crescente ao longo do período de teste.

**Sazonal Naive:** Captura bem o padrão semanal (periodicidade de 7 dias), mas não incorpora tendência nem feriados. Erros maiores em datas especiais (feriados, promoções intensas) e quando a série desvia da sazonalidade média.

**SARIMAX:** Reproduz o padrão semanal de forma suavizada, mas subestima os picos de domingo e não antecipa feriados adequadamente. Os regressores exógenos (onpromotion, oil_price), que melhoram Prophet e XGBoost, não trouxeram ganho ao SARIMAX — provavelmente porque o modelo de espaço de estados tem dificuldade em estimar simultaneamente os parâmetros sazonais e os coeficientes dos regressores numa série com forte tendência multiplicativa. Em domingos com vendas muito acima da média histórica, os erros superam o Naive.

**Prophet:** Captura tendência crescente, sazonalidade semanal e anual, e efeitos de feriados de forma explícita e interpretável. Erra principalmente nos picos extremos de fim de semana — a suavidade imposta pelos priors do modelo limita a capacidade de reproduzir valores atípicos. Segundo melhor modelo e o mais interpretável para uso operacional.

**XGBoost:** Melhor desempenho geral. Os lags curtos (lag_7, lag_14) e as médias móveis permitem ao modelo "ancorá-lo" ao nível recente da série, capturando tanto a sazonalidade semanal quanto os desvios do nível base. Erra principalmente nos domingos de amplitude atípica — casos onde o lag_7 recente não é um bom preditor do valor corrente por ruptura no padrão habitual.

### Viabilidade operacional

**XGBoost:** Melhor precisão, mas requer feature engineering manual, retreino periódico com dados atualizados e fornecimento antecipado de variáveis exógenas (onpromotion, oil_price) para o horizonte de previsão. Adequado para equipes com capacidade técnica de manutenção de pipelines de ML.

**Prophet:** Segundo lugar em precisão, com vantagem substancial em interpretabilidade. Os componentes de tendência, sazonalidade e feriados são diretamente comunicáveis a áreas de negócio sem conhecimento técnico avançado. Requer valores futuros dos regressores exógenos, mas é mais tolerante a irregularidades nos dados históricos. Recomendado quando a explicabilidade é requisito.

**SARIMAX:** Modelo mais leve computacionalmente e sem dependência de feature engineering. Contudo, o desempenho inferior a todos os demais modelos — incluindo os baselines — nesta série específica exclui sua recomendação prática para o problema em questão. Poderia ser reconsiderado em cenários com séries de variância estacionária, sem forte tendência multiplicativa, e com regressores exógenos de relação causal clara e estável.

**Sazonal Naive:** Valor como critério mínimo (*sanity check*). Qualquer modelo de previsão que não supere o Sazonal Naive não agrega valor operacional. Neste estudo, apenas Prophet e XGBoost cumprem esse critério. O SARIMAX ficou abaixo do Sazonal Naive em todas as métricas, e abaixo inclusive do Naive por sMAPE.

---

## Limitações Identificadas

- **SARIMAX com s=7:** captura sazonalidade semanal mas não a anual (índice jan/dez ~1,13 vs. ago ~0,87; amplitude ~29%). s=365 é computacionalmente inviável com 1.684 observações diárias. O resultado inferior a todos os modelos — incluindo os baselines — reflete a combinação dessa limitação com a incompatibilidade entre o modelo de espaço de estados e a natureza multiplicativa da série.

- **Regime de forecast condicional (SARIMAX, Prophet, XGBoost):** os três modelos recebem valores *reais* (não previstos) de `onpromotion` e `oil_price` no período de teste. Na operação real, esses valores precisariam ser previstos ou planejados antecipadamente — o que introduziria erro adicional e provavelmente reduziria o desempenho dos três modelos. As métricas reportadas representam o potencial máximo condicional, não o desempenho end-to-end de um pipeline completo (ver DECISOES.md).

- **Tendência parcialmente estrutural:** a variação de +223,6% na média diária (2013→2017) inclui o efeito de expansão da rede (47 → 54 lojas ativas, +15% em número de lojas). Os modelos capturam a tendência agregada da rede, adequado para o objetivo de previsão de demanda total, mas não é possível isolar crescimento orgânico por loja da expansão estrutural.

- **Escopo de categoria única:** resultados são específicos para BEVERAGES. A generalizabilidade para categorias com diferentes padrões de sazonalidade, esparsidade ou sensibilidade a promoções não foi avaliada.

- **Horizonte único de avaliação:** o hold-out de 90 dias representa uma única janela temporal. A robustez dos modelos em diferentes períodos do histórico (e.g., períodos com menor tendência, estações distintas) não foi testada. O walk-forward backtesting foi listado como melhoria futura em PENDENCIAS.md.

- **Autocorrelação residual no XGBoost:** Durbin-Watson = 1,522 indica leve autocorrelação positiva. O modelo não captura integralmente a estrutura temporal remanescente nos resíduos, o que sugere que abordagens híbridas (e.g., XGBoost + ARIMA nos resíduos) poderiam reduzir ainda mais o erro.

- **oil_price com importance baixa (0,96%):** a variável macroeconômica mostrou-se praticamente irrelevante para previsões de horizonte de 90 dias nesta categoria. Recomenda-se reavaliar sua inclusão; em horizontes mais longos, o nível do preço do petróleo pode ser relevante para a economia equatoriana (ver DECISOES.md). Confirma ausência de aprendizado espúrio no XGBoost — a tendência é capturada pelos lags e features temporais.

- **lag_365 com importance baixa (1,65%):** a sazonalidade anual é modesta em comparação à semanal nesta série. O sinal do ano anterior é parcialmente capturado pelas features de lag mais curtas e pela forte tendência.

- **Terremoto de abril/2016:** evento excepcional tratado como sinal real. O SARIMAX e o XGBoost não recebem flag explícita; o Prophet absorveu via changepoints e `holidays`. O período pós-terremoto não faz parte do conjunto de teste (os últimos 90 dias são de 2017), minimizando o impacto nas métricas de avaliação.
