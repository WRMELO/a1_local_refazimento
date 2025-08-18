

> Documento de referência didática para o MVP de ML/DL no CFB.  
> **Granularidade temporal:** timestamps **horários** (1 amostra por hora).  
> **Dataset base:** `A1_ML_DL.csv` (2 linhas de cabeçalho: **[nome | dimensão]**).

---

## 1) O que temos

- `A1_ML_DL.csv` com amostragem **horária**.  
- Da **coluna 152 (`TAU_DENSA`) até a última**, há variáveis físicas necessárias para **calcular Wr (refratário)** e **Wm (metálico)** a partir do **equacionamento teórico** (PDF interno).  
- **Busca semântica sempre ativa:** qualquer nome de variável citado (PT/EN) é mapeado automaticamente para a coluna real (não exige igualdade literal).

> **Importante:** as colunas **152→fim** servem **apenas** para formar Wr/Wm (rótulos). **Não** entram como features do modelo (para evitar vazamento).

---

## 2) Classes de variáveis

- **Primárias (controle):** sinais manipulados pela operação (ex.: vazões de ar primário/total/SEC, vazão de carvão, setpoints).  
- **Secundárias (resposta):** o processo responde às primárias (temperaturas, pressões, ΔP, O₂ etc.).  
- **Desgaste (alvo):** **Wr** e **Wm**, calculados via relações físico-químicas.

**Objetivo do MVP:** prever a **taxa de desgaste futura** (Wr/Wm em `t+H`) a partir do histórico até `t`, identificar **regimes** e **alavancas operacionais** que influenciam o desgaste.

---

## 3) Geração dos rótulos (Wr/Wm)

1. **Calcular Wr e Wm por hora** usando as variáveis **152→fim** e o equacionamento do PDF.  
   - Usar constantes/termos adimensionais conforme o documento; se necessário, calibrar **um fator global** para alinhar a escala.  
   - Nome sugerido das colunas: `wr_kg_m2_h` e `wm_kg_m2_h`.  
2. **Persistir rótulos** em um novo arquivo: `A1_ML_DL_rotulado.csv` (mantendo a **2ª linha de dimensões**).  
3. **Excluir das features** todas as colunas usadas no cálculo (152→fim).  
4. (Opcional) Criar `wr_future_H`/`wm_future_H` deslocando `H` horas à frente, se formos prever a frente.

> **Checagens:** distribuição, outliers, sazonalidade e estacionariedade de `Wr/Wm` (e dos resíduos) após o cálculo.

---

## 4) Por que **não** embaralhar (shuffle) os dados?

É **série temporal**. Se embaralhar:
- o modelo aprende com **informação do futuro** (vazamento) e as métricas ficam **otimistas**;
- perdemos a noção de **deriva** (mudança de regime/condição ao longo do tempo).

**Correto:** separar por **blocos de tempo** (ex.: Treino = Meses 1–2, Validação = Mês 3, Teste = Mês 4).  
Validar com **forward chaining** (backtesting): treina até `T1`, valida em `T2`; depois avança a janela.

---

## 5) Janelas e horizonte — ajustados para dados **horários**

- **Janela (W)** = horas de histórico usadas para prever:  
  - Inicial: **W = 1h**.  
  - Experimentos: **W = 2h** e **W = 4h** (capturar dinâmicas mais lentas).  
- **Horizonte (H)** = em quantas horas à frente queremos prever:  
  - Inicial: **H = 1h**.  
  - Experimentos: **H = 2h** e **H = 4h**.

> Critério: escolher W/H conforme o erro e a utilidade operacional (tempo de reação).

---

## 6) Preparação dos dados (para ambos os caminhos)

- **Index temporal:** unicidade, ordenação e regularidade **horária**; resample se necessário.  
- **Nulos:** `ffill` curto + mediana; registrar taxa por coluna.  
- **Outliers:** winsorização leve (p1–p99) em sinais contínuos.  
- **Escalonamento:** `RobustScaler` por grupo de **dimensão** (°C, MPa, t/h, %, etc.).  
- **Features por janela (tabular):** média, std, min/max, p10/p90, IQR, slope (tendência), % acima de limite físico, contagem de picos, energia espectral simples.  
- **Relações físicas:** razões (ex.: `vazao_ar_primario` / `vazao_ar_total`), ΔP normalizada por vazão, balanços simplificados.

---

## 7) Caminho **Supervisionado** (principal do MVP)

### 7.1 Modelos baseline
- **Tabular:** **XGBoost/LightGBM** (regressão) com features agregadas por janela.  
- **Sequencial:** **TCN** raso (ou GRU) recebendo a sequência horária da janela.

### 7.2 Alvo
- `Wr(t+H)` e/ou `Wm(t+H)` — prever taxa de desgaste **à frente**.

### 7.3 Métricas e validação
- **Métricas:** MAE, RMSE, MAPE; acerto de **tendência** (subida/descida).  
- **Validação temporal:** blocos cronológicos + backtesting + baseline de persistência.  
- **Interpretabilidade:** SHAP (tabular) e saliency/IG (sequencial) para destacar variáveis relevantes.

> **Antivazamento:** remover do feature set as colunas 152→fim (usadas no cálculo de Wr/Wm) e quaisquer colunas futuras/derivadas.

---

## 8) Caminho **Não-supervisionado** (radar complementar)

**Objetivo:** detectar **mudança de regime** e **anomalias** que podem **preceder** elevação de desgaste, mesmo sem rótulos perfeitos.

- **PCA + Monitoramento:** estatísticas T² (Hotelling) e Q (SPE).  
- **Autoencoder sequencial (TCN-AE):** treinar em janelas “normais”; usar **erro de reconstrução** como índice de anomalia.  
- **Change-points** (PELT/BinSeg): marcar rupturas persistentes (em variáveis-chave e/ou no score do AE).  
- **Clustering de janelas** (k-means/HDBSCAN em embeddings): separar **regimes** e analisar **transições**.

**Uso prático:** cruzar janelas anômalas com **Wr/Wm** calculados. Se o “radar” dispara **antes** de Wr/Wm subir, temos **alerta precoce**.

---

## 9) Modelo em 2 etapas (opcional — para alavancas de controle)

1. **Primárias → Secundárias** (dinâmica do processo).  
2. **Secundárias → Wr/Wm** (desgaste).  

Permite simulações “e se”: alterar primárias e observar o efeito projetado nas secundárias e no desgaste.

---

## 10) Entregáveis do MVP

- `A1_ML_DL_rotulado.csv` (Wr/Wm anexados; 2ª linha de dimensões preservada).  
- **Relatório EDA** (distribuições, nulos, outliers, correlações).  
- **Modelos baseline** (XGBoost e TCN) com **validação temporal**, métricas e gráficos de previsão vs. real.  
- **Explicabilidade** (SHAP/IG).  
- **Radar não-supervisionado** com mapa de regimes/anomalias no tempo.  
- **Recomendações operacionais** (ex.: ajustes em vazões/combustível por regime).

---

## 11) Riscos e mitigações

- **Vazamento:** remover features usadas no cálculo de Wr/Wm; splits temporais.  
- **Deriva:** monitorar distribuição entre blocos; recalibrar normalizadores.  
- **Dados faltantes:** imputar + sinalizar; registrar intervalos sem confiança.  
- **Escalas físicas:** checar unidades; padronizar por **dimensão**.

---

## 12) Próximos passos (execução)

- [ ] Implementar cálculo **Wr/Wm por hora** com variáveis **152→fim**.  
- [ ] Gerar `A1_ML_DL_rotulado.csv` e **excluir 152→fim** das features.  
- [ ] Rodar **EDA dos rótulos** e definir **W/H** iniciais (**W=1h**, **H=1h**; testar **2h/4h**).  
- [ ] Treinar **XGBoost** (tabular) e **TCN** (sequencial) com **split temporal**.  
- [ ] Construir **radar não-supervisionado** (PCA + AE + change-points).  
- [ ] Consolidar **métricas**, gráficos e **recomendações operacionais**.

---

## 13) Convenções e contrato de dados

- **Leitura do CSV:** `header=[0,1]` (linha 1 = nome, linha 2 = dimensão).  
- **Amostragem:** **1 hora**; janelas e horizontes são múltiplos de 1h.  
- **Busca semântica:** nomes de variáveis (PT/EN) mapeados automaticamente.  
- **Armazenamento:**
  - Entrada: `A1_ML_DL.csv`  
  - Saída (rotulado): `A1_ML_DL_rotulado.csv`  
  - Artefatos (modelos/relatórios/gráficos): `/data/analises_preliminares`.

---

### Anexo
- Equacionamento físico: `0_teoria_combustao_e_desgaste.pdf` (referência para calcular Wr/Wm).
