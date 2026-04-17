# ADR-0004: CNPJ como External ID para Idempotência

- **Status:** Accepted
- **Data:** 2026-04-16
- **Autor:** Davi dos Santos Evangelista
- **Tags:** `data-model`, `idempotencia`, `integridade`

## Contexto

Um dos requisitos do briefing (página 18) é:

> *"Verificar se já existe uma conta com o mesmo CNPJ para evitar duplicatas."*

A implementação ingênua deste requisito seria:

```
1. Executar SOQL: SELECT Id FROM Account WHERE CNPJ__c = :cnpj
2. Se retornar > 0 registros → UPDATE
3. Se retornar 0 registros → INSERT
```

Este approach tem 3 problemas:

1. **Race condition:** entre o SELECT e o INSERT, outro usuário pode criar o mesmo CNPJ
2. **Código imperativo:** exige Apex (viola [ADR-0002](0002-100-omnistudio-zero-apex.md))
3. **Complexidade de teste:** necessita cobrir ambos os branches

## Decisão

Configurar o campo `CNPJ__c` no objeto Account com dois atributos essenciais:

- **External ID:** ✅
- **Unique (Case Insensitive):** ✅

E usar o recurso nativo **Upsert** do Data Mapper Load, que:

1. Aceita External ID como chave de identificação
2. Executa INSERT se o valor não existir
3. Executa UPDATE se o valor existir
4. É **atômico** — sem race conditions
5. É declarativo — sem uma linha de código

### Implementação

No Data Mapper Load `cltSvt_DM_Load_AccountUpsert`:

```
Operation Type: Upsert
Upsert Key: CNPJ__c
```

### Armazenamento do CNPJ

O CNPJ é armazenado **sem formatação** (apenas 14 dígitos). A formatação `XX.XXX.XXX/XXXX-XX` é aplicada apenas na camada de apresentação (LWC). Isso garante:

- Comparação consistente (não importa se o input veio formatado ou não)
- Busca eficiente no índice do External ID
- Separation of concerns: dado puro no banco, formatação no front

## Consequências

### Positivas

1. **Idempotência garantida por plataforma** — mesmo chamada chamada N vezes produz o mesmo resultado
2. **Zero chance de duplicata** — constraint Unique no database
3. **Performance** — External ID cria índice automaticamente; lookup O(log n)
4. **Zero código** — reforça [ADR-0002](0002-100-omnistudio-zero-apex.md)
5. **Alinha com Well-Architected → Trusted → Compliant** — integridade referencial garantida por design
6. **Facilita integração futura** — qualquer sistema externo pode enviar CNPJ como chave natural sem precisar saber o Salesforce Id

### Negativas

1. **Case sensitivity:** External ID no Salesforce tem opção Case Sensitive ou Case Insensitive. Como CNPJ é numérico, essa nuance não se aplica, mas é importante documentar
2. **Migração de dados legados:** se houver Accounts sem CNPJ, Upsert por CNPJ__c ignora esses registros

## Alternativas consideradas

| Alternativa | Pro | Contra | Veredito |
|---|---|---|---|
| **SOQL + IF + DML** (imperativo) | Código explícito, fácil debug | Race conditions. Exige Apex. Quebra [ADR-0002](0002-100-omnistudio-zero-apex.md) | Rejeitado |
| **External ID + Upsert** (esta decisão) | Atômico, declarativo, indexado automaticamente | Nenhum trade-off material | **Escolhido** |
| **Duplicate Rules nativas** | Previne duplicatas no UI | Não previne em callouts/integrações. Exige configuração adicional | Complementar, não substituto |

## Referências

- Salesforce Developer Docs. *External IDs and Upsert*. https://developer.salesforce.com/docs/atlas.en-us.api.meta/api/sforce_api_concepts_upsert.htm
- Salesforce Help. *Define Unique Identifiers with External ID*. https://help.salesforce.com/s/articleView?id=sf.custom_field_attributes.htm
- Martin Fowler. *Idempotency in REST*. https://martinfowler.com/articles/rest-api.html#idempotent
