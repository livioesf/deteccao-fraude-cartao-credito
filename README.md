# Detecção de Fraude em Transações de Cartão de Crédito

Projeto do bootcamp **Accenture – Python para Análise e Automação de Dados** (DIO).
Treino e comparação de modelos para detectar fraude numa base em que só ~0,17% das transações são fraudulentas, onde a **acurácia engana** e o foco vai para **recall, precisão e F1 da classe de fraude**.

## Resultado em uma linha
XGBoost com hiperparâmetros escolhidos pela PR-AUC e limiar de decisão 0,8 recuperou **81 das 98 fraudes** do conjunto de teste (recall 0,8265) com **precisão de 0,9000** (F1 0,8617).

## 1. O problema e por que o desbalanceamento muda a forma de avaliar
- Base: `creditcard.csv`, com 284.807 transações e 492 fraudes (0,1727%). `Class` = 1 indica fraude.
- Um modelo que responde "normal" para tudo acerta ~99,8% e não detecta nenhuma fraude. Por isso a avaliação usa **recall** (fraudes reais encontradas), **precisão** (alertas que eram fraude de fato) e **F1**, além da curva Precisão x Recall.

## 2. Preparação dos dados
- `np.log1p` em `Amount`, que é muito assimétrico.
- Divisão 80/20 com `stratify=y`: treino com 227.845 linhas (0,1729% de fraude) e teste com 56.962 (0,1720%, 98 fraudes).
- `StandardScaler` dentro do `Pipeline`, ajustado só no treino, para evitar vazamento de dados do teste.

## 3. Comparação dos modelos (limiar 0,5, classe fraude)
| Modelo | Acurácia | Recall | Precisão | F1 | PR-AUC |
|---|---|---|---|---|---|
| Reg. Logística (baseline) | 0,9992 | 0,6531 | 0,8312 | 0,7314 | 0,7386 |
| Reg. Logística (balanced) | 0,9752 | 0,9184 | 0,0603 | 0,1131 | 0,7113 |
| Random Forest (balanced) | 0,9994 | 0,8061 | 0,8404 | 0,8229 | 0,8314 |
| XGBoost | 0,9995 | 0,8265 | 0,8804 | 0,8526 | 0,8805 |

- A acurácia não separa os modelos (0,975 a 0,9995). O baseline chega a 0,9992 com recall de apenas 0,6531.
- A regressão logística `balanced` tem o maior recall, mas só ~6% dos alertas eram fraude de verdade, o que a torna inviável na prática.
- O XGBoost teve o melhor equilíbrio e a maior PR-AUC. Contra o Random Forest, a diferença de recall é de apenas 2 fraudes (81 contra 79), num teste com 98 fraudes.
- A curva ROC quase não separa os modelos (AUC de 0,96 a 0,98), enquanto a curva Precisão x Recall separa bem, porque ignora o grande volume de transações normais.

## 4. Limiar de decisão e SHAP
- **Limiar escolhido: 0,8**, o de maior F1 (0,8617) na tabela de 0,1 a 0,9, com recall de 0,8265 e precisão de 0,9000. Baixar para 0,1 sobe o recall para 0,8571, mas a precisão cai para 0,7778.
- **Limitação:** o limiar foi escolhido no próprio conjunto de teste, que tem só 98 fraudes; diferenças de 1 ou 2 fraudes entre limiares são ruído. O ideal seria usar um conjunto de validação separado.
- **SHAP:** as variáveis que mais pesaram foram V14, V4, V10 e V12. Valores baixos de V14, V10 e V12 e valores altos de V4 empurram a previsão para fraude. Como `V1`–`V28` são componentes de PCA, a análise mostra quais componentes importam, não um significado de negócio. O SHAP mostra associação, não causa.

## 5. O que eu mudei em relação à Expert
- O roteiro sugeria limiar de 0,3 e `GridSearchCV` pelo recall. Segui, e o modelo ficou com recall de 0,9286, mas com precisão de 0,0443 (1.964 falsos alarmes para 91 fraudes pegas; detalhes no [RUNBOOK](RUNBOOK.md)).
- Troquei o critério da busca para `average_precision` e passei a escolher o limiar pelo maior F1 numa tabela de 0,1 a 0,9.
- Comparei a regressão logística com e sem balanceamento e limitei `max_depth=12` no Random Forest para o treino não ficar lento no Colab.

## Como rodar
1. Abra `deteccao_fraude_cartao.ipynb` no Google Colab.
2. Execute todas as células em ordem (Ambiente de execução → Executar tudo). O treino leva alguns minutos.
3. O dataset é lido direto do link; ele **não** está neste repositório (passa do limite de 100 MB do GitHub).

## Estrutura
```
.
├── deteccao_fraude_cartao.ipynb   # notebook com as saídas salvas
├── README.md
└── RUNBOOK.md                     # diário de erros, causas e correções
```

## Próximos passos
Escolher o limiar num conjunto de validação separado, testar undersampling/oversampling, ampliar a busca de hiperparâmetros e criar variáveis de comportamento ao longo do tempo.
