# Verificação de Adequação – GOLD

- Linhas: **10005**  |  Features numéricas: **144**
- Alvos presentes: `wr_kg_m2_h, wm_kg_m2_h`
- Mediana de faltas nas features: **0.00%**
- Colunas constantes: **1**  | quase-constantes: **0**
- Pares com |corr| ≥ 0.97: **1184**
- Timestamp: **timestamp** | início: **2023-03-01 07:00:00** | fim: **2024-10-14 10:00:00**
- Passo de 1h (≈): **99.59%** | gaps >1h: **41**

## Vazamento (features proibidas detectadas)
- Encontradas: **24**
  - corr_primary_air_vent_motor_a
  - corr_primary_air_vent_motor_b_a
  - corr_secondary_air_vent_motor_a
  - corr_secondary_air_vent_motor_b_a
  - differential_pressao_fornalha_a
  - differential_pressao_fornalha_b
  - fornalha_leito_temp_10_12
  - fornalha_leito_temp_13_15
  - fornalha_leito_temp_16_18
  - fornalha_leito_temp_1_3
  - fornalha_leito_temp_4_6
  - fornalha_leito_temp_7_9
  - lms_pdr_bin_flud_air_htr_temp
  - media_horaria_so2_apos_fgd_mg_nm3
  - pressao_air_in_stm_preaq_ar_paf_a_saida
  - pressao_air_in_stm_preaq_ar_paf_b_saida
  - pressao_fornalha_a_medio
  - pressao_fornalha_b_medio
  - pressao_hot_pri_air_in_preaq_ar_saida
  - pressao_tambor
  - so2_con_clean_flue_gas_rectifi_mg_m3
  - so2_con_raw_flue_gas_rectified_mg_nm3
  - so2_concentration_raw_flue_gas_mg_m3
  - temp_hot_pri_air_in_preaq_ar_saida

## Baseline – Supervisionado (RandomForest multi-alvo)
- Split: train=7003, val=1001, test=2001
- VAL:
  - `wr_kg_m2_h` → R²=0.930 | MAE=5.4e-08
  - `wm_kg_m2_h` → R²=0.921 | MAE=2.7e+04
- TEST:
  - `wr_kg_m2_h` → R²=0.754 | MAE=1.07e-07
  - `wm_kg_m2_h` → R²=0.682 | MAE=6.06e+04

## Não-supervisionado (PCA + Silhouette)
- Componentes p/ 95% variância: **30**
- Variância acumulada (10 comps): 0.54, 0.66, 0.71, 0.74, 0.76, 0.78, 0.79, 0.81, 0.82, 0.83
- Silhouette k=2..6: k=2:0.583, k=3:0.531, k=4:0.343, k=5:0.313, k=6:0.301

## Juízo de adequação (heurístico)
- **Supervisionado**: OK (n_train≥1000 e ≥10 features num.)
- **Não-supervisionado**: OK (PCA ≤ 30 comps p/ 95% var.)