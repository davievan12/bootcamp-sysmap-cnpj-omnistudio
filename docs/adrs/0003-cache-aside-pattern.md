# ADR-0003: Cache-Aside Pattern para Consulta CNPJ

- **Status:** Accepted
- **Data:** 2026-04-16
- **Autor:** Davi dos Santos Evangelista
- **Tags:** `performance`, `integracao`, `design-pattern`

## Contexto

A API ReceitaWS oferece duas limitações críticas no tier gratuito:

1. **Rate limit:** 3 requisições por minuto
2. **Latência:** 2–5 s por chamada (depende da disponibilidade da Receita Federal)

A persona Carlos ([ADR-0001](0001-persona-driven-design.md)) visita 8–12 empresas por dia. Muitas dessas visitas são **re-visitas** a clientes já cadastrados. Se cada consulta sempre chamar a ReceitaWS, temos três problemas:

- Estouro potencial do rate limit em alguns minutos de uso
- Latência desnecessária (2–5 s) para dados que já estão no Salesforce
- Risco de inconsistência se a API ficar indisponível

## Decisão

Implementar o **Cache-Aside Pattern** (também conhecido como *Lazy Loading*) na Integration Procedure `cltSvt_IP_GetAccountByCnpj`, usando o próprio Salesforce como cache.

### Fluxo

```
1. LWC chama IP com cnpj como input
2. IP executa Data Mapper Extract filtrando Account por CNPJ__c
3. Se Account EXISTE:
     → retorna dados existentes (latência <200 ms)
     → LWC oferece CTA opcional: "🔄 Verificar atualizações na Receita"
     → Se usuário optar, IP é invocada novamente com flag forceApi=true
4. Se Account NÃO existe:
     → IP executa HTTP Action para ReceitaWS
     → recebe payload, executa Data Mapper Load (upsert)
     → atualiza campo Last_API_Sync__c
     → retorna dados ao LWC
```

### Invalidação de cache

Cache-aside não tem TTL automático. A invalidação é **explícita pelo usuário**: Carlos decide se quer atualizar via API. Isso respeita a autonomia dele e evita chamadas desnecessárias.

Alternativamente, poderia ser implementada invalidação automática baseada em `Last_API_Sync__c > X dias`, mas foi descartado para o escopo deste projeto (ver seção "Alternativas").

## Consequências

### Positivas

1. **Redução drástica de chamadas à API externa** — revisitas não consomem cota da ReceitaWS
2. **Latência sub-segundo** para Accounts já cadastrados
3. **Graceful degradation** — se ReceitaWS cair, leitura de Accounts existentes continua funcionando
4. **Alinhamento com Well-Architected → Adaptable → Resilient** — sistema continua operando mesmo com falha de dependência externa
5. **Controle de custo** — se no futuro a ReceitaWS virar paga, cada hit no cache é uma chamada economizada

### Negativas

1. **Complexidade adicional na IP** — Conditional Branching é 1 elemento a mais que fluxo linear
2. **Dados potencialmente desatualizados** — se o Carlos confiar só no cache, pode ver dado antigo. Mitigação: UX deixa claro que há CTA de refresh disponível
3. **Sem invalidação automática** — não detecta mudança na Receita proativamente

## Alternativas consideradas

| Alternativa | Pro | Contra | Veredito |
|---|---|---|---|
| **Sempre chamar ReceitaWS** (no cache) | Simples. Dado sempre fresh | Estoura rate limit. Latência alta. Quebra se API cair | Rejeitado |
| **Cache-aside** (esta decisão) | Balanço performance/freshness. Graceful degradation | Complexidade leve na IP | **Escolhido** |
| **Write-through cache** | Dado sempre sincronizado | Complexidade alta. ReceitaWS não suporta callbacks. Não aplicável | Rejeitado |
| **TTL automático (ex: 30 dias)** | Refresh periódico sem intervenção | Scheduled Apex necessário → quebra [ADR-0002](0002-100-omnistudio-zero-apex.md). Fora de escopo | Rejeitado |

## Referências

- Microsoft Learn. *Cache-Aside Pattern*. https://learn.microsoft.com/en-us/azure/architecture/patterns/cache-aside
- Fowler, Martin. *Patterns of Enterprise Application Architecture*. Addison-Wesley, 2002 — capítulo sobre Object-Relational Structural Patterns.
- Salesforce Architects. *Well-Architected: Adaptable → Resilient*. https://architect.salesforce.com/docs/architect/well-architected/adaptable/resilient/overview
- ReceitaWS. *Limites do plano gratuito*. https://receitaws.com.br/api
