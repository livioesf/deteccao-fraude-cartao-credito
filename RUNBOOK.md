# Runbook do projeto: detecção de fraude

Registro dos problemas que apareceram durante o projeto, no formato:
**tentei X → deu o erro Y → descobri a causa Z → corrigi assim → evidência → prevenção.**

---

### RB-01 · Acurácia altíssima, mas fraude mal detectada
| Campo | Conteúdo |
|---|---|
| **Etapa** | Avaliar |
| **Objetivo** | Medir a qualidade dos modelos treinados |
| **Tentativa** | Comparar os modelos pela acurácia (`accuracy_score`) |
| **Sintoma** | O baseline tinha acurácia de 0,9992, mas recall da fraude de só 0,6531. A acurácia ia de 0,9752 a 0,9995 entre os modelos e não os diferenciava |
| **Diagnóstico** | Só 0,17% das transações são fraude; prever "normal" sempre já rende ~99,83% de acurácia. A acurácia mede quase só a classe normal |
| **Correção** | Passei a comparar recall, precisão, F1 e PR-AUC da classe de fraude, e usei a curva Precisão x Recall |
| **Evidência** | Regressão logística `balanced`: recall 0,9184, mas precisão 0,0603 e a menor acurácia (0,9752). XGBoost: recall 0,8265 e precisão 0,8804 |
| **Prevenção** | Definir a métrica da classe rara antes de treinar |

---

### RB-02 · Modelo ajustado pelo recall virou um gerador de falsos alarmes
| Campo | Conteúdo |
|---|---|
| **Etapa** | Ajustar |
| **Objetivo** | Aumentar o recall com `GridSearchCV` e limiar de 0,3, como no roteiro |
| **Tentativa** | `GridSearchCV(..., scoring="recall")` + `LIMIAR = 0.3` |
| **Sintoma** | Recall de 0,9286 (91 fraudes pegas), mas precisão de 0,0443 e **1.964 falsos alarmes**. O F1 caiu de 0,8454 (XGBoost original no limiar 0,3) para 0,0845 |
| **Diagnóstico** | Recall sozinho não penaliza falsos alarmes: a busca escolheu `max_depth=3`, `learning_rate=0.05`, 100 árvores, e com limiar baixo o modelo passou a acusar milhares de transações normais |
| **Correção** | Troquei o critério para `scoring="average_precision"` (PR-AUC, que olha precisão e recall) e passei a escolher o limiar pelo maior F1 numa tabela de 0,1 a 0,9 |
| **Evidência** | Parâmetros escolhidos: `max_depth=5`, `n_estimators=300`, `learning_rate=0.1`. Limiar 0,8: recall 0,8265, precisão 0,9000, F1 0,8617. Perdi 10 fraudes e deixei de bloquear quase 1.950 clientes legítimos |
| **Prevenção** | Em problemas desbalanceados, não otimizar uma métrica isolada: acompanhar recall e precisão juntos. Mudei critério e limiar ao mesmo tempo, então não isolei qual dos dois pesou mais; num próximo ciclo, testaria um de cada vez e escolheria o limiar num conjunto de validação |

---

### RB-03 · `NameError: name 'GridSearchCV' is not defined`
| Campo | Conteúdo |
|---|---|
| **Etapa** | Ajustar |
| **Objetivo** | Rodar a célula corrigida do `GridSearchCV` |
| **Tentativa** | Executar a célula nova diretamente |
| **Sintoma** | `NameError` na primeira linha, mesmo com os imports aparecendo no notebook |
| **Diagnóstico** | A sessão do Colab tinha reiniciado e perdido as variáveis da memória; as saídas antigas na tela eram só resquício |
| **Correção** | Ambiente de execução → **Executar antes**, para refazer imports, dados e treinos, e depois rodar a célula de novo |
| **Evidência** | A célula rodou normalmente (cerca de 1 minuto) |
| **Prevenção** | Depois de qualquer desconexão, reexecutar o notebook desde o início; no fim, validar com "Executar tudo" |

---

## Lições aprendidas
1. Em dados desbalanceados, a acurácia engana: escolha as métricas pela classe rara antes de treinar.
2. Otimizar uma métrica só (recall) pode piorar outra a ponto de inviabilizar o modelo; acompanhe recall, precisão e F1 juntos.
3. Escolher o limiar no conjunto de teste é uma limitação; o ideal é um conjunto de validação separado, principalmente com só 98 fraudes no teste.
