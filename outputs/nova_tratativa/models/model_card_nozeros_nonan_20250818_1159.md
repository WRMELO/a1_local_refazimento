# A1 — MVP (features.csv sem zeros/NaN nos alvos)

**Data**: 2025-08-18T12:00:13.257008

**Entrada (limpa)**: `C:\Users\wilso\MBA_EMPREENDEDORISMO\3AGD\A1_LOCAL_REFAZIMENTO\outputs\nova_tratativa\base_dados\A1_ML_DL_features_nozeros_nonan_20250818_1159.csv`

**Excluídos**:
- zeros: `C:\Users\wilso\MBA_EMPREENDEDORISMO\3AGD\A1_LOCAL_REFAZIMENTO\outputs\nova_tratativa\base_dados\A1_ML_DL_features_onlyzeros_20250818_1159.csv`
- NaN: `C:\Users\wilso\MBA_EMPREENDEDORISMO\3AGD\A1_LOCAL_REFAZIMENTO\outputs\nova_tratativa\base_dados\A1_ML_DL_features_onlynans_20250818_1159.csv`

**Modelo**: RandomForestRegressor (MultiOutput), n_estimators=400

**Split temporal**: 70/10/20 (ordem do CSV ordenado por timestamp)

## Métricas
    split  linhas              inicio                 fim    R2_wr      RMSE_wr      MAE_wr    R2_wm      RMSE_wm       MAE_wm
   treino    3245 2023-03-01 07:00:00 2024-03-27 05:00:00 0.999910  3658.029234 1842.425976 0.986294 5.595303e+11 1.285777e+11
validacao     463 2024-03-27 06:00:00 2024-05-14 19:00:00 0.532544 18156.514433 8142.792474 0.888917 1.176099e+12 5.818540e+11
    teste     928 2024-05-14 20:00:00 2024-10-14 10:00:00 0.142216 45498.704990 5353.494048 0.080925 1.423536e+13 8.715416e+11
