# EXECUCAO.md — Diário de Execução

Registro cronológico do que foi executado em cada etapa. Cada entrada deve conter data, ações realizadas, arquivos gerados e resultado-chave. Este arquivo permite reconstruir o que foi feito sem precisar re-executar o código.

---

## Phase 0 — Setup do Projeto

**Quando:** 2026-04-21

**O que foi feito:**
- Leitura do briefing completo (`instructions.md`).
- CSVs originais confirmados em `data/`: `train.csv` (116 MB), `stores.csv`, `holidays_events.csv`, `oil.csv`, `transactions.csv`.
- Criadas pastas: `data/processed/`, `notebooks/`, `outputs/figures/`, `outputs/tables/`.
- Criados arquivos de documentação base: `README.md`, `DECISOES.md`, `EXECUCAO.md`, `RESULTADOS.md`, `PENDENCIAS.md`.
- Criado `requirements.txt`.
- Registradas 4 decisões iniciais em `DECISOES.md`: rascunho TCC existente, fórmula sMAPE, ordem feature engineering/split, Prophet regressores obrigatórios.

**Arquivos gerados:**
- `README.md`, `DECISOES.md`, `EXECUCAO.md`, `RESULTADOS.md`, `PENDENCIAS.md`, `requirements.txt`
- Estrutura de pastas criada

**Resultado-chave:** Projeto estruturado e pronto para início da Etapa 1.

---

## Etapa 1 — Carregamento e Preparação

**Quando:** 2026-04-21

**O que foi feito:**
- Carregado `train.csv` com 3.000.888 linhas e 6 colunas.
- Filtrado `family == 'BEVERAGES'` → linhas referentes à categoria beverages por loja/dia.
- Agregado por data (soma de `sales` e `onpromotion`) → **série de 1.684 dias** (2013-01-01 a 2017-08-15).
- Verificada continuidade: nenhuma data ausente no período.
- **Zeros:** a série agregada não contém zeros. Mínimo = 810 unidades (contrário ao esperado no briefing — a soma das 54 lojas nunca chega a zero). Nenhum tratamento necessário.
- Carregado `oil.csv`; forward-fill nos 43 NaN (fins de semana/feriados de mercado); backward-fill como fallback para início da série. Nenhum NaN remanescente.
- Carregado `holidays_events.csv`; filtrado `locale == 'National'` e `transferred == False`; criada flag `is_national_holiday`. Total: 136 dias com feriado nacional na série.
- Arquivo final salvo em `data/processed/beverages_daily.csv`.

**Arquivos gerados:**
- `data/processed/beverages_daily.csv` (1.684 linhas × 5 colunas)
- `notebooks/01_preparacao.ipynb` (executado sem erros)

**Resultado-chave:** Série diária limpa, sem NaN, com 5 variáveis: `date`, `sales`, `onpromotion`, `oil_price`, `is_national_holiday`. Pronta para EDA.

**Observações:** A ausência de zeros (mínimo = 810) simplifica o pipeline — não é necessária nenhuma imputação. Registrado em DECISOES.md.

---

## Etapa 2 — Análise Exploratória (EDA)

**Quando:** 2026-04-21

**O que foi feito:**
- Plot da série diária (2013–2017) com anotação do terremoto de 16/04/2016.
- Plot da série mensal resampled.
- Histograma e Q-Q plot da distribuição das vendas diárias.
- Boxplot por dia da semana e por mês.
- Decomposição sazonal aditiva e multiplicativa (período=7); escolhida multiplicativa (ver DECISOES.md).
- Teste ADF: série original (p=0,1085, não estacionária) e primeira diferença (p≈0, estacionária).
- Análise do impacto do terremoto de abril/2016: janela de 30 dias antes e 60 dias depois.

**Arquivos gerados:**
- `notebooks/02_eda.ipynb` (executado sem erros)
- `outputs/figures/01_serie_diaria.png`
- `outputs/figures/02_serie_mensal.png`
- `outputs/figures/03_histograma_vendas.png`
- `outputs/figures/04_boxplot_sazonalidade.png`
- `outputs/figures/05_decomposicao_aditiva.png`
- `outputs/figures/06_decomposicao_multiplicativa.png`
- `outputs/figures/07_impacto_terremoto.png`

**Resultado-chave:**
- Tendência fortemente crescente: +223,6% entre médias diárias de 2013 e 2017 (inclui expansão de 47→54 lojas ativas).
- Sazonalidade semanal evidente: pico domingo (177.977), vale quinta (99.489), amplitude 78,9%.
- Sazonalidade anual: pico em janeiro (1,130) e dezembro (1,129); vale em agosto (0,874). Amplitude ~29%.
- Série original não estacionária (ADF p=0,109); primeira diferença é estacionária (ADF p≈0) → d=1 para SARIMA.
- Terremoto de 04/2016: +28,8% nas vendas (compra preventiva + compras humanitárias institucionais). Mantido na série.
- Decomposição multiplicativa escolhida. SARIMA rodará na escala original sem log (ver DECISOES.md).
- Pico máximo 04/06/2017 (339.352): domingo + promoções acima da média (1.393 itens) — sem tratamento especial.

**Arquivos adicionais gerados:**
- `outputs/figures/08_decomposicao_anual.png` (gerado via script após checkpoint)

**Observações:** Rede cresceu de 47→54 lojas ativas entre 2013-2017. Registrado em RESULTADOS.md como limitação da tendência observada.

---

## Etapa 3 — Feature Engineering

**Quando:** 2026-04-21

**O que foi feito:**
- Criadas 18 features sobre a série completa (sem split prévio — sem leakage):
  - Calendário: year, month, day, day_of_week, day_of_year, week_of_year, quarter
  - Lags: lag_7, lag_14, lag_30, lag_365
  - Rolling (shift(1) para garantir sem leakage): rolling_7_mean, rolling_30_mean, rolling_7_std, rolling_30_std
  - Exógenas: onpromotion, oil_price, is_national_holiday
- Verificação de leakage executada e confirmada: rolling_7_mean[i] = mean(sales[i-7:i-1]) — sem acesso a sales[i].
- Removidas 365 linhas iniciais com NaN gerados pelo lag_365.
- Arquivo salvo.

**Arquivos gerados:**
- `data/processed/beverages_features.csv` (1.319 linhas × 20 colunas)
- `notebooks/03_features.ipynb` (executado sem erros)

**Resultado-chave:** 1.319 observações (2014-01-02 a 2017-08-15), sem NaN.

**Correlação linear das features com sales (ordenada):**

| Feature | Correlação | Feature | Correlação |
|---------|-----------|---------|-----------|
| lag_7 | 0,820 | lag_365 | 0,308 |
| lag_14 | 0,790 | lag_30 | 0,305 |
| rolling_7_mean | 0,715 | month | 0,209 |
| rolling_30_mean | 0,661 | quarter | 0,204 |
| rolling_7_std | 0,542 | day_of_year | 0,202 |
| onpromotion | 0,513 | week_of_year | 0,180 |
| year | 0,511 | is_national_holiday | 0,099 |
| rolling_30_std | 0,488 | day | −0,077 |
| day_of_week | 0,369 | oil_price | −0,433 |

**Observações sobre features específicas:**

- **lag_365** (correlação = 0,308): baixa, esperada. A série cresce +223% no período; o valor de 365 dias atrás reflete o padrão sazonal do ano anterior mas está sistematicamente abaixo do nível atual. O XGBoost usará essa feature junto com `year` e lags mais curtos para compensar o viés de nível. Se a feature importance do lag_365 for baixa na Etapa 6, confirma essa interpretação.
- **oil_price** (correlação = −0,433): correlação negativa. O preço do petróleo caiu fortemente em 2014–2016 (colapso do mercado global) e se recuperou parcialmente em 2017, enquanto as vendas cresciam. A correlação negativa provavelmente reflete covariação temporal (não causalidade direta). Analisar com cautela na discussão dos resultados.
- **is_national_holiday**: 115 dias (8,7% das 1.319 linhas) com valor 1. Variação suficiente para o modelo aprender o padrão de feriados. Correlação linear baixa (0,099) pois feriados podem aumentar ou diminuir as vendas dependendo do tipo — a correlação linear não captura esse comportamento bidirecional; árvores de decisão (XGBoost) sim.
- **onpromotion**: 92,1% dos dias com valor > 0; média = 687,6; desvio padrão = 791,3; máximo = 4.225. Alta variância, distribuição muito assimétrica (mediana = 449 vs. máximo = 4.225). Correlação linear de 0,513 — segundo sinal exógeno mais importante.

**Observações finais:** As 365 linhas removidas por NaN do lag_365 correspondem ao período 2013-01-01 a 2013-12-31 e não afetam o split temporal (período de teste são os últimos 90 dias de 2017).

---

## Etapas 4+5 — Split Temporal e Modelagem (v2 — corrigido)

**Quando:** 2026-04-21

**Correções aplicadas vs. v1 (ver DECISOES.md para detalhe):**
1. SARIMA → **SARIMAX** com `exog=[onpromotion, oil_price]` — comparação justa com Prophet e XGBoost.
2. XGBoost com **protocolo de dois passos** sem look-ahead bias: (1) early stopping em validação interna (últimos 90 dias de train_feat) → `best_n=583`; (2) retreino em train_feat completo com n_estimators=583.
3. **Regime de forecast condicional** documentado: todos os modelos recebem valores reais de onpromotion e oil_price no período de teste.

**O que foi feito:**
- Split temporal: treino até 2017-05-17 (1.594 obs na série diária; 1.229 no dataset com features); teste 2017-05-18 a 2017-08-15 (90 dias).
- **Naive:** previsão constante = último valor de treino (165.675).
- **Sazonal Naive:** y_pred[t] = y[t-7].
- **SARIMAX(1,1,1)(1,1,1,7):** exog=[onpromotion, oil_price]. enforce_stationarity=False, enforce_invertibility=False. AIC=36.239,25, BIC=36.276,79. Treinamento: 1,8s.
- **Prophet:** yearly+weekly seasonality, changepoint_prior_scale=0.05, feriados nacionais + evento terremoto (14 dias), add_regressor obrigatório para onpromotion e oil_price. Treinamento: 0,4s.
- **XGBoost:** n_estimators=583 (encontrado via val interno), lr=0.01, max_depth=5, subsample=0.8, colsample_bytree=0.8, random_state=42.

**Arquivos gerados:**
- `notebooks/04_modelagem.ipynb` (v2, executado sem erros)
- `data/processed/predicoes_teste.csv` (90 linhas × 7 colunas)
- `data/processed/xgb_feature_importance.csv`
- `data/processed/prophet_components.csv`

**Resultado-chave — prévia das métricas (hold-out 90 dias):**

| Modelo | MAE | RMSE | sMAPE (%) |
|--------|-----|------|-----------|
| Naive | 36.748 | 53.083 | 18,58 |
| Sazonal Naive | 21.801 | 28.285 | 10,80 |
| **SARIMAX** | **35.336** | **43.419** | **20,76** |
| Prophet | 18.484 | 24.768 | 9,39 |
| **XGBoost** | **12.944** | **18.506** | **6,40** |

**Feature importance XGBoost v2 (rank completo):**
lag_7 (29,5%) → day_of_week (12,7%) → rolling_7_mean (12,1%) → onpromotion (9,9%) → lag_14 (9,3%) → rolling_7_std (5,4%) → rolling_30_mean (4,3%) → is_national_holiday (3,9%) → day (2,3%) → day_of_year (2,0%) → lag_30 (1,8%) → lag_365 (1,7%) → week_of_year (1,4%) → year (1,0%) → oil_price (1,0%) → rolling_30_std (0,9%) → month (0,9%) → quarter (0,0%)

**Observações:**
- SARIMAX com exog piorou vs. v1 SARIMA univariado (MAE: 35.336 vs. 28.501). Adição de regressores exógenos ao modelo de espaço de estados foi prejudicial nesta série. Discussão aprofundada em RESULTADOS.md.
- SARIMAX ficou abaixo de todos os modelos por MAE/RMSE. Por sMAPE, ficou abaixo também do Naive — o pior resultado geral.
- XGBoost levemente pior que v1 (MAE: 12.944 vs. 12.693) — esperado: v2 usa early stopping honesto, sem otimismo artificial do v1.
- oil_price com importance de 1,0%: confirma ausência de aprendizado espúrio (ver DECISOES.md).
- quarter com importance 0,0%: feature irrelevante dado que month, day_of_year e week_of_year já cobrem a informação trimestral.

---

## Etapa 6 — Avaliação (v2 — reexecutada com outputs corrigidos)

**Quando:** 2026-04-21

**O que foi feito:**
- Re-executado `05_avaliacao.ipynb` sobre `predicoes_teste.csv` gerado pelo notebook v2.
- Calculadas MAE, RMSE, sMAPE para todos os 5 modelos sobre o hold-out de 90 dias.
- Análise residual do XGBoost: viés médio, desvio padrão, Durbin-Watson, top 5 erros.
- Gerados 18 gráficos (sobrescritos com versão v2).

**Arquivos gerados:**
- `notebooks/05_avaliacao.ipynb` (re-executado sem erros, v2)
- `outputs/tables/metricas_comparativas.csv`
- `outputs/figures/09_naive_real_vs_previsto.png`
- `outputs/figures/09_sazonal_naive_real_vs_previsto.png`
- `outputs/figures/09_sarima_real_vs_previsto.png`
- `outputs/figures/09_prophet_real_vs_previsto.png`
- `outputs/figures/09_xgboost_real_vs_previsto.png`
- `outputs/figures/10_todos_modelos_sobrepostos.png`
- `outputs/figures/11_residuos_xgboost.png`
- `outputs/figures/11b_residuos_acf.png`
- `outputs/figures/12_xgboost_feature_importance.png`
- `outputs/figures/13_prophet_componentes.png`

**Resultado-chave:**
- XGBoost: melhor em todas as métricas (MAE=12.944, RMSE=18.506, sMAPE=6,40%).
- Prophet: segundo lugar (MAE=18.484, RMSE=24.768, sMAPE=9,39%). Inalterado vs. v1.
- SARIMAX: pior que todos por sMAPE (20,76%), pior que todos exceto Naive por MAE/RMSE. Adição de exog piorou o modelo vs. v1 SARIMA univariado.
- Resíduos XGBoost: viés +4.451 (leve superestimação), DW=1,522 (leve autocorrelação positiva).
- Top 5 erros do XGBoost: todos em domingos ou sábados — amplitude atípica do ciclo semanal.

**Observações:** RESULTADOS.md, DECISOES.md e EXECUCAO.md atualizados com métricas v2 e análise do comportamento inesperado do SARIMAX. Projeto pronto para handoff à sessão de redação do TCC.
