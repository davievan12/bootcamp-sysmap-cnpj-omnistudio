# Evidências — Screenshots do Projeto

Screenshots organizados por bloco de execução, seguindo a estrutura de trabalho usada durante o desenvolvimento. Cada subpasta corresponde a um bloco do desenvolvimento.

## Inventário

| Pasta | Bloco | Quantidade de prints |
|---|---|---|
| 01_Setup | Setup inicial da Trial Org | 2 |
| 02_Custom_Metadata | Custom Metadata Type + Custom Fields | 6 |
| 03_Remote_Site | Remote Site Settings (ReceitaWS, ViaCEP) | 2 |
| 04_Integration_Procedure | Integration Procedures (GetAccountByCnpj, AtualizarAccount, ConsultaCep) | 11 |
| 05_Data_Mappers | Data Mapper Extract + Load | 3 |
| 06_LWC | Lightning Web Component cnpjLookup | 6 |
| 07_OmniScript | OmniScript BuscaAccountsReceitaWS | 14 |
| 08_Security | Profile + Permission Set + Sharing Rule + OWD | 12 |
| **Total** | | **56** |

## Convenção de nomenclatura

Arquivos devem seguir o padrão:

```
NN_NomeDescritivo.png
```

Onde `NN` é o número sequencial dentro do bloco (começando em 01).

Exemplos:
- `08_Security/01_OWD_Account_Private.png`
- `08_Security/02_Profile_Object_Permission.png`
- `04_Integration_Procedure/IP_atualizacao_1.png`

## Anotações

Cada screenshot técnico crítico (FlexCard/IP/OmniScript Designer, Data Mapper mapping) contém **anotações visuais** (caixas de texto, setas, destaques) feitas no Preview do macOS, conforme requisito explícito do PDF (página 28):

> *"Utilizar caixas de texto ou setas para destacar pontos importantes ou explicar partes específicas."*

As anotações destacam:
- Nome do componente
- Configurações críticas
- Fluxo de dados
- Decisões de design relevantes
