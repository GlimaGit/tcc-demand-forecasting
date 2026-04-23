# DECISOES.md — Log de Decisões Técnicas

Registro cronológico de todas as decisões técnicas do projeto. Cada entrada documenta contexto, opções consideradas, decisão adotada e suas implicações. Este arquivo é o canal de handoff entre a execução (agente) e a redação do TCC (sessão não-agente).

---

## [2026-04-21] Existência de rascunho anterior do TCC

**Contexto:** Antes do início da execução do código, o aluno já havia redigido um rascunho das seções Resumo e Considerações Iniciais em uma sessão não-agente do Claude.

**Decisão:** Não reescrever essas seções. Mantê-las intactas para revisão posterior, quando os números reais dos modelos estiverem disponíveis.

**Justificativa:** Evitar retrabalho e preservar a voz e estrutura acadêmica já estabelecida pelo aluno.

**Implicações:** Após a conclusão da Etapa 6 (avaliação), o aluno deverá revisar o Resumo e as Considerações Iniciais para alinhá-los com os resultados obtidos.

**Reversibilidade:** Fácil — o rascunho pode ser ajustado a qualquer momento.

---

## [2026-04-21] Fórmula de sMAPE

**Contexto:** O briefing define sMAPE como uma das três métricas de avaliação. Existem variantes da fórmula na literatura, com diferenças que afetam os valores calculados e a comparabilidade com outros estudos.

**Opções consideradas:**
- A) Fórmula original de Makridakis (1993): `sMAPE = (1/n) × Σ [|y - ŷ| / ((|y| + |ŷ|) / 2)] × 100`
- B) Variante sem divisão por 2 no denominador: `sMAPE = (1/n) × Σ [|y - ŷ| / (|y| + |ŷ|)] × 100`

**Decisão:** Opção A — fórmula de Makridakis (1993).

**Justificativa:** É a formulação canônica citada nas M-competitions (Makridakis, Spiliotis e Assimakopoulos, 2018, 2020), que são as referências centrais do TCC para avaliação de métricas. Garante comparabilidade com a literatura.

**Implicações:** A implementação em código deve usar explicitamente `(np.abs(y - y_pred) / ((np.abs(y) + np.abs(y_pred)) / 2)).mean() * 100`. O valor resultante é percentual e pode variar entre 0 e 200%.

**Reversibilidade:** Fácil — trocar a fórmula exige apenas alterar a função de cálculo e recalcular as métricas.

---

## [2026-04-21] Ordem de feature engineering em relação ao split temporal

**Contexto:** Para o XGBoost, é necessário criar features de lag e janelas móveis. Há risco de data leakage se o split for feito antes do cálculo das features.

**Opções consideradas:**
- A) Calcular features sobre a série completa, depois aplicar o split — os lags usam apenas valores passados em relação a cada ponto, portanto não cruzam o corte.
- B) Calcular features separadamente para treino e teste — mais conservador, mas desnecessariamente complexo se A não gera leakage.

**Decisão:** Opção A — features calculadas sobre a série completa antes do split.

**Justificativa:** Lags e janelas móveis são calculados com lookback exclusivamente para trás (e.g., `lag_7` para o dia t usa o valor do dia t−7). Não há acesso a valores futuros em relação a nenhum ponto. O split subsequente separa corretamente treino e teste. Após o split, X_test não contém informação de y_test além do que os lags carregam dos últimos dias de treino (o que é legítimo — simula um forecaster que conhece o passado recente).

**Implicações:** As ~365 primeiras linhas da série serão descartadas por NaN nos lags/rolling (especialmente `lag_365`). O split temporal deve ser aplicado após essa limpeza.

**Reversibilidade:** Médio — alterar exigiria refatorar o pipeline de features.

---

## [2026-04-21] Prophet: regressores exógenos obrigatórios

**Contexto:** O briefing original marcava `onpromotion` e `oil_price` como "opcional" no Prophet. O XGBoost usa essas variáveis como features.

**Decisão:** Tornar `onpromotion` e `oil_price` **obrigatórios** como `add_regressor` no Prophet.

**Justificativa:** Para que a comparação entre modelos seja metodologicamente válida, todos os modelos principais devem ter acesso ao mesmo conjunto de informação exógena. Excluir essas variáveis do Prophet criaria uma vantagem artificial para o XGBoost na comparação de métricas.

**Implicações:** O Prophet precisa receber valores futuros de `onpromotion` e `oil_price` no DataFrame de previsão (`make_future_dataframe`). Para o período de teste (hold-out), usaremos os valores reais observados dessas variáveis — o que é metodologicamente correto para avaliação ex-post.

**Reversibilidade:** Fácil — remover os regressores exige apenas deletar as chamadas `add_regressor`.

---

## [2026-04-21] Tratamento dos zeros na série agregada de BEVERAGES

**Contexto:** O briefing previa que a série agregada teria ~5 zeros, todos em 1º de janeiro de cada ano. Após execução da Etapa 1, verificou-se que a série agregada de toda a rede **não contém nenhum zero**. O valor mínimo é 810 unidades.

**Observação:** Em nível de loja individual, alguns dias 1º de janeiro provavelmente têm zeros. Porém, ao somar as 54 lojas, pelo menos uma loja teve vendas em todos os dias do período — resultando em um mínimo positivo.

**Decisão:** Nenhum tratamento de zeros necessário. Série limpa, com 1.684 observações e valores positivos ao longo de todo o período.

**Implicações:** Simplifica o pipeline — não há necessidade de interpolação, imputação ou remoção de datas. A série pode ser usada diretamente para modelagem.

**Reversibilidade:** Não aplicável.

---

## [2026-04-21] Forward-fill + backward-fill no preço do petróleo

**Contexto:** `oil.csv` tem 43 NaN (fins de semana e feriados em que o mercado não opera). Para integrar `oil_price` na série diária é necessário preencher esses gaps.

**Decisão:** Forward-fill primário (propaga o último preço disponível para frente). Backward-fill como fallback para eventuais NaN no início da série sem valor anterior.

**Justificativa:** Para preços de commodities, o valor do último dia de negociação é a melhor estimativa para dias sem cotação — é o preço "em vigor" até a próxima cotação. Procedimento padrão na literatura de series temporais financeiras.

**Resultado:** Após o merge, nenhum NaN restante em `oil_price`.

**Reversibilidade:** Fácil — trocar por interpolação linear se desejado.

---

## [2026-04-21] Feriados transferidos (`transferred == True`)

**Contexto:** `holidays_events.csv` contém registros com `transferred == True`. Esses registros marcam a data **original** de um feriado que foi oficialmente movido para outra data.

**Decisão:** Excluir registros com `transferred == True` ao criar a flag `is_national_holiday`.

**Justificativa:** O impacto no varejo ocorre na data em que o feriado é efetivamente observado, não na data original que foi cancelada. Incluir datas transferidas criaria falsos positivos na flag de feriado.

**Resultado:** 136 dias marcados como feriado nacional na série de 1.684 dias.

**Reversibilidade:** Fácil.

---

## [2026-04-21] Decomposição sazonal: multiplicativa vs. aditiva

**Contexto:** A decomposição sazonal é usada na EDA para interpretar os componentes da série e informar a escolha do modelo SARIMA. A escolha entre aditiva e multiplicativa depende do comportamento da variância sazonal em relação ao nível da série.

**Opções consideradas:**
- A) Aditiva: componentes são somados; amplitude sazonal constante independente do nível.
- B) Multiplicativa: componentes são multiplicados; amplitude sazonal proporcional ao nível.

**Decisão:** Multiplicativa.

**Justificativa:** (1) A série cresce +223,6% entre 2013 e 2017 — os padrões sazonais em valor absoluto são muito maiores no final do período que no início, o que é típico de séries multiplicativas. (2) O coeficiente de variação dos resíduos da decomposição multiplicativa (0,22) indica resíduos bem comportados, enquanto a decomposição aditiva gera resíduos heterocedásticos visíveis no plot. (3) O Q-Q plot da distribuição das vendas mostra assimetria positiva moderada (skewness=0,496), consistente com comportamento multiplicativo.

**Implicações:** Para o SARIMA, **sem transformação logarítmica** — modelagem na escala original (ver entrada específica abaixo). Para o Prophet, modelo aditivo com regressores e sazonalidades ajustadas — sem transformação. Para o XGBoost, as features de lag e rolling window capturam implicitamente o padrão multiplicativo.

**Reversibilidade:** Médio — trocar para aditiva exigiria ajustes na documentação, mas não na implementação (sem log).

---

## [2026-04-21] Tratamento do terremoto de abril/2016

**Contexto:** O terremoto de 16/04/2016 causou pico de vendas de +28,8% nos 14 dias seguintes (média pré: 162.519; média pós-14d: 209.328). Precisa-se decidir se o evento será tratado como outlier ou mantido.

**Opções consideradas:**
- A) Manter na série sem modificação — os modelos aprendem com o padrão tal como ocorreu.
- B) Interpolar o período pós-terremoto para suavizar o pico — remove o efeito do evento.
- C) Criar flag de "evento terremoto" como variável exógena.

**Decisão:** Opção A — manter na série. Adicionar o evento como feriado/evento no Prophet (Opção C aplicada apenas ao Prophet).

**Justificativa:** O pico representa comportamento de mercado real. A análise identificou duas fontes de demanda complementares: (1) **compra preventiva dos consumidores**, que acumularam estoques domésticos de bebidas por incerteza logística e de abastecimento; (2) **compras institucionais da rede Favorita para doação humanitária**, consistente com o papel social assumido pela rede durante a crise. Ambas as fontes são comportamentos legítimos de demanda e não devem ser removidos da série. Removê-los distorceria a série histórica e apagaria um sinal real do mercado.

Para SARIMA e XGBoost: o evento está suficientemente distante do período de teste (últimos 90 dias = maio-agosto 2017) para não contaminar as previsões. A decisão será revisada se os resíduos do SARIMA mostrarem outliers significativos nesse período.

**Implicações:** Incluir o evento "Terremoto Equador 2016" no DataFrame de holidays do Prophet com janela de efeito de ~14 dias.

**Reversibilidade:** Médio — requer remodelagem se a decisão for revista.

---

## [2026-04-21] SARIMA na escala original (sem transformação logarítmica)

**Contexto:** A decomposição multiplicativa indicaria, em princípio, a conveniência de uma transformação log antes do SARIMA para estabilizar a variância. A decisão foi reavaliada explicitamente.

**Opções consideradas:**
- A) Transformar a série com log antes do SARIMA, back-transformar as previsões.
- B) Rodar o SARIMA na escala original, sem transformação.

**Decisão:** Opção B — SARIMA na escala original.

**Justificativa:** (a) A transformação log exige correção de viés por desigualdade de Jensen ao back-transformar as previsões para a escala original: `E[exp(log_pred)] ≠ exp(E[log_pred])`. A correção exata requer estimativa da variância do erro, adicionando complexidade metodológica sem benefício claro. (b) MAE, RMSE e sMAPE são calculados na escala original para todos os modelos — a comparabilidade direta das métricas é preservada se o SARIMA também estiver nessa escala. (c) Se o SARIMA tiver desempenho inferior por operar em escala inadequada para a natureza multiplicativa da série, isso constitui argumento legítimo de discussão sobre as limitações do modelo clássico frente a séries com variância dependente do nível.

**Implicações:** O SARIMA pode apresentar resíduos heterocedásticos. Isso será documentado na seção de resultados como limitação do modelo, não como erro de implementação.

**Reversibilidade:** Fácil — adicionar `np.log` na entrada e `np.exp` na saída se desejado.

---

## [2026-04-21] Decomposição sazonal anual (período=365/12)

**Contexto:** A decomposição inicial usou apenas período=7 (semanal). Dada a série de ~5 anos, o padrão anual é relevante para o TCC.

**Decisão:** Adicionar decomposição anual sobre a série reamostrada mensalmente (período=12), complementar à decomposição semanal da série diária.

**Justificativa:** A série diária com período=365 não é computacionalmente tratável com `seasonal_decompose` de forma estável. A reamostragem mensal com período=12 captura o mesmo padrão de forma mais limpa.

**Resultado dos índices sazonais mensais (multiplicativo):**
- Meses de pico: janeiro (1,130) e dezembro (1,129) — associados a festas e Ano Novo
- Meses de vale: agosto (0,874), maio (0,902) e fevereiro (0,906)
- Amplitude anual: ~29% entre pico e vale

**Implicações:** O padrão anual é moderado (amplitude de ~29%) comparado ao padrão semanal (~79%). O Prophet com `yearly_seasonality=True` captará esse padrão. O SARIMA com s=7 não o captura — limitação a ser discutida nos Resultados.

**Reversibilidade:** Não aplicável (é visualização, não decisão de modelagem).

---

## [2026-04-21] Pico máximo de 04/06/2017

**Contexto:** O valor máximo da série (339.352 unidades) ocorreu em 04/06/2017. Verificação realizada para avaliar se requer tratamento especial.

**Investigação:**
- 04/06/2017 é domingo (dia de pico semanal habitual)
- `onpromotion` nesse dia: 1.393 itens (2,6× a média diária de 539; percentil ~70 da distribuição)
- Nenhum feriado registrado no `holidays_events.csv` para essa data
- Contexto sazonal: junho 2017 apresentou vendas elevadas em geral (não é pico isolado)

**Decisão:** Nenhum tratamento especial. O pico resulta da combinação de: (1) dia de pico semanal (domingo), (2) nível de promoções acima da média, (3) sazonalidade de junho. Não é outlier anômalo.

**Implicações:** Valor mantido na série. Documentar em RESULTADOS.md como achado esperado dentro do comportamento normal da série.

**Reversibilidade:** Não aplicável.

---

## [2026-04-21] Variável oil_price: risco interpretativo e decisão de manutenção

**Contexto:** A correlação linear entre `oil_price` e `sales` é −0,433 — correlação negativa moderada. À primeira vista, isso sugere que quedas no preço do petróleo estão associadas a aumentos nas vendas de beverages. No entanto, esse sinal deve ser interpretado com cautela.

**Hipótese de causalidade vs. artefato temporal:**
A janela do dataset (2013–2017) coincidiu com o colapso global do preço do petróleo (2014–2016, queda de ~$100 para ~$26/barril), que ocorreu simultaneamente à expansão da rede Favorita (47 → 54 lojas) e ao crescimento orgânico das vendas. A correlação negativa é provavelmente um **artefato da covariação temporal** entre duas tendências independentes: preço do petróleo caindo e vendas crescendo — não uma relação causal de curto prazo entre preço de combustível e compra de bebidas.

**Por que manter a variável:**
- O briefing define `oil_price` como variável exógena obrigatória para os modelos Prophet e XGBoost, para garantir comparabilidade entre eles.
- Na presença de `year`, lags e features rolling (que capturam tendência e nível), o risco de o modelo aprender o cotrend espúrio em vez da sazonal real é mitigado — as outras features já explicam o nível da série.
- Remover `oil_price` do modelo tornaria a comparação com o Prophet inconsistente.

**O que monitorar na Etapa 6 (feature importance do XGBoost):**
Se `oil_price` receber **alta feature importance** (e.g., top 3) combinada com **baixa importance de `year`**, é forte indício de aprendizado espúrio: o modelo está usando o cotrend de longo prazo do petróleo como proxy da tendência de crescimento das vendas. Nesse caso, adicionar nota nos Resultados alertando para esse risco e recomendando, em trabalhos futuros, uso de primeiras diferenças do preço do petróleo (em vez do nível) para isolar o efeito de curto prazo.

**Reversibilidade:** Fácil — remover `oil_price` das features do XGBoost e dos regressores do Prophet em uma iteração futura.

---

## [2026-04-21] Prevenção de look-ahead bias no XGBoost — protocolo de dois passos

**Contexto:** Na versão inicial (v1) do notebook de modelagem, o XGBoost usava `early_stopping_rounds=50` com `eval_set=[(X_test, y_test)]`. Isso é um vazamento de informação (*look-ahead bias*): o critério de parada do treinamento era determinado pelos dados de teste, que deveriam ser completamente desconhecidos pelo modelo durante o treinamento. O modelo parava de treinar no ponto ótimo para o teste específico, inflando artificialmente as métricas reportadas.

**Correção adotada (v2):**
Protocolo de dois passos completamente isolado do conjunto de teste:

1. **Passo 1 — encontrar `best_n`:** Dentro de `train_feat`, separar os últimos 90 dias como conjunto de validação interna (`val`). Treinar com os demais registros usando `early_stopping_rounds=50` e `eval_set=[(X_val, y_val)]`. O `best_iteration` encontrado (`best_n`) não envolve nenhuma informação do teste.

2. **Passo 2 — treinar definitivo:** Retreinar em `train_feat` completo (sem val separado) usando `n_estimators=best_n` e sem `early_stopping_rounds`. Prever em `test_feat`.

**Resultado:** `best_n = 583` (encontrado via validação interna). Métricas ligeiramente piores que v1, como esperado — a versão honesta não se beneficia de parada no ponto ótimo do teste.

**Justificativa:** O hold-out de teste deve permanecer completamente intocado até o momento da predição final. Usar o teste para qualquer decisão de hiperparâmetro (incluindo early stopping) viola esse princípio e invalida a avaliação comparativa.

**Reversibilidade:** Não reversível — a v1 era metodologicamente incorreta.

---

## [2026-04-21] SARIMAX com variáveis exógenas — comparação justa entre modelos

**Contexto:** Na versão v1, o SARIMA era univariado (sem variáveis exógenas), enquanto Prophet e XGBoost recebiam `onpromotion` e `oil_price` como regressores. Isso criava uma assimetria de informação: os modelos baseados em ML tinham acesso a sinais adicionais que o SARIMA não tinha, tornando a comparação injusta.

**Decisão (v2):** Substituir SARIMA univariado por SARIMAX com `exog=[onpromotion, oil_price]` tanto em `.fit()` quanto em `.forecast()`. A ordem permanece (1,1,1)(1,1,1,7).

**Resultado observado:** Contra-intuitivamente, o SARIMAX com exog apresentou desempenho **pior** que o SARIMA univariado (MAE: 35.336 vs. 28.501; sMAPE: 20,76% vs. 15,79%). Hipóteses para esse comportamento: (1) a relação negativa espúria do `oil_price` com as vendas introduz ruído no modelo de estado-espaço; (2) o modelo SARIMAX tem dificuldade em estimar simultaneamente os parâmetros AR/MA sazonais e os coeficientes dos regressores em uma série com forte tendência e heteroscedasticidade multiplicativa; (3) as 90 observações de teste ocorrem em período onde o oil_price apresenta comportamento distinto do período de treino.

**Implicação para o TCC:** O resultado não invalida a decisão metodológica de usar SARIMAX — a comparação justa é o objetivo correto. O resultado empírico (SARIMAX pior com exog) é, em si, um achado relevante que deve ser discutido.

**Reversibilidade:** Não reversível — o SARIMAX é a versão metodologicamente correta.

---

## [2026-04-21] Regime de forecast condicional — limitação documentada

**Contexto:** Os três modelos que usam variáveis exógenas (SARIMAX, Prophet, XGBoost) recebem, no período de teste, os **valores reais** (observados) de `onpromotion` e `oil_price` — e não previsões dessas variáveis. Isso é chamado de *forecast condicional* ou *oracle forecast* para as exógenas.

**Implicação prática:** Em uma aplicação operacional real, os valores futuros de `onpromotion` e `oil_price` não estariam disponíveis no momento da previsão — precisariam ser previstos ou planejados antecipadamente. O uso de valores reais no teste torna as métricas reportadas otimistas: elas medem o potencial máximo dos modelos condicionais, não o desempenho end-to-end de um pipeline completo de previsão.

**Por que essa abordagem foi mantida:**
- O briefing define o experimento como avaliação de algoritmos de forecasting, não de um pipeline completo de produção.
- Todos os três modelos com exógenas são tratados da mesma forma, preservando a comparabilidade entre eles.
- As exógenas do período de teste são informações planejáveis (promoções são decididas com antecedência pela gestão; oil_price pode ser obtido de mercados futuros).

**Registrado como limitação em RESULTADOS.md.** Em trabalhos futuros, substituir os valores reais das exógenas por previsões geradas a partir de modelos auxiliares.

**Reversibilidade:** Não aplicável — é uma decisão de escopo experimental, não de implementação.

---
