---
name: mcp-integration
description: Implementar e validar MCP Server e MCP Client com Spring AI 2.0.1 e Streamable HTTP na comunicação exclusiva entre api-atendimentos e api-conhecimento.
---

# Integração MCP sobre Streamable HTTP

## Quando utilizar

Usar ao expor pesquisa semântica como tool, configurar o cliente de conhecimento, descobrir e executar tools ou tratar segurança e falhas MCP. Não usar como autorização para criar ferramentas ou contratos fora da tarefa.

## Responsabilidades

- `api-conhecimento`: MCP Server de pesquisa, validação, autorização e execução da recuperação.
- `api-atendimentos`: MCP Client e adaptação dos resultados para o caso de uso, sem banco vetorial ou geração de embeddings.
- Manter o contrato de ferramentas, erros e transporte independente dos detalhes internos de cada serviço.

## Passo a passo de implementação

1. **Spec:** Ler `AGENTS.md` e as quatro especificações obrigatórias. Confirmar em `specs/06-api-contracts.md` nome e descrição da tool, esquemas, limites, autenticação, endpoint, versão do protocolo e resultados de erro. Lacunas bloqueiam a interface afetada; não inventar nomes ou payloads definitivos.
2. **Plan:** Escolher modo SYNC ou ASYNC e stack WebMVC ou WebFlux conforme arquitetura. Manter Spring AI 2.0.1 pelo BOM. Não bloquear event loop nem misturar tipos de cliente na mesma configuração.
3. No servidor, selecionar `spring-ai-starter-mcp-server-webmvc` ou `spring-ai-starter-mcp-server-webflux` e configurar `spring.ai.mcp.server.protocol=STREAMABLE`. Não substituir por SSE legado, STDIO ou protocolo STATELESS sem revisão da especificação.
4. Registrar explicitamente a tool de pesquisa por um único mecanismo suportado pela versão, como métodos `@Tool` com `MethodToolCallbackProvider` exposto como `ToolCallbackProvider`. Evitar registro duplicado por anotações e callbacks. Manter a operação em um caso de uso, não na camada de transporte.
5. Validar consulta não vazia, tamanho, limites de resultados e campos permitidos antes da pesquisa. Esquema de tool não substitui validação de runtime. Obter identidade de contexto autenticado e aplicar autorização no servidor, nunca confiar em escopo declarado livremente pelo modelo ou pelo usuário.
6. No cliente, usar `spring-ai-starter-mcp-client` ou sua variante `spring-ai-starter-mcp-client-webflux`. Configurar base URL e endpoint em `spring.ai.mcp.client.streamable-http.connections.<nome>.url` e `.endpoint`; definir `spring.ai.mcp.client.request-timeout` conforme orçamento aprovado. Externalizar endereços e credenciais.
7. Inicializar e negociar capacidades pelo SDK MCP; descobrir com `listTools()` e executar com `callTool(...)` no cliente do tipo escolhido, usando os tipos de requisição do SDK transitivo de 2.0.1. Conferir nomes e esquemas contra o contrato e restringir a allowlist. Tratar paginação da descoberta quando aplicável. A orquestração deve garantir busca antes da resposta factual, sem depender de o modelo decidir pesquisar.
8. Se a arquitetura adotar tool calling via ChatClient, adaptar somente tools permitidas com `SyncMcpToolCallbackProvider` ou o equivalente assíncrono da versão e registrar callbacks explicitamente. Descoberta automática não concede autorização de execução. Não expor ferramentas administrativas ou de ingestão ao atendimento.
9. Aplicar segurança no endpoint: os starters não fornecem autenticação/autorização por si. Usar HTTPS em ambientes implantados; validar credenciais, origem quando aplicável e isolamento de sessão. Propagar correlação e contexto de autorização de forma confiável, sem colocá-los como conteúdo não confiável do prompt.
10. Diferenciar falha HTTP/transporte, erro JSON-RPC, resultado da tool com indicação de erro e sucesso sem resultados. Aplicar timeouts de conexão e execução, prazo total e retries limitados para operações elegíveis; não converter indisponibilidade em ausência de conhecimento. Encerrar clientes e sessões pelo ciclo de vida gerenciado.
11. **Test e Review:** Testar servidor e cliente reais por Streamable HTTP, com descoberta, execução e segurança. Executar compilação e testes; revisar contrato, logs e ausência de caminhos alternativos.

## Padrões de codificação

- Adaptadores MCP separados do domínio; DTOs como records; injeção por construtor.
- Retornar somente trechos autorizados e referências previstas no contrato, sem vetores ou detalhes SQL.
- Usar propriedades e classes confirmadas na tag 2.0.1, verificando os tipos do SDK resolvido antes de escrever imports.
- Registrar duração, tool, resultado e correlação, sem argumentos completos ou credenciais.

## Antipadrões proibidos

- Atalho REST/SQL, fallback direto ao banco ou embeddings locais no cliente.
- Confundir GET com SSE opcional do Streamable HTTP com o transporte SSE legado.
- Expor todos os beans como ferramentas ou aceitar filtros/identidades arbitrários do modelo.
- Tratar qualquer HTTP 200 como sucesso da tool; retries sem limite ou timeouts ausentes.

## Critérios de validação

Comprovar descoberta e chamada pelo endpoint real e contrato aprovado; rejeitar parâmetros inválidos, tool desconhecida e acesso não autorizado. Testar timeout, sessão inválida, resposta malformada, erro de tool, indisponibilidade e resultado vazio. Verificar que o atendimento não consulta conhecimento por outro caminho e que não há vazamento entre usuários ou sessões.

## Checklist de conclusão

- [ ] Streamable HTTP e versão MCP conforme contrato.
- [ ] Registro único, esquemas e allowlist de tools verificados.
- [ ] Validação e autorização no servidor.
- [ ] Descoberta, execução e ciclo de vida cobertos.
- [ ] Timeouts, erros e resultados vazios distinguidos.
- [ ] Compilação e testes reais de integração executados e reportados.

## Referências da versão

[MCP Server 2.0.1](https://github.com/spring-projects/spring-ai/blob/v2.0.1/spring-ai-docs/src/main/antora/modules/ROOT/pages/api/mcp/mcp-server-boot-starter-docs.adoc) e [MCP Client 2.0.1](https://github.com/spring-projects/spring-ai/blob/v2.0.1/spring-ai-docs/src/main/antora/modules/ROOT/pages/api/mcp/mcp-client-boot-starter-docs.adoc). Conferir configurações e SDK nessa tag, não em exemplos da branch principal.
