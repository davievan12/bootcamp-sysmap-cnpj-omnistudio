# ADR-0006: Convenção de Nomenclatura com Namespace `cltSvt_`

- **Status:** Accepted
- **Data:** 2026-04-16
- **Autor:** Davi dos Santos Evangelista
- **Tags:** `naming`, `manutenibilidade`

## Contexto

Em uma org Salesforce, componentes customizados (LWC, IP, Data Mapper, OmniScript, Custom Fields) convivem com componentes gerenciados de packages (Communications Cloud, Industries Cloud, apps do AppExchange) e componentes nativos da plataforma. Sem uma convenção de nomenclatura clara, rapidamente fica impossível diferenciar:

- O que é customizado deste projeto
- O que é nativo da Salesforce
- O que é de outro package gerenciado

Este problema é especialmente severo em **Trial Orgs de Communications Cloud**, que já vêm com centenas de componentes OmniStudio pré-instalados.

## Decisão

Adotar a convenção `cltSvt_<TIPO>_<NomeDescritivo>` para todos os componentes customizados criados neste projeto.

### Padrão

```
cltSvt_<TIPO>_<NomeDescritivo>
│      │        └── PascalCase, descritivo em inglês
│      └── Abreviação do tipo de componente
└── Prefixo de namespace (fixo)
```

### Significado do prefixo

`cltSvt` = **cl**iente **Sv**e**t** (referência cruzada a Sevet Company, empresa do autor). Em cenário real, seria o código do cliente na consultoria. Como proxy de um namespace organizacional, evita colisão com componentes de package.

### Tabela de tipos

| Componente | Abreviação | Exemplo |
|---|---|---|
| Integration Procedure | `IP` | `cltSvt_IP_GetAccountByCnpj` |
| Data Mapper Extract | `DM_Extract` | `cltSvt_DM_Extract_AccountByCnpj` |
| Data Mapper Load | `DM_Load` | `cltSvt_DM_Load_AccountUpsert` |
| OmniScript | `OS` | `cltSvt_OS_BuscaAccountsReceitaWS` |
| Lightning Web Component | `LWC` | `cltSvt_LWC_cnpjLookup` |
| Custom Metadata Type | *(sufixo `__mdt` é obrigatório)* | `cltSvt_ReceitaWS_Config__mdt` |

### Exceções

**Entidades de segurança seguem nomenclatura de negócio em português**, refletindo a linguagem do briefing (página 12):

| Tipo | Nome | Justificativa |
|---|---|---|
| Profile | `Representante de Vendas` | Briefing cita literalmente entre aspas |
| Permission Set | `Cadastro CNPJ Representante` | Legível para admins e RH |
| Sharing Rule | `Compartilhamento Account Gerente` | Clareza operacional |
| Role Hierarchy | `Sales Manager`, `Sales Rep` | Padrão Salesforce global |

**Campos custom do Account seguem convenção Salesforce padrão** (`NomeCampo__c`), sem prefixo cltSvt, porque são extensões de objeto nativo e não componentes próprios do projeto:

| Campo | Justificativa |
|---|---|
| `CNPJ__c` | Extensão natural do modelo Account |
| `Status_Cadastral__c` | Informação da Receita |
| `Last_API_Sync__c` | Auditoria de sync |

## Consequências

### Positivas

1. **Descoberta imediata** — qualquer avaliador consegue filtrar todos os artefatos deste projeto buscando por `cltSvt`
2. **Evita colisão** — nenhum package gerenciado usa esse prefixo
3. **Consistente com padrões enterprise** — times grandes usam prefixos similares (ex: `acme_`, `corp_`)
4. **Facilita deploy seletivo** — SFDX retrieve/deploy com wildcard `cltSvt_*` puxa apenas o que é do projeto
5. **Storytelling** — no vídeo, é possível mostrar rapidamente "olha aqui, tudo que é meu tem esse prefixo"

### Negativas

1. **Nomes ficam longos** — `cltSvt_IP_GetAccountByCnpj` tem 30 caracteres vs 20 de `IP_GetAccountByCnpj`
2. **Não é auto-explicativo** — quem vê `cltSvt` pela primeira vez não sabe o que significa sem ler esta ADR

## Alternativas consideradas

| Alternativa | Pro | Contra | Veredito |
|---|---|---|---|
| **Sem prefixo** (`IP_GetAccountByCnpj`) | Nomes curtos | Colisão potencial com componentes de package | Rejeitado |
| **Prefixo `Custom_`** | Descritivo | Genérico, muitos packages também usam | Rejeitado |
| **Managed Namespace real** (`sysmap__`) | Padrão AppExchange | Exige namespace registrado. Não faz sentido para projeto de bootcamp | Rejeitado |
| **`cltSvt_` como proxy de namespace** (esta decisão) | Clareza, unicidade, simula padrão enterprise | Nomes longos | **Escolhido** |

## Referências

- Salesforce Developer Docs. *Namespace Prefix*. https://developer.salesforce.com/docs/atlas.en-us.pkg2_dev.meta/pkg2_dev/sfdx_dev2gp_namespace.htm
- Salesforce Architects. *Well-Architected: Easy → Intentional*. https://architect.salesforce.com/docs/architect/well-architected/easy/intentional/overview — consistência de naming é princípio explícito
- Trailheadapps. *Naming convention observations*. https://github.com/trailheadapps/lwc-recipes — referência do padrão de sample apps oficiais Salesforce
