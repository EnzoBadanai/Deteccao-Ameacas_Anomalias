# 🧠 Regressão Logística × DNN — Detecção de Intrusão (CIC-IDS2017)

Notebook: [`RL_x DNN.ipynb`](./RL%20x%20DNN.ipynb)

Comparação lado a lado entre um modelo linear (Regressão Logística) e uma rede neural profunda (DNN) para classificação binária de tráfego de rede — **Normal vs Ataque** — usando o dataset **CIC-IDS2017**.

---

## 📦 Dataset

- **Fonte:** [`dhoogla/cicids2017`](https://www.kaggle.com/datasets/dhoogla/cicids2017) (Kaggle, via `kagglehub`)
- **8 arquivos Parquet** (Benign, Botnet, Bruteforce, DDoS, DoS, Infiltration, Portscan, WebAttacks)
- **2.231.806 registros** após concatenação e remoção de duplicatas, **78 colunas**

## 🏗️ Pipeline de pré-processamento

1. Remoção de valores infinitos/`NaN`
2. Remoção de **8 features constantes** (variância zero)
3. Remoção de **8 features duplicadas** (ex.: `SYN Flag Count`, `Subflow Fwd Packets`)
4. Remoção manual de **7 features com correlação > 0.95** (ex.: `Subflow Bwd Bytes`, `Avg Packet Size`, `Fwd IAT Max`)
5. **56 features finais**
6. Alvo binário: `Benign = 0` (84.92%) / `Ataque = 1` (15.08%)
7. Split estratificado 80/20 (treino: 1.785.444 · teste: 446.362)
8. `StandardScaler` ajustado apenas no treino

## 🤖 Modelos

### 1. Regressão Logística
```python
LogisticRegression(max_iter=3000, class_weight='balanced', random_state=42)
```
Uso de `class_weight='balanced'` para compensar o desbalanceamento de classes.

### 2. DNN (Keras/TensorFlow)
```
Input(56) → Dense(128, relu) → Dropout(0.3)
          → Dense(64, relu)  → Dropout(0.2)
          → Dense(32, relu)
          → Dense(1, sigmoid)
```
- Otimizador: Adam (`lr=0.001`) · Loss: `binary_crossentropy`
- `EarlyStopping` + `ReduceLROnPlateau` (monitor `val_loss`)
- 20 épocas · batch size 1024 · `validation_split=0.1`
- Threshold de decisão: **0.1** (priorizando recall na classe Ataque)

## 📊 Resultados (conjunto de teste, 446.362 amostras)

| Métrica | Regressão Logística | DNN |
|---|---|---|
| Accuracy | 0.9431 | **0.9889** |
| F1 Macro | 0.9015 | **0.9788** |
| ROC-AUC | 0.9922 | **0.9996** |
| PR-AUC | — | 0.9981 |
| Precisão (Ataque) | 0.74 | **0.94** |
| Recall (Ataque) | 0.97 | **0.99** |

**Matrizes de confusão (TN / FP / FN / TP):**

- Regressão Logística: `355565 / 23498 / 1882 / 65417`
- DNN: `374540 / 4523 / 451 / 66848`

## 🔍 Conclusão

A Regressão Logística, mesmo balanceada, não consegue separar as fronteiras não lineares do tráfego malicioso, gerando ~25% de falsos positivos na classe Ataque. A DNN reduz drasticamente esses falsos positivos mantendo recall equivalente, funcionando como classificador de alta precisão para ataques conhecidos — a Regressão Logística permanece útil como *baseline* leve para triagem de baixo custo computacional.

## ▶️ Como executar

```bash
pip install -r RL_DNN-requirements.txt
```

Requer uma conta Kaggle configurada (`kagglehub`) para baixar o dataset automaticamente na primeira execução.
