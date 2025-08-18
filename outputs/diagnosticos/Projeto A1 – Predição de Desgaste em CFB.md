# Projeto A1 – Predição de Desgaste em CFB

## Relatório técnico (versão MVP) — engenharia de dados, modelagem física, índices e diagnóstico estatístico

> Este documento consolida **o que foi desenvolvido** no refazimento do Projeto A1, incluindo: consolidação de dados (SSOT local), normalização semântica, proxies físico-baseadas de desgaste (**Wr**, **Wm**), definição de **referências** (v1, v2 e v3-p5), construção de **índices** adimensionais, gráficos e **ROI estatística** (retângulo ±k·σ e elipse de Mahalanobis χ²). No anexo constam **todos os caminhos** de arquivos utilizados e gerados.

---

## 1. Sumário executivo

* **SSOT local** montada a partir dos CSVs originais, eliminando a triplicação por *zona* e padronizando nomes/unidades.
  Resultado base: **11.757** timestamps únicos.

* **Modelagem física (MVP)**: criamos duas grandezas relativas ao desgaste:

  * **Wr** (refratário) e **Wm** (metálico), funções de **vazão de ar (v)**, **queda de pressão (|Δ|)** e **temperatura (τ)**, com pesos distintos para cada material.
  * Introdução de **|Δ|** (magnitude com ε) para eliminar efeitos de sinal e garantir domínio para a raiz de Wm.

* **Referências e índices**:

  * **v1**: referências dos arquivos de *refs*; gerou índices, mas havia sinais negativos em Wr por uso de Δ com sinal.
  * **v2**: referências pelos **mínimos absolutos** observados de Wr/Wm (garantindo índices ≥ 1).
  * **v3 (p5)**: referências **robustas** por **5º percentil (p5)** dentro do *baseline* mais longo (ou no dataset, como fallback), produzindo **Wr\_idx\_p5** e **Wm\_idx\_p5** centrados em **1,0** no p5.

* **Diagnóstico estatístico**:

  * Construímos ROI “normal” por **retângulo ±k·σ** e por **elipse χ²(95%)** (Mahalanobis) **centradas na referência** (v1 para análises históricas e (1,1) no espaço dos índices p5).
  * O **quadrante Q1 (Wr↑, Wm↑)** concentra a maioria dos pontos fora da ROI, explicável por **aumentos de vazão de ar** (peso cúbico em Wm e quadrático em Wr), **maior |Δ|** e **temperaturas mais baixas** (termo exponencial).

* **Resultados operacionais**:

  * Tabelas consolidadas (Parquet/CSV) com **Wr/Wm** e **índices**;
  * Gráficos **temporais** e **Wr×Wm** (bruto e índices), **quadrantes**, **ROI** e arquivos CSV com **outliers** e **drivers**.

---

## 2. Engenharia de dados (SSOT) e padronização

1. **Fontes**:
   `A1_SECONDARIES_FOR_PEDRA_MODEL(_old).csv`, `A1_SECONDARIES_REFS.csv`, `DELTA_PROXY_DIAGNOSTICS.csv`, e artefatos de baseline (quando existentes).

2. **Problema corrigido**: a base original vinha **triplicada** por zona (*densa, diluída, backpass*) — três linhas por timestamp.
   **Solução**: verificamos ausência de conflitos e **colapsamos** em **uma linha por timestamp**, criando colunas por grandeza (p.ex., `tau_densa`, `tau_diluida`, `tau_backpass`) e **média** `tau_mean_C`.

3. **Padronização semântica**:

   * **vazão de ar total** (`total_air_flow_knm3_h`) preferida; *fallback* = vazão primária;
   * **ΔP** tratado como **magnitude**: `delta_mag = max(|delta_proxy|, ε)` com ε ≈ 1e-9;
   * **temperatura absoluta**: `tau_K = mean(tau_*) + 273.15`.

4. **SSOT local**: os derivados consolidados são gravados em
   `...A1_LOCAL_REFAZIMENTO\data\derivadas\*.parquet|.csv` (ver Anexo A).

---

## 3. Modelagem físico-baseada (MVP)

### 3.1. Equações (v2/v3)

Com **parâmetros padrão** (ajustáveis):
$n_r=2,\; m_r=1,\; n_m=3,\; m_m=0.5,\; T_{crit}=1200\,K,\; T_{opt}=1250\,K$.

$$
\begin{aligned}
&Wr \;\propto\; v^{\,2}\cdot \Delta_{\text{mag}}^{\,1}\cdot \exp\!\Big(-\frac{\tau_K}{T_{crit}}\Big) \\
&Wm \;\propto\; v^{\,3}\cdot \Delta_{\text{mag}}^{\,0.5}\cdot \exp\!\Big(-\frac{\tau_K}{T_{opt}}\Big)
\end{aligned}
$$

* **v** = vazão (preferência: total de ar).
* **$\Delta_{\text{mag}}$** = $|\Delta P|$ com ε.
* **$\tau_K$** = temperatura em Kelvin.
* O termo exponencial **reduz** desgaste quando a operação está mais quente (dentro do regime seguro).
* **Wm** depende mais fortemente de **v** (expoente 3), portanto responde com maior inclinação a mudanças de fluxo.

### 3.2. Análise dimensional (resumo)

Se $v:[L^3T^{-1}]$ e $\Delta:[ML^{-1}T^{-2}]$, então
$[Wr]\sim [M^1L^{5}T^{-4}],\quad [Wm]\sim [M^{1/2}L^{17/2}T^{-4}]$.
**Conclusão**: Wr e Wm são **proxies** com unidade composta; em operação usamos os **índices** (adimensionais) para comparação temporal.

---

## 4. Referências e índices

### 4.1. v1 — referências dos artefatos de *refs*

* Uso direto dos valores em `A1_SECONDARIES_REFS.csv` como $v_{ref}, \Delta_{ref}, \tau_{ref}$.
* Problema: Δ com sinal → **Wr** podia ficar **negativo**; **Wm** tinha **NaN** em $\sqrt{\Delta<0}$.

### 4.2. v2 — referências pelos **mínimos absolutos**

* Com $|\Delta|$: **Wr\_ref = min(Wr)**, **Wm\_ref = min(Wm)** ⇒ índices **≥ 1** por construção.
* Benefícios: remove sinais negativos e maximiza cobertura de Wm.
* Aviso: se o **mínimo** for outlier, índices podem inflar.

### 4.3. v3 — referências **robustas p5**

* Escolhemos a **janela de baseline** com **maior duração** (quando existente).
* **Wr\_ref\_p5** = p5(Wr na janela); **Wm\_ref\_p5** = p5(Wm na janela).
* Índices: **Wr\_idx\_p5 = Wr/Wr\_ref\_p5**, **Wm\_idx\_p5 = Wm/Wm\_ref\_p5**.
* Vantagem: reduz sensibilidade a pontos extremos e mantém leitura “> 1” = mais agressivo que a referência.

> Os valores numéricos, a janela usada e estatísticas de **vazão de ar** e **vazão de carvão (linha c)** da janela estão registrados em
> `...outputs\diagnosticos\physics_refs_p5_summary.json`.

---

## 5. ROI (“normalidade”) e gráficos

### 5.1. Conceito de ROI

* **Retângulo ±k·σ** (por eixo), centrado na referência: simples e interpretável.
* **Elipse χ²** (Mahalanobis, 2 g.l.): leva em conta a **correlação Wr–Wm**; mais adequada ao seu caso (nuvem alongada).

A elipse é **estimada no baseline** e **aplicada** a todo o conjunto, portanto **não contém 95% do dataset inteiro**, e sim \~95% do **baseline** — fora dele é normal ver mais pontos **fora**.

### 5.2. Gráficos produzidos

* **Séries temporais** de `Wr_idx`/`Wm_idx`, com **baseline sombreado**.
* **Wr×Wm (bruto)** com eixos na **referência p5**, elipse χ²(95%) e **contagem por quadrantes** fora da elipse.
* **Wr\_idx\_p5×Wm\_idx\_p5 (índices)** com eixos em **(1,1)**, elipse χ²(95%) e quadrantes.

> Os arquivos `.png` estão em `...outputs\diagnosticos\` e os **outliers** correspondentes em `.csv`.

---

## 6. Interpretação dos resultados

1. **Por que Q1 domina?**

   * A **vazão de ar** tem **peso elevado** (³ em Wm, ² em Wr), deslocando a nuvem para **cima e à direita** quando aumenta.
   * **|Δ|** reforça o efeito (lin. em Wr; raiz em Wm).
   * **Temperaturas mais baixas** aumentam desgaste via $e^{-\tau/T}$.

2. **Cobertura menor que 11,7k no gráfico de índices**

   * O gráfico de dispersão exige **ambos** os índices presentes; timestamps sem alguma variável primária (v, Δ, τ) viram **NaN** e são descartados na visualização.
   * O *script* de diagnóstico (entregue) contabiliza **quantos** ficaram de fora e **por qual motivo** (vazão ausente, Δ ausente, τ ausentes).

3. **Efeito do p5**

   * **Estabiliza** a referência e mitiga “otimismo” quando o mínimo absoluto é um ponto atípico.
   * **Interpretação direta**: índices > 1 indicam operação mais agressiva que o **p5** do baseline; **2** = \~100% acima da referência.

---

## 7. Uso prático (MVP)

* **Dashboards**: plote `Wr_idx_p5` e `Wm_idx_p5` **no tempo** com a elipse/retângulo como *overlay* no plano Wr×Wm (índices).
* **Alarmes**: dispare para pontos **fora da elipse** e com **índices** acima de *thresholds* (p.ex., 95º/99º percentis).
* **Investigação**: use o CSV de outliers e o relatório de **drivers** (contribuições relativas de v, |Δ| e τ) para priorizar ações.
* **Calibração futura**: quando houver **medidas reais de desgaste** (mm), ajustam-se $\alpha,\beta$ e convertem-se as proxies para **unidades físicas**.

---

## 8. Limitações e próximos passos

* **Sem rótulo** de desgaste físico medido, os resultados são **proxies**; ainda assim, úteis para **comparar condições** e detectar **regimes agressivos**.
* **MinIO**: a SSOT está **local**; caso deseje, replicamos para um **bucket** MinIO com *schema* versionado e *catalog* (p.ex., Hive/Glue) para consumo por BI/ML.
* **Robustez**: opcional trocar p5 por **p10** em campanhas muito ruidosas; manter a elipse χ² como padrão da ROI.
* **Features**: incluir *lags*, dérivadas de v e |Δ|, e marcadores operacionais (startups, blowers etc.) para enriquecer a análise.

---

## Anexo A — Caminhos de arquivos (entrada, derivados, gráficos)

**Fontes (A1\_LOCAL):**

* `C:\Users\wilso\MBA_EMPREENDEDORISMO\3AGD\A1_LOCAL\outputs\pedra\A1_SECONDARIES_FOR_PEDRA_MODEL_old.csv`
* `C:\Users\wilso\MBA_EMPREENDEDORISMO\3AGD\A1_LOCAL\outputs\pedra\A1_SECONDARIES_FOR_PEDRA_MODEL.csv`
* `C:\Users\wilso\MBA_EMPREENDEDORISMO\3AGD\A1_LOCAL\outputs\pedra\A1_SECONDARIES_REFS.csv`
* `C:\Users\wilso\MBA_EMPREENDEDORISMO\3AGD\A1_LOCAL\outputs\pedra\DELTA_PROXY_DIAGNOSTICS.csv`
* (quando existentes)
  `C:\Users\wilso\MBA_EMPREENDEDORISMO\3AGD\A1_LOCAL\outputs\baseline_datasets\physics_baseline_proxies.csv`
  `C:\Users\wilso\MBA_EMPREENDEDORISMO\3AGD\A1_LOCAL\outputs\baseline_mask.csv`

**SSOT e derivados (A1\_LOCAL\_REFAZIMENTO):**

* `...\data\derivadas\tabela_wr_wm_padrao_dedup.parquet`
* `...\data\derivadas\tabela_wr_wm_com_desgaste.parquet`  *(v1)*
* `...\data\derivadas\tabela_wr_wm_com_desgaste_v2.parquet` *(|Δ|, mínimos absolutos)*
* `...\data\derivadas\tabela_wr_wm_com_desgaste_p5.parquet` *(referência p5)*
* CSVs correspondentes (`.csv`) para cada Parquet.

**Diagnósticos e logs:**

* `...\outputs\diagnosticos\physics_params_mvp.json` *(v1)*
* `...\outputs\diagnosticos\physics_params_mvp_v2.json` *(v2)*
* `...\outputs\diagnosticos\physics_refs_p5_summary.json` *(v3, p5 + estatísticas de ar e carvão)*

**Modelos/Anomalias (quick-win):**

* `...\data\derivadas\anomalias_isoforest.parquet`
* `...\outputs\modelos\isoforest_baseline.pkl`

**Gráficos (seleção):**

* `...\outputs\diagnosticos\wr_idx_timeline.png`
* `...\outputs\diagnosticos\wm_idx_timeline.png`
* `...\outputs\diagnosticos\wr_idx_timeline_v2.png`
* `...\outputs\diagnosticos\wm_idx_timeline_v2.png`
* `...\outputs\diagnosticos\wr_wm_quadrantes_ref_anterior.png`
* `...\outputs\diagnosticos\wr_wm_quadrantes_ref_anterior_com_roi.png`
* `...\outputs\diagnosticos\wr_wm_quadrantes_ref_anterior_com_roi_labels_ok.png`
* `...\outputs\diagnosticos\wr_wm_roi_estatistica_quadrantes.png`
* `...\outputs\diagnosticos\wr_wm_elipse_ref_p5_raw.png`
* `...\outputs\diagnosticos\wr_wm_elipse_ref_p5_idx.png`
* Outliers e diagnósticos:
  `...\outputs\diagnosticos\wr_wm_outliers_elipse_raw.csv`
  `...\outputs\diagnosticos\wr_wm_outliers_elipse_idx.csv`
  `...\outputs\diagnosticos\drivers_q1.csv`

---

## Anexo B — Equações e índices (para referência rápida)

1. **Proxies de desgaste**

$$
Wr \;\propto\; v^{2}\cdot |\Delta|^{1}\cdot \exp\!\Big(-\frac{\tau_K}{1200}\Big),\qquad
Wm \;\propto\; v^{3}\cdot |\Delta|^{0.5}\cdot \exp\!\Big(-\frac{\tau_K}{1250}\Big)
$$

2. **Referências**

* v2: $Wr_{ref}=\min(Wr),\; Wm_{ref}=\min(Wm)$
* v3: $Wr_{ref,p5}=\text{p5}(Wr|\text{baseline}),\; Wm_{ref,p5}=\text{p5}(Wm|\text{baseline})$

3. **Índices (adimensionais)**

$$
Wr_{idx,p5}=\frac{Wr}{Wr_{ref,p5}},\qquad
Wm_{idx,p5}=\frac{Wm}{Wm_{ref,p5}}
$$

4. **ROI estatística**

* Retângulo: $[Wr_{ref}\pm k\sigma_{Wr}] \times [Wm_{ref}\pm k\sigma_{Wm}]$
* Elipse (Mahalanobis, 2 g.l.): $d^2 \le \chi^2_{0.95}=5{,}991$, com $\Sigma$ estimada no **baseline**.

---

### Encerramento

Com a **SSOT consolidada**, **proxies físico-baseadas**, **referências robustas (p5)**, **índices adimensionais** e **ROI estatística** operacionalizados, o MVP atende ao objetivo: **comparar condições de operação**, **priorizar investigações** e criar base para, quando houver rótulos, calibrar para **desgaste em mm/tempo** mantendo a mesma arquitetura.
