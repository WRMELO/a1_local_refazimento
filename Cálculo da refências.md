ótima pauta. Minha proposta é transformar “estabilidade + baixo desgaste (Wr/Wm)” em um critério **objetivo e reprodutível** por janelas de tempo, e selecionar o (ou os) melhores trechos. Abaixo está o desenho — com defaults sensatos — para você adotar como “referência operacional”.

# 1) Pré-filtro (higiene dos dados)

- Use a base atual **sem zeros e sem NaN** nos alvos (Wr/Wm).
    
- Opcional (recomendado): `status_operação == 1` se existir na base.
    
- Para Wm (cauda pesada), trabalhe em **log10(Wm)** quando fizer medidas geométricas (Mahalanobis etc.), mas reporte sempre em escala linear no final.
    

# 2) Conjunto de variáveis “internas da fornalha”

Selecione apenas medições **internas** (temperaturas, pressões, vazões, aberturas, correntes dos ventiladores, tiragem etc.) que **não** foram usadas direta/indiretamente no cálculo de Wr/Wm (mantendo o leak-guard).  
Regra prática de seleção (padrão): colunas que comecem por `temp_`, `pressao_`, `ar_`, `vent_`, `flow_`/`flw_`, `corr_`, `tiragem_`, `o2_`, `so2_`, exceto as explicitamente marcadas como parte do bloco físico.

# 3) Janela temporal e amostragem

- **Tamanho de janela (L):** 12 horas (bom trade-off). Também avalie L=6h e 24h para multi-escala.
    
- **Passo (stride):** 1 amostra (deslizante).
    
- **Duração mínima de referência:** ≥ 6 amostras (ou ≥ L/2, o que for maior).
    
- **Tolerância a buracos:** permitir pequenos gaps (ex.: até 1 amostra) sem quebrar a janela.
    

# 4) Métrica de **estabilidade** (S) por janela

Combine 3 vistas robustas (todas padronizadas por **mediana/IQR** por variável antes de agregar):

1. **Variabilidade intra-janela** (Sᵥ):
    
    - Para cada variável interna, compute **MAD** ou desvio-padrão robusto na janela (após robust-zscore).
        
    - Agregue pela **mediana** entre variáveis. → menor = mais estável.
        
2. **Rugosidade/dinâmica** (Sᵣ):
    
    - Para cada variável, calcule o **mediano de |Δx|** (diferença ponto a ponto) na janela (após robust-zscore).
        
    - Agregue pela **mediana** entre variáveis. → menor = mais suave.
        
3. **Compacidade multivariada** (Sᶜ):
    
    - No espaço das variáveis internas padronizadas, compute a **covariância robusta** da janela.
        
    - Use `log det(Σ)` ou o **maior autovalor** como escalar de “espalhamento”. → menor = mais compacto.
        

Normalize cada componente para [0,1] (min-max ao longo de todas as janelas) e combine:

S=0.4 Sv+0.4 Sr+0.2 ScS = 0.4\,Sᵥ + 0.4\,Sᵣ + 0.2\,Sᶜ

(Pesos ajustáveis; defaults equilibram nível e dinâmica, com leve peso para compacidade.)

# 5) Métrica de **baixo desgaste** (W) por janela

Use **valores medianos** da janela: `Wr̃`, `Wm̃`. Construa W de duas formas (e fique com a melhor/mais estável):

**Opção A (quantílica, simples e robusta):**

- Compute os quantis **apenas entre positivos** das distribuições globais:  
    `q_wr = F⁺(Wr̃)`, `q_wm = F⁺(Wm̃)` (entre 0 e 1).
    
- Combine por **média geométrica**: `W = √(q_wr · q_wm)` (menor é melhor).
    

**Opção B (geométrica 2D):**

- Trabalhe no espaço `(Wr, log10 Wm)` para reduzir assimetria.
    
- Estime um **“centro de baixa”** global com base na interseção de quantis baixos (ex.: p10 de Wr e p10 de Wm⁺).
    
- Calcule a **Mahalanobis²** da janela (usando a covariância das amostras “baixas” globais).
    
- Normalize para [0,1] por min-max: isso é o `W`. (menor = mais “baixo/casado”).
    

Se B e A discordarem, dê preferência ao que **melhor correlacionar** com sua intuição operacional (rapidamente verificável num scatter `(Wr, Wm)` com color-map pelo score).

# 6) Critério final e seleção da referência

Duas formas equivalentes; eu recomendo a **constrangida**:

**(i) Constrangida em 2 passos (recomendado):**

1. **Estabilidade primeiro:** mantenha janelas no **melhor decil** de S (ex.: `S ≤ p10(S)`), garantindo estabilidade forte.
    
2. Entre as sobreviventes, escolha as de **menor W** (top-K, e em especial a mais longa/contígua).
    

**(ii) Escalar único (se quiser score único):**

J=α S+(1−α) Wcom α=0.5J = \alpha\,S + (1-\alpha)\,W \quad \text{com } \alpha=0.5

Escolha janelas com **J mínimo** + **maior duração contínua**.

Em ambos os casos, reporte:

- **Janela “referência oficial”** (início, fim, nº amostras, duração, medianas Wr̃/Wm̃, e desvios);
    
- **Top-5 candidatas** (úteis para auditoria/rollback);
    
- **Parâmetros do método** (L, stride, pesos, quantis usados).
    

# 7) Sanity checks e salváveis

- **Sanity**: mostrar onde a janela cai no scatter `(Wr, Wm)` e sobre o tempo (faixa azul no timeline).
    
- **Salvar** em `FREEZE_DIR`:
    
    - `reference_period.json` (parâmetros + janela oficial + top-K)
        
    - `reference_scores.csv` (uma linha por janela com S, W, J, duração)
        
    - `reference_plots/` (timeline com faixa, scatter com ROI, histos)
        
    - `roi_low_zone.json` (centro/cov da zona baixa usada em W-Mahalanobis, se você optar por B)
        

# 8) Defaults sugeridos (prontos para uso)

- L = **12h**, stride = 1 amostra; duração mínima ≥ **6 amostras**; tolerância de gap = **1**.
    
- Estabilidade: robust-zscore por variável; **MAD** para Sᵥ, mediana de |Δx| para Sᵣ, `log det` para Sᶜ.
    
- W: **Opção A** (quantílica) como base; **Opção B** para confirmação.
    
- Seleção: **constrangida** (S no **p10**) → **menor W**, empatando por **maior duração**.
    

---

Se estiver de acordo, eu já te entrego uma **célula única** que implementa exatamente esse fluxo (com os nomes de pastas do seu projeto), calcula S/W/J em **L=6/12/24h**, escolhe a **referência oficial**, e salva **JSON/CSV/plots** no FREEZE atual.