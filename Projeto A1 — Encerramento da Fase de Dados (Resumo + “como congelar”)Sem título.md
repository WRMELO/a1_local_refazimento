

> Documento para você colar no novo chat/notebook e continuar direto no **modelamento**.

---

## 1) O que fizemos (linha do tempo resumida)

- **Base escolhida:** `a1_physics_informed_proxies.csv` → padronizada para `A1_ML_DL.csv` (2 linhas de cabeçalho: **nome** | **dimensão**).
    
- **Semântica de cabeçalhos:** ajustes e auditoria; removemos colunas auxiliares que não tinham significado confiável.
    
- **Rótulos físicos (MVP):** calculamos `wr_kg_m2_h` e `wm_kg_m2_h` a partir de proxies físicas:
    
    - Vazão real de gás `Q_real` a partir de **QN_total (kNm³/h)** com correções de **T** e **P**.
        
    - **ν (velocidade)** via área efetiva.
        
    - **ΔP (robusto):** usa `dp_fornalha` se for ΔP; caso contrário, `|pressao_fornalha_a_inf − pressao_fornalha_b_inf|`.
        
    - **ρ_g** com mistura ar/CO₂ (γ default 0,25 no MVP), **μ_gas** via Sutherland (TK robusto mesmo com gaps).
        
    - **δ (diâmetro efetivo)** aproximado a partir de ΔP (balanceamento tipo Ergun) com ganho `C_DELTA`.
        
    - Fórmulas:
        
        - `Wr = α · ν² · δ¹ · exp(−T/Tcrit) · g`
            
        - `Wm = β · ν³ · √δ · exp(−T/Topt) · h`  
            com constantes base (MVP) e fatores `g = σ_mat/ρ_fuel`, `h = σ_steel/μ_gas`.
            
- **Correções críticas:**
    
    - **QN_total preenchido**: `air_total_knm3_h` → `air_primary + air_secondary` (linha a linha) → média entre bordas do bloco (reduziu NaNs).
        
    - **μ_gas robusto:** evitamos usar `o2_medio` como densidade de CO₂; quando T faltou, usamos `TK_mu = 1100 K` (seguro).
        
- **Versões dos arquivos:**
    
    - **v2/v3/v4** até estabilizarmos **Wr/Wm**.
        
    - **GOLD** = filtros sem imputar ΔP além do robusto e sem “forçar” QN — 10.005 linhas válidas.
        
    - **SILVER2** = imputação adicional de **QN** (interp ≤24h + mediana móvel) **e** ΔP (interp ≤6h + `k·ρ_g·ν²` com clipping) — 11.749 linhas válidas.
        
- **Anti-vazamento (Leak-guard):** removemos das features todas as colunas usadas direta/indiretamente no cálculo dos rótulos, incluindo o bloco físico (`TAU_DENSA → fim`) e `air_total_knm3_h_filled`.
    
- **Diagnósticos & adequação (outputs):**
    
    - **GOLD** (10.005 linhas, 144 features numéricas, miss=0%):
        
        - RF baseline (teste): R² **Wr=0.754**, **Wm=0.682**; Silhouette k=2 ≈ **0.58**.
            
    - **SILVER2** (11.749 linhas, 144 features numéricas, miss=0%):
        
        - RF baseline (teste): R² **Wr=0.506**, **Wm=0.627**; Silhouette k=2 ≈ **0.58**.
            
    - Conclusão: **GOLD** como conjunto principal; **SILVER2** para robustez/ablação.
        

---

## 2) Artefatos finais e caminhos (consolidados)

**Dados “curated”:**

- `C:\Users\wilso\MBA_EMPREENDEDORISMO\3AGD\A1_LOCAL_REFAZIMENTO\data\curated\A1_ML_DL_rotulado_v4.csv`
    
- `C:\Users\wilso\MBA_EMPREENDEDORISMO\3AGD\A1_LOCAL_REFAZIMENTO\data\curated\A1_ML_DL_features_v4.csv`
    

**Conjuntos finais para modelagem:**

- **GOLD**
    
    - Rotulado: `...\curated\A1_ML_DL_rotulado_v4_gold.csv`
        
    - Features: `...\curated\A1_ML_DL_features_v4_gold.csv`
        
- **SILVER2**
    
    - Rotulado: `...\curated\A1_ML_DL_rotulado_v4_silver2.csv`
        
    - Features: `...\curated\A1_ML_DL_features_v4_silver2.csv`
        

**Relatórios/diagnósticos:**

- `C:\Users\wilso\MBA_EMPREENDEDORISMO\3AGD\A1_LOCAL_REFAZIMENTO\outputs\diagnosticos\`
    
    - `gold_relatorio.md`, `silver2_relatorio.md`
        
    - `gold_*.csv`, `silver2_*.csv` (missingness, leakage, constantes, colinearidade, etc.)
        

---

## 3) Sugestão de **congelamento** (freeze) para trabalhar em outro notebook

> Cria uma pasta “versão” com cópia dos artefatos, um **manifesto** e **checksums**; opcionalmente zipa.  
> Ajuste `TAG` se quiser versionar por data/versão.

```python
# ===================== FREEZE DOS DADOS PARA MODELAGEM =====================
import os, shutil, json, hashlib, pandas as pd
from datetime import datetime

BASE = r"C:\Users\wilso\MBA_EMPREENDEDORISMO\3AGD\A1_LOCAL_REFAZIMENTO"
CUR  = os.path.join(BASE, "data", "curated")
OUTS = os.path.join(BASE, "outputs", "diagnosticos")

# Tag de versão (ex.: v1_YYYYmmdd_HHMM)
TAG = "v1_" + datetime.now().strftime("%Y%m%d_%H%M")
FREEZE_DIR = os.path.join(BASE, "outputs", "freeze", TAG)
os.makedirs(FREEZE_DIR, exist_ok=True)

# Arquivos que vamos congelar (adicione se quiser mais)
FILES = [
    os.path.join(CUR, "A1_ML_DL_rotulado_v4_gold.csv"),
    os.path.join(CUR, "A1_ML_DL_features_v4_gold.csv"),
    os.path.join(CUR, "A1_ML_DL_rotulado_v4_silver2.csv"),
    os.path.join(CUR, "A1_ML_DL_features_v4_silver2.csv"),
    os.path.join(CUR, "A1_ML_DL_rotulado_v4.csv"),
    os.path.join(CUR, "A1_ML_DL_features_v4.csv"),
]

# Copiar diagnósticos principais (MDs)
DIAG_MD = [
    os.path.join(OUTS, "gold_relatorio.md"),
    os.path.join(OUTS, "silver2_relatorio.md"),
]

def sha256_of(path, chunk=1<<20):
    h = hashlib.sha256()
    with open(path, "rb") as f:
        while True:
            b = f.read(chunk)
            if not b: break
            h.update(b)
    return h.hexdigest()

# 1) Copia arquivos
copied = []
for p in FILES + DIAG_MD:
    if os.path.exists(p):
        dst = os.path.join(FREEZE_DIR, os.path.basename(p))
        shutil.copy2(p, dst)
        copied.append(dst)

# 2) Gera checksums
ck_path = os.path.join(FREEZE_DIR, "checksums_sha256.txt")
with open(ck_path, "w", encoding="utf-8") as f:
    for p in copied:
        f.write(f"{sha256_of(p)}  {os.path.basename(p)}\n")

# 3) Manifesto com contagem de linhas/colunas dos CSV principais
manifest = {"tag": TAG, "generated_at": datetime.now().isoformat(), "files": []}
for p in copied:
    item = {"file": os.path.basename(p), "path": p}
    if p.lower().endswith(".csv"):
        try:
            df = pd.read_csv(p, header=[0,1], engine="python")
            item["rows"] = int(df.shape[0])
            item["cols"] = int(len(df.columns))
            # registra nomes das colunas e dimensões (primeiras 10 colunas como amostra)
            cols = [str(c[0]) for c in df.columns]
            dims = [str(c[1]) for c in df.columns]
            item["sample_columns"] = cols[:10]
            item["sample_dims"]    = dims[:10]
        except Exception:
            pass
    manifest["files"].append(item)

with open(os.path.join(FREEZE_DIR, "manifest.json"), "w", encoding="utf-8") as f:
    json.dump(manifest, f, indent=2, ensure_ascii=False)

print("✅ Freeze concluído em:", FREEZE_DIR)

# 4) (Opcional) Criar um ZIP da pasta
# import shutil
# shutil.make_archive(FREEZE_DIR, "zip", FREEZE_DIR)
# print("ZIP gerado em:", FREEZE_DIR + ".zip")
# ========================================================================== 
```

**Como usar depois no notebook de modelagem:**

- Aponte seus loaders para `outputs/freeze/<TAG>/A1_ML_DL_features_v4_gold.csv` (principal)  
    e opcionalmente `..._silver2.csv` (robustez).
    
- Os relatórios `gold_relatorio.md` e `silver2_relatorio.md` vão junto para consulta rápida.
    

---

## 4) Splits e pré-processamento (opcional, já pronto para o próximo notebook)

> Se quiser, já deixe prontos os índices de **train/val/test** e um **pipeline** (imputer + scaler) salvos no mesmo `FREEZE_DIR`.

```python
# =============== OPCIONAL: SPLITS + PIPELINE (para reprodutibilidade) ===============
import os, joblib, numpy as np, pandas as pd
from sklearn.impute import SimpleImputer
from sklearn.preprocessing import StandardScaler

FREEZE_DIR = r"C:\Users\wilso\MBA_EMPREENDEDORISMO\3AGD\A1_LOCAL_REFAZIMENTO\outputs\freeze\__COLOQUE_A_TAG_AQUI__"
PATH_FEATS = os.path.join(FREEZE_DIR, "A1_ML_DL_features_v4_gold.csv")

df = pd.read_csv(PATH_FEATS, header=[0,1], engine="python")
df.columns = [c for (c,_) in df.columns]  # usa só a linha de nomes
ycols = ["wr_kg_m2_h","wm_kg_m2_h"]
X = df.drop(columns=ycols).select_dtypes(include=[np.number])
y = df[ycols].apply(pd.to_numeric, errors="coerce")

n = len(df)
idx = np.arange(n)
train_end = int(0.7*n); val_end = int(0.8*n)
splits = {
    "train": idx[:train_end].tolist(),
    "val":   idx[train_end:val_end].tolist(),
    "test":  idx[val_end:].tolist(),
}
joblib.dump(splits, os.path.join(FREEZE_DIR, "splits_gold_70_10_20.joblib"))

imp = SimpleImputer(strategy="median")
scaler = StandardScaler()
X_train = imp.fit_transform(X.iloc[splits["train"]])
scaler.fit(X_train)
joblib.dump(imp,    os.path.join(FREEZE_DIR, "imputer_gold_median.joblib"))
joblib.dump(scaler, os.path.join(FREEZE_DIR, "scaler_gold_standard.joblib"))
print("✅ Splits e pipeline salvos em:", FREEZE_DIR)
# ====================================================================================
```

> Troque `__COLOQUE_A_TAG_AQUI__` pela pasta criada no passo de freeze.

---

## 5) Próximos passos (modelagem)

- **Supervisionado (principal):** inicie com **GOLD**. Modelos base:
    
    - Árvores/GBTs (XGBoost/LightGBM) com `log1p` nos alvos _se_ cauda pesada;
        
    - Série temporal: janelas (ex.: 24–48 h) com **TCN/GRU**; validação temporal 70/10/20.
        
- **Não-supervisionado:** PCA + KMeans / DBSCAN; detecção de anomalias (Isolation Forest/LOF) sobre features **não vazadas**.
    
- **Ablação/robustez:** replique com **SILVER2** e compare métricas; se ganhos forem consistentes sem degradar períodos medidos, considere ensembles.
    

---

### Tudo certo com nomes e paths?

Sim — estão **consolidados** acima. Se quiser, eu também preparo um “starter notebook de modelagem” já apontando para o `FREEZE_DIR` e carregando `splits` + `pipeline`.