# Projeto A1 – Predição de Desgaste em CFB

## Relatório técnico (MVP) — dados, modelagem física, referências (p5), índices e ROI estatística

> Este documento consolida **o que foi desenvolvido** no refazimento do Projeto A1: montagem da SSOT local, padronização semântica/unidades, construção de proxies físico-baseadas de desgaste (**Wr**, **Wm**), definição de **referências** (mínimo e **p5**), normalização em **índices**, e diagnóstico com **ROI estatística** (retângulo ±k·σ e **elipse χ²**).  
> Formato: **Markdown** com fórmulas em LaTeX (pronto para exportar a PDF).

---

## 1) Sumário executivo

- **SSOT local** a partir dos CSVs originais; correção da triplicação por _zona_ (densa, diluída, backpass) e **uma linha por timestamp** (11.757 instantes).
    
- **Proxy de desgaste refratário (Wr)** e **metálico (Wm)** modeladas por **vazão de ar (v)**, **queda de pressão (|\Delta|)** e **temperatura (τ)**, com pesos distintos para cada material.
    
- Passagem de Δ\Delta para **magnitude** ∣Δ∣|\Delta| (com ε\varepsilon) para remover efeitos de sinal e garantir domínio da raiz em **Wm**.
    
- **Referências**:
    
    - **v2**: mínimos absolutos min⁡(Wr),min⁡(Wm)\min(Wr), \min(Wm) ⇒ índices ≥1\ge 1.
        
    - **v3** (**p5**): **5º percentil** no _baseline_ mais longo ⇒ índices centrados no **p5** (mais robusto a outliers).
        
- **ROI estatística**:
    
    - **Retângulo ±k·σ** por eixo (simples e interpretável).
        
    - **Elipse χ²** (Mahalanobis, 2 g.l.) centrada na referência (melhor quando há correlação Wr–Wm).
        
- **Resultados operacionais**: tabelas Parquet/CSV com Wr/Wm e índices; gráficos temporais e Wr×Wm (bruto e índices), quadrantes, elipse/retângulo; CSVs de outliers e _drivers_.
    

---

## 2) Engenharia de dados (SSOT) e padronização

**Fontes**: arquivos `A1_SECONDARIES_FOR_PEDRA_MODEL(_old).csv`, `A1_SECONDARIES_REFS.csv`, `DELTA_PROXY_DIAGNOSTICS.csv` e, quando disponíveis, artefatos de _baseline_.

**Problema original**: três linhas por timestamp (uma por zona).  
**Correção**: ausência de conflitos entre zonas e **colapso** para **uma linha por timestamp**, mantendo colunas `tau_densa`, `tau_diluida`, `tau_backpass` e adicionando:

- **Temperatura média** (°C):  
    τC=mean{τdensa,τdiluida,τbackpass}\tau_{\mathrm{C}}=\mathrm{mean}\{\tau_{\text{densa}},\tau_{\text{diluida}},\tau_{\text{backpass}}\}
    
- **Temperatura absoluta** (K):  
    τK=τC+273,15\tau_{\mathrm{K}}=\tau_{\mathrm{C}}+273{,}15
    
- **Vazão de ar**: preferência por `total_air_flow_knm3_h`; _fallback_ = `total_paf_air_flow_knm3_h`.
    
- **Queda de pressão** (magnitudes):  
    Δmag=max⁡(∣Δ∣, ε),ε≈10−9\Delta_{\text{mag}}=\max\big(|\Delta|,\ \varepsilon\big),\quad \varepsilon\approx10^{-9}
    

Saídas consolidadas gravadas em `...A1_LOCAL_REFAZIMENTO\data\derivadas\*.parquet|.csv`.

---

## 3) Modelagem físico-baseada (MVP)

### 3.1 Variáveis e parâmetros

- vv — vazão de ar (proxy de velocidade): **kNm³/h** (internamente tratada como fluxo).
    
- Δmag\Delta_{\text{mag}} — magnitude de queda de pressão (Pa ou equivalente).
    
- τK\tau_{\mathrm{K}} — temperatura absoluta (K).
    
- Parâmetros (ajustáveis):  
    nr=2, mr=1,nm=3, mm=0,5,Tcrit=1200 K, Topt=1250 Kn_r=2,\ m_r=1,\quad n_m=3,\ m_m=0{,}5,\quad T_{\mathrm{crit}}=1200\,\mathrm{K},\ T_{\mathrm{opt}}=1250\,\mathrm{K}
    

### 3.2 Equações (v2/v3)

- **Desgaste refratário (Wr)**  
    Wr ∝ v 2⋅Δmag 1⋅exp⁡ ⁣(−τKTcrit)\boxed{Wr \ \propto\ v^{\,2}\cdot \Delta_{\text{mag}}^{\,1}\cdot \exp\!\Big(-\frac{\tau_{\mathrm{K}}}{T_{\mathrm{crit}}}\Big)}
    
- **Desgaste metálico (Wm)**  
    Wm ∝ v 3⋅Δmag 0,5⋅exp⁡ ⁣(−τKTopt)\boxed{Wm \ \propto\ v^{\,3}\cdot \Delta_{\text{mag}}^{\,0{,}5}\cdot \exp\!\Big(-\frac{\tau_{\mathrm{K}}}{T_{\mathrm{opt}}}\Big)}
    

**Interpretação:** aumentar vv eleva **fortemente** Wm (expoente 3) e também Wr (expoente 2); maior ∣Δ∣|\Delta| aumenta desgaste; operar mais “quente” reduz desgaste via o termo exponencial (dentro do regime seguro).

### 3.3 Análise dimensional (o que “são” Wr e Wm)

- vv tem dimensão [L3T−1][L^3T^{-1}]; Δ\Delta tem [ML−1T−2][ML^{-1}T^{-2}].  
    Então:  
    [vnΔm]=[Mm L 3n−m T−n−2m][v^n\Delta^m]=\big[M^m\ L^{\,3n-m}\ T^{-n-2m}\big]
    
- Para **Wr** (n=2,m=1n=2,m=1): [M1L5T−4][M^1L^{5}T^{-4}].
    
- Para **Wm** (n=3,m=0,5n=3,m=0{,}5): [M1/2L17/2T−4][M^{1/2}L^{17/2}T^{-4}].
    

> Conclusão: Wr e Wm são **proxies** proporcionais à agressividade do escoamento/abrasão; em operação comparamos **índices** (adimensionais) no tempo.  
> Se desejado converter para **mm/tempo**, introduzem-se constantes de calibração α,β\alpha,\beta com unidades compensatórias e ajustadas a **medições reais**.

---

## 4) Referências e índices

### 4.1 v2 — referência por mínimos absolutos

- Wrref=min⁡tWr(t),Wmref=min⁡tWm(t)Wr_{\text{ref}}=\min_t Wr(t),\qquad Wm_{\text{ref}}=\min_t Wm(t)
    
- Índices:  
    Wridx=WrWrref≥1,Wmidx=WmWmref≥1Wr_{\text{idx}}=\dfrac{Wr}{Wr_{\text{ref}}}\ge 1,\qquad Wm_{\text{idx}}=\dfrac{Wm}{Wm_{\text{ref}}}\ge 1
    
- Pro: elimina negativos e maximiza cobertura.
    
- Contra: sensível a **mínimos extremos** (ruído).
    

### 4.2 v3 — referência **robusta p5** (baseline)

- Janela [t0,t1][t_0,t_1] = _baseline_ de **maior duração** (ou dataset inteiro se vazio).
    
- Referências:  
    Wrref,p5=p5⁡ ⁣(Wr∣t∈[t0,t1]),Wmref,p5=p5⁡ ⁣(Wm∣t∈[t0,t1])Wr_{\text{ref},p5}=\operatorname{p5}\!\big(Wr\mid t\in[t_0,t_1]\big),\quad Wm_{\text{ref},p5}=\operatorname{p5}\!\big(Wm\mid t\in[t_0,t_1]\big)
    
- Índices:  
    Wridx,p5=WrWrref,p5,Wmidx,p5=WmWmref,p5Wr_{\text{idx},p5}=\frac{Wr}{Wr_{\text{ref},p5}},\qquad Wm_{\text{idx},p5}=\frac{Wm}{Wm_{\text{ref},p5}}
    

**Leitura operacional**: valor 1,0 representa o **p5** do _baseline_; >1>1 é mais agressivo que a referência; p.ex., Wridx,p5=2Wr_{\text{idx},p5}=2 ≈ **100% acima** do p5.

---

## 5) ROI (“normalidade”) e diagnóstico

### 5.1 Retângulo ±k·σ (centrado na referência)

- Em eixos brutos (Wr,Wm)(Wr,Wm) com centro (Wrref,Wmref)(Wr_{\text{ref}},Wm_{\text{ref}}), ou em índices com centro (1,1)(1,1):  
    Wr∈[Wrref±k σWr],Wm∈[Wmref±k σWm]Wr\in[Wr_{\text{ref}}\pm k\,\sigma_{Wr}],\quad Wm\in[Wm_{\text{ref}}\pm k\,\sigma_{Wm}]
    
- σ\sigma estimado **no baseline**; k∈[1,2]k\in[1,2] cobre ∼\sim68–95% (univariado).
    

### 5.2 Elipse χ² (Mahalanobis, 2 g.l.)

- Para vetor z=[Wr,Wm]⊤z=[Wr,Wm]^\top (ou [Wridx,Wmidx]⊤[Wr_{\text{idx}},Wm_{\text{idx}}]^\top), centro cc e matriz de covariância Σ\Sigma (estimada no **baseline**):  
    d2=(z−c)⊤Σ−1(z−c) ≤ χ2, p2d^2=(z-c)^\top \Sigma^{-1}(z-c) \ \le\ \chi^2_{2,\ p}
    
- Valores usuais: p=0,95⇒χ2=5,991p=0{,}95\Rightarrow \chi^2=5{,}991 (2 g.l.).
    
- **Vantagem**: respeita a **correlação** Wr–Wm (nuvem alongada), definindo uma normalidade elíptica mais realista.
    

> Observação: a elipse cobre ∼p\sim p do **baseline**; aplicada ao **conjunto inteiro**, naturalmente haverá mais pontos “fora” se a operação se afastou daquele regime.

### 5.3 Quadrantes e _drivers_

- Com eixos cruzando na referência (bruta ou índices), contamos pontos **fora** da ROI por quadrante:  
    Q1 (↑,↑), Q2 (↓,↑), Q3 (↓,↓), Q4 (↑,↓).
    
- Pela forma das equações, **Q1 tende a concentrar** quando há **aumento de vv** (peso 3 em Wm e 2 em Wr), reforçado por maior ∣Δ∣|\Delta| e por **temperatura mais baixa** (exponencial).
    
- Um relatório de _drivers_ avalia contribuições relativas Δlog⁡v\Delta\log v, Δlog⁡∣Δ∣\Delta\log |\Delta| e termos −Δτ/T-\Delta\tau/T.
    

---

## 6) Resultados e uso prático

- **Tabelas**: `tabela_wr_wm_com_desgaste_v2.parquet` (mínimos) e `tabela_wr_wm_com_desgaste_p5.parquet` (**p5**), com `Wr, Wm, Wr_ref(_p5), Wm_ref(_p5), Wr_idx(_p5), Wm_idx(_p5)`.
    
- **Gráficos**: temporais dos índices; dispersões Wr×Wm (bruto e índices) com elipse χ²(95%), quadrantes e contagem de **outliers**.
    
- **Logs**: arquivo JSON registra janela de baseline, valores de referência (p5), e estatísticas de **vazão de ar** e **vazão de carvão (linha c)** no período.
    

**Operação (MVP):**

1. Monitorar **Wr_idx_p5** e **Wm_idx_p5** no tempo;
    
2. Sinalizar **pontos fora** da elipse (95%) e/ou acima de _thresholds_ (p95/p99 dos índices);
    
3. Priorizar investigação com base nos **drivers** (qual variável puxou para Q1).
    

**Próximos passos**:

- Ingerir **medidas reais** de desgaste para calibrar α,β\alpha,\beta (converter para **mm/tempo**).
    
- Versionar a SSOT em **MinIO** (bucket, _schema_, _catalog_), liberando consumo por BI/ML.
    
- Enriquecer com _lags_, derivadas de vv e ∣Δ∣|\Delta| e _flags_ operacionais.
    

---

## Anexo A — Caminhos de arquivos (usados/gerados)

**Fontes (A1_LOCAL):**

- `C:\Users\wilso\MBA_EMPREENDEDORISMO\3AGD\A1_LOCAL\outputs\pedra\A1_SECONDARIES_FOR_PEDRA_MODEL_old.csv`
    
- `C:\Users\wilso\MBA_EMPREENDEDORISMO\3AGD\A1_LOCAL\outputs\pedra\A1_SECONDARIES_FOR_PEDRA_MODEL.csv`
    
- `C:\Users\wilso\MBA_EMPREENDEDORISMO\3AGD\A1_LOCAL\outputs\pedra\A1_SECONDARIES_REFS.csv`
    
- `C:\Users\wilso\MBA_EMPREENDEDORISMO\3AGD\A1_LOCAL\outputs\pedra\DELTA_PROXY_DIAGNOSTICS.csv`
    
- (quando existentes)  
    `C:\Users\wilso\MBA_EMPREENDEDORISMO\3AGD\A1_LOCAL\outputs\baseline_datasets\physics_baseline_proxies.csv`  
    `C:\Users\wilso\MBA_EMPREENDEDORISMO\3AGD\A1_LOCAL\outputs\baseline_mask.csv`
    

**SSOT e derivados (A1_LOCAL_REFAZIMENTO):**

- `...\data\derivadas\tabela_wr_wm_padrao_dedup.parquet`
    
- `...\data\derivadas\tabela_wr_wm_com_desgaste.parquet` _(v1)_
    
- `...\data\derivadas\tabela_wr_wm_com_desgaste_v2.parquet` _(|Δ|, mínimos)_
    
- `...\data\derivadas\tabela_wr_wm_com_desgaste_p5.parquet` _(referência p5)_
    
- CSVs correspondentes (`.csv`) para cada Parquet.
    

**Diagnósticos e logs:**

- `...\outputs\diagnosticos\physics_params_mvp.json` _(v1)_
    
- `...\outputs\diagnosticos\physics_params_mvp_v2.json` _(v2)_
    
- `...\outputs\diagnosticos\physics_refs_p5_summary.json` _(v3, p5 + estatísticas de ar e carvão)_
    

**Gráficos e relatórios auxiliares:**

- `...\outputs\diagnosticos\wr_idx_timeline*.png`
    
- `...\outputs\diagnosticos\wm_idx_timeline*.png`
    
- `...\outputs\diagnosticos\wr_wm_quadrantes_ref_anterior*.png`
    
- `...\outputs\diagnosticos\wr_wm_roi_estatistica_quadrantes.png`
    
- `...\outputs\diagnosticos\wr_wm_elipse_ref_p5_raw.png`
    
- `...\outputs\diagnosticos\wr_wm_elipse_ref_p5_idx.png`
    
- `...\outputs\diagnosticos\wr_wm_outliers_elipse_raw.csv`
    
- `...\outputs\diagnosticos\wr_wm_outliers_elipse_idx.csv`
    
- `...\outputs\diagnosticos\drivers_q1.csv`
    

---

## Anexo B — Fórmulas de bolso

**Proxies de desgaste**

- Wr ∝ v2 ∣Δ∣1 exp⁡ ⁣(−τK1200)Wr \ \propto\ v^{2}\,|\Delta|^{1}\,\exp\!\Big(-\frac{\tau_{\mathrm{K}}}{1200}\Big)
    
- Wm ∝ v3 ∣Δ∣0,5 exp⁡ ⁣(−τK1250)Wm \ \propto\ v^{3}\,|\Delta|^{0{,}5}\,\exp\!\Big(-\frac{\tau_{\mathrm{K}}}{1250}\Big)
    

**Referências**

- v2: Wrref=min⁡(Wr),  Wmref=min⁡(Wm)Wr_{\text{ref}}=\min(Wr),\; Wm_{\text{ref}}=\min(Wm)
    
- v3: Wrref,p5=p5(Wr∣baseline),  Wmref,p5=p5(Wm∣baseline)Wr_{\text{ref},p5}=\mathrm{p5}(Wr\mid \text{baseline}),\; Wm_{\text{ref},p5}=\mathrm{p5}(Wm\mid \text{baseline})
    

**Índices (adimensionais)**

- Wridx,p5=WrWrref,p5,Wmidx,p5=WmWmref,p5Wr_{\text{idx},p5}=\dfrac{Wr}{Wr_{\text{ref},p5}},\qquad Wm_{\text{idx},p5}=\dfrac{Wm}{Wm_{\text{ref},p5}}
    

**ROI estatística**

- Retângulo: [Wrref±kσWr]×[Wmref±kσWm][Wr_{\text{ref}}\pm k\sigma_{Wr}] \times [Wm_{\text{ref}}\pm k\sigma_{Wm}]
    
- Elipse (2 g.l.): d2≤χ2,0,952=5,991d^2 \le \chi^2_{2,0{,}95}=5{,}991 com Σ\Sigma estimada no **baseline**.