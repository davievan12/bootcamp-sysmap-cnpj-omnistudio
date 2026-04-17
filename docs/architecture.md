# Arquitetura — Cadastro CNPJ via ReceitaWS

Documento técnico detalhado da arquitetura da solução. Este documento complementa o [README principal](../README.md) e deve ser lido em conjunto com os [ADRs](adrs/).

---

## 1. Visão de alto nível

A solução segue uma arquitetura **camada-a-camada**, onde cada camada tem responsabilidade única:

```
┌───────────────────────────────────────────────────────────────┐
│ APRESENTAÇÃO                                                  │
│   OmniScript cltSvt_OS_BuscaAccountsReceitaWS                 │
│   LWC cltSvt_LWC_cnpjLookup                                   │
└──────────────────────────┬────────────────────────────────────┘
                           │
┌──────────────────────────▼────────────────────────────────────┐
│ ORQUESTRAÇÃO                                                  │
│   Integration Procedure cltSvt_IP_GetAccountByCnpj            │
│   Integration Procedure cltSvt_IP_AtualizarAccount            │
│   Integration Procedure cltSvt_IP_ConsultaCep                 │
└──────────┬───────────────────────────────────────┬────────────┘
           │                                       │
┌──────────▼──────────────────────┐   ┌────────────▼───────────┐
│ PERSISTÊNCIA                    │   │ INTEGRAÇÃO EXTERNA     │
│   Data Mapper Extract           │   │   HTTP Action          │
│   Data Mapper Load (Upsert)     │   │   → ReceitaWS API      │
└──────────┬──────────────────────┘   └────────────┬───────────┘
           │                                       │
┌──────────▼──────────────────────┐   ┌────────────▼───────────┐
│ DADO                            │   │ CONFIG                 │
│   Account (sObject)             │   │   cltSvt_ReceitaWS_    │
│   + Custom Fields               │   │   Config__mdt          │
└─────────────────────────────────┘   └────────────────────────┘
```

---

## 2. Sequence diagram — Cache Hit (Account já existe)

```
Carlos         LWC              IP              DM Extract       Account
  │             │                │                   │              │
  │ digita CNPJ │                │                   │              │
  ├────────────►│                │                   │              │
  │             │ valida DV (JS) │                   │              │
  │             ├─────────┐      │                   │              │
  │             │◄────────┘      │                   │              │
  │ clica buscar│                │                   │              │
  ├────────────►│                │                   │              │
  │             │  invoke(cnpj)  │                   │              │
  │             ├───────────────►│                   │              │
  │             │                │ extract(cnpj)     │              │
  │             │                ├──────────────────►│              │
  │             │                │                   │ SELECT       │
  │             │                │                   ├─────────────►│
  │             │                │                   │ Account      │
  │             │                │                   │◄─────────────┤
  │             │                │◄──────────────────┤              │
  │             │◄───────────────┤                   │              │
  │             │ exibe dados    │                   │              │
  │◄────────────┤ + CTA refresh  │                   │              │
  │             │                │                   │              │
  │ opt refresh?│                │                   │              │
  │ NÃO         │                │                   │              │
  │ edita       │                │                   │              │
  │ salva       │                │                   │              │
  └─────────────┘                │                   │              │
                                 │ (upsert flow)     │              │
```

Latência típica: **<200 ms** (sem chamada externa).

---

## 3. Sequence diagram — Cache Miss (Account não existe)

```
Carlos         LWC              IP           HTTP Action        ReceitaWS        DM Load      Account
  │             │                │                │                 │               │            │
  │ digita CNPJ │                │                │                 │               │            │
  ├────────────►│                │                │                 │               │            │
  │             │ valida DV (JS) │                │                 │               │            │
  │ clica buscar│                │                │                 │               │            │
  ├────────────►│                │                │                 │               │            │
  │             │  invoke(cnpj)  │                │                 │               │            │
  │             ├───────────────►│                │                 │               │            │
  │             │                │ extract → vazio│                 │               │            │
  │             │                ├────┐           │                 │               │            │
  │             │                │◄───┘           │                 │               │            │
  │             │                │ GET /cnpj/:id │                 │               │            │
  │             │                ├───────────────►│                 │               │            │
  │             │                │                │ HTTP GET        │               │            │
  │             │                │                ├────────────────►│               │            │
  │             │                │                │ JSON (2–5s)     │               │            │
  │             │                │                │◄────────────────┤               │            │
  │             │                │◄───────────────┤                 │               │            │
  │             │◄───────────────┤                │                 │               │            │
  │             │ exibe dados    │                │                 │               │            │
  │◄────────────┤                │                │                 │               │            │
  │ revisa/edita│                │                │                 │               │            │
  │ clica salvar│                │                │                 │               │            │
  ├────────────►│                │                │                 │               │            │
  │             │ invoke(payload)│                │                 │               │            │
  │             ├───────────────►│                │                 │               │            │
  │             │                │ load(payload) │                 │               │            │
  │             │                ├────────────────────────────────────────────────►│            │
  │             │                │                │                 │               │ UPSERT     │
  │             │                │                │                 │               ├───────────►│
  │             │                │                │                 │               │ OK         │
  │             │                │                │                 │               │◄───────────┤
  │             │                │◄───────────────────────────────────────────────┤            │
  │             │◄───────────────┤                │                 │               │            │
  │             │ toast sucesso  │                │                 │               │            │
  │◄────────────┤                │                │                 │               │            │
```

Latência típica: **2–6 s** (dependente da ReceitaWS).

---

## 4. Modelo de dados

### Account (objeto estendido)

| Campo | API Name | Tipo | Propriedades | Origem do dado |
|---|---|---|---|---|
| Razão Social | `Name` | Text(255) | Standard | ReceitaWS `nome` |
| CNPJ | `CNPJ__c` | Text(14) | **External ID, Unique** | input do usuário / `cnpj` |
| Logradouro completo | `BillingStreet` | TextArea | Standard | Concatenação `logradouro + numero + complemento + bairro` |
| Município | `BillingCity` | Text | Standard | `municipio` |
| UF | `BillingState` | Text(2) | Standard | `uf` |
| CEP | `BillingPostalCode` | Text(8) | Standard | `cep` (sem formatação) |
| Telefone | `Phone` | Phone | Standard | `telefone` |
| Situação Cadastral | `Status_Cadastral__c` | Picklist | Custom | `situacao` |
| Última Sync API | `Last_API_Sync__c` | DateTime | Custom, **Read-Only** via Permission Set | Timestamp de cada sync |
| Data de Abertura | `Data_Abertura__c` | Date | Custom (bonus) | `abertura` |
| Email Empresa | `Email_Empresa__c` | Email | Custom (bonus) | `email` |
| Complemento | `Complemento__c` | Text | Custom (bonus) | `complemento` |

### Mapeamento Data Mapper (ReceitaWS → Account)

O Data Mapper Load `cltSvt_DM_Load_AccountUpsert` define o mapeamento declarativo:

| JSON da ReceitaWS | Account Field | Transformação |
|---|---|---|
| `$.nome` | `Name` | Trim |
| `$.cnpj` | `CNPJ__c` | Remove pontuação (14 dígitos) |
| `$.logradouro + $.numero + $.complemento + $.bairro` | `BillingStreet` | Concatenação com separadores |
| `$.municipio` | `BillingCity` | — |
| `$.uf` | `BillingState` | Uppercase |
| `$.cep` | `BillingPostalCode` | Remove pontuação |
| `$.telefone` | `Phone` | — |
| `$.situacao` | `Status_Cadastral__c` | Match contra picklist |
| `NOW()` | `Last_API_Sync__c` | Timestamp server-side |

---

## 5. Configuração externa via Custom Metadata

Evitar hardcode de endpoint é princípio do Well-Architected (Easy → Intentional). O Custom Metadata Type `cltSvt_ReceitaWS_Config__mdt` centraliza:

| Campo | Valor atual | Descrição |
|---|---|---|
| `Endpoint__c` | `https://www.receitaws.com.br/v1/cnpj/` | URL base. Trocar para SERPRO, Casa dos Dados, etc. sem alterar código |
| `Timeout_Seconds__c` | `30` | Timeout da requisição HTTP |
| `Rate_Limit_Per_Minute__c` | `3` | Limite do plano gratuito (documentação) |

Para habilitar o callout, `Remote Site Settings` precisa ter o endpoint cadastrado (feito no Bloco 2 do projeto).

---

## 6. Princípios arquiteturais aplicados

Esta solução foi avaliada contra os princípios do [Salesforce Well-Architected Framework](https://architect.salesforce.com/docs/architect/well-architected/guide/overview.html):

| Pilar → Behavior | Princípio aplicado | Evidência no projeto |
|---|---|---|
| **Trusted → Secure** | Defense in depth | 4 camadas de segurança (ver [security.md](security.md)) |
| **Trusted → Compliant** | Auditabilidade | `Last_API_Sync__c` readonly; External ID imutável |
| **Trusted → Reliable** | Graceful degradation | Erros granulares, leitura funciona sem API |
| **Easy → Intentional** | Decisões documentadas | 6 ADRs; persona explícita |
| **Easy → Automated** | Config over code | 0 linhas de Apex; Custom Metadata para endpoint |
| **Easy → Engaging** | Mobile-first | SLDS; máscaras em tempo real; feedback <300ms |
| **Adaptable → Resilient** | Cache-aside | Reduz dependência da API externa |
| **Adaptable → Composable** | Reutilização | IP invocável de qualquer frontend; Permission Set desacoplado |

---

## Referências

- Salesforce Architects. *Well-Architected Framework*. https://architect.salesforce.com/docs/architect/well-architected/guide/overview.html
- Microsoft Learn. *Cache-Aside Pattern*. https://learn.microsoft.com/en-us/azure/architecture/patterns/cache-aside
- Salesforce Help. *OmniStudio Integration Procedures*. https://help.salesforce.com/s/articleView?id=sf.os_integration_procedures.htm
- Salesforce Help. *OmniStudio Data Mappers*. https://help.salesforce.com/s/articleView?id=sf.os_data_mappers_27964.htm
