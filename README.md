# 🛡️ Detecção de Intrusão e Ameaças em Redes (NIDS): Estudo Comparativo

Este repositório contém um estudo comparativo sobre o uso de **Aprendizado de Máquina** e **Deep Learning** para Detecção de Intrusões e Ameaças em Redes de Computadores (NIDS), utilizando o dataset de referência **CIC-IDS2017**.

O projeto avalia três paradigmas de Inteligência Artificial no contexto de um **Centro de Operações de Segurança (SOC)**: de um modelo linear supervisionado ultra-rápido a uma rede neural profunda, passando por uma camada não supervisionada de detecção de anomalias.

---

## 📌 Visão Geral da Arquitetura e Modelos

A solução foi estruturada em três abordagens complementares que simulam uma estratégia de **Defesa em Camadas (Defense-in-Depth)**:

1. **Regressão Logística (`class_weight='balanced'`)**
   - **Tipo:** Supervisionado Linear.
   - **Função no SOC:** *Baseline* leve e de baixíssimo custo computacional para filtragem e triagem preliminar de tráfego no perímetro de rede.

2. **DNN Classificadora (Multi-Layer Perceptron)**
   - **Tipo:** Supervisionado Não-Linear Profundo (Keras/TensorFlow).
   - **Função no SOC:** Classificação de alta precisão para assinaturas e categorias de ataques conhecidos, minimizando alarmes falsos.

3. **Autoencoder (detecção de anomalias)**
   - **Tipo:** Não Supervisionado (treinado apenas com tráfego normal).
   - **Função no SOC:** Segunda linha de defesa em cascata após a DNN, capturando padrões de tráfego nunca vistos (potenciais ataques de dia zero) que escapariam de um classificador puramente supervisionado.

---

## 🏗️ Pipeline de Engenharia e Pré-processamento de Dados

Pipeline padronizado e compartilhado por todos os modelos, garantindo reprodutibilidade e evitando vazamento de dados (*data leakage*):

- **Dataset:** CIC-IDS2017 (`dhoogla/cicids2017`, via `kagglehub`) — 8 arquivos `.parquet` consolidados, 2.231.806 registros.
- **Limpeza de Dados:** remoção de duplicatas, tratamento de valores infinitos/ausentes (`inf`, `NaN`) e descarte de colunas com variância zero.
- **Redução de Features:** eliminação de features duplicadas e de pares com correlação > 0.95 → **56 features finais**.
- **Alvo binário:** `Benign = 0` (84.92%) vs `Attack = 1` (15.08%).
- **Divisão Estratificada:** treino/teste 80/20 mantendo a proporção real das classes (`stratify=y`).
- **Escalonamento:** `StandardScaler` ajustado estritamente nos dados de treino.

---
## 📊 Tabela Comparativa de Métricas e Desempenho

Resultados no conjunto de teste (446.362 amostras na DNN/RL e 1.854.979 amostras no Autoencoder isolado):

| Modelo / Métrica | Regressão Logística | DNN Classificadora | Autoencoder (isolado, P97) | Cascata DNN → AE |
|---|---|---|---|---|
| **Tipo de Aprendizado** | Supervisionado Linear | Supervisionado Não-Linear | Não Supervisionado | Híbrido |
| **Dados de Treino** | Normal + Ataque | Normal + Ataque | Somente Normal | Normal + Ataque |
| **Acurácia Geral** | 94.31% | 98.89% | 91.70% | 98.89% |
| **ROC-AUC** | 0.9922 | 0.9996 | — | — |
| **Recall (Ataque)** | 97% | 99% | 76% | 99% |
| **Precisão (Ataque)** | 74% | 94% | 78% | 94% |
| **F1-Score (Ataque)** | 0.84 | 0.96 | 0.77 | 0.96 |

> A cascata DNN → Autoencoder atua como uma **segunda camada de proteção**: o tráfego classificado como Normal pela DNN é auditado pelo Autoencoder. Com o limiar calibrado no **percentil 97 (P97)**, o Autoencoder reduz significativamente os falsos negativos sem comprometer a capacidade de detectar anomalias e potenciais ameaças de dia zero (*zero-day*).
---

## 🔍 Análise Crítica dos Resultados (Cibersegurança & SOC)

- **Trade-off da Regressão Logística:** O uso de `class_weight='balanced'` eleva o Recall para 97%, mas a incapacidade de separar fronteiras não lineares mantém a Precisão em apenas 74%. Isso gera mais de 23.000 alarmes falsos que sobrecarregariam um SOC (*Security Operations Center*) com fadiga de alertas.
- **Superioridade da DNN Classificadora:** Ao mapear as interações complexas e não lineares entre as 56 features de tráfego, a DNN reduz os Falsos Positivos em mais de 80% em relação ao baseline, elevando a Precisão para 94% e o Recall para 99% (ROC-AUC de 0.9996).
- **Calibração e Papel do Autoencoder:** Com a calibração do limiar no **percentil 97 (P97)** do erro de reconstrução (MSE), o Autoencoder atinge um equilíbrio muito mais robusto e operacional: Precisão de 78% e Recall de 76% na classe Ataque (F1-Score de 0.77). O seu valor estratégico não reside em substituir a DNN, mas em atuar como uma **segunda linha de defesa não supervisionada**: por ser treinado exclusivamente com tráfego benigno, não depende de assinaturas nem de rótulos prévios, sendo capaz de detetar anomalias estatísticas e potenciais ataques de dia zero (*zero-day*) que passariam despercebidos pelo classificador supervisionado.
  
---

## 🎓 Conclusões do Estudo

1. **A complexidade da rede neural é demonstrada e justificada:** converter uma solução linear (Regressão Logística) em não linear (DNN) elimina o gargalo de alarmes falsos mantendo a detecção de invasões no nível máximo.
2. **Defesa em Camadas:** o modelo linear serve para descarte rápido no perímetro, a DNN resolve a classificação exata de ameaças conhecidas, e o Autoencoder cobre a lacuna de ataques desconhecidos/zero-day antes de escalar para investigação humana.

---

## 📂 Estrutura de Documentos do Repositório

- [`RL x DNN.ipynb`](./RL_ML%20x%20DNN.ipynb) — Notebook comparando Regressão Logística e DNN.
- [`RL_DNN-README.md`](./RL_DNN-README.md) — Documentação detalhada da Regressão Logística e da DNN Classificadora.
- [`DNN_AE.ipynb`](./DNN_AE.ipynb) — Notebook com a DNN, o Autoencoder e a cascata DNN → Autoencoder.
- [`DNN_AUTOENCODER.md`](./DNN_AE-README.md) — Documentação detalhada do Autoencoder e da cascata de detecção.
- [`requirements.txt`](./DNN_AE-requirements.txt) — Dependências dos notebooks Regressão Logística + DNN + Autoencoder.

---

## ▶️ Como executar

```bash
pip install -r requirements.txt  

```

Requer uma conta Kaggle configurada (`kagglehub`) para baixar o dataset CIC-IDS2017 automaticamente na primeira execução.
