---
name: spring-ai
description: Configurar OpenAI e implementar geração de respostas com Spring AI 2.0.1 no COTI AI Atendimento, incluindo ChatClient, prompts, saída estruturada e falhas do provedor.
---

# Spring AI e OpenAI

## Quando utilizar

Usar em tarefas de configuração de modelos, geração contextualizada, prompts, respostas estruturadas e tratamento de falhas de OpenAI. Para recuperação e embeddings, combinar com `skills/rag-pgvector/SKILL.md`; para acesso ao conhecimento, consultar `skills/mcp-integration/SKILL.md`.

## Responsabilidades

- Concentrar geração de respostas na `api-atendimentos` e embeddings na `api-conhecimento`.
- Manter políticas de resposta na aplicação, isolando o provedor em adaptadores testáveis.
- Preservar autorização, evidências, rastreabilidade e configuração segura.

## Passo a passo de implementação

1. **Spec:** Ler `AGENTS.md`, constituição, requisitos, arquitetura e `specs/06-api-contracts.md`. Não implementar contratos ou versões de Spring Boot que estejam indefinidos. Aplicar Java 21 e Spring AI 2.0.1, sem copiar APIs de versões posteriores.
2. **Plan:** Identificar modelos e capacidades necessários, orçamento de contexto, saída esperada e limites de tempo e tentativas. Usar o BOM `org.springframework.ai:spring-ai-bom:2.0.1` e o starter `spring-ai-starter-model-openai`, conforme o build aprovado.
3. Configurar `spring.ai.openai.api-key` por variável de ambiente e os modelos em `spring.ai.openai.chat.options.model` e `spring.ai.openai.embedding.options.model` somente no serviço responsável. Desabilitar modalidades não utilizadas conforme as propriedades da versão. Validar configuração obrigatória na inicialização, sem valores secretos padrão.
4. Obter `org.springframework.ai.chat.client.ChatClient.Builder` por injeção e construir o cliente no adaptador ou configuração. Preservar a autoconfiguração e sua observabilidade. Utilizar `prompt()`, mensagens `system` e `user`, `call()` e `content()` ou `chatResponse()` conforme a necessidade.
5. Recuperar evidências autorizadas via MCP antes de elaborar uma resposta factual. Tratar resultados insuficientes ou conflitantes no caso de uso. Separar instruções da aplicação dos documentos, delimitando estes como dados não confiáveis e limitando seu tamanho; não interpolar documentos como instruções de sistema.
6. Para DTO de saída, preferir record e conversão com `call().entity(Tipo.class)`. Quando o modelo suportar, avaliar `useProviderStructuredOutput()` e `validateSchema()` nas opções de `entity` da versão 2.0.1. Validar ainda campos, referências e regras de domínio: JSON válido não comprova veracidade. Limitar também as tentativas de correção de esquema.
7. Distinguir timeout, autenticação inválida, limitação de taxa, falha transitória, recusa do modelo, resposta vazia e saída inválida. Mapear ao contrato sem retornar detalhes internos. Aplicar retries limitados apenas a falhas elegíveis e dentro do prazo total; considerar as tentativas do SDK e do Spring AI para evitar multiplicação e custo inesperado.
8. **Test e Review:** Executar compilação e testes relevantes. Revisar configurações, conteúdo dos logs, fontes utilizadas, insuficiência e resistência a prompt injection. Reportar evidências e pendências conforme `AGENTS.md`.

## Padrões de codificação

- Injeção por construtor; DTOs imutáveis; configuração tipada e validada; prompts versionados e testáveis.
- Identificar modelo, versão do prompt, fontes e correlação sem registrar perguntas completas, documentos ou segredos por padrão.
- Validar que toda referência retornada pertence ao conjunto autorizado recuperado. Rejeitar saída sem sustentação; citações sozinhas não garantem suporte factual.
- Tratar parâmetros específicos de modelos como configuração, sem assumir que todos aceitam temperatura ou saída estruturada.

## Antipadrões proibidos

- Criar embeddings ou acessar PostgreSQL/pgvector na `api-atendimentos`.
- Responder sobre cursos, preços, datas e horários a partir da memória do modelo ou preencher lacunas por suposição.
- Usar advisors de vector store local no atendimento para contornar MCP.
- Hardcode de credenciais, log de prompts completos, retries ilimitados ou captura de exceções convertida em resposta de sucesso.
- Aceitar instruções documentais para trocar políticas, executar ferramentas ou divulgar contexto restrito.

## Critérios de validação

Comprovar configuração correta e falha explícita sem credenciais; testar resposta contextualizada e referência desconhecida; simular saída malformada, recusa, timeout, erro de autenticação, 429 e 5xx. Verificar que ausência de evidências impede resposta factual e que documentos hostis não ampliam permissões. Usar a skill de testes para separar testes determinísticos de avaliações reais com OpenAI.

## Checklist de conclusão

- [ ] Especificações e versões respeitadas; somente o escopo implementado.
- [ ] Configuração externa e modalidades isoladas por serviço.
- [ ] Prompts delimitam documentos e não expõem segredos.
- [ ] Saída estruturada e fontes validadas além do formato.
- [ ] Falhas, prazos, retries e insuficiência cobertos por testes.
- [ ] Compilação e testes executados; relatório final fiel às evidências.

## Referências da versão

Consultar a documentação da tag fixa antes de alterar chamadas: [ChatClient 2.0.1](https://github.com/spring-projects/spring-ai/blob/v2.0.1/spring-ai-docs/src/main/antora/modules/ROOT/pages/api/chatclient.adoc), [OpenAI Chat 2.0.1](https://github.com/spring-projects/spring-ai/blob/v2.0.1/spring-ai-docs/src/main/antora/modules/ROOT/pages/api/chat/openai-chat.adoc). Não utilizar a documentação corrente como prova de compatibilidade com 2.0.1.
