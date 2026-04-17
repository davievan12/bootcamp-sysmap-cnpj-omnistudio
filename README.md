# Cadastro CNPJ via ReceitaWS — OmniStudio + LWC

> Projeto final do **Bootcamp Sysmap Trainee de Excelência Salesforce — 7ª edição** (Março/2026)
> Autor: **Davi dos Santos Evangelista** · [davievangelista@gmail.com](mailto:davievangelista@gmail.com)

[![Platform](https://img.shields.io/badge/platform-Salesforce-00A1E0?logo=salesforce&logoColor=white)](https://developer.salesforce.com/)
[![Stack](https://img.shields.io/badge/stack-OmniStudio%20%2B%20LWC-1589EE)](https://help.salesforce.com/s/articleView?id=sf.os_omnistudio.htm)
[![Apex](https://img.shields.io/badge/Apex-0%20lines-success)]()
[![Well-Architected](https://img.shields.io/badge/Salesforce-Well--Architected-blue)](https://architect.salesforce.com/docs/architect/well-architected/guide/overview.html)

---

## O que é isto

Uma automação de cadastro de Accounts (pessoa jurídica) no Salesforce a partir do CNPJ. O vendedor digita o CNPJ, o sistema consulta a ReceitaWS e o ViaCEP em tempo real, pré-preenche os campos, permite revisão e salva — tudo em cerca de 30 segundos.

A solução foi implementada com **OmniStudio** (Integration Procedures, Data Mappers, OmniScript) e um **Lightning Web Component** customizado para entrada e validação. **Nenhuma linha de Apex.**

A arquitetura segue os princípios do [Salesforce Well-Architected Framework](https://architect.salesforce.com/docs/architect/well-architected/guide/overview.html) — *Trusted, Easy, Adaptable* — e cada decisão significativa está registrada como um [ADR](docs/adrs/) no formato proposto por [Michael Nygard](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions).

---

## Índice

- [Problema](#problema)
- [Quem é o Carlos](#quem-é-o-carlos)
- [Arquitetura](#arquitetura)
- [Alinhamento com o Well-Architected Framework](#alinhamento-com-o-well-architected-framework)
- [Componentes entregues](#componentes-entregues)
- [Fluxo](#fluxo)
- [Modelo de segurança](#modelo-de-segurança)
- [Tratamento de erros](#tratamento-de-erros)
- [Decisões arquiteturais (ADRs)](#decisões-arquiteturais-adrs)
- [Uso de IA](#uso-de-ia)
- [Acesso à Trial Org](#acesso-à-trial-org)
- [Referências](#referências)

---

## Problema

Representantes de vendas externos B2B cadastravam manualmente cada cliente novo, gastando 10 a 15 minutos por registro. Isso gerava três problemas:

- **Erros de digitação** em CNPJ, endereço e telefone
- **Duplicatas**, porque o representante não tinha como saber se já existia um registro
- **Tempo perdido** em campo, entre uma visita e outra

A solução entregue reduz esse tempo para aproximadamente 30 segundos ponta a ponta, elimina duplicatas por design (upsert via External ID) e valida o CNPJ no próprio dispositivo antes de chamar qualquer API.

---

## Quem é o Carlos

Antes de configurar o primeiro componente, defini uma persona de referência. A ideia é simples: toda decisão técnica deste projeto precisa responder a uma necessidade real de alguém — não à conveniência do desenvolvedor.

**Carlos tem 32 anos e é representante de vendas externo B2B.** Trabalha 70% do tempo no iPad ou no celular, visita de 8 a 12 empresas por dia. Precisa cadastrar ou atualizar clientes em menos de 2 minutos, sem erro, de qualquer dispositivo.

As decisões de UX que seguem foram derivadas dessa persona: busca no Salesforce antes de chamar a API externa (porque Carlos faz revisita), CNPJ readonly após salvar (porque é chave única), validação do dígito verificador em JavaScript antes de qualquer chamada server (porque Carlos não pode esperar 3 segundos para descobrir que digitou um CNPJ inválido), mobile-first em todo o fluxo.

A persona completa e as 7 regras derivadas estão em [ADR-0001](docs/adrs/0001-persona-driven-design.md).

---

## Arquitetura

```
                     Carlos (iPad em campo)
                              │
                     ┌────────▼─────────┐
                     │    OmniScript    │
                     │  BuscaAccounts-  │
                     │   ReceitaWS      │
                     │  (GuidedFlow,    │
                     │     pt-BR)       │
                     └────────┬─────────┘
                              │
                     ┌────────▼─────────┐
                     │  LWC cnpjLookup  │
                     │                  │
                     │ • Máscara        │
                     │   CNPJ/CEP/Phone │
                     │ • Validação DV   │
                     │ • 13 campos,     │
                     │   3 seções       │
                     │ • Dropdowns      │
                     │   Status/UF      │
                     └────────┬─────────┘
                              │ omniRemoteCall
                     ┌────────▼─────────┐
                     │ IP GetAccount-   │
                     │   ByCnpj         │
                     │ (cache-aside)    │
                     └───┬─────────┬────┘
                         │         │
              cache hit  │         │  cache miss
                         │         │
        ┌────────────────▼┐      ┌─▼───────────────┐
        │ DM Extract      │      │ HTTP Action     │
        │ AccountByCnpj   │      │ → ReceitaWS API │
        └────────┬────────┘      │ (endpoint via   │
                 │               │ Custom Metadata)│
                 │               └─┬───────────────┘
                 │                 │
                 │       ┌─────────┘
                 │       │
                 │  ┌────▼──────────┐
                 │  │ IP Consulta-  │
                 │  │   Cep         │
                 │  │ → ViaCEP API  │
                 │  └────┬──────────┘
                 │       │
                 └───┬───┘
                     │
            ┌────────▼─────────┐
            │ DM Load Account- │
            │ Upsert           │
            │ (External ID:    │
            │      CNPJ__c)    │
            └────────┬─────────┘
                     │
            ┌────────▼─────────┐
            │  Account         │
            │  (Salesforce)    │
            └──────────────────┘
```

**Como ler o diagrama.** Quando Carlos digita um CNPJ, a Integration Procedure procura primeiro no Salesforce. Se o Account já existe, retorna em menos de 1 segundo sem consumir cota da API externa. Se não existe, vai na ReceitaWS, complementa o endereço via ViaCEP quando necessário, e persiste com upsert — eliminando duplicatas por design da plataforma, não por código.

Diagramas de sequência detalhados em [docs/architecture.md](docs/architecture.md).

---

## Alinhamento com o Well-Architected Framework

A Salesforce publica o [Well-Architected Framework](https://architect.salesforce.com/docs/architect/well-architected/guide/overview.html) como o guia oficial para projetar soluções saudáveis na plataforma. O framework é organizado em três pilares — Trusted, Easy, Adaptable — e cada um se desdobra em comportamentos observáveis.

Este projeto foi desenhado com os três pilares como checkpoints explícitos:

**Trusted.** A solução protege dados e usuários. O modelo de segurança tem quatro camadas independentes: OWD Account como Private, Profile com menor privilégio, Permission Set com FLS granular (o campo `Last_API_Sync__c` é readonly para preservar a auditoria), e Sharing Rule explícita entre roles. Os erros da API externa são tratados em quatro cenários distintos, cada um com mensagem específica para o usuário, sem nunca vazar stack trace.

**Easy.** A solução entrega valor rápido. Zero linhas de Apex significa que qualquer admin consegue ajustar a lógica sem precisar de skills de desenvolvedor. O endpoint da ReceitaWS está em Custom Metadata Type, não hardcoded — trocar por outra API (SERPRO, Casa dos Dados) não exige deploy. O LWC aplica máscaras em tempo real e dá feedback em menos de 300 ms, respeitando o tempo de atenção do Carlos.

**Adaptable.** A solução evolui com o negócio. O padrão cache-aside reduz a dependência da API externa (cujo plano gratuito permite apenas 3 requisições por minuto) e mantém a leitura de dados funcional mesmo se a ReceitaWS ficar indisponível. A Integration Procedure é reutilizável em qualquer OmniScript, FlexCard ou Flow futuro. Profile e Permission Set são desacoplados: habilitar o cadastro CNPJ para uma nova persona é uma atribuição, não uma reconfiguração.

---

## Componentes entregues

Todos os componentes customizados usam o prefixo `cltSvt_` como proxy de namespace organizacional — decisão detalhada em [ADR-0006](docs/adrs/0006-naming-convention.md).

### OmniStudio

| Componente | API Name | Função |
|---|---|---|
| Integration Procedure | `cltSvt_IP_GetAccountByCnpj` | Orquestra o cache-aside: busca o Salesforce primeiro; se não achar, chama a ReceitaWS e persiste |
| Integration Procedure | `cltSvt_IP_AtualizarAccount` | Executa upsert do Account a partir do payload revisado pelo usuário |
| Integration Procedure | `cltSvt_IP_ConsultaCep` | Consulta a API ViaCEP (`GET /ws/{cep}/json/`) para enriquecer o endereço quando o CEP é alterado manualmente pelo usuário |
| Data Mapper Extract | `cltSvt_DM_Extract_AccountByCnpj` | Busca Account existente filtrando por `CNPJ__c` |
| Data Mapper Load | `cltSvt_DM_Load_AccountUpsert` | Upsert atômico usando `CNPJ__c` como External ID |
| OmniScript | `cltSvt_OS_BuscaAccountsReceitaWS` | GuidedFlow em português, SubType TicketRegistration, guia o usuário pelos passos |

### Lightning Web Component

| Componente | API Name | Função |
|---|---|---|
| LWC | `cltSvt_LWC_cnpjLookup` | Interface principal: 13 campos distribuídos em 3 seções, máscaras automáticas para CNPJ/CEP/telefone, dropdowns para Status Cadastral e UF, validação de dígito verificador em JavaScript antes de chamar o server |

### Metadata

| Tipo | Nome | Função |
|---|---|---|
| Custom Metadata Type | `cltSvt_ReceitaWS_Config__mdt` | Armazena endpoint, timeout e rate limit — evita hardcode |
| Custom Fields (Account) | `CNPJ__c`, `Status_Cadastral__c`, `Last_API_Sync__c` | Extensões do modelo de negócio; CNPJ com External ID + Unique |
| Remote Site Setting | `ReceitaWS` | Habilita callout para `https://www.receitaws.com.br` |
| Remote Site Setting | `ViaCEP` | Habilita callout para `https://viacep.com.br` |

### Segurança

| Tipo | Nome | Função |
|---|---|---|
| OWD | Account → Private | Base restritiva por default |
| Profile | `Representante de Vendas` | Clone do Standard User, aplicando menor privilégio |
| Permission Set | `Cadastro CNPJ Representante` | FLS granular nos custom fields, acesso ao Custom Metadata Type |
| Role Hierarchy | `Sales Manager` → `Sales Rep` | Pré-requisito da Sharing Rule |
| Sharing Rule | `Compartilhamento Account Gerente` | Compartilha Accounts do Sales Rep com o Sales Manager, Read/Write |

---

## Fluxo

```
1. Carlos abre o OmniScript no iPad.
2. LWC aplica máscara em tempo real no campo CNPJ.
3. JavaScript valida o dígito verificador antes de qualquer chamada server.
   Se o DV estiver errado, mensagem imediata e botão Buscar desabilitado.

4. Carlos clica Buscar.
5. LWC invoca cltSvt_IP_GetAccountByCnpj via omniRemoteCall.

6. A Integration Procedure executa o cache-aside:
   - Data Mapper Extract busca Account por CNPJ__c.
     - Existe: retorna dados em <1s, oferece CTA opcional de refresh na API.
     - Não existe: prossegue para o HTTP Action.

7. HTTP Action chama a ReceitaWS. Endpoint vem do Custom Metadata Type.
   - status OK: prossegue.
   - status ERROR: retorna erro granular ao LWC.
   - timeout ou HTTP 429: retorna erro específico.

8. Se Carlos alterar o CEP manualmente, IP ConsultaCep chama a ViaCEP e
   atualiza logradouro, bairro, cidade e UF.

9. Carlos revisa os dados. CNPJ fica readonly após a primeira carga.
10. Data Mapper Load executa upsert usando CNPJ__c como External ID —
    idempotência garantida pela plataforma, sem código.

11. Last_API_Sync__c é atualizado com o timestamp.
12. Toast de sucesso e navegação para o registro criado.
```

---

## Modelo de segurança

Quatro camadas independentes, seguindo o princípio de *defense in depth* descrito no [NIST SP 800-160](https://csrc.nist.gov/publications/detail/sp/800-160/vol-1/final).

**Camada 1 — OWD.** Account está configurado como Private. Por padrão, cada Account só é visível para o owner do registro. Sem esta camada, qualquer usuário com permissão de Read no objeto veria tudo.

**Camada 2 — Profile.** O Profile `Representante de Vendas` é um clone do Standard User com ajustes cirúrgicos. Accounts e Contacts têm Read/Create/Edit mas não Delete — Carlos é rep externo em campo, não deveria poder apagar cliente com um clique. Leads, Opportunities, Cases e Campaigns foram explicitamente desmarcados — estão fora do escopo do fluxo CNPJ, e o briefing do projeto (página 12) pede literalmente que usuários não tenham acesso a objetos desnecessários.

**Camada 3 — Permission Set.** O Permission Set `Cadastro CNPJ Representante` concentra a configuração específica do recurso: FLS dos custom fields (o campo `Last_API_Sync__c` é readonly, para que o usuário não possa adulterar manualmente um dado que deveria ser auditoria pura) e o acesso ao Custom Metadata Type. Essa separação entre Profile (quem o usuário é) e Permission Set (o que o usuário pode fazer no recurso) é o padrão moderno Salesforce — modular, auditável, reversível.

**Camada 4 — Sharing Rule.** A rule `Compartilhamento Account Gerente` compartilha Accounts criadas por membros da role `Sales Rep` com membros da role `Sales Manager`, Read/Write. Dá ao gerente a autonomia de corrigir um dado do subordinado sem depender do rep voltar ao sistema.

Detalhamento em [docs/security.md](docs/security.md).

---

## Tratamento de erros

A API ReceitaWS pode falhar de quatro formas. Cada uma tem tratamento específico na Integration Procedure, sem try/catch em Apex — porque não há Apex.

| Cenário | Detecção | Mensagem para o usuário |
|---|---|---|
| CNPJ com DV inválido | Validação em JavaScript no LWC | CNPJ inválido. Verifique os dígitos. |
| CNPJ não encontrado na Receita | Response com `status === "ERROR"` | Empresa não localizada na Receita Federal. |
| Rate limit (3 req/min no plano grátis) | HTTP 429 ou mensagem específica | Limite de consultas atingido. Aguarde um minuto. |
| Timeout ou erro de rede | Timeout superior a 30 s ou HTTP 5xx | Serviço temporariamente indisponível. Tente novamente. |

O princípio é *fail-safe defaults*: se a API cair, a leitura de Accounts existentes continua funcionando — Carlos pode editar manualmente, a funcionalidade degrada em vez de quebrar.

---

## Decisões arquiteturais (ADRs)

Decisões significativas estão registradas como ADRs seguindo o [template de Michael Nygard](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions), consolidado em [adr.github.io](https://adr.github.io/).

| Número | Decisão |
|---|---|
| [ADR-0001](docs/adrs/0001-persona-driven-design.md) | Persona-driven design com Carlos como referência |
| [ADR-0002](docs/adrs/0002-100-omnistudio-zero-apex.md) | OmniStudio + LWC, zero Apex |
| [ADR-0003](docs/adrs/0003-cache-aside-pattern.md) | Cache-aside pattern na consulta de CNPJ |
| [ADR-0004](docs/adrs/0004-external-id-upsert.md) | CNPJ como External ID para idempotência |
| [ADR-0005](docs/adrs/0005-lwc-over-flexcard.md) | LWC customizada em vez de FlexCard |
| [ADR-0006](docs/adrs/0006-naming-convention.md) | Prefixo `cltSvt_` como namespace organizacional |

---

Uso de IA
O briefing (página 8) solicita justificativa explícita sobre uso de IA. Usei Claude (Anthropic) nas seguintes frentes:

Debugging e criação de código — auxílio na identificação de erros, criação de trechos de código e fórmulas que implementei durante a realização do projeto.
Sugestões de mapeamento entre o JSON da ReceitaWS e os campos do Account no Data Mapper — validei contra o contrato real da API.
Estruturação de fórmulas complexas no Integration Procedure Designer.
Estruturação da documentação — draft inicial, que revisei tecnicamente e ajustei.

O que não foi feito com IA: a arquitetura da solução, as decisões de segurança, a configuração dos componentes OmniStudio (IP, Data Mappers, OmniScript). A IA foi consultada pontualmente para aumentar minha eficiência codando e no acesso aos comportamentos de elementos específicos da plataforma, mas o desenho da solução e toda a implementação foram execução própria.
O princípio que segui: IA acelera tarefas executáveis — geração de boilerplate, redação, estruturação. Não substitui pensamento arquitetural, mas alinhada ao conhecimento prévio é uma grande ferramenta de desenvolvimento, já que torna o processo consideravelmente mais rápido e produtivo. OmniStudio, em particular, é uma plataforma onde a IA ainda tem profundidade limitada, e o valor diferencial está no julgamento humano.

---

## Acesso à Trial Org

### A limitação de licenciamento

A Trial Org Communications Cloud (Enterprise Edition) disponibiliza apenas uma licença Salesforce full, consumida pelo usuário admin. Por essa restrição, não foi possível criar usuários nominais para os examinadores seguindo o pattern `email+.sales9` solicitado — as licenças em volume disponíveis na Trial (Chatter External, Identity) não dariam acesso funcional a Accounts, Contacts ou OmniStudio, inviabilizando a avaliação prática.

### Acesso pleno para avaliação

As credenciais do admin estão disponibilizadas no campo de mensagem do envio via Gupy.

| Parâmetro | Valor |
|---|---|
| URL | `https://sales9.my.salesforce.com` |
| Username | `davievangelist-7a1l@force.com` |
| Senha | fornecida no envio |
| Validade | até aproximadamente 14/05/2026 |

Em ambiente não-Trial, os examinadores receberiam licença Salesforce com Profile System Administrator — o "perfil Premium" do briefing.

---

## Referências

### Framework e padrões

- Salesforce Architects. **Well-Architected Framework.** https://architect.salesforce.com/docs/architect/well-architected/guide/overview.html
- Salesforce Architects. **Patterns & Anti-Patterns Library.** https://architect.salesforce.com/docs/architect/well-architected-tools/guide/patterns
- Nygard, Michael. **Documenting Architecture Decisions.** 2011. https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions
- ADR GitHub Organization. **Architectural Decision Records.** https://adr.github.io/
- NIST SP 800-160 Vol. 1. **Systems Security Engineering.** https://csrc.nist.gov/publications/detail/sp/800-160/vol-1/final
- Microsoft Azure Architecture Center. **Cache-Aside Pattern.** https://learn.microsoft.com/en-us/azure/architecture/patterns/cache-aside

### Documentação oficial Salesforce

- **OmniStudio Developer Guide.** https://help.salesforce.com/s/articleView?id=sf.os_omnistudio.htm
- **OmniStudio Integration Procedures.** https://help.salesforce.com/s/articleView?id=sf.os_integration_procedures.htm
- **OmniStudio Data Mappers.** https://help.salesforce.com/s/articleView?id=sf.os_data_mappers_27964.htm
- **Lightning Web Components Developer Guide.** https://developer.salesforce.com/docs/component-library/documentation/en/lwc
- **Salesforce Lightning Design System.** https://www.lightningdesignsystem.com/
- **Custom Metadata Types.** https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/custommetadatatypes_about.htm
- **External IDs and Upsert.** https://developer.salesforce.com/docs/atlas.en-us.api.meta/api/sforce_api_concepts_upsert.htm
- **Record-Level Security: Sharing Rules.** https://help.salesforce.com/s/articleView?id=sf.security_sharing_rules.htm

### Sample apps de referência (padrão trailheadapps)

- [trailheadapps/lwc-recipes](https://github.com/trailheadapps/lwc-recipes)
- [trailheadapps/dreamhouse-lwc](https://github.com/trailheadapps/dreamhouse-lwc)
- [trailheadapps/apex-recipes](https://github.com/trailheadapps/apex-recipes)

### APIs externas

- **ReceitaWS** — Consulta pública de CNPJ. https://receitaws.com.br/
- **ViaCEP** — Consulta pública de CEP. https://viacep.com.br/
