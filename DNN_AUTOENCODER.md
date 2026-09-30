# 🛡️ DNN + Autoencoder — Detecção em Cascata (CIC-IDS2017)

Notebook: [`DNN_AE.ipynb`](./DNN_AE.ipynb)

Extensão do classificador DNN com uma segunda camada **não supervisionada**: um Autoencoder treinado apenas com tráfego normal, usado para capturar anomalias/possíveis ataques de dia zero (*zero-day*) que escapem do classificador supervisionado. As duas camadas operam em **cascata**.

---

## 📦 Dataset e pré-processamento

Pipeline do projeto: **CIC-IDS2017** via `kagglehub` (`dhoogla/cicids2017`), 2.231.806 registros → 56 features finais após remoção de valores inválidos, features constantes/duplicadas e correlação > 0.95. Split estratificado 80/20 (treino: 1.785.444 · teste: 446.362), `StandardScaler` ajustado no treino[cite: 3.2].

---

## 🤖 Camada 1 — DNN Classificadora (supervisionada)

Arquitetura do projeto (`Dense 128-64-32`, Dropout 0.3/0.2, saída sigmoid), treinada com Normal + Ataque, threshold de decisão **0.1**[cite: 3.2].

| Métrica | Valor |
|---|---|
| Accuracy | 0.9889 |
| Precisão / Recall (Ataque) | 0.94 / 0.99 |
| F1 (Ataque) | 0.96 |
| ROC-AUC | 0.9996 |

---

## 🧩 Camada 2 — Autoencoder (não supervisionada)

Treinado **somente com tráfego normal** (o modelo aprende a reconstruir o padrão "normal"; nunca vê ataques no treino).

- **Dados de teste avaliados:** 1.854.979 amostras (incluindo todo o tráfego benigno de validação + tráfego de ataque)[cite: 3.2].
- **Arquitetura:**
(saída **linear**, não sigmoid, pois os dados escalonados com `StandardScaler` podem ser negativos)
- Loss: `MSE` · Otimizador: Adam (`lr=0.001`) · `EarlyStopping` (paciência 5)
- Treinamento: convergiu na época 14 (até 50 épocas)
- **Threshold de anomalia:** calibrado no **percentil 97 (P97)** do erro de reconstrução (MSE) no conjunto de validação normal → `0.0004`.

### Resultado do Autoencoder isolado (Threshold P97)

| Métrica | Normal | Ataque |
|---|---|---|
| Precisão | 0.95 | **0.78** |
| Recall | 0.95 | **0.76** |
| F1 | 0.95 | **0.77** |

Accuracy geral: **0.9170**

---

## 🔗 Cascata: DNN → Autoencoder

Regra de decisão (camada de proteção em duas etapas):

1. Se a **DNN classifica como Ataque** → decisão final = Ataque (bloqueio direto).
2. Se a **DNN classifica como Normal** → o tráfego passa pelo Autoencoder:
   - Autoencoder também diz Normal → decisão final = Normal.
   - Autoencoder aponta anomalia → escalado para investigação (tratado como potencial Ataque/Zero-Day).

---

## 🔍 Conclusão

A DNN supervisionada atinge desempenho excelente isoladamente (**ROC-AUC de 0.9996** e **Recall de 0.99** na classe Ataque). Ao reajustar o limiar do Autoencoder para o **percentil 97 (P97)**, obtém-se um equilíbrio ideal entre Precisão (0.78) e Recall (0.76) sem depender de rótulos de ataques conhecidos no treino[cite: 3.2]. O ganho real da cascata é atuar como uma **segunda linha de defesa não supervisionada** para capturar anomalias desconhecidas (*zero-day*) que passariam despercebidas por um classificador puramente supervisionado.

---
