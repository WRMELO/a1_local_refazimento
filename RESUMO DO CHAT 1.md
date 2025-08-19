# Projeto A1 – REFAZIMENTO (MVP)

## Tudo o que foi feito neste chat — resumo técnico em Markdown

> Objetivo: sair de múltiplos CSVs e notebooks desconexos para um **MVP reprodutível**, com **SSOT local**, **proxies físico-baseadas de desgaste (Wr/Wm)**, **referências robustas (p5)**, **índices adimensionais**, e **diagnóstico estatístico** (ROI retangular ±k·σ e elipse χ²) com gráficos e artefatos salvos em disco.

---

## 1) Organização do projeto e versionamento

1. **Git**
    
    - Criamos `.gitignore` adequado (ignorando `.venv/`, `.obsidian/`, `data/` bruta, temporários etc.).
        
    - Inicializamos o repositório, adicionamos `origin` e **subimos para** `https://github.com/WRMELO/a1_local_refazimento`.
        
    - Fluxo: tudo que é **fonte original** continua em `A1_LOCAL`; tudo que é **refazimento/MVP** vai para `A1_LOCAL_REFAZIMENTO`.
        
2. **Ambiente**
    
    - `.venv` configurado; instalamos pacotes de ciência de dados e gráficos.
        
    - Notebooks existentes foram usados como **insumos** (não reescritos), reprocessando dados via scripts/patches.
        

---

## 2) Engenharia de dados (SSOT local)

**Problema original:** a base estava **triplicada por zona** (`densa`, `diluida`, `backpass`), resultando em ~35k linhas (3×).  
**Solução:** verificamos que não havia conflitos entre as zonas e **colapsamos** para **1 linha por timestamp** (11.757 instantes), criando colunas por zona e uma **média de temperatura**:

- Temperatura média (°C):  
    τC=mean{τdensa, τdiluida, τbackpass}\tau_{\mathrm{C}}=\mathrm{mean}\{\tau_{\text{densa}},\ \tau_{\text{diluida}},\ \tau_{\text{backpass}}\}
    
- Conversão para Kelvin:  
    τK=τC+273.15\tau_{\mathrm{K}}=\tau_{\mathrm{C}}+273.15
    
- **Vazão de ar**: preferência por `total_air_flow_knm3_h`; _fallback_ = `total_paf_air_flow_knm3_h`.
    
- **Queda de pressão**: passamos a usar a **magnitude** com ε para evitar zero/negativo:  
    Δmag=max⁡ ⁣(∣Δ∣, ε),  ε≈10−9\Delta_{\text{mag}}=\max\!\big(|\Delta|,\ \varepsilon\big),\ \ \varepsilon\approx 10^{-9}
    

**Artefatos**

- `...data\derivadas\tabela_wr_wm_padrao.parquet` (35.271×18) ➜ **dedup** para  
    `...data\derivadas\tabela_wr_wm_padrao_dedup.parquet` (**11.757×17**).
    
- Exportamos CSV/Parquet consolidados.
    

---

## 3) Detecção rápida de anomalias (dados brutos)

Implementamos um **baseline de anomalias** com `IsolationForest` (contamination ≈3%), em _pipeline_ com imputação e padronização.

- Corrigimos a incompatibilidade do NumPy 2.0 (`.ptp` ➜ `np.ptp`).
    
- Arquivos:
    
    - `...data\derivadas\anomalias_isoforest.parquet` / `.csv`
        
    - Modelo: `...outputs\modelos\isoforest_baseline.pkl`
        
- **Uso**: triagem operacional de outliers antes da modelagem física.
    

---

## 4) Modelagem físico-baseada de desgaste (MVP)

### 4.1 Variáveis/Parâmetros

- vv: vazão de ar (kNm³/h no dado).
    
- Δmag\Delta_{\text{mag}}: magnitude da queda de pressão.
    
- τK\tau_{\mathrm{K}}: temperatura absoluta (K).
    
- Parâmetros (ajustáveis):  
    nr=2, mr=1, nm=3, mm=0.5, Tcrit=1200 K, Topt=1250 Kn_r{=}2,\ m_r{=}1,\ n_m{=}3,\ m_m{=}0.5,\ T_{\mathrm{crit}}{=}1200\,K,\ T_{\mathrm{opt}}{=}1250\,K.
    

### 4.2 Equações (v2/v3)

- **Desgaste refratário (Wr)**  
    Wr ∝ v 2⋅Δmag 1⋅exp⁡ ⁣(−τKTcrit)Wr\ \propto\ v^{\,2}\cdot \Delta_{\text{mag}}^{\,1}\cdot \exp\!\Big(-\frac{\tau_{\mathrm{K}}}{T_{\mathrm{crit}}}\Big)
    
- **Desgaste metálico (Wm)**  
    Wm ∝ v 3⋅Δmag 0.5⋅exp⁡ ⁣(−τKTopt)Wm\ \propto\ v^{\,3}\cdot \Delta_{\text{mag}}^{\,0.5}\cdot \exp\!\Big(-\frac{\tau_{\mathrm{K}}}{T_{\mathrm{opt}}}\Big)
    

**Interpretação física**

- **Vazão de ar** domina Wm (expoente 3) e pesa forte em Wr (expoente 2);
    
- **|ΔP|** aumenta desgaste (linear em Wr; raiz em Wm);
    
- **Quanto mais quente**, **menor desgaste** (termo exponencial), dentro do regime seguro.
    

### 4.3 Dimensionalidade (por que usamos índices)

Se v:[L3T−1]v:[L^3T^{-1}] e Δ:[ML−1T−2]\Delta:[ML^{-1}T^{-2}], então  
[vnΔm]=[MmL3n−mT−n−2m][v^n\Delta^m]=[M^m L^{3n-m}T^{-n-2m}]. Logo:

- [Wr]∼[M1L5T−4][Wr]\sim [M^1 L^5 T^{-4}],
    
- [Wm]∼[M1/2L17/2T−4][Wm]\sim [M^{1/2} L^{17/2} T^{-4}].  
    Como são magnitudes compostas, **normalizamos em índices** para comparabilidade temporal.
    

---

## 5) Referências e índices

### 5.1 v1 (histórica)

Usava valores de referência dos arquivos _refs_. Como Δ tinha **sinal**, **Wr** podia ficar **negativo** e **Wm** dava **NaN** (raiz de Δ<0).

### 5.2 v2 (mínimos absolutos)

Com ∣Δ∣|\Delta|:

- Wrref=min⁡(Wr)Wr_{\text{ref}}=\min(Wr), Wmref=min⁡(Wm)Wm_{\text{ref}}=\min(Wm) ⇒ índices **≥ 1**.
    
- Pro: remove negativos. Contra: **sensível a mínimos atípicos**.
    

### 5.3 v3 (robusta p5, **adotada**)

- Janela [t0,t1][t_0,t_1] = **baseline mais longo** (se vazio, usa o dataset todo).
    
- Referências:  
    Wrref,p5=p5(Wr∣[t0,t1])Wr_{\text{ref},p5}=\text{p5}(Wr|[t_0,t_1]), Wmref,p5=p5(Wm∣[t0,t1])Wm_{\text{ref},p5}=\text{p5}(Wm|[t_0,t_1]).
    
- Índices:  
    Wridx,p5=Wr/Wrref,p5Wr_{\text{idx},p5}=Wr/Wr_{\text{ref},p5}, Wmidx,p5=Wm/Wmref,p5Wm_{\text{idx},p5}=Wm/Wm_{\text{ref},p5}.
    
- **Vantagem**: estabilidade (resiste a outliers); leitura direta: **(1,1)** é o **p5** do baseline.
    

**Artefatos**

- `...data\derivadas\tabela_wr_wm_com_desgaste.parquet` (v1)
    
- `...data\derivadas\tabela_wr_wm_com_desgaste_v2.parquet` (|Δ| + mínimos)
    
- `...data\derivadas\tabela_wr_wm_com_desgaste_p5.parquet` (**final**)
    
- Log com janela e estatísticas operacionais:  
    `...outputs\diagnosticos\physics_refs_p5_summary.json`
    

---

## 6) Visualizações e ROI estatística

### 6.1 Séries temporais

- **Wr_idx** e **Wm_idx** ao longo do tempo, com **baseline sombreado**.
    
- Observamos períodos de maior agressividade, picos e drifts.
    

### 6.2 Dispersão Wr×Wm (bruto e índices)

- **Eixos cruzando na referência** (em bruto: Wrref,p5,Wmref,p5Wr_{\text{ref},p5},Wm_{\text{ref},p5}; em índices: (1,1)).
    
- **Quadrantes**: Q1 (↑,↑), Q2 (↓,↑), Q3 (↓,↓), Q4 (↑,↓) **só para pontos fora da ROI**.
    
- **ROI retangular ±k·σ** e **ROI elíptica χ²(95%)** (Mahalanobis) **estimadas no baseline**:
    
    - Retângulo é simples e legível;
        
    - **Elipse** respeita a **correlação Wr–Wm** (nuvem alongada) — **padrão recomendado**.
        

**Constatações**

- **Q1** ficou superpovoado fora da ROI, coerente com **aumento de vazão** (expoentes altos) + **|Δ|** + **temperaturas mais baixas**.
    
- Ao usar elipse estimada no baseline e aplicar **no conjunto todo**, é **normal** ter muito ponto **fora da elipse** (mudança de regime).
    

**Arquivos**

- `...outputs\diagnosticos\wr_idx_timeline*.png`, `wm_idx_timeline*.png`
    
- `...outputs\diagnosticos\wr_wm_quadrantes_ref_anterior*.png`
    
- `...outputs\diagnosticos\wr_wm_roi_estatistica_quadrantes.png`
    
- `...outputs\diagnosticos\wr_wm_elipse_ref_p5_raw.png`
    
- `...outputs\diagnosticos\wr_wm_elipse_ref_p5_idx.png`
    
- Outliers: `wr_wm_outliers_elipse_raw.csv`, `wr_wm_outliers_elipse_idx.csv`
    

---

## 7) Diagnósticos adicionais

1. **Por que poucos pontos em Wm em alguns gráficos?**  
    Porque gráficos de dispersão exigem **ambos** os índices válidos; linhas com falta de **v**, **Δ** ou **τ** viram **NaN** e são descartadas. Foi entregue um _script_ de diagnóstico listando **quantas** linhas ficaram de fora **e por quê** (vazão ausente, Δ ausente, τ ausentes).
    
2. **Definição de “normalidade” não arbitrária**  
    Substituímos a ROI “±10%” por métricas **estatísticas**:
    
    - **Retângulo ±k·σ** (k≈1–2) com σ\sigma do **baseline**;
        
    - **Elipse χ²(95%)** (2 g.l.) com matriz de covariância **do baseline**.
        
3. **Rótulos e calibração**  
    Sem medidas reais de desgaste, **Wr/Wm** são **proxies**. Assim que houver rótulos (mm/tempo), calibramos constantes α,β\alpha,\beta para converter **Wr/Wm → mm/tempo** sem mudar a arquitetura.
    

---

## 8) Entregáveis e caminhos (seleção)

**Fontes (A1_LOCAL)**

- `...\A1_LOCAL\outputs\pedra\A1_SECONDARIES_FOR_PEDRA_MODEL_old.csv`
    
- `...\A1_LOCAL\outputs\pedra\A1_SECONDARIES_FOR_PEDRA_MODEL.csv`
    
- `...\A1_LOCAL\outputs\pedra\A1_SECONDARIES_REFS.csv`
    
- `...\A1_LOCAL\outputs\pedra\DELTA_PROXY_DIAGNOSTICS.csv`
    
- (quando presentes) `...\A1_LOCAL\outputs\baseline_datasets\physics_baseline_proxies.csv`, `baseline_mask.csv`
    

**SSOT e derivados (A1_LOCAL_REFAZIMENTO)**

- `...\data\derivadas\tabela_wr_wm_padrao.parquet` → **dedup** → `...\tabela_wr_wm_padrao_dedup.parquet`
    
- `...\data\derivadas\tabela_wr_wm_com_desgaste.parquet` (v1)
    
- `...\data\derivadas\tabela_wr_wm_com_desgaste_v2.parquet` (|Δ| + mínimos)
    
- `...\data\derivadas\tabela_wr_wm_com_desgaste_p5.parquet` (**referência p5**)
    
- (CSV correspondentes em cada etapa)
    

**Modelos/diagnóstico**

- `...\data\derivadas\anomalias_isoforest.parquet`
    
- `...\outputs\modelos\isoforest_baseline.pkl`
    
- `...\outputs\diagnosticos\physics_params_mvp.json` (v1)
    
- `...\outputs\diagnosticos\physics_refs_p5_summary.json` (janela, refs p5, estatísticas de AR e carvão)
    
- `...\outputs\diagnosticos\wr_wm_outliers_elipse_raw.csv`
    
- `...\outputs\diagnosticos\wr_wm_outliers_elipse_idx.csv`
    
- `...\outputs\diagnosticos\drivers_q1.csv`
    

**Gráficos**

- `wr_idx_timeline.png`, `wm_idx_timeline.png`,  
    `wr_wm_quadrantes_ref_anterior*.png`,  
    `wr_wm_roi_estatistica_quadrantes.png`,  
    `wr_wm_elipse_ref_p5_raw.png`, `wr_wm_elipse_ref_p5_idx.png`.
    

---

## 9) O que ficou decidido (estado final do MVP)

- **Referência adotada:** **p5 do baseline** (v3) para Wr e Wm.
    
- **Índices oficiais:** `Wr_idx_p5` e `Wm_idx_p5` (**adimensionais**, referência em 1,0).
    
- **ROI padrão:** **Elipse χ²(95%)** estimada no baseline (retângulo ±k·σ como auxiliar).
    
- **SSOT:** consolidada e deduplicada por timestamp; nomenclatura e unidades padronizadas.
    
- **Fluxo reprodutível:** scripts/patches salvando **tabelas**, **gráficos** e **logs** nos caminhos do projeto.
    
- **Próximos passos (quando houver):** ingestão de **rótulos de desgaste** para calibrar α,β\alpha,\beta e reportar em **mm/tempo**; opcional: publicar SSOT em **MinIO** com _schema_ versionado.
    

---

### Apêndice — Equações de bolso

- Wr∝v2 ∣Δ∣1 exp⁡(−τK/1200)Wr \propto v^{2}\,|\Delta|^{1}\,\exp(-\tau_{\mathrm{K}}/1200)
    
- Wm∝v3 ∣Δ∣0.5 exp⁡(−τK/1250)Wm \propto v^{3}\,|\Delta|^{0.5}\,\exp(-\tau_{\mathrm{K}}/1250)
    
- Wrref,p5=p5(Wr∣baseline)Wr_{\text{ref},p5}=\mathrm{p5}(Wr|baseline), Wmref,p5=p5(Wm∣baseline)Wm_{\text{ref},p5}=\mathrm{p5}(Wm|baseline)
    
- Wridx,p5=Wr/Wrref,p5Wr_{\text{idx},p5}=Wr/Wr_{\text{ref},p5}, Wmidx,p5=Wm/Wmref,p5Wm_{\text{idx},p5}=Wm/Wm_{\text{ref},p5}
    
- Elipse (2 g.l.): d2≤χ0.952=5,991d^2\le \chi^2_{0.95}=5{,}991, com Σ\Sigma do **baseline**.