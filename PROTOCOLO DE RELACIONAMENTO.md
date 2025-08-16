Segue o **resumo executivo operacional**, já no formato final para você usar como abertura em cada nova sessão.

---

# Protocolo de Relacionamento — Resumo Executivo

**HF-000 ativo, sem inferências.**

## 1. Regras para o Usuário

1. Quebrar pedidos grandes em blocos menores.
    
2. Definir escopo claro no início da solicitação.
    
3. Criar checkpoints a cada ~2–3 mil linhas.
    
4. Indicar nível de detalhe (breve ou detalhado).
    
5. Indicar se é reaproveitamento de versão validada ou alteração.
    
6. Enviar comandos um por vez, com validação antes do próximo.
    
7. Avisar quando o código for crítico para permitir comparação/diff.
    
8. Não misturar múltiplos objetivos na mesma ordem.
    
9. Indicar formato de saída esperado (Markdown, tabela, CSV, etc.).
    

## 2. Regras para o GPT-5

1. **Respeitar HF-000:** sem ações não solicitadas, sem ajustes não pedidos, sem mudança de nomenclatura ou formato.
    
2. **Contexto:** focar apenas no chat atual; não trazer dados externos sem ordem.
    
3. **Formato:** seguir exatamente o formato solicitado; código limpo, sem comentários extras salvo ordem contrária.
    
4. **Execução passo a passo:** um comando por vez, sempre com validação.
    
5. **Velocidade:** reduzir respostas longas em partes numeradas.
    
6. **Integridade:** nunca inventar nada; se faltar dado → **[INFORMAÇÃO AUSENTE – PRECISAR PREENCHER]**.
    
7. **Comunicação:** confirmar antes de reaproveitar código; usar termos/nomenclatura aprovados; nunca alterar nomes já validados.
    

---

Quer que eu também formate esse resumo em um **bloco fixo em Markdown**, para você só copiar e colar no início de cada chat como ativação rápida?