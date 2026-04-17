# ADR-0001: Persona-Driven Design

- **Status:** Accepted
- **Data:** 2026-04-16
- **Autor:** Davi dos Santos Evangelista
- **Tags:** `ux`, `produto`, `metodologia`

## Contexto

O briefing do bootcamp descreve os requisitos técnicos e funcionais do projeto, mas não prescreve como priorizar trade-offs. Sem um norte claro, decisões de UX e arquitetura podem ser tomadas por conveniência técnica em vez de valor real para o usuário final — um anti-pattern explicitamente identificado pelo Well-Architected Framework no pilar **Easy → Intentional**.

Um feedback recorrente mencionado nas aulas do bootcamp foi o caso de uma aluna (Juliana) que construiu uma solução OmniStudio tecnicamente correta, mas **não conseguiu comunicar valor** no vídeo porque falou apenas de tecnologia, sem conectar a decisões de produto.

## Decisão

Adotar uma **persona-driven design** para guiar todas as decisões de UX e arquitetura. Antes de configurar o primeiro componente, foi definida a persona **Carlos**, e cada decisão subsequente deve responder a uma necessidade específica dele.

### Persona de referência

> **Carlos, 32 anos — Representante de Vendas Externo B2B**
>
> - Trabalha 70% mobile (iPad/celular em campo), 30% desktop
> - Visita 8–12 empresas por dia
> - Cadastra manualmente cada novo cliente → 10–15 min de retrabalho
> - Frequentemente digita CNPJ errado ou cria duplicatas
> - Precisa cadastrar ou atualizar em <2 min, sem erro, em qualquer dispositivo

### 7 Regras de Produto derivadas

| # | Regra | Justificativa |
|---|---|---|
| 1 | Buscar **primeiro no Salesforce** (cache) | Carlos faz revisita. Se cliente já existe, mostrar em <1 s sem consumir cota de API |
| 2 | Permitir editar todos os campos **exceto CNPJ** | Carlos corrige telefone/endereço em campo. CNPJ é chave única — readonly após salvar |
| 3 | Quando CNPJ existe, **perguntar** antes de chamar API | Não fazer Carlos esperar 3–5 s desnecessariamente. Oferecer refresh opt-in |
| 4 | Quando atualizar, **mostrar o que mudou** | Carlos quer ver diferenças (telefone? endereço?), não substituir tudo cegamente |
| 5 | **Mobile-first sempre** | SLDS responsivo, inputs grandes, touch targets ≥44px, sem scroll horizontal |
| 6 | **Feedback instantâneo** (<300 ms) | Validação de DV em JS, spinner em toda chamada, toast em toda ação |
| 7 | **Botão Voltar sempre disponível** | Se digitar CNPJ errado, Carlos não fica preso |

## Consequências

### Positivas

- **Alinhamento com Well-Architected (Easy → Intentional):** o framework Salesforce exige que decisões sejam intencionais e alinhadas ao valor de negócio
- **Coerência:** elimina decisões conflitantes. Se uma escolha técnica favorece o desenvolvedor mas prejudica o Carlos, ela é descartada

### Negativas

- **Risco de over-engineering se persona virar dogma:** mitigação → persona é referência, não lei. Decisões técnicas objetivas (ex: naming, security) não precisam citar Carlos

## Alternativas consideradas

1. **Requirements-driven puro** — seguir o PDF ao pé da letra sem persona. Rejeitado: gera soluções tecnicamente corretas mas com UX medíocre, e sem narrativa forte para a venda do produto.
2. **Data-driven design** — basear decisões em métricas. Rejeitado: projeto de bootcamp não tem baseline histórico de uso.

## Referências

- Salesforce Architects. *Well-Architected Framework: Easy → Intentional*. https://architect.salesforce.com/docs/architect/well-architected/easy/intentional/overview
- Cooper, Alan. *The Inmates Are Running the Asylum*. 1999. Fundação do conceito de persona em UX.
