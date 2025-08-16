
## 1. Objetivo
Estabelecer uma linguagem e métrica únicas para avaliar e comparar taxas de desgaste de **refratário** e **metal** na caldeira CFB, aplicáveis tanto à linha *Physics-Based* quanto à linha *ML/DL*.

---

## 2. Definições Fundamentais

### Wr – *Wear rate – refractory*
Taxa de desgaste do **refratário**, representando a velocidade de perda de material ou degradação estrutural do revestimento refratário.

### Wm – *Wear rate – metal*
Taxa de desgaste do **metal**, representando a velocidade de perda de material ou degradação das partes metálicas (erosão, corrosão, abrasão).

---

## 3. Normalização por Referência

Para permitir comparabilidade e padronização, as taxas são normalizadas por referências de operação ideal:

$$
Wr_{idx} = \frac{Wr}{Wr_{ref}}
$$

$$
Wm_{idx} = \frac{Wm}{Wm_{ref}}
$$

Onde:

- **Wr_ref** → taxa de desgaste de referência para refratário.
- **Wm_ref** → taxa de desgaste de referência para metal.

**Benefícios da normalização:**
- Coloca variáveis em escala comparável, independente da unidade original.
- Permite análise de desvios em relação à operação projetada/ótima.
- Facilita integração com diferentes modelos e períodos operacionais.

---

## 4. Situação Atual (MVP)

- **Não há medição direta** de desgaste acumulado (mm ou kg) no momento.
- Avaliaremos **variações nas taxas** (`Wr`, `Wm`) e não o desgaste físico total.
- As referências (**Wr_ref**, **Wm_ref**) serão definidas a partir de condições estáveis de operação.
- Quando medições reais estiverem disponíveis, as referências serão recalibradas.

---

## 5. Aplicações

### Physics-Based
- Baseia-se em proxies físicos mínimos:
  - `v_proxy` → carga global (média z-score de fluxos primários).
  - `delta_proxy` → comportamento fluidodinâmico (média z-score de ΔP).
  - `tau_*` → regimes térmicos regionais (média z-score de temperaturas por região).
- Modela e calibra a relação entre condições operacionais e `Wr_idx`/`Wm_idx`.

### ML/DL
- Utiliza variáveis primárias + secundárias para treinar modelos preditivos.
- `Wr_idx` e `Wm_idx` funcionam como *targets*.
- Captura padrões complexos, mantendo alinhamento com as mesmas métricas do Physics-Based.

---

## 6. Essência do Conceito

A métrica **Wr/Wr_ref** e **Wm/Wm_ref** é a **língua comum** entre Physics-Based e ML/DL.

- Garante que ambos os fluxos compartilhem a mesma escala e objetivo.
- Permite validação cruzada: avanços em um fluxo podem ser auditados pelo outro.
- Facilita a transição do MVP (taxas relativas) para um modelo calibrado com desgaste real.

---

## 7. Quadro-Resumo

| Componente          | Símbolo       | Descrição                                               | Referência          | Métrica Normalizada |
|--------------------|--------------|-------------------------------------------------------|--------------------|--------------------|
| Desgaste Refratário | Wr           | Taxa de desgaste do refratário                        | Wr_ref (ideal)     | Wr / Wr_ref        |
| Desgaste Metal      | Wm           | Taxa de desgaste do metal                             | Wm_ref (ideal)     | Wm / Wm_ref        |

---

## 8. Observações

- `Wr_ref` e `Wm_ref` podem ser definidos por condições operacionais estáveis, históricas ou de projeto.
- A unidade de `Wr` e `Wm` (mm/ano, kg/dia, etc.) não interfere na razão, desde que Wr e Wr_ref compartilhem a mesma unidade (idem para Wm).
- O conceito será mantido e aplicado de forma idêntica nas duas linhas do projeto.
