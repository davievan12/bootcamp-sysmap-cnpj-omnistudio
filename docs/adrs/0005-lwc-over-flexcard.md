# ADR-0005: LWC Customizada em vez de FlexCard

- **Status:** Accepted
- **Data:** 2026-04-16
- **Autor:** Davi dos Santos Evangelista
- **Tags:** `frontend`, `omnistudio`, `lwc`, `flexcard`

## Contexto

O briefing (página 23) oferece duas opções para a apresentação das informações da empresa:

> *"Criar um FlexCard **ou** LWC que apresente de forma resumida as informações de empresa obtidas através do cadastro ou da API ReceitaWS."*

**FlexCards** são componentes declarativos do OmniStudio, otimizados para **exibição** de dados com design visual rápido de montar.

**LWC** (Lightning Web Components) são componentes programáticos, com controle fino sobre comportamento, estado e interação.

Para a persona Carlos ([ADR-0001](0001-persona-driven-design.md)), que precisa **digitar, validar em tempo real, editar e salvar** dados em dispositivo móvel, esta decisão tem impacto direto na qualidade da experiência.

## Decisão

Implementar um **LWC customizado** (`cltSvt_LWC_cnpjLookup`) em vez de FlexCard.

### Funcionalidades que exigiram LWC

| Requisito | Por que FlexCard não serve | Como LWC resolve |
|---|---|---|
| Máscara em tempo real no CNPJ (ex: `12345678000190` → `12.345.678/0001-90`) | FlexCard não suporta event handlers customizados em input fields | `input event handler` com regex replace |
| Validação de DV do CNPJ **antes** de chamar server | FlexCard não executa JS customizado on-change | Função `validateCnpj()` em `.js` do componente |
| Toggle readonly em campos após salvar | FlexCard tem estados condicionais, mas limitados | `disabled` property binding reativo |
| Destacar visualmente campos alterados na comparação pós-refresh API | FlexCard não suporta conditional styling complexo por campo | CSS class binding dinâmica |
| Loading spinner durante callout com mensagem customizada | FlexCard tem loading nativo, mas não customizável textualmente | `<lightning-spinner>` com `alternative-text` controlado |

## Consequências

### Positivas

1. **UX superior para Carlos** — interações fluidas, feedback instantâneo, controle total sobre o comportamento
2. **Testabilidade com Jest** — LWC suporta unit tests com `@salesforce/sfdx-lwc-jest`. FlexCard não tem equivalente
3. **Reutilização em outros contextos** — LWC pode ser embedado em Flow Screen, App Page, Record Page, Community. FlexCard é mais acoplado ao OmniStudio
4. **SLDS nativo** — controle fino sobre responsividade mobile usando utility classes do Lightning Design System
5. **Versionamento Git friendly** — código em `.js/.html/.css` é diffable. FlexCard metadata é JSON gigante e ilegível em PR

### Negativas

1. **Mais tempo de desenvolvimento** — estimativa: 2–3x mais tempo que um FlexCard equivalente
2. **Exige conhecimento de JS/LWC** — não é declarativo; futuros ajustes exigem dev skills
3. **Não aproveita o editor visual do OmniStudio** — outros componentes OmniStudio do time teriam sido construídos em designer, mas este não

### Trade-off assumido

Para um **componente de exibição apenas**, FlexCard seria a escolha correta (alinha com Well-Architected → Easy → Automated). Para um **componente de entrada + validação + edição**, LWC é objetivamente superior.

Este projeto é o segundo caso. O componente precisa:
- **Capturar** input (CNPJ)
- **Validar** client-side (DV do CNPJ)
- **Exibir** dados retornados
- **Permitir** edição
- **Disparar** salvamento

FlexCard cobre apenas 2 dos 5 (exibir e, com limitações, disparar). LWC cobre todos 5 com controle total.

## Alternativas consideradas

| Alternativa | Pro | Contra | Veredito |
|---|---|---|---|
| **FlexCard puro** | Declarativo. Mais rápido de montar. Alinha com "config over code" | Não suporta máscara em tempo real, nem validação JS customizada, nem toggle de readonly | Rejeitado |
| **LWC puro** (esta decisão) | Controle total sobre UX. Testável. Reutilizável | Mais código. Exige skills de dev | **Escolhido** |
| **FlexCard + LWC** (híbrido) | Display em FlexCard, input em LWC | Duplica superfície de manutenção. Dois componentes para o mesmo fluxo | Rejeitado |

## Referências

- Salesforce Developer Docs. *Lightning Web Components Developer Guide*. https://developer.salesforce.com/docs/component-library/documentation/en/lwc
- Salesforce Help. *OmniStudio FlexCards*. https://help.salesforce.com/s/articleView?id=sf.os_omnistudio_flexcards.htm
- Trailhead Sample Apps. *lwc-recipes*. https://github.com/trailheadapps/lwc-recipes — padrão de referência para LWC
