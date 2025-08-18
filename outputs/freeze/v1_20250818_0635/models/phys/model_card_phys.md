# Projeto A1 — Model Card (Physics-Based MVP)

## 1) Identificação
- **Modelo:** PhysicsBasedModel (determinístico)
- **Tag:** MVP_PHYSICS
- **Freeze dir:** C:\Users\wilso\MBA_EMPREENDEDORISMO\3AGD\A1_LOCAL_REFAZIMENTO\outputs\freeze\v1_20250818_0635
- **Arquivo:** models\phys\model_phys_mvp.joblib
- **Gerado em:** 2025-08-18

## 2) Fórmulas
- Wr = α · ν² · δ¹ · exp(−T/Tcrit) · (σ_mat/ρ_fuel)
- Wm = β · ν³ · √δ · exp(−T/Topt) · (σ_steel/μ_gas)

## 3) Insumos esperados (nomes canônicos)
nu_m_s, delta_eff_m, T_gas_K, mu_gas_Pa_s, sigma_mat, rho_fuel_kg_m3, sigma_steel

## 4) Constantes (MVP)
α=1.0, β=1.0, Tcrit=1400.0 K, Topt=1100.0 K

## 5) Notas
- Unidade esperada: ν[m/s], δ[m], T[K], μ_gas[Pa·s], ρ_fuel[kg/m³].
- O artefato **não aprende**; aplica equações físicas. Calibração de α/β/Tcrit/Topt pode ser feita em futura versão.
- Se o CSV tiver nomes diferentes, renomeie ou mapeie para os canônicos antes de inferir.

## 6) Verificação
- Executada: Não
- Estatísticas: —
- Amostra de predição: —
