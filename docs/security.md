# Modelo de Segurança — Defense in Depth

Documento detalhado das 4 camadas de segurança implementadas no projeto, seguindo o princípio **Defense in Depth** do [NIST SP 800-160](https://csrc.nist.gov/publications/detail/sp/800-160/vol-1/final) e o pilar **Trusted → Secure** do [Salesforce Well-Architected Framework](https://architect.salesforce.com/docs/architect/well-architected/trusted/secure/overview).

---

## Modelo conceitual

```
┌─────────────────────────────────────────────────────────┐
│                    Carlos (Sales Rep)                   │
│                     ↓ tenta acessar                     │
└─────────────────────────┬───────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────┐
│ CAMADA 1 — OWD (Record-level default: Private)          │
│ Pergunta: Carlos pode ver este Account?                 │
│ Resposta: Só se ele for o owner OU se houver sharing    │
└─────────────────────────┬───────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────┐
│ CAMADA 2 — Profile (Object-level scope)                 │
│ Pergunta: Carlos tem permissão no objeto Account?       │
│ Resposta: Read/Create/Edit sim. Delete não.             │
└─────────────────────────┬───────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────┐
│ CAMADA 3 — Permission Set (Field-level)                 │
│ Pergunta: Carlos pode ler/editar CNPJ__c?               │
│ Resposta: Read + Edit. Last_API_Sync__c: Read only.     │
└─────────────────────────┬───────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────┐
│ CAMADA 4 — Sharing Rule (Role hierarchy)                │
│ Pergunta: Alguém mais pode ver este Account?            │
│ Resposta: Sim — Sales Manager do Carlos, Read/Write.    │
└─────────────────────────────────────────────────────────┘
```

---

## Camada 1 — OWD (Organization-Wide Defaults)

**Configuração:** Account → Private (Default Internal Access + Default External Access)

**Efeito:** Por padrão, cada Account é visível **apenas para o owner** do registro. Nenhum outro usuário consegue ver, mesmo que tenha permissão de Read no objeto.

**Por que Private e não Public Read/Write:**
- Sales Reps são independentes — o cliente do Carlos não deveria ser visível para outros Reps concorrentes
- Regulamentação LGPD (Lei Geral de Proteção de Dados) sugere minimização de acesso a dados de pessoa jurídica sensíveis
- Base restritiva permite escalar — é fácil abrir acesso com Sharing Rule; quase impossível restringir um modelo aberto depois de dados criados

**Print de evidência:** `docs/evidencias/08_Security/01_OWD_Account_Private.png`

---

## Camada 2 — Profile `Representante de Vendas`

**Configuração:** Clone do Standard User, com ajustes cirúrgicos baseados no princípio de **menor privilégio**.

### Standard Object Permissions

| Objeto | Read | Create | Edit | Delete | View All | Modify All |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| **Accounts** | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| **Contacts** | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| Leads | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Opportunities | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Cases | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Campaigns | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |

**Delete negado em Account/Contact:** Carlos é rep externo trabalhando no celular. Não deve ter poder de apagar cliente com 1 clique — decisão operacional é do gestor, não do rep em campo.

**Leads/Opportunities/Cases removidos:** fora do escopo do fluxo CNPJ. Menor privilégio reduz superfície de ataque e cumpre requisito literal do PDF página 12: *"Assegurar que os usuários não tenham acesso a objetos ou campos desnecessários."*

### Administrative Permissions

Todas **desmarcadas**:
- ❌ Modify All Data
- ❌ View All Data
- ❌ Manage Users
- ❌ Customize Application

### General User Permissions

- ✅ **API Enabled** — crítico. Sem isso o LWC não consegue invocar a Integration Procedure.
- ✅ Run Reports — útil para Carlos ver métricas de cadastros próprios.

---

## Camada 3 — Permission Set `Cadastro CNPJ Representante`

**Princípio arquitetural:** separação Profile (wide) vs Permission Set (específico). Esta separação é padrão moderno do Salesforce e está explicitamente recomendada no Well-Architected Framework.

Enquanto o Profile define **quem o usuário é** (Representante de Vendas), o Permission Set define **o que o usuário pode fazer no recurso específico** (cadastro via CNPJ).

### Field-Level Security (custom fields do Account)

| Campo | Read | Edit | Justificativa |
|---|:---:|:---:|---|
| `CNPJ__c` | ✅ | ✅ | Necessário para busca e cadastro inicial |
| `Status_Cadastral__c` | ✅ | ✅ | Pode precisar de correção manual se API divergir |
| `Last_API_Sync__c` | ✅ | ❌ | **Readonly** — auditoria. Preenchido apenas pela IP |
| `Complemento__c` | ✅ | ✅ | Campo editável padrão |
| `Data_Abertura__c` | ✅ | ✅ | Campo editável padrão |
| `Email_Empresa__c` | ✅ | ✅ | Campo editável padrão |

**Readonly em `Last_API_Sync__c`:** se Carlos pudesse editar este campo manualmente, a confiança na auditoria se perderia. FLS é a última barreira de proteção — mais granular que Profile.

### Custom Metadata Types

- ✅ `cltSvt_ReceitaWS_Config__mdt` (Read)

Necessário para a Integration Procedure ler o endpoint em runtime.

### System Permissions

- ✅ API Enabled (defense in depth — redundância intencional com o Profile)
- ✅ Run Reports

### Manage Assignments

O Permission Set é atribuído **explicitamente** aos usuários via `Manage Assignments`. Isso força auditoria: cada usuário com acesso foi autorizado deliberadamente, com data e autor registrados.

---

## Camada 4 — Sharing Rule `Compartilhamento Account Gerente`

### Role Hierarchy (pré-requisito)

```
Sales (root)
  └── Sales Manager
       └── Sales Rep
```

**Por que em inglês:** hierarquia de role é estrutura organizacional genérica, não vocabulário específico do projeto. Padrão Salesforce global.

### Sharing Rule

| Atributo | Valor |
|---|---|
| Label | Compartilhamento Account Gerente |
| Rule Type | Based on record owner |
| Owned by members of | Roles: **Sales Rep** |
| Share with | Roles: **Sales Manager** |
| Default Account, Contract and Asset Access | **Read/Write** |

**Por que Read/Write e não Read Only:** o Gerente precisa poder corrigir Accounts dos subordinados rapidamente — cenário real: Carlos reporta via WhatsApp que cadastrou telefone errado, Gerente corrige sem depender do Carlos voltar ao sistema. Decisão de produto, não técnica.

### Complementaridade com Grant Access Using Hierarchies

O Salesforce, por padrão, concede leitura por hierarquia — Sales Manager já veria Accounts do Sales Rep via `Grant Access Using Hierarchies`. A Sharing Rule explícita:

1. Garante **Read/Write** (não só Read)
2. Documenta a regra de negócio (auditável, reversível)
3. Sobrevive se `Grant Access Using Hierarchies` for desmarcado no futuro

Isso é **defense in depth aplicado ao modelo de sharing**.

---

## Matriz de proteção — qual camada bloqueia qual ataque

| Ameaça | Camada que bloqueia |
|---|---|
| Sales Rep A tentar ver Account do Sales Rep B | Camada 1 (OWD Private) |
| Sales Rep tentar deletar Account | Camada 2 (Profile sem Delete) |
| Sales Rep acessar Leads (fora de escopo) | Camada 2 (Profile sem permissão) |
| Sales Rep adulterar `Last_API_Sync__c` | Camada 3 (FLS readonly) |
| Usuário sem Permission Set acessar cadastro CNPJ | Camada 3 (Custom Metadata sem acesso) |
| Sales Manager não ter visibilidade da equipe | Camada 4 (Sharing Rule explícita) |

Nenhuma camada protege contra tudo. Juntas, cobrem vetores complementares.

---

## Limitações reconhecidas

### Usuários dos examinadores

O requisito do briefing (página 4) solicita criar usuários nominais para os examinadores no pattern `email+.sales9` com "perfil Premium". A Trial Org Communications Cloud disponibiliza apenas **1 licença Salesforce full**, já consumida pelo admin.

**Decisão tomada:** não criar usuários com licenças limitadas (Chatter External não daria acesso funcional) e compartilhar credenciais admin no envio via Gupy. Documentado no [README principal](../README.md#acesso-à-trial-org-examinadores).

Em ambiente produtivo (não-Trial), os examinadores receberiam licença `Salesforce` com Profile `System Administrator`.

### Campos que não têm FLS explícita

Campos standard do Account (Name, Phone, BillingStreet, etc.) herdam FLS do Profile base (Standard User clonado). Isso é intencional — reconfigurar FLS para todos os 80+ campos standard seria over-engineering para este projeto.

---

## Referências

- Salesforce Architects. *Well-Architected: Trusted → Secure*. https://architect.salesforce.com/docs/architect/well-architected/trusted/secure/overview
- Salesforce Help. *Record-Level Security: Sharing Rules*. https://help.salesforce.com/s/articleView?id=sf.security_sharing_rules.htm
- Salesforce Help. *Field-Level Security*. https://help.salesforce.com/s/articleView?id=sf.admin_fls.htm
- NIST. *SP 800-160 Vol. 1 — Systems Security Engineering*. https://csrc.nist.gov/publications/detail/sp/800-160/vol-1/final
- OWASP. *Defense in Depth*. https://owasp.org/www-community/controls/Defense_in_Depth
