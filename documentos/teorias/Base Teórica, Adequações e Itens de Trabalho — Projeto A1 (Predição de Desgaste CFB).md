
---

**Data:** 2025-08-10  
**Protocolo ativo:** PROTOCOLOS_UNIFICADOS_V2 — MODO ULTRA-HF-000  
**Referências:**

- Documento "Teoria da Combustão e Fenômenos de Desgaste em Caldeiras CFB"
    
- "Plano de Ataque A1"
    
- Histórico e checkpoints A1
    

---

## 1. Objetivo

Estabelecer a base técnica e o roteiro operacional para calcular e aplicar modelos de desgaste **Wr** (refratário) e **Wm** (metálico) em caldeiras CFB, visando:

1. **Estimativa** da taxa de desgaste por região (fase densa, diluída, backpass).
    
2. **Orientação operacional** para reduzir desgaste sem comprometer carga, eficiência ou emissões.
    
3. Disponibilizar **duas versões** do trabalho:
    
    - **Baseline explicável** (physics-informed).
        
    - **Modelagem ML/DL** de maior capacidade.
        

---

## 2. Fundamentos Teóricos — Wr/Wm

### 2.1 Equação geral do desgaste

W=k⋅νn⋅δm⋅exp⁡(−τTcrit)⋅f(p1,p2,…)W = k \cdot \nu^n \cdot \delta^m \cdot \exp\left(- \frac{\tau}{T_{\text{crit}}} \right) \cdot f(p_1, p_2, \ldots)

**Termos:**

- **W**: taxa de desgaste [M·L⁻²·T⁻¹]
    
- **k**: constante de proporcionalidade
    
- **ν**: velocidade do fluxo gás-sólido [L·T⁻¹]
    
- **δ**: diâmetro médio das partículas [L]
    
- **τ**: temperatura de operação [K]
    
- **Tcrit**: temperatura crítica [K]
    
- **f(·)**: função adimensional com propriedades do material e condições de operação
    
- **n, m**: expoentes que indicam a contribuição de velocidade e tamanho de partículas.
    

### 2.2 Especializações

**Refratário (Wr):**

Wr=α⋅ν2⋅δ1⋅exp⁡(−τTcrit)⋅g(σmat,ρfuel)Wr = \alpha \cdot \nu^2 \cdot \delta^1 \cdot \exp\left(- \frac{\tau}{T_{\text{crit}}} \right) \cdot g(\sigma_{\text{mat}}, \rho_{\text{fuel}})

**Metálico (Wm):**

Wm=β⋅ν3⋅δ0.5⋅exp⁡(−τTopt)⋅h(σsteel,μgas)Wm = \beta \cdot \nu^3 \cdot \delta^{0.5} \cdot \exp\left(- \frac{\tau}{T_{\text{opt}}} \right) \cdot h(\sigma_{\text{steel}}, \mu_{\text{gas}})

**g(·)** e **h(·)** são funções adimensionais:

- g=σmat/ρfuelg = \sigma_{\text{mat}} / \rho_{\text{fuel}}
    
- h=σsteel/μgash = \sigma_{\text{steel}} / \mu_{\text{gas}}
    

---

## 3. Adequações para MVP

### 3.1 Índices adimensionais

Como não temos dados absolutos de desgaste (mm ou kg) para calibrar α e β, adotaremos índices relativos:

Ir(t)=W^r(t)Wrref,Im(t)=W^m(t)WmrefI_r(t) = \frac{\widehat{W}_r(t)}{W_r^{\text{ref}}}, \quad I_m(t) = \frac{\widehat{W}_m(t)}{W_m^{\text{ref}}}

**Definição de Wref:**

- Mediana de W^\widehat{W} durante operação normal (status_operacao = 1) por zona.
    
- Alternativas: percentil 60–70 ou janela "benchmark".
    

Quando houver evento medido de desgaste, recalibraremos Wref para converter índices em valores absolutos.

### 3.2 Proxies

- **ν_proxy**: derivado de vazões de ar, ΔP em fornalha e recirculação.
    
- **δ_proxy**: de indicadores de peneiramento, loop seal, ΔP ciclone↔fornalha.
    
- **τ**: médias ponderadas por zona (densa/diluída/backpass).
    

---

## 4. Itens e Artefatos

### 4.1 Arquivos base

- **Consolidado:** `dados_operacao_consolidado.csv` (17.448 × 150)
    
- **Derivados:**
    
    - `dados_em_operacao_normal.csv` (status_operacao = 1)
        
    - `dados_fora_operacao_normal.csv` (status_operacao = 0)
        
    - `resumo_split_operacao.csv` (auditoria)
        

### 4.2 Resultados esperados

- `correlacoes_altas_*.csv` (pares com |r| ≥ 0,90)
    
- `mapa_grupos_correlacao.csv` (grupos + membros + agregados)
    
- `dados_agreg_wr.csv` (agregados mean/std por grupo)
    
- `features_fisicas.csv` (ν_proxy, δ_proxy, τ por zona)
    
- `dicionario_proxies.csv` (fórmulas e variáveis)
    
- `indices_adimensionais.csv` (Ir, Im por zona)
    

---

## 5. Caminho de Trabalho (Plano de Ataque)

### Caminho A — Physics-informed

- Regressão log-linear + GAM para estimar parâmetros.
    
- Alta interpretabilidade e baseline para comparação.
    

### Caminho B — ML/DL

- Modelos como GBM, TCN, Transformers temporais para risco e RUL.
    
- Incorporação fraca da teoria via features físicas.
    

### Caminho C — Híbrido

- Núcleo teórico multiplicado por correções aprendidas.
    
- Usado para otimização multiobjetivo de setpoints.
    

---

## 6. Próximos Passos

1. **EDA final**:
    
    - Matriz de correlação.
        
    - Formação de grupos (|r| ≥ 0,90).
        
    - Criação de agregados mean/std mantendo `flw_total_*` e descartando parciais.
        
2. **Proxies físicos**:
    
    - Calcular ν_proxy, δ_proxy, τ.
        
    - Gerar features_fisicas.csv e dicionario_proxies.csv.
        
3. **Índices adimensionais**:
    
    - Calcular Wr_hat, Wm_hat e Ir, Im.
        
    - Salvar indices_adimensionais.csv.
        

---

## 7. Disclaimer

- Índices são relativos até calibração.
    
- Proxies iniciais serão refinados com dados futuros.
    
- PCA e autoencoder só para a trilha ML/DL.
    

---

Se quiser, já posso usar esse documento como **referência fixa** para as próximas etapas do EDA e geração dos datasets curados.

aqui está o **checkpoint** no seu padrão, pronto para copiar e colar no `HISTÓRICO.md`:

---

# CHECKPOINT – Normalização de Features e Auditoria de Fontes (A1)

**Data/Hora:** 16/08/2025 (America/Sao_Paulo)  
**Responsável:** Wilson Melo  
**Protocolo ativo:** Documento Unificado de Operação — GPT-5 (V0)  
**MODO ULTRA-HF-000:** ATIVADO

---

## 🎯 Objetivo

Padronizar nomes de features para **português curto e único** (`nome_pt`) e garantir que **todas as fontes de dados** do A1 passem pelo processo de normalização com **backup** e **auditoria**.

---

## ✅ Ações executadas nesta atualização

1. **Dicionário canônico consolidado**
    
    - Arquivo: `C:\Users\wilso\MBA_EMPREENDEDORISMO\3AGD\A1_LOCAL\features_padronizadas.csv`
        
    - SHA256 atual (ambiente local): `73078b08b9ab591a9871889bdcb0e69c04329ab83d069534ffd21dc19ef8a5df`
        
    - Markdown de referência foi atualizado **apenas na tabela** (coluna `nome_pt` incluída), **sem alterar os significados**:  
        `C:\Users\wilso\MBA_EMPREENDEDORISMO\3AGD\A1_LOCAL\NOME DAS VARIAVEIS E SEUS SIGNIFICADOS_COM_NOME_PT.md`
        
2. **Auditoria e normalização das fontes** (com **backup simples** `_old.csv`)
    
    - Inventário: `C:\Users\wilso\MBA_EMPREENDEDORISMO\3AGD\A1_LOCAL\outputs\auditoria_normalizacao_features.csv`
        
    - Total de arquivos processados na execução reportada: **26**
        
    - Exemplos do inventário:
        
        - `data\curated\a1_physics_informed.csv` → **backup_e_normalizacao** (mapeadas: 150, não mapeadas: 0)
            
        - `data\curated\a1_physics_informed_enriched.csv` → **backup_e_normalizacao** (150, 4)
            
        - `data\curated\a1_physics_informed_proxies.csv` → **backup_e_normalizacao** (150, 16)
            
        - `outputs\baseline_datasets\physics_baseline_proxies.csv` → **backup_e_normalizacao** (150, 16)
            
        - `outputs\baseline_datasets\physics_offbaseline_proxies.csv` → **backup_e_normalizacao** (150, 16)
            
        - `outputs\baseline_datasets\compare_baseline_vs_off.csv` → **somente_auditoria** (0, 4)
            
        - `outputs\baseline_datasets\stats_baseline.csv` → **somente_auditoria** (0, 9)
            
        - `outputs\baseline_datasets\stats_offbaseline.csv` → **somente_auditoria** (0, 9)
            
        - `outputs\baseline_mask.csv` → **backup_e_normalizacao** (1, 1)
            
        - `outputs\eda\matriz_correlacao_20250809_115344.csv` → **backup_e_normalizacao** (149, 1)
            
        - `outputs\pedra\A1_SECONDARIES_FOR_PEDRA_MODEL.csv` → **backup_e_normalizacao** (4, 6)
            
        - `outputs\pedra\A1_SECONDARIES_REFS.csv` → **somente_auditoria** (0, 6)
            
    - _Obs.: ver o inventário para a lista completa._
        
3. **Tratamento das “não mapeadas”**
    
    - Coletas e propostas geradas a partir do inventário; novas entradas foram **anexadas** ao dicionário com **backup simples**: `features_padronizadas_old.csv`.
        
    - **Higienização adicional (executada na sessão de trabalho):** removidos itens não relacionados a processo de combustão/fornalha (placeholders e administrativos) e **completadas dimensões** quando vazias (regras por sufixos e sentido do texto).
        
    - **Status:** artefatos dessa higienização estão prontos para replicação controlada no diretório oficial.
        

---

## 📂 Arquivos de referência

- Dicionário canônico:  
    `C:\Users\wilso\MBA_EMPREENDEDORISMO\3AGD\A1_LOCAL\features_padronizadas.csv`
    
- Inventário de auditoria:  
    `C:\Users\wilso\MBA_EMPREENDEDORISMO\3AGD\A1_LOCAL\outputs\auditoria_normalizacao_features.csv`
    
- Markdown com `nome_pt` incluído:  
    `C:\Users\wilso\MBA_EMPREENDEDORISMO\3AGD\A1_LOCAL\NOME DAS VARIAVEIS E SEUS SIGNIFICADOS_COM_NOME_PT.md`
    

---

## ⚠️ Pendências e riscos

- Há fontes com **colunas não mapeadas** (por ex.: +4, +16) que devem ser **incorporadas ao dicionário** ou **descartadas**.
    
- A versão “limpa” (apenas variáveis de processo) do dicionário foi gerada na sessão; requer **replicação consciente** para o caminho oficial, evitando divergência.
    

---

## 🔜 Próximas ações (imediatas)

1. **Replicar** o dicionário “limpo” (apenas processo) para `A1_LOCAL\features_padronizadas.csv` **após validação**.
    
2. **Reexecutar a normalização** com backup `_old.csv` para atualizar cabeçalhos em todas as fontes.
    
3. **Regerar o inventário** e confirmar que `data\curated\*` está 100% mapeado.
    
4. **Registrar** o novo SHA256 do dicionário neste histórico a cada alteração aprovada.
    

---

## ✅ Estado de controle

- **Backup de fontes:** `*_old.csv` criado automaticamente antes de qualquer alteração.
    
- **Ponto de retomada oficial:** este checkpoint.
    

---