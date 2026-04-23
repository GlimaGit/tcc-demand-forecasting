# BRIEFING DE PROJETO — TCC: Previsão de Demanda em Varejo

## Preâmbulo: contexto desta comunicação (LEIA PRIMEIRO)

Este documento foi produzido em uma **sessão não-agente do Claude** (chat comum em claude.ai, sem acesso direto ao sistema de arquivos do usuário, sem capacidade de instalar pacotes, sem execução persistente de código). Nessa sessão, o planejamento metodológico, a escolha do dataset, a escolha do nicho (BEVERAGES) e a estrutura do TCC foram discutidos e decididos em conjunto com o aluno.

**Você, que está lendo agora, é um agente** (Claude Code, ou equivalente) com capacidade de ler e escrever arquivos no sistema do usuário, executar código Python, instalar bibliotecas e iterar sobre o projeto. Seu papel é **executar o plano descrito aqui**, não redesenhá-lo.

### Handoff entre as duas sessões

- A sessão não-agente **não tem acesso ao que você (agente) executa**. Cada vez que o aluno volta para a sessão não-agente (para pedir ajuda na redação do TCC, por exemplo), ela recomeça sem memória do que foi feito por você.
- Por isso, **toda decisão que você tomar, todo arquivo que você gerar, toda escolha técnica, toda divergência em relação a este briefing, precisa estar registrada em arquivos `.md` dentro do projeto**. A sessão não-agente vai ler esses `.md` quando o aluno colar o conteúdo deles no chat.
- Pense assim: a documentação que você escreve no projeto é o **canal de comunicação entre você e a sessão não-agente**. Sem ela, a sessão não-agente não consegue ajudar o aluno a revisar, ajustar ou escrever o TCC com base no que foi feito.

---

## 1. Quem é o aluno e o que ele está fazendo

- Aluno de MBA em Data Science e Analytics (USP/Esalq).
- Trabalho: TCC com template de "Implementação de Algoritmo(s) de Machine Learning".
- Estrutura obrigatória do TCC: Resumo → Considerações Iniciais → Implementação de Algoritmo(s) de Machine Learning → Resultados e Discussão → Conclusão → Referências.
- Tema: Previsão de demanda em varejo com comparação de modelos e análise de desempenho.
- Prazo curto. Foco prioritário em **documentação acadêmica de qualidade**, não na sofisticação máxima do código.
- Norma de citação: ABNT (autor-data).
- Idioma: português acadêmico.

## 2. Problema prático

Erros de previsão de demanda no varejo geram dois efeitos:
- Ruptura de estoque — perda de vendas, insatisfação do cliente.
- Excesso de estoque — custo de armazenagem, capital imobilizado, risco de obsolescência.

Padrões de consumo variam entre categorias de produto, logo uma análise agregada mistura sinais heterogêneos. A orientadora recomendou trabalhar com um nicho/subcategoria de produto para que a análise seja coerente.

## 3. Objetivo

**Geral:** comparar modelos de previsão de demanda em um nicho específico do varejo, avaliando desempenho preditivo e viabilidade de aplicação.

**Específicos:**
1. Construir série temporal de demanda a partir de dados reais.
2. Aplicar baselines (Naive, Sazonal Naive) como referência mínima.
3. Implementar ARIMA/SARIMA, Prophet e XGBoost.
4. Comparar os modelos via MAE, RMSE e sMAPE, sob validação temporal.
5. Discutir qual modelo é mais adequado ao contexto, considerando acurácia e viabilidade operacional.

## 4. Dataset escolhido

**Store Sales — Time Series Forecasting (Corporación Favorita, Equador)**
Arquivos utilizados:
- `train.csv` — 3.000.888 linhas, 6 colunas: `id`, `date`, `store_nbr`, `family`, `sales`, `onpromotion`
- `stores.csv` — metadados de 54 lojas (cidade, estado, tipo, cluster)
- `holidays_events.csv` — 350 feriados/eventos classificados por tipo e escopo (nacional/regional/local)
- `oil.csv` — preço diário do petróleo WTI (proxy macroeconômica do Equador)
- `transactions.csv` — transações diárias por loja (uso complementar)

Período do `train.csv`: **01/01/2013 a 15/08/2017** (1.684 dias).

> O arquivo `test.csv` da competição não tem a coluna `sales` (era para submissão). Não usar. O split treino/teste é feito a partir do próprio `train.csv`.

## 5. Recorte de nicho escolhido

- Família: `BEVERAGES` (bebidas).
- Justificativa: 2ª família com maior volume (~20% das vendas totais da rede), baixa esparsidade (~8% de zeros, quase todos em 1º de janeiro por fechamento de loja), sazonalidade semanal e anual esperadamente fortes, sensibilidade a feriados e eventos.
- Agregação: série **diária, total de toda a rede** (soma das 54 lojas). 1.684 observações.

## 6. Modelos e metodologia

**Baselines (obrigatórios):**
- Naive: previsão = último valor observado
- Sazonal Naive: previsão(t) = valor(t − 7)

**Modelos principais:**
- ARIMA/SARIMA — estatístico clássico (`statsmodels`)
- Prophet — modelo aditivo decomponível (`prophet`)
- XGBoost — ML de regressão sobre features engenheiradas (`xgboost`)

**Validação:**
- Hold-out temporal com as últimas 90 dias como teste (simula previsão de 1 trimestre à frente).
- Backtesting adicional com walk-forward em 3 a 5 janelas, se o tempo permitir.

**Métricas:**
- MAE, RMSE, sMAPE.

## 7. Plano de execução

Cada etapa deve ser concluída com os entregáveis descritos na Seção 10 (política de documentação) antes de avançar para a próxima.

### Etapa 1 — Carregamento e preparação
1. Carregar `train.csv`, filtrar `family == 'BEVERAGES'`, agregar por data (soma de `sales` e de `onpromotion`).
2. Verificar integridade (datas contínuas, zeros = apenas 1º de janeiro de cada ano).
3. Decidir tratamento dos zeros — remover, interpolar ou manter. **Documentar a decisão com justificativa.**
4. Enriquecer a série com variáveis exógenas:
   - `onpromotion` agregado
   - Preço do petróleo (`oil.csv`) — forward-fill para dias faltantes (43 NaN no arquivo original)
   - Flag de feriado nacional (a partir de `holidays_events.csv`, filtrando `locale == 'National'` e `transferred == False`)

### Etapa 2 — EDA
- Plot da série diária e mensal.
- Histograma da distribuição das vendas diárias.
- Boxplot por dia da semana e por mês.
- Decomposição sazonal (`seasonal_decompose`, aditiva ou multiplicativa — testar ambas e escolher com justificativa).
- Teste ADF para estacionariedade (original e primeira diferença).
- Identificar e registrar o impacto do terremoto de abril/2016.

### Etapa 3 — Feature engineering (para XGBoost)
Calendário: `year`, `month`, `day`, `day_of_week`, `day_of_year`, `week_of_year`, `quarter`.
Lags: `lag_7`, `lag_14`, `lag_30`, `lag_365`.
Janelas móveis: `rolling_7_mean`, `rolling_30_mean`, `rolling_7_std`, `rolling_30_std`.
Exógenas: `onpromotion`, `oil_price`, `is_national_holiday`.
Remover linhas com NaN gerados pelos lags/rolling.

### Etapa 4 — Split temporal
- Treino: toda a série até T − 90 dias.
- Teste: últimos 90 dias.
- Sem random split — é séries temporais, split aleatório é data leakage.

### Etapa 5 — Modelagem

Baselines:
- Naive: `y_pred[t] = y[t-1]`
- Sazonal Naive: `y_pred[t] = y[t-7]`

SARIMA:
- Começar com `order=(1,1,1)`, `seasonal_order=(1,1,1,7)` (sazonalidade semanal — `s=365` é computacionalmente inviável).
- Usar `SARIMAX(..., enforce_stationarity=False, enforce_invertibility=False)` para estabilidade.
- Registrar trade-off: s=7 captura padrão semanal mas perde sazonalidade anual — discutir nos Resultados.
- Opcional: grid search pequeno em `(p,d,q)` via AIC.

Prophet:
- `yearly_seasonality=True`, `weekly_seasonality=True`, `daily_seasonality=False`.
- `changepoint_prior_scale=0.05` como baseline.
- Adicionar feriados via `holidays` (DataFrame com `ds` e `holiday`).
- Opcional: adicionar `onpromotion` e `oil_price` como `add_regressor`.

XGBoost:
- `n_estimators=1000`, `learning_rate=0.01`, `max_depth=5`, `subsample=0.8`, `colsample_bytree=0.8`, `early_stopping_rounds=50`, `random_state=42`.
- Treinar em `X_train`, validar em `X_test` para early stopping.
- Extrair feature importance (gain-based).

### Etapa 6 — Avaliação
DataFrame comparativo: Modelo, MAE, RMSE, sMAPE.
Gráficos:
1. Real vs previsto por modelo (4 gráficos separados).
2. Gráfico único com os 5 modelos sobrepostos sobre o real no período de teste.
3. Resíduos do melhor modelo.
4. Feature importance do XGBoost (top 10).
5. Componentes do Prophet (trend, weekly, yearly, holidays).

### Etapa 7 — Discussão
- Qual modelo teve melhor desempenho nas três métricas?
- Os baselines foram superados? Por quanto?
- Onde cada modelo acerta / erra (picos, vales, feriados)?
- Viabilidade operacional: Prophet é mais interpretável, XGBoost costuma ser mais preciso, SARIMA é o mais leve.
- Limitações do estudo.

## 8. Regras de escrita acadêmica

- Português acadêmico, objetivo e técnico.
- Evitar rodeios, clichês de LLM ("é importante destacar que", "vale salientar"), autopromoção.
- Não voltar ao tema antigo de "agentes de IA" (descartado).
- Tempo verbal na seção de Implementação: pretérito perfeito, forma impessoal ("foi carregado", "aplicou-se", "construiu-se").
- Citações ABNT autor-data: `(Hyndman; Athanasopoulos, 2021)`.
- **Não citar blogs nem artigos do Medium como fontes.** Fontes aceitáveis: livros, papers peer-reviewed, documentação oficial.
- Referências essenciais:
  - Hyndman & Athanasopoulos (2021) — *Forecasting: Principles and Practice*
  - Box, Jenkins, Reinsel & Ljung (2015) — ARIMA
  - Taylor & Letham (2018) — Prophet
  - Chen & Guestrin (2016) — XGBoost
  - Makridakis, Spiliotis & Assimakopoulos (2018, 2020) — métricas e M-competitions
  - Syntetos, Boylan & Croston (2005) — demand forecasting

## 9. Decisões já tomadas que NÃO devem ser revistas

- Dataset: Store Sales (Favorita).
- Família: BEVERAGES.
- Agregação: rede inteira.
- 5 modelos: Naive, Sazonal Naive, SARIMA, Prophet, XGBoost.
- 3 métricas: MAE, RMSE, sMAPE.
- Validação: hold-out dos últimos 90 dias (obrigatório) + walk-forward 3-5 janelas (se tempo permitir).
- Norma: ABNT autor-data.
- Idioma do TCC: português.

---

## 10. Política de documentação (OBRIGATÓRIA — leia com atenção)

Esta seção é o coração do handoff entre você (agente) e a sessão não-agente. **Nada do que você fizer importa se não estiver documentado aqui.**

### 10.1. Princípio geral

**Cada decisão técnica deve ser justificada por escrito, no momento em que é tomada.** Não basta o código rodar — a sessão não-agente precisa entender *por que* ele roda daquele jeito para poder:
- Escrever a seção de Metodologia do TCC com fidelidade ao que foi feito.
- Defender cada escolha em eventual banca.
- Pedir a você, numa sessão futura, modificações pontuais sem precisar reconstruir o raciocínio.

### 10.2. Estrutura de arquivos obrigatória

Na raiz do projeto, manter os seguintes arquivos `.md`:

```
projeto_tcc/
├── README.md                    # visão geral e como reproduzir
├── DECISOES.md                  # log cronológico de todas as decisões técnicas
├── EXECUCAO.md                  # o que foi executado, em que ordem, com que resultado
├── RESULTADOS.md                # números, métricas e interpretação dos modelos
├── PENDENCIAS.md                # o que ficou por fazer ou precisa de decisão do bin
├── data/                        # CSVs originais (não modificar)
├── notebooks/ ou src/           # código
└── outputs/                     # gráficos, tabelas, artefatos
```

### 10.3. O que cada arquivo deve conter

**`README.md`** — a página de rosto do projeto.
- Título do projeto, nome do aluno, contexto do TCC.
- Dataset usado e onde está.
- Como reproduzir (versão do Python, dependências, comandos para rodar).
- Índice apontando para os outros `.md`.

**`DECISOES.md`** — o registro de cada escolha técnica.
Para **cada** decisão, registrar em uma entrada com este formato:

```
## [AAAA-MM-DD] Título curto da decisão

**Contexto:** o que estava em jogo no momento (o problema que a decisão resolve).
**Opções consideradas:** A, B, C (com prós e contras de cada uma).
**Decisão:** qual opção foi adotada.
**Justificativa:** por que essa e não as outras (técnica, metodológica, prazo, etc.).
**Implicações:** o que isso afeta nas etapas seguintes.
**Reversibilidade:** fácil ou difícil de voltar atrás. Em que condições reconsiderar.
```

Exemplos de decisões que DEVEM ser registradas:
- Como tratar os 5 zeros da série (manter? interpolar? remover?).
- Decomposição aditiva vs multiplicativa (qual e por quê).
- Hiperparâmetros escolhidos para cada modelo (por que esses e não outros).
- Definição do bloco de teste em 90 dias.
- Tratamento do terremoto de abril/2016.
- Forward-fill no preço do petróleo.
- Quais feriados entraram no Prophet e como.
- Qual critério decidiu empate entre modelos (se houver).

Se você tiver que escolher entre duas alternativas razoáveis e seguir por uma sem consultar o aluno, **o registro na DECISOES.md é obrigatório**. Sem isso, a decisão está invisível.

**`EXECUCAO.md`** — diário de bordo do que foi rodado.
Formato cronológico, por etapa:

```
## Etapa 1 — Carregamento e preparação

**Quando:** AAAA-MM-DD
**O que foi feito:**
- Carregado `train.csv` com pandas, 3.000.888 linhas.
- Filtrado `family == 'BEVERAGES'` → 87.048 linhas.
- Agregado por data → série de 1.684 dias.
- Identificados 5 zeros (todos em 1º de janeiro).
- Merge com `oil.csv` via left join, forward-fill nos 43 NaN originais.

**Arquivos gerados:**
- `data/processed/beverages_daily.csv`

**Resultado-chave:** série pronta, integridade confirmada.
**Observações:** [qualquer coisa inesperada que tenha aparecido]
```

Uma entrada para cada etapa concluída. Nada de "rodei, deu certo" — o aluno precisa poder ler esse arquivo e reconstruir o que aconteceu.

**`RESULTADOS.md`** — os números e a leitura deles.
Estrutura sugerida:

```
## Estatísticas descritivas da série
- [valores]

## Testes estatísticos
- ADF original: t-stat = ..., p-valor = ... → interpretação
- ADF com 1ª diferença: ... → interpretação

## Métricas dos modelos (hold-out de 90 dias)
| Modelo        | MAE  | RMSE | sMAPE |
| Naive         | ...  | ...  | ...   |
| Sazonal Naive | ...  | ...  | ...   |
| SARIMA        | ...  | ...  | ...   |
| Prophet       | ...  | ...  | ...   |
| XGBoost       | ...  | ...  | ...   |

## Ranking e interpretação
[leitura acadêmica dos resultados, sem hype]

## Limitações identificadas
[o que ficou fora do modelo e como isso afeta as conclusões]
```

Este é o arquivo que a sessão não-agente vai usar para escrever a seção de Resultados e Discussão do TCC. **Precisa estar em linguagem acadêmica, não conversacional.**

**`PENDENCIAS.md`** — o que ficou por resolver.
Três seções fixas:

```
## Perguntas pendentes ao aluno
- [questões em que você precisou escolher sem consultar — destacadas para revisão]

## Melhorias não implementadas
- [coisas que seriam boas mas não couberam no prazo/escopo]

## Bugs conhecidos ou comportamentos estranhos
- [qualquer coisa que precisa de atenção numa próxima iteração]
```

### 10.4. Regras operacionais

1. **Atualização contínua, não retrospectiva.** Ao final de cada etapa (antes de seguir para a próxima), atualizar os `.md` relevantes. Não deixar para o fim.
2. **Sem sumários vagos.** "Fiz a EDA" é inútil. "A EDA confirmou tendência crescente (+38% entre 2013 e 2017) e sazonalidade semanal com pico aos sábados" é útil.
3. **Números explícitos.** Toda afirmação quantitativa precisa do número. "MAE baixo" não serve; "MAE = 4.93" serve.
4. **Se desviar do briefing, documentar.** Se você precisar fazer algo diferente do que este documento manda (por limitação técnica, por descoberta na EDA, etc.), registre em `DECISOES.md` com contexto e motivo. Não silencie divergências.
5. **Código e documentação caminham juntos.** Se o código mudar, a doc também muda. Não deixar `.md` mentindo sobre o estado real do projeto.
6. **Linguagem da documentação:** português técnico, mesmo padrão do TCC. Pode ser menos formal que o TCC em si (são notas internas), mas sem gírias nem clichês.

### 10.5. Teste prático para saber se está bom

Ao final de cada etapa, faça este teste mental: *"Se o aluno colar o conteúdo dos `.md` em uma nova sessão do Claude (não-agente), ela consegue escrever o TCC sem precisar fazer perguntas?"*

- Se sim: a documentação está boa.
- Se não: identifique o que falta e preencha antes de avançar.

---

## 11. Ambiente de execução

- Python 3.10+
- Bibliotecas: `pandas`, `numpy`, `matplotlib`, `seaborn`, `scipy`, `statsmodels`, `prophet`, `xgboost`, `scikit-learn`
- Fixar `random_state=42` e `np.random.seed(42)` para reprodutibilidade
- Salvar versão de cada biblioteca em `requirements.txt` ou mencionar em `README.md`

## 12. Como você (agente) deve trabalhar

- Executar em blocos pequenos e validar cada etapa antes de avançar.
- **Não inventar números.** Se um modelo não rodou, dizer isso e parar.
- Salvar gráficos em `outputs/` com nomes descritivos.
- Ao terminar uma etapa, atualizar os `.md` (ver Seção 10) **antes** de avançar.
- Fazer perguntas ao aluno quando houver múltiplos caminhos razoáveis, em vez de seguir por suposição silenciosa. Se seguir por suposição (por agilidade), registrar em `PENDENCIAS.md` para revisão.

---

## Próximo passo sugerido ao iniciar

Mensagem para você (agente) enviar ao aluno ao começar uma nova sessão:

> "Li o briefing completo. Confirmei a localização dos CSVs em `<caminho>`. Vou criar a estrutura de pastas e os `.md` base (README, DECISOES, EXECUCAO, RESULTADOS, PENDENCIAS) antes de tocar no código. Depois começo pela Etapa 1. Ok?"
