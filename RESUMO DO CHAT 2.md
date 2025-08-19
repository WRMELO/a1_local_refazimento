# Projeto A1 — Registro completo do que foi feito neste chat (fase de dados)

## Sumário executivo

- Padronizamos a base principal (`a1_physics_informed_proxies.csv`) e consolidamos tudo em `A1_ML_DL.csv` (2 linhas de cabeçalho: **nome** | **dimensão**).
    
- Fizemos auditoria semântica de variáveis, corrigimos cabeçalhos e removemos colunas auxiliares problemáticas.
    
- Definimos **rótulos físicos** de desgaste (`wr_kg_m2_h`, `wm_kg_m2_h`) com base em proxies dimensionais e em equações físicas (MVP).
    
- Resolvemos **lacunas de dados**: vazão total de ar (**Q_N**) e pressão diferencial (**ΔP**) com estratégias robustas.
    
- Implementamos **anti-vazamento** (remoção das colunas usadas nos rótulos) e garantimos consistência temporal (timestamp horário).
    
- Geramos **duas versões finais** para modelagem:
    
    - **GOLD** (sem imputações adicionais além das físicas/robustas): 10.005 linhas válidas.
        
    - **SILVER2** (imputa Q_N e ΔP de forma conservadora): 11.749 linhas válidas.
        
- Avaliamos adequação (supervisionado e não-supervisionado), com relatórios e métricas; **GOLD** ficou como conjunto principal.
    
- “Congelamos” artefatos para modelagem em `outputs\freeze\v1_20250818_0635` e preparamos _starter_ de modelagem.
    

---

## Linha do tempo (o que fizemos, na prática)

### 1) Preparação e auditoria

- **Leitura e padronização** da base: criamos `A1_ML_DL.csv` com duas linhas de cabeçalho (nome | dimensão), corrigindo o problema de múltiplas linhas no header.
    
- **Auditoria de cabeçalhos**: listamos todas as variáveis e dimensões; removemos `nome_pt_curto`/`significado` quando não tinham correspondência confiável; geramos planilhas de auditoria para inspeção.
    

### 2) EDA inicial

- Geração de **amostras transpostas de cabeçalhos** + exemplos de valores.
    
- **Histogramas** e estatísticas descritivas (status de operação, `total_flow_c`, vazão de ar primário etc.).
    
- Confirmação: timestamps são **horários**.
    

### 3) Definição dos rótulos (MVP físico)

Rótulos:

- **Wr** (desgaste por erosão):
    
    Wr=α⋅ν2⋅δ1⋅e−T/Tcrit⋅gW_r = \alpha \cdot \nu^{2}\cdot \delta^{1}\cdot e^{-T/T_{crit}}\cdot g
- **Wm** (desgaste por corrosão/oxidação):
    
    Wm=β⋅ν3⋅δ⋅e−T/Topt⋅hW_m = \beta \cdot \nu^{3}\cdot \sqrt{\delta}\cdot e^{-T/T_{opt}}\cdot h

Proxies e grandezas:

- **Temperatura** TT (K) a partir de `tau_*`/`leito_temp_*` (conversão °C→K quando aplicável).
    
- **Pressão absoluta** PabsP_{abs}: conversões automáticas (MPa, kPa, bar… → Pa); soma de PatmP_{atm} quando necessário.
    
- **ΔP robusto**: usar `dp_fornalha` **se for de fato ΔP**; caso contrário, |pressao_fornalha_a_inf - pressao_fornalha_b_inf|. _Fallback_ linha a linha.
    
- **Vazão normal (kNm³/h)** → **Q_real (m³/s)** com correções por TT/PP; velocidade **ν=Qreal/Aef\nu = Q_{real}/A_{ef}** com ajuste fraco por ΔP.
    
- **Mistura do gás** (ar/CO₂): fração γ\gamma default **0,25** no MVP (evitamos usar O₂ como densidade de CO₂).
    
- **Densidade do gás** ρg\rho_g via mistura ideal; **viscosidade** μgas\mu_{gas} por Sutherland (air/CO₂).
    
    - **Robustez**: quando TT falha para μ\mu, usamos **T=1100 KT=1100\,K** (faixa típica de leito).
        
- **Diâmetro efetivo** δ\delta derivado de balanço tipo Ergun com ganho CδC_{\delta} (trazendo δ\delta para ordem de décimos de mm).
    

Constantes MVP (ajustáveis):  
α=4.2×10−3\alpha=4.2\times10^{-3}, β=1.2×10−4\beta=1.2\times10^{-4}, Tcrit=1200 KT_{crit}=1200\,K, Topt=1250 KT_{opt}=1250\,K, Aef=15 m2A_{ef}=15\,m^2, Lref=3 mL_{ref}=3\,m, Cδ=10−3C_{\delta}=10^{-3}, σmat=6.0×109\sigma_{mat}=6.0\times10^9, ρfuel=700\rho_{fuel}=700, σsteel=2.5×109\sigma_{steel}=2.5\times10^9.

### 4) Tratamento de faltas (QN e ΔP) e estabilização de Wm

- **QN_total (kNm³/h)**:
    
    1. usar `air_total_knm3_h` quando disponível;
        
    2. senão, `air_primary_knm3_h + air_secondary_knm3_h` na linha;
        
    3. **blocos de NaN** preenchidos com **média** entre o último valor válido antes e o primeiro depois (bordas ficam NaN).
        
    
    - Persistimos **`air_total_knm3_h_filled`** (mas **excluída das features**, para evitar vazamento).
        
- **Wm caindo para NaN**: detectamos que `rho_co2` havia sido **mapeado para `o2_medio`**; corrigimos (γ default) e **μgas\mu_{gas}** virou robusto (sem NaN).
    
- Com esses ajustes, obtivemos **10.005 válidos** em ambos os alvos.
    

### 5) Versões de saída e anti-vazamento

- **v2/v3/v4**: iterações até estabilizar Wr/Wm com as correções.
    
- **GOLD**: sem imputações extras além das robustecedoras → **10.005 linhas**.
    
- **SILVER2**: imputação adicional e conservadora de:
    
    - **Q_N** (interp ≤24h → mediana móvel centrada (48) → mediana global);
        
    - **ΔP** (interp ≤6h; e, quando necessário, ΔP^=k⋅ρgν2\widehat{\Delta P} = k \cdot \rho_g \nu^2 com **clipping** ao [P5, P95]).
        
    - Resultado: **11.749 linhas** válidas.
        
- **Anti-vazamento**: retiramos das features **todas** as colunas do bloco físico (`TAU_DENSA → fim`), colunas de ar (`air_*` incluindo a `_filled`), O₂/ΔP/pressões usadas, etc. Só ficam variáveis exógenas/seguras + rótulos.
    

### 6) Adequação (diagnósticos)

- Scripts geraram relatórios e CSVs em:  
    `...\outputs\diagnosticos\`  
    (`gold_*` e `silver2_*`: faltas, constantes, colinearidades, leakage flags, relatórios `.md`).
    
- **GOLD** (10.005 linhas; 144 features num.; faltas medianas = 0%):
    
    - RF baseline (teste): **Wr R²=0.754**, **Wm R²=0.682**.
        
    - Silhouette: **k=2 ≈ 0.58** (boa separação); passo temporal 1h ≈ **99,6%**.
        
- **SILVER2** (11.749 linhas; 144 features num.; faltas medianas = 0%):
    
    - RF baseline (teste): **Wr R²=0.506**, **Wm R²=0.627**.
        
    - Silhouette ≈ **0.58** em k=2; passo temporal 1h ≈ **99,6%**.
        
- **Decisão**: **GOLD** é o **conjunto principal** para treino; **SILVER2** serve para **robustez/ablação**.
    

### 7) Congelamento e handoff para modelagem

- **Freeze** dos artefatos em:  
    `C:\Users\wilso\MBA_EMPREENDEDORISMO\3AGD\A1_LOCAL_REFAZIMENTO\outputs\freeze\v1_20250818_0635`
    
    - Inclui: `A1_ML_DL_features_v4_gold.csv`, `A1_ML_DL_rotulado_v4_gold.csv`, `A1_ML_DL_features_v4_silver2.csv`, `A1_ML_DL_rotulado_v4_silver2.csv`, relatórios `.md`, `manifest.json`, `checksums`.
        
- **Splits** (70/10/20), **imputer** (mediana) e **scaler** (Standard) prontos para reuso.
    
- Entregue também um **starter** de modelagem (RF baseline multi-alvo) apontando para o `FREEZE_DIR`.
    

---

## Arquivos finais e caminhos (consolidação)

**Curated (base e versões):**

- `...\data\curated\A1_ML_DL_rotulado_v4.csv`
    
- `...\data\curated\A1_ML_DL_features_v4.csv`
    

**Conjuntos para modelagem:**

- **GOLD**:  
    `...\data\curated\A1_ML_DL_rotulado_v4_gold.csv`  
    `...\data\curated\A1_ML_DL_features_v4_gold.csv`
    
- **SILVER2**:  
    `...\data\curated\A1_ML_DL_rotulado_v4_silver2.csv`  
    `...\data\curated\A1_ML_DL_features_v4_silver2.csv`
    

**Diagnósticos:**

- `...\outputs\diagnosticos\gold_relatorio.md`
    
- `...\outputs\diagnosticos\silver2_relatorio.md`
    
- `...\outputs\diagnosticos\gold_*.csv`, `silver2_*.csv`
    

**Freeze para modelagem:**

- `...\outputs\freeze\v1_20250818_0635\`  
    (inclui dados, relatórios, `manifest.json`, checksums, `splits`, `imputer`, `scaler`, e pasta `models` para salvamento de treinados)
    

---

## Recomendações para o notebook de modelagem

- Usar **GOLD** como padrão; **SILVER2** para testes de robustez.
    
- Começar com **árvores/GBTs** (XGBoost/LightGBM); avaliar `log1p` nos alvos se houver caudas pesadas.
    
- Para **temporal** (GRU/TCN), criar janelas (ex.: 24–48h) e manter validação temporal (70/10/20).
    
- Registrar **versões** de dados e pipeline (hashes já estão no freeze).
    
- Monitorar **importâncias**/SHAP e estabilidade entre GOLD vs. SILVER2.
    

---

**Status:** fase de dados **encerrada** e pronta para **modelagem**. ✅