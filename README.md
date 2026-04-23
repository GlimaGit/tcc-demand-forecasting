# TCC — Previsão de Demanda em Varejo com Modelos de Machine Learning

**Aluno:** MBA em Data Science e Analytics — USP/Esalq  
**Tema:** Comparação de modelos de previsão de demanda no segmento BEVERAGES (bebidas) do varejo equatoriano  
**Dataset:** Store Sales — Time Series Forecasting (Corporación Favorita, Equador)

---

## Contexto

Este projeto implementa e compara cinco modelos de previsão de demanda aplicados à série temporal diária de vendas da categoria BEVERAGES, agregada para toda a rede de 54 lojas da Corporación Favorita (2013–2017).

---

## Dataset

Fonte: [Kaggle — Store Sales Time Series Forecasting](https://www.kaggle.com/competitions/store-sales-time-series-forecasting)

Arquivos em `data/`:
| Arquivo | Descrição |
|---------|-----------|
| `train.csv` | 3.000.888 linhas; vendas diárias por loja e família (2013-01-01 a 2017-08-15) |
| `stores.csv` | Metadados das 54 lojas (cidade, estado, tipo, cluster) |
| `holidays_events.csv` | 350 feriados/eventos por tipo e escopo |
| `oil.csv` | Preço diário do petróleo WTI (proxy macroeconômica) |
| `transactions.csv` | Transações diárias por loja |

> `test.csv` **não é utilizado** — não contém a coluna `sales`. O split treino/teste é feito a partir do próprio `train.csv`.

---

## Estrutura do Projeto

```
tcc-demand-forecasting/
├── data/
│   ├── *.csv                  # CSVs originais
│   └── processed/             # Dados processados pelo pipeline
├── notebooks/
│   ├── 01_preparacao.ipynb    # Etapa 1: carregamento e preparação
│   ├── 02_eda.ipynb           # Etapa 2: análise exploratória
│   ├── 03_features.ipynb      # Etapa 3: feature engineering
│   ├── 04_modelagem.ipynb     # Etapas 4+5: split temporal e modelagem
│   └── 05_avaliacao.ipynb     # Etapa 6: avaliação e gráficos
├── outputs/
│   ├── figures/               # Gráficos gerados
│   └── tables/                # Tabelas exportadas
├── README.md                  # Este arquivo
└── requirements.txt           # Dependências Python
```

---

## Como Reproduzir

### Requisitos
- Python 3.10+
- Instalar dependências: `pip install -r requirements.txt`

### Execução (ordem obrigatória)
```bash
jupyter notebook
```
Executar os notebooks em ordem: `01_preparacao` → `02_eda` → `03_features` → `04_modelagem` → `05_avaliacao`

> Fixação de aleatoriedade: `random_state=42` e `np.random.seed(42)` em todo o pipeline.

---

## Modelos Comparados

| Modelo | Tipo |
|--------|------|
| Naive | Baseline |
| Sazonal Naive | Baseline |
| SARIMA (1,1,1)(1,1,1,7) | Estatístico clássico |
| Prophet | Modelo aditivo decomponível |
| XGBoost | ML de regressão |

**Métricas:** MAE, RMSE, sMAPE (Makridakis, 1993)  
**Validação:** hold-out temporal — últimos 90 dias como teste
