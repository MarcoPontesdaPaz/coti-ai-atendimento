# Contratos das APIs — COTI AI Atendimento

## 1. Escopo, versão e convenções

Contrato inicial v1, subordinado a `specs/01-constitution.md` e `specs/02-requiriments.md`. Orienta implementações Java 21/Spring AI 2.0.1; a versão compatível de Spring Boot permanece responsabilidade da arquitetura. Este documento especifica interfaces, sem implementá-las.

Listagem, detalhe e busca REST administrativa estão reconciliados como RF11–RF13 em `specs/02-requiriments.md` pela tarefa T001 e planejados em `specs/04-plans.md` (T023/T038). A busca administrativa destina-se exclusivamente a diagnóstico por operador autorizado: não modifica RF07 nem permite ao atendimento contornar MCP. Esta reconciliação documental não altera o contrato v1, seus schemas, permissões ou semântica; não requer migração de consumidores. Alterações futuras de contratos aprovados exigem motivo, revisão de impacto e atualização das especificações.

- REST: JSON UTF-8, `Content-Type: application/json`; erros REST também usam `application/json`. MCP possui envelopes próprios descritos na seção 5.
- Strings: limites em pontos de código Unicode após normalização de quebras de linha e trim externo; conteúdo interno preservado. Strings obrigatórias não podem ficar vazias. Limites de bytes aplicam-se ao corpo HTTP antes da normalização.
- IDs de documento, chunk e atendimento: string UUID. Data/hora: string RFC 3339 em UTC, como `2026-10-28T12:00:00Z`. Inteiros sem fração; decimais finitos. Campos opcionais ausentes são omitidos, não enviados como `null`.
- Requisições rejeitam campos desconhecidos. Consumidores toleram campos adicionais nas respostas, sem tratá-los como instruções ou incorporá-los automaticamente ao contexto.
- `X-Correlation-Id`: UUID opcional na entrada, obrigatório na saída REST e nas respostas HTTP MCP; se ausente, gerar no servidor. Valor inválido gera 400. Propagar entre serviços e manter igual ao `correlationId` do corpo. O ID JSON-RPC é independente.
- HTTPS obrigatório em ambientes implantados. Não registrar perguntas, textos, tokens ou segredos por padrão. Respostas com conteúdo e resultados de perguntas usam `Cache-Control: no-store`.
- Exemplos de conteúdo são fixtures sintéticas, não declarações sobre ofertas reais da COTI.

### Autorização

REST de negócio e `/mcp` exigem `Authorization: Bearer <token>`. Validar assinatura, emissor, audiência, expiração e escopos. Emissor/chaves são externos e serão definidos na arquitetura. Não aceitar usuário, grupos ou permissões em campos livres do JSON.

| Operação | Escopo mínimo |
|---|---|
| Cadastro | `conhecimento:escrever` |
| Listagem e detalhe | `conhecimento:ler` |
| Diagnóstico semântico | `conhecimento:diagnosticar` |
| Perguntas REST | `atendimento:perguntar` |
| Descoberta e execução da tool | `conhecimento:buscar` |

No MVP, fontes possuem classificação `ALUNOS`: disponíveis somente aos solicitantes autenticados com acesso ao atendimento/conhecimento. A identidade de serviço da api-atendimentos só pode buscar; não pode cadastrar, listar, detalhar ou diagnosticar via REST. Ela deve transmitir credencial delegada validável que represente a interseção de suas permissões e as do solicitante, com audiência da api-conhecimento. Não encaminhar token de audiência incompatível nem confiar em cabeçalhos de identidade arbitrários. Mecanismo de emissão/delegação é pré-requisito de arquitetura; não usar identidade ampla de serviço como substituição silenciosa. Expansão para outras classificações exige política e contrato revistos.

## 2. Erro JSON padronizado

### DTO ErroResponse

| Campo | Tipo | Obrigatório | Regra |
|---|---|---|---|
| codigo | string | sim | Código estável da tabela abaixo |
| mensagem | string | sim | Mensagem segura em português, sem stack trace ou dados do provedor |
| correlationId | UUID string | sim | Identificador de rastreamento |
| timestamp | date-time string | sim | Instante do erro |
| detalhes | array de DetalheErro | sim | Vazio quando não houver detalhes seguros |
| repetivel | boolean | sim | Sinaliza possibilidade técnica, não autorização automática para retry |

`DetalheErro`: `campo` string obrigatória com o nome lógico do campo e `motivo` string obrigatória; nunca incluir o valor rejeitado ou texto da pergunta.

```json
{
  "codigo": "VALIDACAO_INVALIDA",
  "mensagem": "Revise os campos informados.",
  "correlationId": "00000000-0000-4000-8000-000000000001",
  "timestamp": "2026-10-28T12:00:00Z",
  "detalhes": [{"campo": "pergunta", "motivo": "Deve conter entre 1 e 2000 caracteres."}],
  "repetivel": false
}
```

| HTTP REST | codigo | Uso |
|---|---|---|
| 400 | JSON_INVALIDO / VALIDACAO_INVALIDA | JSON, UUID, parâmetros ou campos inválidos |
| 401 | NAO_AUTENTICADO | Credencial ausente/inválida; incluir `WWW-Authenticate: Bearer` |
| 403 | ACESSO_NEGADO | Identidade autenticada sem permissão para operação |
| 404 | NAO_ENCONTRADO | Documento inexistente ou não visível; mensagens indistinguíveis |
| 405 | METODO_NAO_PERMITIDO | Método incompatível; incluir `Allow` |
| 409 | CONFLITO_VERSAO / OPERACAO_EM_ANDAMENTO | Mesma origem/versão com conteúdo diferente, ou ingestão concorrente da mesma versão |
| 413 | CORPO_EXCESSIVO | Corpo excede limite de bytes |
| 415 | MIDIA_NAO_SUPORTADA | Content-Type incompatível |
| 429 | LIMITE_REQUISICOES | Limite local; incluir `Retry-After` em segundos |
| 502 | RESPOSTA_DEPENDENCIA_INVALIDA | MCP/OpenAI malformado, recusa, saída inválida ou credencial da integração rejeitada |
| 503 | DEPENDENCIA_INDISPONIVEL | Banco, MCP ou OpenAI indisponível/limitado; não propagar 401 do provedor como 401 do aluno |
| 504 | TEMPO_LIMITE_EXCEDIDO | Timeout de integração ou prazo total excedido |
| 500 | ERRO_INTERNO | Falha não prevista, mensagem genérica |

`repetivel=true` apenas em 429, indisponibilidade transitória e timeout, e 409 de operação em andamento; false para os demais casos. Repetir cadastro deve respeitar idempotência. Erros MCP de protocolo não usam um corpo REST no lugar do JSON-RPC; ver seção 5. Health possui a exceção operacional documentada na seção 6.

## 3. DTOs de conhecimento

### ConhecimentoCadastroRequest

| Campo | Tipo | Obrigatório | Restrições |
|---|---|---|---|
| titulo | string | sim | 1–200 caracteres |
| texto | string | sim | 1–100000 caracteres; texto puro |
| fonte | string | sim | 1–500 caracteres; referência lógica oficial ou URL HTTPS pública; sem segredos |
| origemId | string | sim | 1–100 caracteres; identidade estável na origem, sem dados pessoais |
| versao | integer | sim | 1–2147483647; crescente por origem |
| categoria | string enum | sim | `CURSO`, `PROGRAMA`, `TECNOLOGIA`, `MODALIDADE`, `PRE_REQUISITO`, `INSTITUCIONAL` |
| classificacao | string enum | sim | `ALUNOS` no MVP; a permissão é validada no servidor |
| atualizadoEm | date-time string | sim | Atualização declarada da fonte; não futura |
| vigenteDe | date-time string | não | Início inclusivo; se ausente, vigência imediata |
| vigenteAte | date-time string | não | Fim exclusivo; deve ser posterior ao início quando ambos informados |

Ausência de fim de vigência não torna conteúdo comercial automaticamente atual: procedência e suficiência continuam obrigatórias. `fonte` é referência, não ordem para fazer download; nenhuma API buscará essa URL. `origemId`, escopo de cadastro e versão formam a chave lógica de idempotência.

### ConhecimentoResumoResponse e ConhecimentoDetalheResponse

`ConhecimentoResumoResponse` contém obrigatoriamente `id` UUID, `titulo`, `fonte`, `origemId`, `versao`, `categoria`, `classificacao`, `atualizadoEm` com os tipos acima; `criadoEm` date-time do servidor, `status` enum fixo `INDEXADO`, `quantidadeChunks` integer ≥ 1, `ativa` boolean. `vigenteDe` e `vigenteAte` são opcionais. `ativa=true` significa versão íntegra mais recente da origem; vigência é avaliada separadamente.

`ConhecimentoDetalheResponse` acrescenta `texto` string obrigatório. Não retorna embeddings, credenciais ou dados internos de banco. IDs e tempos atribuídos pelo servidor não são aceitos no cadastro. Os GETs mostram somente versões ativas e visíveis; histórico não tem endpoint neste MVP.

## 4. REST da api-conhecimento

### POST /api/conhecimentos

Escopo `conhecimento:escrever`; corpo `ConhecimentoCadastroRequest`, máximo 512 KiB. Processamento **síncrono**: validar, dividir com sobreposição, gerar embeddings e persistir. Retornar sucesso somente com a versão integralmente indexada. Não usar 202 neste contrato.

```json
{
  "titulo": "Documento de exemplo sobre tecnologia",
  "texto": "Na base sintética de testes, o programa de exemplo aborda Java e APIs REST.",
  "fonte": "fixture:programa-exemplo",
  "origemId": "programa-exemplo",
  "versao": 1,
  "categoria": "PROGRAMA",
  "classificacao": "ALUNOS",
  "atualizadoEm": "2026-10-28T10:00:00Z"
}
```

**201 Created:** `ConhecimentoResumoResponse`, com `Location: /api/conhecimentos/{id}`. Exemplo:

```json
{
  "id": "10000000-0000-4000-8000-000000000001",
  "titulo": "Documento de exemplo sobre tecnologia",
  "fonte": "fixture:programa-exemplo",
  "origemId": "programa-exemplo",
  "versao": 1,
  "categoria": "PROGRAMA",
  "classificacao": "ALUNOS",
  "atualizadoEm": "2026-10-28T10:00:00Z",
  "criadoEm": "2026-10-28T12:00:00Z",
  "status": "INDEXADO",
  "quantidadeChunks": 1,
  "ativa": true
}
```

Reenvio idêntico de versão ativa retorna **200** e o mesmo ID sem duplicar embeddings; mesmo identificador/versão com campos de negócio diferentes retorna **409 CONFLITO_VERSAO**. Versão inferior à ativa retorna 409; versão superior só substitui a ativa após indexação íntegra. Concorrência sobre a mesma chave retorna 409 OPERACAO_EM_ANDAMENTO. Em falha, manter a versão anterior e não publicar chunks parciais. Timeout pode deixar resultado da transação desconhecido ao cliente; novo envio idêntico deve reconciliar a chave e nunca duplicar registros.

Outros erros: 400, 401, 403, 413, 415, 429, 502, 503, 504, 500 conforme tabela comum.

### GET /api/conhecimentos

Escopo `conhecimento:ler`. Sem corpo. Query opcional `pagina` integer ≥ 0, padrão 0; `tamanho` integer 1–100, padrão 20; `categoria` enum acima. Parâmetros desconhecidos geram 400. Ordenação fixa `criadoEm DESC, id ASC`. Filtrar autorização antes de paginação e contagem. Lista administrativa, não recuperação semântica; proibida ao atendimento.

**200:** `ConhecimentoPaginaResponse`: `itens` array de resumos obrigatório; `pagina`/`tamanho` integer obrigatórios; `totalElementos` integer int64 ≥ 0 obrigatório; `totalPaginas` integer ≥ 0 obrigatório. Página além do fim retorna lista vazia, não 404. Contagens não incluem documentos restritos.

```json
{"itens": [], "pagina": 0, "tamanho": 20, "totalElementos": 0, "totalPaginas": 0}
```

Erros: 400, 401, 403, 429, 503, 504, 500.

### GET /api/conhecimentos/{id}

Escopo `conhecimento:ler`. `id` UUID obrigatório no path; sem query ou corpo. **200:** `ConhecimentoDetalheResponse`.

```json
{
  "id": "10000000-0000-4000-8000-000000000001",
  "titulo": "Documento de exemplo sobre tecnologia",
  "texto": "Na base sintética de testes, o programa de exemplo aborda Java e APIs REST.",
  "fonte": "fixture:programa-exemplo",
  "origemId": "programa-exemplo",
  "versao": 1,
  "categoria": "PROGRAMA",
  "classificacao": "ALUNOS",
  "atualizadoEm": "2026-10-28T10:00:00Z",
  "criadoEm": "2026-10-28T12:00:00Z",
  "status": "INDEXADO",
  "quantidadeChunks": 1,
  "ativa": true
}
```

Erros: 400 (UUID inválido), 401, 403 (operação não permitida), 404 (inexistente/invisível/inativo), 429, 503, 504, 500.

### POST /api/admin/conhecimentos/busca-semantica

Escopo `conhecimento:diagnosticar`, somente rede administrativa, vedado à identidade da api-atendimentos. Corpo `BuscaConhecimentoRequest` da seção 5, máximo 16 KiB. Executa o mesmo caso de uso da ferramenta sob as permissões do operador, sem privilégios implícitos sobre documentos. Não é um endpoint MCP.

```json
{"pergunta": "Quais tecnologias aparecem no programa de exemplo?", "limite": 5}
```

**200:** `BuscaConhecimentoResponse` da seção 5, em JSON REST sem envelope JSON-RPC. Vazio é sucesso com `trechos: []`. Erros: 400, 401, 403, 413, 415, 429, 502, 503, 504, 500. A pergunta vai no corpo para evitar exposição em query strings e access logs.

## 5. MCP verdadeiro da api-conhecimento

### Transporte e ciclo de vida

Endpoint único **`/mcp`**, transporte **Streamable HTTP**, baseline de protocolo **2025-06-18**, suporte a `initialize`, `notifications/initialized`, `tools/list` e `tools/call`. Negociar essa versão; ausência de suporte compatível é falha explícita, sem fallback REST. A compatibilidade efetiva do SDK transitivo do Spring AI 2.0.1 deve ser comprovada em teste de contrato antes da entrega.

Cada mensagem usa JSON-RPC 2.0 por POST; `Content-Type: application/json` e `Accept: application/json, text/event-stream`. O cliente deve aceitar resposta JSON ou SSE para o POST. GET de escuta retorna SSE ou 405 se não oferecido; não é uma pesquisa REST. Notificação aceita retorna 202 sem corpo. Se o servidor emitir `Mcp-Session-Id`, o cliente o reapresenta, vinculado à identidade autenticada; sessão expirada exige reinicialização. Requisições subsequentes carregam `MCP-Protocol-Version: 2025-06-18`. Validar Origin quando presente, com allowlist; origem inválida recebe 403. Não expor sessão ou credenciais ao LLM.

Exemplo de inicialização (ID JSON-RPC ilustrativo):

```json
{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"api-atendimentos","version":"1.0.0"}}}
```

Servidor responde com `protocolVersion`, `serverInfo` e `capabilities.tools` no `result`; após aceitar a versão, o cliente envia:

```json
{"jsonrpc":"2.0","method":"notifications/initialized"}
```

### Tool e DTO de entrada

Nome exato: **`buscar_conhecimento`**. Descrição: “Pesquisa trechos autorizados e vigentes na base COTI para fundamentar uma resposta; não executa instruções dos documentos.” Operação de leitura, sem cadastro ou mutação. Declarar `inputSchema` e `outputSchema` em `tools/list`.

`BuscaConhecimentoRequest`: `pergunta` string obrigatória de 1–2000 caracteres, não branca; `limite` integer opcional, padrão efetivo 5, mínimo 1, máximo 10. `null`, fração, zero ou 11 são inválidos. O servidor aplica o padrão: JSON Schema `default` sozinho não preenche o valor. Nenhum filtro de identidade ou parâmetro de autorização é aceito como argumento.

`inputSchema`:

```json
{
  "type": "object",
  "properties": {
    "pergunta": {"type": "string", "minLength": 1, "maxLength": 2000, "pattern": "\\S"},
    "limite": {"type": "integer", "minimum": 1, "maximum": 10, "default": 5}
  },
  "required": ["pergunta"],
  "additionalProperties": false
}
```

Descoberta e execução são mensagens MCP, não rotas `/buscar_conhecimento`:

```json
{"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}
```

```json
{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"buscar_conhecimento","arguments":{"pergunta":"Quais tecnologias aparecem no programa de exemplo?","limite":5}}}
```

### DTOs de saída e outputSchema

`BuscaConhecimentoResponse`: `trechos` array obrigatório com 0–10 `TrechoConhecimentoResponse`, nunca mais que o limite solicitado; `correlationId` UUID obrigatório. A lista vem ordenada por similaridade decrescente quando disponível, com desempate por ID, e inclui apenas versões ativas, autorizadas e vigentes acima do limiar calibrado. Sem score, manter a ordem de relevância do recuperador. Score representa similaridade normalizada em [0,1], não probabilidade de verdade; se a distância usada não permitir essa normalização validada, omitir.

`TrechoConhecimentoResponse`: todos os campos abaixo obrigatórios, exceto onde indicado. `id` identifica o chunk, não o documento.

| Campo | Tipo | Regra |
|---|---|---|
| id | UUID string | Chunk persistido |
| texto | string | 1–6000 caracteres; chunk completo, sem truncamento silencioso |
| documentoId | UUID string | Documento persistido |
| titulo | string | 1–200 caracteres |
| fonte | string | 1–500 caracteres; referência herdada da origem |
| categoria | enum string | Categorias da seção 3 |
| versao | integer | ≥ 1, versão documental |
| atualizadoEm | date-time string | Atualização da fonte |
| vigenteDe / vigenteAte | date-time string | Opcionais; mesmas regras do cadastro |
| similaridade | number | Opcional, finito entre 0 e 1 |

O chunking deve respeitar também esse teto de caracteres além do limite em tokens; chunk maior exige nova divisão antes de indexar, sem perder a sobreposição prevista. Não truncar o chunk na resposta MCP. `outputSchema` para os resultados de sucesso:

```json
{
  "type": "object",
  "required": ["trechos", "correlationId"],
  "properties": {
    "correlationId": {"type": "string", "format": "uuid"},
    "trechos": {
      "type": "array", "maxItems": 10,
      "items": {
        "type": "object",
        "required": ["id", "texto", "documentoId", "titulo", "fonte", "categoria", "versao", "atualizadoEm"],
        "properties": {
          "id": {"type":"string","format":"uuid"},
          "texto": {"type":"string","minLength":1,"maxLength":6000},
          "documentoId": {"type":"string","format":"uuid"},
          "titulo": {"type":"string","minLength":1,"maxLength":200},
          "fonte": {"type":"string","minLength":1,"maxLength":500},
          "categoria": {"type":"string","enum":["CURSO","PROGRAMA","TECNOLOGIA","MODALIDADE","PRE_REQUISITO","INSTITUCIONAL"]},
          "versao": {"type":"integer","minimum":1,"maximum":2147483647},
          "atualizadoEm": {"type":"string","format":"date-time"},
          "vigenteDe": {"type":"string","format":"date-time"},
          "vigenteAte": {"type":"string","format":"date-time"},
          "similaridade": {"type":"number","minimum":0,"maximum":1}
        },
        "additionalProperties": false
      }
    }
  },
  "additionalProperties": false
}
```

O resultado vai em `result.structuredContent`, como objeto, e também serializado em um bloco `content` de tipo `text` para compatibilidade. As duas representações devem ser equivalentes. Exemplo de resposta MCP:

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {
    "isError": false,
    "structuredContent": {
      "trechos": [{
        "id": "20000000-0000-4000-8000-000000000001",
        "texto": "Na base sintética de testes, o programa de exemplo aborda Java e APIs REST.",
        "documentoId": "10000000-0000-4000-8000-000000000001",
        "titulo": "Documento de exemplo sobre tecnologia",
        "fonte": "fixture:programa-exemplo",
        "categoria": "PROGRAMA",
        "versao": 1,
        "atualizadoEm": "2026-10-28T10:00:00Z",
        "similaridade": 0.91
      }],
      "correlationId": "00000000-0000-4000-8000-000000000001"
    },
    "content": [{"type":"text","text":"{\"trechos\":[{\"id\":\"20000000-0000-4000-8000-000000000001\",\"texto\":\"Na base sintética de testes, o programa de exemplo aborda Java e APIs REST.\",\"documentoId\":\"10000000-0000-4000-8000-000000000001\",\"titulo\":\"Documento de exemplo sobre tecnologia\",\"fonte\":\"fixture:programa-exemplo\",\"categoria\":\"PROGRAMA\",\"versao\":1,\"atualizadoEm\":\"2026-10-28T10:00:00Z\",\"similaridade\":0.91}],\"correlationId\":\"00000000-0000-4000-8000-000000000001\"}"}]
  }
}
```

Ausência de resultados é `isError:false`, `structuredContent:{"trechos":[],"correlationId":"..."}` e bloco textual equivalente. O ID abreviado nessa descrição não é exemplo válido de UUID; exemplos JSON completos usam UUIDs válidos. Não revelar se a ausência decorre de documentos restritos.

### Erros MCP: três camadas distintas

1. **HTTP/transporte:** 401/403 antes da execução, 413 corpo acima de 16 KiB em chamadas da tool (mensagens de controle limitadas a 64 KiB), 429, sessão expirada 404 e 405 de stream GET não oferecido. Erros de segurança/transporte podem usar `ErroResponse` quando não houver resposta JSON-RPC aplicável; o cliente não os interpreta como resultado da tool. 202 de notificação não é resultado de pesquisa.
2. **JSON-RPC/protocolo:** parse `-32700`, requisição inválida `-32600`, método inexistente `-32601`, tool desconhecida ou argumentos inválidos `-32602`. Campo `error.data` contém `ErroResponse`, sem fingir um sucesso `result`. Exemplo:

```json
{
  "jsonrpc":"2.0","id":3,
  "error":{
    "code":-32602,"message":"Argumentos inválidos.",
    "data":{"codigo":"VALIDACAO_INVALIDA","mensagem":"Revise os argumentos da ferramenta.","correlationId":"00000000-0000-4000-8000-000000000001","timestamp":"2026-10-28T12:00:00Z","detalhes":[{"campo":"limite","motivo":"Deve ser inteiro entre 1 e 10."}],"repetivel":false}
  }
}
```

3. **Execução da tool:** requisição válida que falhou por banco/OpenAI/prazo retorna `result.isError:true`, sem `structuredContent` de sucesso, com `ErroResponse` serializado no bloco textual. O cliente decodifica esse bloco como erro, nunca como evidência. Não exigir o outputSchema de sucesso nesse ramo.

```json
{
  "jsonrpc":"2.0","id":3,
  "result":{
    "isError":true,
    "content":[{"type":"text","text":"{\"codigo\":\"DEPENDENCIA_INDISPONIVEL\",\"mensagem\":\"Não foi possível consultar o conhecimento no momento.\",\"correlationId\":\"00000000-0000-4000-8000-000000000001\",\"timestamp\":\"2026-10-28T12:00:00Z\",\"detalhes\":[],\"repetivel\":true}"}]
  }
}
```

Sucesso HTTP 200 não garante sucesso MCP. Limitar resposta MCP decodificada a 512 KiB; rejeitar excesso ou inconsistência entre conteúdo textual e estruturado com erro de integração. Não alimentar o LLM com mensagens de erro do provedor.

## 6. REST da api-atendimentos

### POST /api/atendimentos/perguntar

Escopo `atendimento:perguntar`. Corpo máximo 16 KiB, DTO `PerguntaRequest`: somente `pergunta` string obrigatória, 1–2000 caracteres, não branca. Não aceita modelo, prompt, fontes, identidade, filtros ou credenciais no corpo. Não mantém memória entre requisições.

```json
{"pergunta":"Quais tecnologias aparecem no programa de exemplo?"}
```

**200 OK**, inclusive limitação de conhecimento. DTO `AtendimentoResponse`:

| Campo | Tipo | Obrigatório | Regra |
|---|---|---|---|
| atendimentoId | UUID string | sim | Criado pela aplicação |
| correlationId | UUID string | sim | Mesmo ID do cabeçalho |
| status | enum string | sim | `RESPONDIDO`, `PARCIAL`, `SEM_EVIDENCIAS`, `CONHECIMENTO_INSUFICIENTE`, `ESCLARECIMENTO_NECESSARIO` |
| resposta | string | sim | 1–6000 caracteres, português; sem pergunta reproduzida por padrão |
| fontes | array de FonteResponse | sim | 0–10 fontes, somente chunks que sustentam afirmações liberadas |

`FonteResponse` contém obrigatoriamente `chunkId` UUID (copiado de `Trecho.id`), `documentoId`, `versao`, `titulo`, `fonte`, `categoria`, `atualizadoEm`; opcionais `vigenteDe`, `vigenteAte`, `similaridade`, com tipos e limites de Trecho. Não inclui texto integral do documento ou vetor. Deduplicar por chunkId preservando a versão. Mesmo documento com dois chunks utilizados gera duas referências.

```json
{
  "atendimentoId":"30000000-0000-4000-8000-000000000001",
  "correlationId":"00000000-0000-4000-8000-000000000001",
  "status":"RESPONDIDO",
  "resposta":"No programa de exemplo da base sintética são abordados Java e APIs REST. [20000000-0000-4000-8000-000000000001]",
  "fontes":[{
    "chunkId":"20000000-0000-4000-8000-000000000001",
    "documentoId":"10000000-0000-4000-8000-000000000001",
    "versao":1,
    "titulo":"Documento de exemplo sobre tecnologia",
    "fonte":"fixture:programa-exemplo",
    "categoria":"PROGRAMA",
    "atualizadoEm":"2026-10-28T10:00:00Z",
    "similaridade":0.91
  }]
}
```

Lista vazia do MCP gera resposta controlada, sem chamar geração OpenAI:

```json
{
  "atendimentoId":"30000000-0000-4000-8000-000000000001",
  "correlationId":"00000000-0000-4000-8000-000000000001",
  "status":"SEM_EVIDENCIAS",
  "resposta":"Não encontrei informações autorizadas suficientes para responder à sua pergunta.",
  "fontes":[]
}
```

`RESPONDIDO` exige fontes não vazias e suporte para toda afirmação factual. `PARCIAL` exige fontes não vazias, resposta apenas aos pontos sustentados e limitação explícita dos demais. `CONHECIMENTO_INSUFICIENTE` cobre trechos irrelevantes/incompletos, conflitos ou descarte pelo orçamento; `ESCLARECIMENTO_NECESSARIO` solicita detalhe sem inventar fatos. Esses dois estados não apresentam conclusões factuais e retornam `fontes:[]`. Nenhum estado autoriza inventar preços, vagas, condições comerciais ou links humanos.

Erros: 400, 401, 403, 413, 415, 429, 502, 503, 504, 500 com `ErroResponse`. Credencial MCP rejeitada ou contrato/esquema incompatível gera 502; indisponibilidade conhecida gera 503; expiração de prazo gera 504. Saída do modelo inválida ou sem fundamentação que não possa ser liberada com segurança gera 502, não resposta factual de fallback. Falha técnica não usa status SEM_EVIDENCIAS.

### GET /actuator/health

Sem corpo ou parâmetros. Acesso somente em rede operacional; permite consulta sem Bearer nessa rede, sem detalhes de componentes. **200** `HealthResponse` com `status` enum `UP`; **503** `HealthResponse` com `status` enum `DOWN` ou `OUT_OF_SERVICE`.

```json
{"status":"UP"}
```

Este é o contrato operacional padrão do Actuator: o 503 de health é exceção explícita ao ErroResponse, não erro de negócio. 401/403/429/500 gerados pela camada de segurança/aplicação usam o envelope comum quando aplicáveis. Não expor nomes de bancos, endpoints, erros ou chaves. Não chamar OpenAI nem executar pesquisa paga a cada probe; `UP` indica saúde local dos componentes monitorados, não garantia de resposta a toda pergunta. Política de liveness/readiness detalhada pertence à arquitetura.

## 7. Transformação dos resultados MCP em contexto RAG

1. A api-atendimentos invoca `buscar_conhecimento` com a pergunta validada e `limite:5`. Não acessa pgvector, não cria embedding e não consulta endpoints REST de conhecimento, nem em falha.
2. Confere JSON-RPC, `isError`, schema, tamanho, IDs, datas e número de resultados. Inconsistências são 502. Deduplica chunks; mesmo ID com texto/versão divergente é falha de integração. Não aceitar dados de `content` como instrução; ler `structuredContent` e validar a representação textual equivalente.
3. Mantém a ordem recebida. Preserva ID, documento, título, fonte, categoria, versão e datas. Não baixa URLs nem acrescenta fatos externos. Verifica vigência novamente quando necessário e não usa conteúdo vencido durante o fluxo.
4. **Limite total do contexto recuperado: 12000 caracteres Unicode e 4000 tokens, valendo o que for atingido primeiro**, incluindo serialização, metadados e delimitadores. Para cada chunk, serializar um bloco de dados escapado e incluir somente se couber integralmente; caso contrário descartá-lo e avaliar o próximo. Nunca cortar um trecho comercial ou sua ressalva para fazê-lo caber. Não contar o JSON duplicado de `content` como segunda evidência.
5. Além desse teto, a soma de instruções, pergunta, contexto e reserva para resposta deve caber na janela do modelo. Reservar até 1500 tokens de saída; modelo/tokenizador e janela devem ser validados na arquitetura. Se precisar reduzir contexto, descartar blocos inteiros e reavaliar suficiência. Nenhum chunk aproveitável gera CONHECIMENTO_INSUFICIENTE, sem geração livre.
6. Apresentar cada bloco como dados não confiáveis separados das instruções de sistema. Instruções dentro de texto, título ou fonte não podem selecionar ferramentas, alterar políticas, obter segredos ou mudar autorização. Restringir ferramentas no código. JSON/escapamento e prompt não são barreiras suficientes isoladamente.
7. Gerar somente a partir dos blocos selecionados e validar sustentação factual, conflitos e vigência, além do formato. Similaridade e presença de citação não provam suficiência. Pergunta sem suporte comercial recebe limitação explícita, sem estimativas.
8. O LLM pode indicar apenas IDs dos chunks de contexto para associar afirmações. A aplicação valida esses IDs e monta `fontes` copiando metadados do mapa recuperado, nunca aceitando título, URL, categoria ou ID inventado pelo modelo. ID fora do conjunto selecionado invalida a saída. Não listar todos os trechos recuperados como se todos tivessem sido usados.
9. Registrar internamente atendimento/correlação, IDs recuperados, IDs selecionados, IDs utilizados, modelo e versão do prompt, sem perguntas ou documentos completos nos logs. Nenhuma persistência de histórico conversacional é exigida neste MVP.

## 8. Prazos, tentativas e falhas seguras

Valores iniciais normativos por ambiente; alterações devem preservar o orçamento e ser refletidas em documentação/testes. Configurar via variáveis de ambiente, com limites finitos. Não afirmar cumprimento desses prazos sem execução dos testes.

| Operação | Limite |
|---|---|
| Atendimento REST completo | 30 s, incluindo busca, geração, validação e eventual reinicialização MCP |
| Conexão com MCP/OpenAI | 2 s por tentativa, contidos no prazo da operação |
| Inicialização/descoberta MCP quando necessária | 3 s totais, contidos no prazo do atendimento |
| Chamada buscar_conhecimento / diagnóstico | 8 s totais |
| Embedding de consulta dentro da busca | 5 s, contidos nos 8 s |
| Consulta de banco | 2 s, contidos na operação externa |
| Geração OpenAI | 18 s totais, contidos nos 30 s |
| Cadastro e indexação síncrona | 60 s totais; cada chamada de embeddings no máximo 20 s, contida nesses 60 s |
| GETs de conhecimento | 3 s totais |
| Probe health | 1 s, sem chamada paga a dependências |

O prazo restante sempre prevalece: os máximos não são somados para ampliar o total. Cancelar trabalho quando possível e impedir publicação indevida após timeout; cadastro com resultado incerto continua sujeito à reconciliação idempotente.

No MVP não há retry automático de geração, cadastro ou falha de autenticação/validação. Leituras MCP podem ter no máximo uma repetição para falha transitória de conexão, com espera de 100 ms, dentro dos mesmos 8 s e 30 s; considerar tentativas do SDK/Spring AI, sem multiplicação. Sessão inválida permite no máximo uma reinicialização no mesmo orçamento. Se não houver tempo, falhar imediatamente. Rate limit com Retry-After superior ao restante gera erro, sem espera prolongada. Correção automática de saída estruturada deve estar desabilitada neste contrato inicial para não introduzir chamadas ilimitadas/ocultas.

## 9. Validação e rastreabilidade dos contratos

- RF01–RF03: cadastro, idempotência, conflito, sobreposição e indexação íntegra; testar sem expor versão parcial em timeout.
- RF04–RF05: diagnóstico isolado e MCP real, com descoberta, schemas, limite padrão 5/máximo 10, ausência de score e lista vazia.
- RF06–RF09: REST de perguntas, busca obrigatória MCP, teto de contexto, referências montadas pela aplicação e ausência de dados vetoriais no atendimento.
- RF10: distinguir vazio/insuficiência de 502/503/504; verificar envelope de erros, JSON-RPC e tool errors separadamente.
- Testar autenticação, fonte invisível como 404, escopo insuficiente, campos inválidos, MIME e corpo excessivo; nenhum erro reproduz o valor recebido.
- Testar corpus hostil, fontes conflitantes/vencidas, chunk que excede orçamento e referências inventadas. Critérios funcionais exigem verificação semântica, não apenas schemas.
- Executar testes HTTP, integração com PostgreSQL/pgvector em Docker Desktop/Testcontainers e cliente/servidor MCP reais compatíveis com Spring AI 2.0.1. Gerar/validar OpenAPI das rotas REST durante a implementação; MCP mantém seus schemas próprios.

Permanecem pré-requisitos da arquitetura: emissor e delegação de identidade, versão de Spring Boot, modelos/tokenizador, calibração de relevância/suficiência, dimensões/índice e infraestrutura. Este contrato não comprova que esses componentes foram implementados ou testados.

## 10. Referências de protocolo

As distinções entre descoberta, chamada, `structuredContent`, erros de ferramenta e erros JSON-RPC seguem [MCP Tools — revisão 2025-06-18](https://modelcontextprotocol.io/specification/2025-06-18/server/tools). POST/GET, negociação de conteúdo, sessões e tratamento de notificações seguem [MCP Streamable HTTP — revisão 2025-06-18](https://modelcontextprotocol.io/specification/2025-06-18/basic/transports). Não trocar essas regras por um endpoint REST que apenas receba o nome da ferramenta.
