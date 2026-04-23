# PENDENCIAS.md — Pendências e Melhorias

---

## Perguntas Pendentes ao Aluno

_Questões em que o agente precisou escolher sem consultar o aluno — destacadas para revisão antes da redação final do TCC._

- [ ] **Terremoto de abril/2016:** tratar como outlier (interpolar ou excluir o período) ou manter na série com anotação? A decisão atual é manter e anotar. Confirmar se a orientadora tem preferência metodológica.
- [ ] **Rascunho do Resumo e Considerações Iniciais:** após a conclusão da Etapa 6, revisar esses rascunhos com os números reais. O agente não os modificará sem instrução explícita.

---

## Melhorias Não Implementadas

_Itens que seriam metodologicamente interessantes mas foram deixados de fora por escopo ou prazo._

- [ ] **Walk-forward (backtesting com 3–5 janelas):** o briefing menciona como "se o tempo permitir". Implementar somente se todas as etapas principais estiverem concluídas e houver tempo. Adicionaria robustez à avaliação ao reduzir dependência de um único período de teste.
- [ ] **Grid search no SARIMA:** explorar combinações de (p,d,q) via AIC para verificar se (1,1,1)(1,1,1,7) é de fato o melhor ajuste. Fixado por simplicidade e prazo.
- [ ] **Tuning de hiperparâmetros do Prophet:** testar diferentes valores de `changepoint_prior_scale` (0.01, 0.05, 0.3) e `seasonality_prior_scale`. Fixado em 0.05 como baseline conservador.
- [ ] **Intervalos de confiança / predição:** incluir bandas de incerteza nos gráficos de previsão. Prophet fornece nativamente; SARIMA e XGBoost exigiriam abordagem adicional (bootstrap ou conformal prediction).
- [ ] **Análise por loja:** o estudo usa a série agregada da rede inteira. Uma extensão natural seria modelar cada loja individualmente ou usar modelos hierárquicos.

---

## Bugs Conhecidos ou Comportamentos Estranhos

_a preencher durante a execução_

- _nenhum registrado até o momento_
