# Requisitos — COTI AI Atendimento

## Objetivo e referências

A plataforma deve responder, em português, às dúvidas de alunos sobre cursos, programas, tecnologias, modalidades, pré-requisitos e informações institucionais da COTI Informática, utilizando exclusivamente conhecimento recuperado e autorizado. Não pode inventar valores, datas, horários, vagas, descontos ou condições comerciais.

Este documento segue `specs/01-constitution.md`, `AGENTS.md` e as skills `spring-ai`, `mcp-integration`, `rag-pgvector` e `testing`. O nome canônico permanece `specs/02-requiriments.md`, conforme orientação do solicitante; as referências operacionais foram reconciliadas em T001, sem renomear nem duplicar a especificação.

Os requisitos descrevem comportamento e dados lógicos. Endpoints REST, campos serializados, códigos HTTP, esquemas MCP e autenticação estão especificados em `specs/06-api-contracts.md`; este documento não os substitui. Decisões técnicas e valores operacionais constam em `specs/03-architecture.md`, cujos gates de qualificação continuam obrigatórios antes da implementação afetada. O plano é `specs/04-plans.md`.

## Atores e fronteiras

- **Responsável pelo conhecimento:** pessoa ou integração autorizada pela COTI a cadastrar fontes oficiais. É responsável pela procedência, classificação de acesso e vigência do conteúdo.
- **Operador de conhecimento:** pessoa ou integração autorizada a listar/detalhar documentos ou executar diagnóstico semântico, conforme o escopo de cada operação. A função não concede acesso irrestrito às fontes e não é exercida pela identidade de serviço da api-atendimentos.
- **Solicitante:** aluno ou consumidor autorizado da API de Atendimentos; sua identidade e permissões vêm de contexto autenticado, não de instruções na pergunta.
- **api-conhecimento:** cadastro REST, chunking, embeddings de documentos e consultas, PostgreSQL/pgvector, recuperação semântica e MCP Server.
- **api-atendimentos:** entrada REST de perguntas, MCP Client, composição do contexto, geração OpenAI e resposta com fontes. Não possui acesso ou dependência de PostgreSQL/pgvector.

## Regras transversais

1. Informações factuais da COTI somente podem ser apresentadas quando sustentadas por fontes recuperadas, autorizadas, relevantes e vigentes quando houver restrição temporal.
2. Informações comerciais ausentes, vencidas ou conflitantes não podem ser inferidas a partir da memória do modelo, de ofertas anteriores ou da insistência do usuário.
3. Documentos e perguntas são dados não confiáveis; seus comandos não alteram políticas, permissões ou ferramentas.
4. Toda busca de conhecimento do atendimento passa exclusivamente pela ferramenta MCP `buscar_conhecimento`, por Streamable HTTP. Não há fallback por REST de busca, SQL, índice local ou pesquisa externa.
5. Ausência de conhecimento, restrição de acesso e falha de dependência devem ser tratadas sem revelar conteúdo ou existência de fontes restritas. Falha técnica não equivale a busca sem resultados.
6. IDs, versões e referências devem ser produzidos ou validados pela aplicação; o modelo não é autoridade para criá-los.
7. Os cenários Given/When/Then abaixo são critérios de aceite a implementar e executar, não evidências de testes já realizados.

## Requisitos funcionais

### RF01 — Cadastrar documentos textuais via API REST

**Identificador:** RF01. **Responsável:** api-conhecimento.

**Descrição:** Permitir que um responsável autorizado cadastre conteúdo textual oficial para disponibilização na base consultável.

**Entrada:** Texto, título, referência de origem, identificador/versão de origem quando disponíveis e metadados permitidos de classificação, atualização e vigência. Identidade e permissão de cadastro provenientes da autenticação.

**Processamento:** Autenticar e autorizar; validar campos e limites; normalizar texto sem alterar seu significado; atribuir identificação e versão; iniciar RF02 e RF03. Não disponibilizar documentos parcialmente processados para pesquisa.

**Saída:** Identificador do documento, versão, resultado do cadastro e situação da indexação, ou erro explícito. Se houver processamento assíncrono aprovado, o contrato deve definir como acompanhar seu resultado; recebimento não significa indexação concluída.

**Regras de negócio:** Somente texto no MVP, sem extração de arquivos binários. Conteúdo vazio ou origem ausente é inválido. Classificação de acesso deve ser autorizada no servidor. Reenvio da mesma origem/versão e conteúdo deve ser idempotente; divergência de conteúdo sob a mesma versão deve ser sinalizada, nunca sobrescrita silenciosamente.

**Critérios de aceite:**

- **Given** responsável autorizado e documento textual válido, **When** o cadastro e a indexação terminam, **Then** retornar ID/versão e permitir busca apenas do conjunto integralmente indexado.
- **Given** texto vazio, campos fora dos limites ou solicitante sem permissão, **When** ocorrer o cadastro, **Then** rejeitar a operação sem chamar embeddings nem publicar chunks.
- **Given** a mesma origem/versão e conteúdo já cadastrados, **When** houver reenvio, **Then** não criar duplicatas pesquisáveis.

### RF02 — Dividir documentos em chunks com sobreposição

**Identificador:** RF02. **Responsável:** api-conhecimento.

**Descrição:** Dividir textos em trechos limitados por tokens, preservando contexto entre trechos adjacentes.

**Entrada:** Documento validado, versão, metadados, tamanho máximo C e sobreposição O em tokens, definidos na configuração aprovada.

**Processamento:** Aplicar tokenização compatível com o modelo e divisão determinística. Para documentos maiores que C, utilizar janelas com avanço C − O; cada janela subsequente inclui os O tokens finais da anterior, até a última janela. Não gerar janela sem conteúdo novo. Vincular cada chunk ao documento, versão e posição.

**Saída:** Chunks identificados, ordenados e rastreáveis, com tamanho limitado e metadados herdados de forma segura.

**Regras de negócio:** Exigir 0 < O < C e C compatível com o limite do modelo. Documento com até C tokens gera um único chunk. O texto normalizado deve ser integralmente coberto, descontada a repetição intencional. A estratégia deve funcionar com Spring AI 2.0.1 sem presumir suporte nativo a overlap em APIs posteriores.

**Critérios de aceite:**

- **Given** texto com mais de C tokens e configuração válida, **When** for dividido, **Then** cada chunk terá no máximo C tokens e chunks adjacentes compartilharão os O tokens previstos, sem perda do texto normalizado.
- **Given** documento curto, **When** for dividido, **Then** produzir um único chunk, sem vazios ou duplicação adicional.
- **Given** O igual a zero, O ≥ C ou C inválido, **When** a configuração for validada, **Then** impedir processamento e informar a configuração inválida.

### RF03 — Gerar embeddings com OpenAI e persistir no pgvector

**Identificador:** RF03. **Responsável:** api-conhecimento.

**Descrição:** Converter chunks em embeddings OpenAI e persistir vetores e metadados no PostgreSQL com pgvector.

**Entrada:** Chunks de RF02, modelo de embeddings, dimensão e configuração do índice aprovados; credenciais externas.

**Processamento:** Gerar embeddings via Spring AI, armazenar usando PgVectorStore e verificar consistência de dimensão, IDs e versões. Controlar falhas parciais e publicar a versão para busca somente após conclusão íntegra.

**Saída:** Confirmação da versão indexada com documento e chunks associados, ou falha de indexação rastreável.

**Regras de negócio:** Embeddings ficam exclusivamente no conhecimento. Não misturar modelos ou dimensões incompatíveis. Atualização de origem deve evitar servir simultaneamente versões obsoletas como atuais. Reindexação necessária por mudança de modelo deve preservar a versão íntegra anterior até a troca validada, quando existente. Não gerar vetores fictícios em produção.

**Critérios de aceite:**

- **Given** chunks válidos e dependências disponíveis, **When** a indexação concluir, **Then** todos terão embeddings de dimensão esperada e metadados persistidos recuperáveis.
- **Given** falha de OpenAI ou banco durante a operação, **When** o processamento terminar com erro, **Then** nenhuma versão parcial será pesquisável e o erro não será apresentado como sucesso.
- **Given** dimensão incompatível com o índice, **When** houver tentativa de persistência, **Then** rejeitar a operação e indicar necessidade de correção/reindexação sem corromper o índice ativo.

### RF04 — Consultar documentos por similaridade semântica

**Identificador:** RF04. **Responsável:** api-conhecimento.

**Descrição:** Recuperar chunks semanticamente relevantes à consulta dentro do escopo autorizado.

**Entrada:** Consulta textual, contexto de autorização confiável e parâmetros de recuperação limitados pela política do servidor.

**Processamento:** Validar consulta; gerar seu embedding no conhecimento; executar similaritySearch com top-K, limiar de relevância e filtros de autorização/vigência; ordenar e limitar resultados. Avaliar conflitos e insuficiência sem tratar score como garantia de veracidade.

**Saída:** Conjunto de trechos com referência de documento, versão, chunk, origem e metadados permitidos, incluindo score quando previsto no contrato; ou resultado vazio/insuficiente explícito.

**Regras de negócio:** Aplicar autorização na recuperação antes de entregar conteúdo ao cliente ou modelo. Não aceitar filtros arbitrários ou identidade declarada pelo usuário. Top-K e limiar devem ser calibrados em corpus de referência; não diminuir requisitos de relevância para forçar resultado. RF04 é capacidade interna exposta por RF05, sem endpoint REST de busca para o atendimento.

**Critérios de aceite:**

- **Given** consulta e chunks relevantes autorizados no corpus de referência, **When** a busca executar, **Then** retornar os resultados esperados dentro do limite e com referências corretas.
- **Given** documentos restritos, vencidos ou abaixo do limiar, **When** a busca executar, **Then** excluí-los sem revelar sua existência ao solicitante.
- **Given** nenhuma evidência relevante, **When** a pesquisa concluir normalmente, **Then** retornar ausência de evidências, distinta de erro de banco ou embeddings.

### RF05 — Disponibilizar ferramenta MCP buscar_conhecimento

**Identificador:** RF05. **Responsável:** api-conhecimento.

**Descrição:** Expor a pesquisa semântica pela tool MCP com nome exato `buscar_conhecimento`.

**Entrada:** Requisição MCP com argumentos de pesquisa previstos no contrato e contexto autenticado.

**Processamento:** Atender inicialização e descoberta; publicar esquema da ferramenta; autenticar, autorizar e validar argumentos em runtime; executar RF04; traduzir resultado ou erro ao contrato MCP.

**Saída:** Tool descobrível com esquema aprovado e resultado estruturado com trechos e fontes, ou erro explícito conforme a camada responsável.

**Regras de negócio:** Utilizar Streamable HTTP e segurança explícita do endpoint. Não expor cadastro, credenciais ou ferramentas administrativas ao atendimento. Distinguir erro de protocolo, erro de execução da tool e sucesso sem resultados. Os nomes dos campos e a versão do protocolo serão definidos em `specs/06-api-contracts.md`.

**Critérios de aceite:**

- **Given** cliente autenticado compatível, **When** inicializar e listar tools, **Then** descobrir `buscar_conhecimento` com o esquema aprovado.
- **Given** chamada válida da tool, **When** RF04 concluir, **Then** devolver resultado conforme contrato pelo transporte Streamable HTTP.
- **Given** argumento inválido ou acesso não autorizado, **When** houver invocação, **Then** rejeitar antes da recuperação sem expor conteúdo interno.

### RF06 — Receber perguntas na API de Atendimentos

**Identificador:** RF06. **Responsável:** api-atendimentos.

**Descrição:** Receber uma pergunta textual por API REST e iniciar um atendimento independente.

**Entrada:** Pergunta e campos permitidos pelo contrato; identidade e permissões provenientes de autenticação.

**Processamento:** Validar conteúdo, tamanho e autenticação; atribuir identificador de atendimento e correlação; iniciar RF07 e, havendo evidências suficientes, RF08 e RF09.

**Saída:** Resultado do atendimento com identificadores e, conforme o desfecho, resposta fundamentada com fontes, insuficiência ou erro tratado.

**Regras de negócio:** Perguntas vazias ou acima do limite são rejeitadas antes de MCP/OpenAI. O MVP não exige histórico conversacional: perguntas devem ser autocontidas; ambiguidade pode gerar pedido de esclarecimento. Perguntas completas não são registradas por padrão. Instruções do usuário não concedem permissão ou fonte oficial.

**Critérios de aceite:**

- **Given** solicitante autorizado e pergunta válida, **When** a API receber a requisição, **Then** iniciar atendimento rastreável e executar o fluxo MCP.
- **Given** pergunta vazia, excessiva ou acesso inválido, **When** a requisição for validada, **Then** rejeitar sem chamadas externas de pesquisa ou geração.
- **Given** duas requisições independentes, **When** forem processadas, **Then** não compartilhar contexto ou evidências entre solicitantes.

### RF07 — Consultar a ferramenta MCP da API de Conhecimento

**Identificador:** RF07. **Responsável:** api-atendimentos.

**Descrição:** Recuperar conhecimento exclusivamente invocando `buscar_conhecimento` no MCP Server da api-conhecimento.

**Entrada:** Pergunta validada, correlação e contexto de autorização confiável.

**Processamento:** Inicializar conexão e verificar disponibilidade/esquema da tool; executar chamada MCP com limites e prazo; validar o resultado recebido e separar evidência disponível, insuficiência e falha.

**Saída:** Evidências autorizadas com fontes para RF08, ou desfecho tratado por RF10.

**Regras de negócio:** A pesquisa deve anteceder qualquer resposta factual da COTI. A decisão de pesquisar não pode depender exclusivamente da escolha do modelo. Não consultar banco, gerar embedding local, usar índice em memória ou contornar MCP por REST. Somente a ferramenta autorizada pode ser invocada; credenciais não integram argumentos livres do modelo.

**Critérios de aceite:**

- **Given** pergunta válida e tool disponível, **When** o atendimento buscar conhecimento, **Then** invocar `buscar_conhecimento` por Streamable HTTP antes da geração factual.
- **Given** MCP indisponível ou ferramenta/esquema incompatível, **When** a consulta falhar, **Then** acionar RF10 sem caminho alternativo de busca.
- **Given** resposta MCP malformada ou com campos obrigatórios ausentes, **When** for recebida, **Then** tratá-la como falha de integração, não como evidência válida.

### RF08 — Construir contexto RAG e gerar resposta com OpenAI

**Identificador:** RF08. **Responsável:** api-atendimentos.

**Descrição:** Gerar resposta clara em português usando Spring AI/OpenAI e apenas as evidências recuperadas autorizadas.

**Entrada:** Pergunta, trechos e referências válidos de RF07, políticas de resposta e limites de contexto.

**Processamento:** Selecionar e organizar evidências sem ultrapassar orçamento de contexto; separar instruções de dados não confiáveis; avaliar suficiência e conflitos; gerar resposta e validar estrutura, referências e sustentação das afirmações antes de liberá-la.

**Saída:** Resposta fundamentada com ligação às evidências utilizadas, ou limitação/erro previsto em RF10.

**Regras de negócio:** Score elevado e JSON válido não bastam para assegurar suporte factual. Não completar informações com memória do modelo. Não inventar valores, datas, horários, vagas ou condições comerciais. Se somente parte de uma pergunta tiver suporte, responder apenas essa parte com a limitação explícita; não apresentar resposta parcial como completa. Contradições não resolvidas impedem conclusão sobre o ponto conflitante.

**Critérios de aceite:**

- **Given** evidências suficientes sobre programa e pré-requisitos de um curso, **When** a resposta for gerada, **Then** suas afirmações corresponderão às fontes autorizadas, sem acréscimos comerciais.
- **Given** pergunta sobre preço ou vagas sem suporte, **When** houver processamento, **Then** declarar a falta de informação sem estimativas ou números inventados.
- **Given** documento que ordena ignorar regras ou revelar segredos, **When** integrar o contexto, **Then** o comando não alterará políticas, ferramentas ou permissões.
- **Given** saída com referência inexistente ou afirmação sem evidência, **When** ocorrer validação, **Then** não liberar essa saída como resposta fundamentada.

### RF09 — Devolver fontes, identificadores e metadados utilizados

**Identificador:** RF09. **Responsáveis:** ambas as APIs, com composição final na api-atendimentos.

**Descrição:** Permitir ao consumidor e à operação rastrear as evidências efetivamente utilizadas em cada resposta.

**Entrada:** Resultado da recuperação, seleção do contexto e referências associadas às afirmações liberadas.

**Processamento:** Preservar e validar IDs de documento, versão e chunk, origem e metadados permitidos; associar referências às partes da resposta; distinguir trechos recuperados, enviados ao modelo e citados. Deduplicar fontes sem perder a relação com os chunks utilizados.

**Saída:** Resposta com ID de atendimento/correlação, referências verificáveis e metadados autorizados de origem, versão e atualização/vigência quando disponíveis. A rastreabilidade interna registra modelo e versão do prompt; sua exposição pública depende do contrato.

**Regras de negócio:** Não apresentar todos os resultados de busca como se sustentassem a resposta. Nenhuma referência pode ser inventada pelo modelo ou apontar conteúdo restrito. Fonte sem URL pública pode ser identificada por referência lógica; não fabricar links. Não expor vetores, credenciais, localização interna do banco ou metadados sensíveis.

**Critérios de aceite:**

- **Given** resposta fundamentada em dois chunks, **When** retornar ao consumidor, **Then** incluir referências que identifiquem os documentos, versões e chunks realmente utilizados.
- **Given** resultados recuperados e descartados do contexto, **When** montar as fontes da resposta, **Then** não atribuir a eles sustentação de afirmações que não apoiaram.
- **Given** origem sem URL pública ou metadado não permitido, **When** serializar a resposta, **Then** usar referência lógica válida e omitir dados restritos.

### RF10 — Tratar ausência de evidência ou falhas de dependências

**Identificador:** RF10. **Responsáveis:** ambas as APIs em suas fronteiras; api-atendimentos na resposta final.

**Descrição:** Produzir desfechos explícitos e seguros quando não for possível responder com evidências suficientes ou concluir o processamento técnico.

**Entrada:** Resultado vazio, insuficiente, ambíguo ou conflitante; falhas de OpenAI, MCP ou PostgreSQL; recusa/saída inválida do modelo; timeout ou prazo total excedido.

**Processamento:** Classificar o desfecho, interromper etapas sem condições de execução e aplicar tentativas limitadas apenas a falhas elegíveis dentro do orçamento total. Registrar causa técnica sanitizada e correlação. Não consultar fontes alternativas não autorizadas.

**Saída:** Limitação de conhecimento, pedido de esclarecimento ou erro técnico distinto, com identificador rastreável e mensagem segura, conforme contrato. Orientação para atendimento humano somente se houver canal oficial cadastrado e autorizado.

**Regras de negócio:** Falha de dependência não significa que a COTI não oferece um curso. Não revelar existência de documentos inacessíveis. Sem evidência alguma, devolver mensagem controlada sem depender de nova geração OpenAI. Não devolver stack trace ou credenciais, nem fontes fictícias. Falha de indexação não publica versão incompleta.

**Critérios de aceite:**

- **Given** busca concluída sem evidências, **When** finalizar o atendimento, **Then** informar insuficiência sem resposta factual inventada e sem chamar geração para produzir uma resposta livre.
- **Given** timeout MCP, falha OpenAI ou banco indisponível, **When** o prazo/limite de tentativas for atingido, **Then** retornar falha técnica rastreável, distinta de ausência de conhecimento.
- **Given** fontes contraditórias sem precedência válida, **When** responder sobre o ponto conflitante, **Then** explicitar a limitação em vez de escolher arbitrariamente um valor.
- **Given** canal humano não cadastrado, **When** ocorrer insuficiência, **Then** não inventar telefone, endereço ou link de encaminhamento.

### RF11 — Listar documentos para operação autorizada

**Identificador:** RF11. **Responsável:** api-conhecimento.

**Descrição:** Listar resumos de documentos ativos e visíveis para operador autorizado, conforme `GET /api/conhecimentos` da seção 4 do contrato. Não é recuperação semântica para atendimento.

**Entrada:** Contexto autenticado com escopo `conhecimento:ler` e parâmetros de paginação/categoria permitidos pelo contrato.

**Processamento:** Validar parâmetros; filtrar autorização e versões ativas antes de contar e paginar; ordenar por criação decrescente e ID crescente. Não gerar embeddings nem executar chat.

**Saída:** Página de resumos com contagens somente do conjunto visível; página além do fim retorna lista vazia. Não expor texto integral, vetores, credenciais ou detalhes internos.

**Regras de negócio:** A identidade de serviço da api-atendimentos não pode listar, inclusive em falha MCP. Vigência e estado ativo são distintos: não acrescentar filtro de vigência não previsto para a listagem administrativa. Campos, limites, defaults e erros são os do contrato.

**Critérios de aceite:**

- **Given** operador autorizado e documentos ativos visíveis/invisíveis, **When** listar, **Then** retornar apenas resumos visíveis, com contagens e paginação posteriores à autorização.
- **Given** categoria válida ou página além do fim, **When** consultar, **Then** aplicar filtro/ordenação contratados ou retornar página vazia, sem 404 pelo fim da paginação.
- **Given** parâmetros inválidos/desconhecidos ou identidade sem permissão, incluindo atendimento, **When** listar, **Then** rejeitar conforme contrato, sem revelar documentos ou contagens restritas.

### RF12 — Detalhar documento para operação autorizada

**Identificador:** RF12. **Responsável:** api-conhecimento.

**Descrição:** Obter documento ativo e visível e seu texto por `GET /api/conhecimentos/{id}`, conforme seção 4 do contrato.

**Entrada:** UUID do documento e contexto autenticado com escopo `conhecimento:ler`.

**Processamento:** Validar identidade e UUID, localizar documento ativo e aplicar autorização antes de expor conteúdo ou metadados. Não gerar embeddings nem executar chat.

**Saída:** Detalhe com texto e metadados permitidos, sem vetores ou informações internas. Documento inexistente, invisível ou inativo retorna o mesmo desfecho 404.

**Regras de negócio:** A rota não oferece histórico, edição ou exclusão e é proibida à identidade da api-atendimentos. A leitura administrativa de versão ativa não implica vigência para recuperação semântica; preservar a distinção contratual.

**Critérios de aceite:**

- **Given** operador autorizado e documento ativo visível, **When** detalhar, **Then** retornar texto e metadados contratuais, sem vetores ou dados internos.
- **Given** documento inexistente, invisível ou inativo, **When** operador com permissão para a operação consultar, **Then** retornar 404 indistinguível, sem revelar existência de fonte restrita.
- **Given** UUID inválido ou identidade sem permissão, incluindo atendimento, **When** detalhar, **Then** rejeitar conforme contrato antes de expor conteúdo.

### RF13 — Diagnosticar recuperação semântica em rede administrativa

**Identificador:** RF13. **Responsável:** api-conhecimento.

**Descrição:** Avaliar recuperação pelo endpoint `POST /api/admin/conhecimentos/busca-semantica`, conforme seção 4 do contrato.

**Entrada:** Consulta/limite contratuais no corpo, identidade autenticada com escopo `conhecimento:diagnosticar` e acesso pela rede administrativa.

**Processamento:** Validar rede, autenticação, autorização e argumentos; executar o mesmo caso de uso de RF04, com embedding de consulta e filtros de autorização/publicação/vigência. Não ampliar permissões do operador, chamar chat ou baixar fontes.

**Saída:** Trechos/referências do schema de pesquisa contratado em JSON REST sem envelope JSON-RPC; lista vazia é sucesso distinto de falha técnica.

**Regras de negócio:** Essa rota não é MCP e não altera RF05/RF07. Atendimento não pode acessá-la, inclusive quando MCP estiver indisponível. Não publicá-la como tool adicional ou ferramenta administrativa ao modelo. Consulta no corpo evita exposição em URLs; logs não registram pergunta completa por padrão.

**Critérios de aceite:**

- **Given** operador na rede administrativa com escopo e consulta válidos, **When** diagnosticar, **Then** executar RF04 e retornar somente chunks ativos, autorizados e vigentes, com referências e limites contratuais.
- **Given** consulta sem resultados ou falha de embeddings/banco, **When** diagnosticar, **Then** distinguir lista vazia bem-sucedida de erro técnico sanitizado/rastreável.
- **Given** acesso fora da rede administrativa, sem escopo, com argumentos inválidos ou pela identidade do atendimento, **When** invocar, **Then** rejeitar sem recuperação indevida ou conteúdo restrito.

## Rastreabilidade da reconciliação T001

RF11–RF13 explicitam capacidades já presentes na seção 4 de `06-api-contracts.md`; não introduzem rotas, campos, permissões ou comportamentos novos nesse contrato. RF01–RF10 e RNF01–RNF11 conservam seus identificadores e obrigações. A identidade de serviço do atendimento continua limitada à busca MCP; acesso REST administrativo não constitui fallback.

| Requisito | Contrato existente | Plano/backlog | Matriz de aceite |
|---|---|---|---|
| RF11 — listagem | GET /api/conhecimentos | Fase 3 / T023 | A detalhar em T002 |
| RF12 — detalhe | GET /api/conhecimentos/{id} | Fase 3 / T023 | A detalhar em T002 |
| RF13 — diagnóstico | POST /api/admin/conhecimentos/busca-semantica | Fase 5 / T038 | A detalhar em T002 |

Impacto: alinhamento documental aditivo nos requisitos, com critérios de aceite para operações já contratadas. Não há alteração do contrato v1, migração de consumidores, mudança de dados ou reindexação decorrente desta reconciliação. Testes dessas capacidades permanecem obrigatórios antes da entrega; não foram executados nesta tarefa documental.

## Requisitos não funcionais

| ID | Categoria | Requisito obrigatório | Validação de aceite |
|---|---|---|---|
| RNF01 | Segurança | Autenticar e autorizar cadastro, perguntas e ferramentas; validar identidade/escopo no servidor; aplicar menor privilégio e HTTPS em ambientes implantados. Banco de conhecimento inacessível à api-atendimentos. | Testes positivos e negativos de acesso, falsificação de escopo e isolamento entre solicitantes; inspeção de permissões e rede sem vazamento. |
| RNF02 | Observabilidade | Logs estruturados, métricas e traces correlacionados entre REST, MCP, recuperação e geração. Medir duração, erros, timeouts, resultados insuficientes e consumo de modelos. Não registrar segredos, documentos ou perguntas completas por padrão; definir acesso e retenção. | Acompanhar atendimento ponta a ponta e falhas simuladas; verificar correlação, indicadores e ausência de conteúdo sensível. |
| RNF03 | Manutenibilidade | Java 21, Spring AI 2.0.1 e Spring Boot com compatibilidade comprovada e versão definida na arquitetura; módulos com responsabilidades claras, dependências controladas, records para DTOs e injeção por construtor como preferências. | Build reproduzível, revisão arquitetural e dependências resolvidas; justificativas para exceções; nenhum contrato alterado silenciosamente. |
| RNF04 | Tempo limite das integrações | Todas as chamadas MCP, OpenAI e banco devem ter timeout finito; definir prazo total de atendimento e indexação, limites de tentativas e backoff. Tentativas e esperas devem caber no prazo total e considerar retries do SDK. Cancelamento deve impedir publicação tardia indevida. | Com dependência lenta, finalizar até o prazo configurado mais tolerância técnica aprovada; comprovar contagem de tentativas e ausência de loop ilimitado. Valores e tolerância devem ser definidos na arquitetura antes da implementação. |
| RNF05 | Validação de entrada | Validar tipos, obrigatoriedade, tamanho de documentos/perguntas, limites de tokens, top-K, sobreposição, referências e metadados. Rejeitar filtros, ferramentas e campos não permitidos conforme o contrato antes de operações externas. | Casos de limite, vazio, formato inválido e argumentos maliciosos sem chamadas indevidas ou gravação parcial. |
| RNF06 | Testabilidade | Critérios automatizados com JUnit 5 e Mockito; integração real de PostgreSQL/pgvector via Testcontainers e Docker Desktop, HTTP e MCP Streamable HTTP; cenários de falhas, insuficiência e prompt injection. Avaliações reais OpenAI separadas e controladas. | Executar compilação e suítes relevantes em ambiente definido; registrar contagens, falhas e ignorados. Mocks não substituem evidências de integração. Nunca afirmar sucesso sem execução. |
| RNF07 | Separação dos microsserviços | Build, configuração e implantação independentes. Somente api-conhecimento possui PostgreSQL/pgvector, chunking e embeddings; toda pesquisa do atendimento usa MCP. Nenhum compartilhamento de entidades de persistência ou acesso interno ao outro serviço. | Inspeção de dependências/configurações e testes de arquitetura; atendimento inicia sem credenciais de banco e retorna erro tratado quando MCP falha. |
| RNF08 | Prompt injection | Perguntas, fontes e metadados são dados não confiáveis. Delimitar contexto, limitar ferramentas e verificar permissões no código. Comandos documentais não alteram políticas nem autorizam divulgação ou ações. | Corpus hostil tentando inventar preços, exfiltrar marcadores fictícios ou executar ferramentas indevidas; nenhum comando amplia acesso ou libera resposta sem suporte. |
| RNF09 | Variáveis de ambiente | Credenciais obrigatoriamente externas, por variáveis de ambiente, inclusive via gerenciador de segredos. Endereços, modelos, limites e timeouts devem ser configuráveis por ambiente. Falhar explicitamente para configuração obrigatória ausente/inválida, sem segredos padrão. | Inicialização sem configuração obrigatória falha de forma sanitizada; troca de valores sem mudar código; varredura de segredos e inspeção de imagens/configurações. |
| RNF10 | Contratos e documentação | Documentar interfaces REST em OpenAPI e ferramenta MCP em esquema próprio; respeitar `specs/06-api-contracts.md`. Informar motivos e atualizar especificações antes de mudar contratos aprovados. | Validação de esquemas e testes de produtor/consumidor, cobrindo desfechos de RF10; documentação consistente com comportamento real. |
| RNF11 | Infraestrutura de dados | Construir e executar containers e serviços de banco via Docker Desktop, com imagens fixadas, migrações e healthchecks. Não substituir silenciosamente por instalação local nem remover volumes persistentes sem autorização. | Subir ambiente reproduzível, verificar extensão pgvector e migrações; executar integração em banco isolado sem alterar dados externos. |

## Escopo do MVP

Todos os requisitos RF01–RF13 e RNF01–RNF11 são obrigatórios no MVP, nas seguintes fronteiras:

- Cadastro de textos por responsáveis autorizados, com origem, versão e política de acesso; normalização, chunks sobrepostos e indexação íntegra e idempotente.
- Listagem/detalhe de documentos ativos e visíveis e diagnóstico semântico para operação autorizada, conforme RF11–RF13; rotas proibidas à identidade de serviço da api-atendimentos.
- Base PostgreSQL/pgvector exclusiva da api-conhecimento, embeddings OpenAI, recuperação semântica autorizada e ferramenta `buscar_conhecimento` por Streamable HTTP.
- Atendimento REST por pergunta independente em português, com contexto RAG, resposta OpenAI, fontes verificáveis e desfechos explícitos de insuficiência e falha.
- Respostas sobre cursos, programas, tecnologias, modalidades, pré-requisitos e informações institucionais. Informações comerciais somente quando explicitamente presentes em fonte autorizada e vigente; nenhum cálculo de oferta, previsão ou confirmação de disponibilidade em tempo real.
- Segurança, observabilidade, configuração externa, documentação de contratos e testes automatizados; ambiente de dados em containers via Docker Desktop.

## Fora do escopo do MVP

- Interface web, aplicativo móvel, WhatsApp, voz, e-mail e integração com redes sociais.
- Importação de PDF, Word, áudio, imagens, OCR, crawler e busca na internet.
- Histórico conversacional persistente, memória entre atendimentos, personalização por perfil e atendimento multilíngue.
- Matrículas, pagamentos, reservas, atualização de vagas, cotações, descontos e execução de transações comerciais.
- Integrações com ERP/CRM e abertura automática de chamados; é permitida apenas orientação para canal oficial já cadastrado.
- Console editorial, workflow de aprovação, edição/exclusão REST e sincronização automática de fontes. A exclusão desses canais não elimina os controles internos de versão, vigência e publicação íntegra exigidos no MVP.
- Fine-tuning, treinamento de modelos próprios, provedores alternativos e arquitetura multiagente.
- Alta disponibilidade, múltiplas regiões e metas de escala não especificadas. Timeouts, isolamento e tratamento de falhas continuam obrigatórios.

## Decisões necessárias antes da implementação

1. Referências e escopo administrativo reconciliados em T001, mantendo `specs/02-requiriments.md` como fonte única. A matriz de testes/aceite, incluindo RF11–RF13, permanece para T002.
2. Definir na arquitetura a versão compatível de Spring Boot, modelos OpenAI, dimensões/índice, orçamento de contexto, tamanhos de entrada, C/O, top-K e critérios de relevância e suficiência.
3. Definir autenticação, matriz de permissões, responsáveis pelas fontes, regras de vigência e tratamento de versões conflitantes.
4. Especificar contratos REST/MCP, transporte Streamable HTTP, versão MCP, schemas, IDs/metadados públicos, erros e modo síncrono ou assíncrono da ingestão.
5. Fixar timeouts, prazo total, tentativas/backoff, tolerância dos testes, imagens Docker e política de acesso/retenção da telemetria.
6. Montar corpus sintético de avaliação, perguntas e evidências esperadas; estabelecer métricas e limiares de qualidade antes da validação de entrega.

Essas decisões detalham os requisitos e não autorizam funcionalidades adicionais. A especificação documental pode ser concluída com essas pendências explícitas; a implementação dependente delas não deve presumir valores ou contratos.
