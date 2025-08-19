Abaixo está um **relato completo e encadeado** do que fizemos neste chat — do fechamento da fase de dados à escolha do MVP, criação de uma referência operacional e descoberta de dois regimes distintos, com todos os artefatos e caminhos.

# 1) Fechamento da fase de dados e “freeze”

- Partimos da base consolidada (cabeçalho em dois níveis) e dos rótulos físicos `wr_kg_m2_h` e `wm_kg_m2_h`, calculados a partir dos proxies e correções físicas (vazão real, ΔP robusto, propriedades do gás, etc.), garantindo **anti-vazamento** nas features.
    
- Definimos dois conjuntos para modelagem:
    
    - **GOLD** (principal): 10.005 linhas válidas, 144 features numéricas (miss=0%).
        
    - **SILVER2** (robustez/ablação): 11.749 linhas válidas.
        
- Executamos um **freeze/versionamento**: copiamos artefatos para `outputs/freeze/v1_20250818_0635`, com manifesto, checksums e relatórios. Isso viabiliza reprodutibilidade e troca de notebook sem “quebrar” caminhos.
    

# 2) Baselines supervisionados e escolha do MVP

- **Split temporal 70/10/20** e pipeline (imputer mediana + standard scaler).
    
- **Baseline RF multi-alvo (E0)** no **GOLD**:
    
    - Validação (10%): `R²_wr ≈ 0.931`, `R²_wm ≈ 0.925`.
        
    - Teste (20%): `R²_wr ≈ 0.756`, `R²_wm ≈ 0.698`.
        
- Fizemos ablações dirigidas:
    
    - **E1 (regime como feature)**: ~empate vs. E0.
        
    - **E2 (reweight por regime)** e **E3 (lags + reweight)**: pioraram generalização.
        
    - **E4 (HistGradientBoosting com absolute_loss)**: **ruim** na nossa configuração.
        
- Decisão: **manter E0** como **MVP supervisionado** (multi-alvo, sem log nos alvos).
    
- Artefatos salvos:
    
    - **MVP supervisionado (E0)**  
        `...\outputs\freeze\v1_20250818_0635\models\mvp\model_mvp.joblib`  
        `...\outputs\freeze\v1_20250818_0635\models\mvp\manifest_mvp.json`  
        `...\outputs\freeze\v1_20250818_0635\models\mvp\model_card.md`
        
    - **Modelo físico (MVP)** (implementação leve/portável para cálculo dos rótulos)  
        `...\outputs\freeze\v1_20250818_0635\models\phys\model_phys_mvp.joblib`  
        `...\outputs\freeze\v1_20250818_0635\models\phys\manifest_phys_mvp.json`  
        `...\outputs\freeze\v1_20250818_0635\models\phys\model_card_phys.md`
        

# 3) Revisão de qualidade dos alvos e nova tratativa

- Em uma outra fonte (“features” sem sufixo _vX), detectamos **zeros e NaNs** em `wr`/`wm` — o que é **incompatível** com status de operação.
    
- Geramos três bases derivadas e **separamos os problemas**:
    
    - `...A1_ML_DL_features_onlyzeros_*.csv`
        
    - `...A1_ML_DL_features_onlynans_*.csv`
        
    - `...A1_ML_DL_features_nozeros_nonan_*.csv` (**base limpa**, usada a seguir)
        
- Com a **base limpa** treinamos novamente um RF multi-alvo; fizemos também um **split 80/20 aleatório** (além do temporal), e salvamos tudo em:
    
    - Modelos e métricas:  
        `...\outputs\nova_tratativa\models\model_mvp_nozeros_nonan_*.joblib`  
        `...\outputs\nova_tratativa\models\metrics_nozeros_nonan_*.csv`  
        `...\outputs\nova_tratativa\models\model_card_nozeros_nonan_*.md`
        
- Testamos **treino por metas individuais** e não observamos ganho relevante — **mantida a opção multi-alvo**.
    

# 4) Período de referência operacional

- Objetivo: **ancorar a operação** num trecho de **baixa variabilidade interna** e **menor abrasividade** (`wr` e `wm` baixos).
    
- Varremos janelas **L = 6h, 12h, 24h** e usamos um critério composto (estabilidade interna + baixos `wr`/`wm`).
    
- **Janela vencedora (L = 6h)**:  
    **2024-06-13 19:00 → 2024-06-14 00:00 (n=6)**  
    Medianas de referência: **Wr̃ ≈ 1.930,887** e **Wm̃ ≈ 4.891×10¹¹**.
    
- Artefatos:
    
    - `...\reference_selection\reference_period.json` (metadados da escolha)
        
    - Tabelas de scores por janela (`reference_scores_L*.csv`) e `reference_top5_by_L_*.csv`
        
    - Plots de apoio em `...\reference_selection\plots\`
        

# 5) Normalização por referência e diagnóstico de proximidade

- Criamos `...\reference_selection\dados_ref.csv` adicionando **duas colunas-chave**:
    
    - `wr/wrref = wr / Wr̃`
        
    - `wm/wmref = wm / Wm̃`
        
- Plotamos as **séries em dupla escala** (mesmo gráfico, dois eixos-y) e o **plano (wr/wrref, wm/wmref)** com **elipse de Mahalanobis (95%)**, suprimindo um outlier extremo para melhorar a leitura. A métrica “% dentro da elipse” virou **indicador de quão “próximo da referência”** está a operação.
    

# 6) Dois regimes operacionais (não supervisionado)

- Com as **variáveis internas** (pressões, temperaturas, fluxos, correntes, etc.):
    
    - Tratamos ausências (sem descartar muita informação): **drop colunas** com NaN ≥ 40% (nenhuma caiu), **imputação por mediana**, **robust-z (mediana/IQR)** e **KMeans (k=2)**.
        
    - **Separação muito forte**: **silhouette = 0,957**.
        
    - **Tamanho**: cluster 0 (n=3.483) e cluster 1 (n=1.153).
        
    - **Abrasividade relativa (medianas)**:
        
        - **Cluster 1 (regime “bom”)**: `wr/wrref ≈ 3,33`, `wm/wmref ≈ 2,59`
            
        - **Cluster 0 (regime “alto desgaste”)**: `wr/wrref ≈ 391,62`, `wm/wmref ≈ 8,58`
            
- **Quais sensores diferenciam** (ranking combinado por |Δ|/IQR, KS e MI):  
    em síntese, o regime bom aparece com **pressões mais altas** em certos **ramos de ar secundário (A/C, heads)**, **menor velocidade** no **ventilador B**, e **temperaturas mais baixas** em aquecedores/reaquecimento.  
    (Planilha completa em `diffvars_k2_*.csv`.)
    
- **PCA por cluster**: geramos loadings completos (PC1/PC2) e “top-15” por cluster, além das curvas de variância explicada (**scree**). Isso ajuda a ler **quais sensores dominam a variabilidade interna** de cada regime.  
    (Arquivos `pca_loadings_cluster{0,1}_*.csv` e `plots\pca_*`.)
    
- **Quadro comparativo p10/p50/p90** (top-10 variáveis mais distintas) por cluster:  
    arquivo `comparativo_top10_p10_p50_p90_*.{csv,md}` em `...\reference_selection\cluster2\`.
    

# 7) “Cédula operacional” e diretrizes

Entregamos uma **carta de navegação** (“cédula”) para a equipe, com:

- alvo de operação em **regime menos abrasivo** (manter `wr/wrref ≲ 5`, `wm/wmref ≲ 3`);
    
- **indicadores online**: `% dentro da elipse 95%` (proximidade da referência), pressões A/C, velocidade do vent. B, temperaturas de aquecedores;
    
- **sugestões de setpoint** (distribuição de ar secundário e bandas de temperatura) e **alarmes suaves** caso as razões disparem por tempo prolongado;
    
- observações de **qualidade de dado** (unidades/offsets suspeitos em alguns instrumentos).
    

# 8) Onde está cada coisa (mapa rápido)

- **Freeze oficial**: `...\outputs\freeze\v1_20250818_0635\`
    
    - **MVP supervisionado (E0)**: `models\mvp\model_mvp.joblib` (+ manifesto e card)
        
    - **Modelo físico**: `models\phys\model_phys_mvp.joblib` (+ manifesto e card)
        
    - **Referência**: `reference_selection\` (JSON, tabelas, plots, `dados_ref.csv`)
        
    - **Clustering (2 regimes)**: `reference_selection\cluster2\`
        
        - `labels_k2_*.csv`, `diffvars_k2_*.csv`, `centroids_k2_*.csv`,
            
        - `comparativo_top10_p10_p50_p90_*.{csv,md}`,
            
        - `plots\pca2_k2.png`, `plots\pca_top15_pc1_c{0,1}_*.png`, `plots\pca_scree_c{0,1}_*.png`.
            
- **Nova tratativa (limpeza de zeros/NaNs)**:
    
    - Bases derivadas: `...\outputs\nova_tratativa\base_dados\A1_ML_DL_features_*_*.csv`
        
    - Modelo e métricas (base limpa): `...\outputs\nova_tratativa\models\*nozeros_nonan_*.{joblib,csv,md}`
        

# 9) O que ficou decidido

- **Modelo para MVP**: **Random Forest multi-alvo sem log** (E0) sobre o **GOLD**.
    
- **Referência operacional**: janela **L=6h (2024-06-13 19:00 → 2024-06-14 00:00)**, com **Wr̃** e **Wm̃** para normalização.
    
- **Indicador de proximidade**: elipse **Mahalanobis 95%** no plano `(wr/wrref, wm/wmref)`.
    
- **Regimes operacionais**: há **dois universos bem separados**; o **cluster 1** é o **menos abrasivo**.
    
- **Foco de atuação**: distribuição de **ar secundário** (pressões A/C ↑, velocidade B ↓) e **temperaturas** (faixa baixa) — sempre respeitando limites de processo e confirmando unidades/instrumentação.
    

# 10) Próximos passos sugeridos

1. **Telemetria on-line** dessas métricas (wr/wrref, wm/wmref, %Mahalanobis, pressões A/C, vel. B, temperaturas) com limites de atenção.
    
2. **Experimentos de setpoint** pequenos e controlados (A/B) para validar causalidade e quantificar ganho real em abrasividade.
    
3. **Banda operacional recomendada** (p10–p90 do cluster 1) publicada em planilha e telas da sala de controle.
    
4. **Drift e saúde de sensores**: monitorar PSI/KS das features de maior impacto (lista do `diffvars_k2_*.csv`) e revisar instrumentos com unidades/offset suspeitos.
    

Se quiser, eu já **compilo a planilha de bandas recomendadas (p10–p90 do cluster 1)** a partir do comparativo e deixo no mesmo diretório da referência.