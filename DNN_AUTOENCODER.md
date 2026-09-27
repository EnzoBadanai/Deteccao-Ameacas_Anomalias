# 🛡️ DNN + Autoencoder — Detecção em Cascata (CIC-IDS2017)

Notebook: [`DNN_AE.ipynb`](./DNN_AE.ipynb)

Extensão do classificador DNN com uma segunda camada **não supervisionada**: um Autoencoder treinado apenas com tráfego normal, usado para capturar anomalias/possíveis ataques de dia zero que escapem do classificador supervisionado. As duas camadas operam em **cascata**.

---

## 📦 Dataset e pré-processamento

Mesmo pipeline do projeto RL × DNN: **CIC-IDS2017** via `kagglehub` (`dhoogla/cicids2017`), 2.231.806 registros → 56 features finais após remoção de valores inválidos, features constantes/duplicadas e correlação > 0.95. Split estratificado 80/20 (treino: 1.785.444 · teste: 446.362), `StandardScaler` ajustado no treino.

## 🤖 Camada 1 — DNN Classificadora (supervisionada)

Mesma arquitetura do projeto RL × DNN (`Dense 128-64-32`, Dropout, saída sigmoid), treinada com Normal + Ataque, threshold de decisão **0.1**.

| Métrica | Valor |
|---|---|
| Accuracy | 0.99 |
| Precisão / Recall (Ataque) | 0.93 / 0.99 |
| F1 (Ataque) | 0.96 |
| ROC-AUC | 0.9996 |
| PR-AUC | 0.9981 |

## 🧩 Camada 2 — Autoencoder (não supervisionada)

Treinado **somente com tráfego normal** (o modelo aprende a reconstruir o padrão "normal"; nunca vê ataques no treino).

- **Dados de treino:** 1.364.625 registros normais · **validação:** 151.626 (também só normal, usada para calibrar o threshold)
- **Arquitetura:**
```
Input(56) → Dense(64, relu) → Dense(32, relu) → Dense(16, relu, bottleneck)
          → Dense(32, relu) → Dense(64, relu) → Dense(56, linear)
```
  (saída **linear**, não sigmoid, pois os dados escalonados com `StandardScaler` podem ser negativos)
- Loss: `MSE` · Otimizador: Adam (`lr=0.001`) · `EarlyStopping` + `ReduceLROnPlateau`
- Até 50 épocas · batch size 1024
- **Threshold de anomalia:** percentil 99.99 do erro de reconstrução (MSE) calculado só na validação normal → `2.4072`

### Resultado do Autoencoder isolado

| Métrica | Normal | Ataque |
|---|---|---|
| Precisão | 0.87 | **1.00** |
| Recall | 1.00 | **0.17** |
| F1 | 0.93 | 0.29 |

Accuracy geral: 0.87 · **ROC-AUC (score = erro de reconstrução): 0.9411**

Sozinho, o Autoencoder é um detector conservador: como o threshold é calibrado num percentil muito alto (99.99%) para minimizar falsos positivos em tráfego normal, ele só sinaliza como anomalia os ataques com erro de reconstrução mais extremo — por isso o recall baixo (0.17) na classe Ataque quando usado isoladamente.

## 🔗 Cascata: DNN → Autoencoder

Regra de decisão (camada de proteção em duas etapas):

1. Se a **DNN classifica como Ataque** → decisão final = Ataque (bloqueio direto).
2. Se a **DNN classifica como Normal** → o tráfego passa pelo Autoencoder:
   - Autoencoder também diz Normal → decisão final = Normal.
   - Autoencoder aponta anomalia → escalado para investigação (tratado como Ataque na avaliação da cascata).

### Resultado da cascata completa

| Métrica | Normal | Ataque |
|---|---|---|
| Precisão | 1.00 | 0.93 |
| Recall | 0.99 | 0.99 |
| F1 | 0.99 | 0.96 |

Accuracy geral: 0.99 · **40 pacotes escalados para investigação manual (0.01% do conjunto de teste)**.

## 🔍 Conclusão

Como a DNN já atinge recall/precisão muito altos isoladamente, a cascata neste conjunto de teste reproduz praticamente as mesmas métricas globais — o ganho real do Autoencoder está em atuar como uma **segunda linha de defesa não supervisionada**: ele não depende de rótulos de ataques conhecidos, então tende a capturar padrões de tráfego nunca vistos (potenciais ataques de dia zero) que passariam despercebidos por um classificador puramente supervisionado, escalando esses poucos casos para investigação em vez de descartá-los como Normal.

## ▶️ Como executar

```bash
pip install -r DNN_AE-requirements.txt
```

Requer uma conta Kaggle configurada (`kagglehub`) para baixar o dataset automaticamente na primeira execução.

Artefatos exportados ao final do notebook: `model_dnn.keras`, `model_ae.keras`, `scaler.pkl`, `feature_columns.json`, `threshold_ae.json`.
