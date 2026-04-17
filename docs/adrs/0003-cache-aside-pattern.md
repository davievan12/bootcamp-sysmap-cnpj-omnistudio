# Plano de Testes — Bootcamp Sysmap OmniStudio

> Todos os testes foram executados via **OmniStudio IP Preview** e **OmniScript Preview**.
> Não há testes Apex — a solução é 100% OmniStudio declarativo (ADR-001).

---

## Método de teste

| Ferramenta | Uso |
|---|---|
| IP Preview (Integration Procedure Designer) | Testar IPs isoladamente com inputs JSON manuais |
| OmniScript Preview (OmniScript Designer) | Testar fluxo end-to-end com LWC embedado |
| Console do Navegador (DevTools) | Debug de respostas do `omniRemoteCall` no LWC |
| Salesforce UI (Account Detail) | Verificar persistência dos dados no registro |

---

## Cenário 1: Cache Hit (Account já cadastrado)

**Descrição:** Consultar um CNPJ que já existe como Account no Salesforce. O IP deve retornar os dados do cache sem chamar a API ReceitaWS.

**Input:**
```json
{
  "cnpj": "60701190000104"
}
```

**Execution Sequence esperada:**
```
BuscarNoSalesforce → VerificaSeExiste → RetornarCache
```

**Output esperado:**
- `fonte: "cache"`
- `empresa.razaoSocial: "ITAU UNIBANCO S.A."`
- Todos os campos preenchidos do Salesforce

**Latência esperada:** ~100-150ms

**Validação:**
- [x] IP retorna `VerificaSeExisteStatus: true`
- [x] `RetornarCacheStatus: true`
- [x] Dados corretos no node `empresa`
- [x] API ReceitaWS NÃO foi chamada (sem `ChamarReceitaWS` no execution sequence)
- [x] Rate limit preservado

**Evidência:** `docs/evidencias/01-cache-hit.png`

---

## Cenário 2: Cache Miss (Account não existe)

**Descrição:** Consultar um CNPJ que NÃO existe no Salesforce. O IP deve chamar a API ReceitaWS, salvar o Account e retornar os dados.

**Input:**
```json
{
  "cnpj": "92754738000162"
}
```

**Execution Sequence esperada:**
```
BuscarNoSalesforce → SeNaoExiste → ChamarReceitaWS → VerificaRespostaAPI → SalvarAccount → RetornarNovo
```

**Output esperado:**
- `fonte: "api"`
- `empresa.nome: "LOJAS RENNER S.A."`
- Account criado no Salesforce com CNPJ limpo (14 dígitos)

**Latência esperada:** ~500-700ms

**Validação:**
- [x] IP retorna `SeNaoExisteStatus: true`
- [x] `ChamarReceitaWSStatus: true` (HTTP 200, status "OK")
- [x] `SalvarAccountStatus: true` (UpsertSuccess: true)
- [x] Account visível no Salesforce com dados corretos
- [x] `Last_API_Sync__c` preenchido com data/hora atual

**Evidência:** `docs/evidencias/02-cache-miss.png`

---

## Cenário 3: forceApi (Bypass de cache via "Verificar atualizações")

**Descrição:** Consultar um CNPJ que JÁ existe no Salesforce, mas com `forceApi=true`. O IP deve ignorar o cache e ir direto pra API ReceitaWS.

**Input:**
```json
{
  "cnpj": "47960950000121",
  "forceApi": "true"
}
```

**Execution Sequence esperada:**
```
BuscarNoSalesforce → SeNaoExiste → ChamarReceitaWS → VerificaRespostaAPI → SalvarAccount → RetornarNovo
```

**Output esperado:**
- `fonte: "api"` (mesmo tendo dados no cache)
- `VerificaSeExisteStatus: false` (cache bypass)
- `SeNaoExisteStatus: true` (entrou no caminho da API)
- Dados frescos da ReceitaWS

**Validação:**
- [x] `VerificaSeExisteStatus: false` (condição AND com forceApi funcionou)
- [x] `SeNaoExisteStatus: true` (condição OR com forceApi funcionou)
- [x] Dados atualizados no Account (ex: telefone Magazine Luiza mudou de "55333" para "(16) 3711-2002")
- [x] `Last_API_Sync__c` atualizado

**Condições do IP:**
- VerificaSeExiste: `AND(NOT(ISBLANK(BuscarNoSalesforce)), %forceApi% != "true")`
- SeNaoExiste: `OR(ISBLANK(BuscarNoSalesforce), %forceApi% == "true")`

**Evidência:** `docs/evidencias/03-force-api.png`

---

## Cenário 4: CNPJ Inválido (Validação client-side no LWC)

**Descrição:** Carlos digita um CNPJ com dígito verificador inválido. A validação acontece no LWC (JavaScript) e o IP NÃO é chamado.

**Inputs testados:**

| Input | Resultado esperado |
|---|---|
| `12345678901234` | "CNPJ inválido (dígito verificador incorreto)." |
| `11111111111111` | "CNPJ inválido (todos dígitos iguais)." |
| `1234567890` | Botão "Consultar" desabilitado (< 14 dígitos) |
| `ABC12345678901` | Máscara remove letras, aceita só dígitos |

**Validação:**
- [x] Mensagem de erro aparece abaixo do campo CNPJ
- [x] Botão "Consultar" permanece desabilitado
- [x] NENHUMA chamada ao IP (zero governor limits consumidos)
- [x] Nenhum callout à ReceitaWS (rate limit preservado)

**Algoritmo DV implementado:** pesos [5,4,3,2,9,8,7,6,5,4,3,2] e [6,5,4,3,2,9,8,7,6,5,4,3,2] — padrão oficial Receita Federal.

**Evidência:** `docs/evidencias/04-cnpj-invalido.png`

---

## Cenário 5: CEP Auto-complete (API ViaCEP via OmniStudio)

**Descrição:** Carlos edita manualmente o campo CEP. Após 2 segundos (debounce), o LWC chama o IP `ConsultaCep_Busca` que consulta a API ViaCEP e preenche automaticamente logradouro, cidade, UF e complemento.

**Input:**
```json
{
  "cep": "01001000"
}
```

**Output esperado (do IP ConsultaCep_Busca):**
```json
{
  "ChamarViaCep": {
    "logradouro": "Praça da Sé",
    "complemento": "lado ímpar",
    "localidade": "São Paulo",
    "uf": "SP",
    "cep": "01001-000"
  }
}
```

**CEPs testados:**

| CEP | Logradouro | Cidade | UF |
|---|---|---|---|
| 01001000 | Praça da Sé | São Paulo | SP |
| 20040020 | Rua do Ouvidor | Rio de Janeiro | RJ |
| 30130000 | Avenida Afonso Pena | Belo Horizonte | MG |
| 32604390 | (Betim) | Betim | MG |

**Regra importante:** Só consulta se o CEP foi digitado manualmente pelo usuário. Se veio da API do CNPJ ou do cache, NÃO consulta (flag `cepDigitadoPeloUsuario`).

**Validação:**
- [x] Campos preenchidos automaticamente após 2 segundos
- [x] Spinner de loading aparece durante a consulta
- [x] CEP da API do CNPJ NÃO dispara auto-complete
- [x] CEP editado manualmente dispara auto-complete

**Evidência:** `docs/evidencias/05-cep-autocomplete.png`

---

## Cenário 6: Comparação inteligente (Verificar atualizações com diff)

**Descrição:** Carlos clica "Verificar se há atualizações via Receita Federal". O LWC compara os dados atuais (cache/editados) com os dados frescos da API campo a campo.

**Sub-cenário 6A — Sem alterações:**
- Badge muda para "Sem nenhuma alteração" (cinza)
- Nenhum painel de diferenças aparece

**Sub-cenário 6B — Com alterações:**
- Badge muda para "Dados atualizados" (verde)
- Painel amarelo aparece com tabela: Campo | Formulário | Receita Federal
- Valores antigos riscados em vermelho, novos em verde
- Dois botões: "Usar dados do governo" e "Manter atual"

**Validação (6A):**
- [x] Dados iguais → badge "Sem nenhuma alteração"
- [x] Nenhum painel de diferenças

**Validação (6B):**
- [x] Dados diferentes → badge "Dados atualizados"
- [x] Painel lista campos que mudaram com valores antes/depois
- [x] "Usar dados do governo" → mantém dados novos da API
- [x] "Manter atual" → restaura dados originais de antes da verificação

**Evidência:** `docs/evidencias/06-comparacao-inteligente.png`

---

## Cenário 7: Salvar dados editados (IP AtualizarAccount)

**Descrição:** Carlos edita campos no formulário e clica "Salvar dados". O LWC chama o IP `AtualizarAccount_Manual` que persiste as edições no Salesforce via Data Mapper Load.

**Input (exemplo):**
```json
{
  "cnpj": "60701190000104",
  "nome": "ITAU TESTE EDITADO",
  "situacao": "ATIVA",
  "telefone": "(11) 9999-9999",
  "logradouro": "RUA TESTE",
  "municipio": "SAO PAULO",
  "uf": "SP",
  "cep": "01000-000"
}
```

**Validação:**
- [x] Toast verde "Dados salvos com sucesso!" aparece por 2 segundos
- [x] Campos limpam automaticamente para nova consulta
- [x] Account atualizado no Salesforce com dados editados
- [x] Re-consulta do mesmo CNPJ retorna dados editados do cache

**Evidência:** `docs/evidencias/07-salvar-dados.png`

---

## Cenário 8: Flag de dados desatualizados (Last_API_Sync > 7 dias)

**Descrição:** Quando o campo `Last_API_Sync__c` do Account tem mais de 7 dias, o LWC exibe uma mensagem vermelha "Dados desatualizados, atualização recomendada."

**Configuração do teste:**
- Editou manualmente `Last_API_Sync__c` do Itaú para 10 dias atrás

**Validação:**
- [x] Mensagem vermelha aparece abaixo do botão de verificar atualizações
- [x] Dados com menos de 7 dias NÃO mostram a mensagem
- [x] Após clicar "Verificar atualizações", `Last_API_Sync__c` é atualizado e a mensagem some

**Evidência:** `docs/evidencias/08-dados-desatualizados.png`

---

## Resumo de performance

| Cenário | Latência medida | Componentes executados |
|---|---|---|
| Cache Hit | ~113ms | Data Mapper Extract |
| Cache Miss (API) | ~700ms | Extract + HTTP + Load |
| forceApi (bypass) | ~550ms | Extract + HTTP + Load |
| CEP Auto-complete | ~400ms | HTTP (ViaCEP) |
| Salvar dados | ~200ms | Data Mapper Load |
| Validação CNPJ | 0ms (client-side) | Nenhum (JavaScript) |

**Ganho de performance com cache:** ~84% redução de latência (700ms → 113ms)
**Preservação de rate limit:** 3 chamadas/min da ReceitaWS consumidas apenas em cache miss e forceApi

---

## Ferramentas de teste do OmniStudio

O OmniStudio não possui framework de testes unitários como Apex (sem `@isTest`). Os testes são realizados via:

1. **IP Preview** — execução isolada com inputs manuais, retorna execution sequence + debug log + timing
2. **OmniScript Preview** — execução do fluxo completo com LWC embedado, simula experiência do usuário
3. **Data JSON** — painel lateral que mostra o estado do OmniScript em tempo real
4. **Action Debugger** — rastreamento de chamadas IP dentro do OmniScript
5. **Console do Navegador** — `console.log` no LWC para debug de respostas do `omniRemoteCall`

Esta abordagem de teste é a padrão para projetos 100% OmniStudio e está documentada no Trailhead "OmniStudio Integration Procedures" e na documentação oficial Salesforce.