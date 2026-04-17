# ADR-0002: 100% OmniStudio, Zero Apex

- **Status:** Accepted
- **Data:** 2026-04-16
- **Autor:** Davi dos Santos Evangelista
- **Tags:** `arquitetura`, `omnistudio`, `apex`, `config-over-code`

## Contexto

O briefing do projeto (seção 4 do PDF) lista Apex como um dos conceitos avaliados, sugerindo uma classe Apex `@AuraEnabled` para fazer callout à ReceitaWS. A seção 5 do briefing trata OmniStudio em separado, como se fossem trilhas alternativas.

No entanto, a página 25 do mesmo briefing contém um requisito crítico ao tratar do LWC + OmniStudio:

> *"Integration procedure deve buscar na API ReceitaWS as informações da empresa. Data load deve ser utilizado para criar ou atualizar accounts existentes."*

Esse requisito específico, combinado com o conceito **"Config over Code"** do Well-Architected Framework (pilar **Easy → Automated**), me obrigou a uma decisão arquitetural mais escalável: **eliminar Apex completamente** e fazer 100% do trabalho em OmniStudio.

## Decisão

Implementar a solução **sem nenhuma linha de Apex**. Toda a lógica de callout, persistência, tratamento de erro e orquestração fica em OmniStudio:

- **HTTP callout** → `HTTP Action` dentro da Integration Procedure 
- **Persistência** → Data Mapper Load com upsert por External ID (declarativo)
- **Cache/lookup** → Data Mapper Extract (declarativo)
- **Lógica condicional** → Conditional Branching na IP (declarativo)
- **Validação de DV do CNPJ** → JavaScript no LWC (client-side, instantâneo)

## Consequências

### Positivas

1. **Menor TCO de manutenção** — qualquer admin Salesforce consegue ajustar IP/DM/OmniScript sem skills de Apex. Config over code é princípio explícito do Well-Architected.
2. **Zero governor limits de CPU/Heap no server-side customizado** — OmniStudio runtime é otimizado pela Salesforce e não consome limites de Apex.
3. **Testabilidade sem mock callouts** — elimina a necessidade de escrever `HttpCalloutMock` e classes de teste com 75% de cobertura.
4. **Reutilização** — a Integration Procedure é invocável por qualquer outro OmniScript, FlexCard, Flow ou Platform Event.
5. **Deploy mais limpo** — metadata OmniStudio via SFDX, sem dependências de classes Apex ou test classes.
6. **Alinhamento com pilar "Easy → Automated"** do Well-Architected: automação declarativa escala melhor que código customizado.

### Negativas

2. **Debugging OmniStudio é menos robusto** que debug de Apex.

### Trade-off assumido

O briefing tem dois requisitos aparentemente conflitantes:
- Requisito 4 (Apex): *"Classe Apex `@AuraEnabled` que chama ReceitaWS"*
- Requisito 5 (OmniStudio, p. 25): *"Integration Procedure deve buscar na API ReceitaWS"*

**Esta decisão prioriza o requisito 5** porque (a) é mais específico e recente, (b) alinha-se ao movimento da plataforma Salesforce para declarativo, e (c) demonstra pensamento arquitetural maduro — **saber quando ultilizar Core ou OMNISTUDIO é uma skill esperada para se trabalar com SalesForce**.

## Alternativas consideradas

| Alternativa | Pro | Contra | Veredito |
|---|---|---|---|
| **Híbrido** (Apex para callout + OmniStudio para UI) | Atende literalmente os requisitos 4 e 5 | Dupla superfície de manutenção. Adiciona governor limits. Complexidade sem valor real para o Trabalho | Rejeitado |
| **100% Apex + Screen Flow** (requisito 2) | Mais tradicional, também funcional | Não é a melhor stack para o projeto considerando alguns critérios supracitados. | Rejeitado |
| **OmniStudio + LWC no Front-END** (esta decisão) | Config over code. Reutilizável. Sem governor limits de Apex. | Perde pontos se avaliador pontuar rigidez ao requisito 4 | **Escolhido** |

## Referências

- Salesforce Architects. *Well-Architected: Easy → Automated*. https://architect.salesforce.com/docs/architect/well-architected/easy/automated/overview
- Salesforce Help. *OmniStudio Integration Procedures*. https://help.salesforce.com/s/articleView?id=sf.os_integration_procedures.htm
- Salesforce Help. *OmniStudio Data Mappers*. https://help.salesforce.com/s/articleView?id=sf.os_data_mappers_27964.htm
