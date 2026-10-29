---
name: testing
description: Criar e executar testes Java do COTI AI Atendimento com JUnit 5, Mockito, HTTP, contratos MCP, Testcontainers PostgreSQL/pgvector e cenários de falhas, evidências e prompt injection.
---

# Testes e critérios de aceite

## Quando utilizar

Usar ao implementar ou alterar comportamentos, corrigir defeitos, validar integrações ou revisar critérios de aceite. Para tarefas apenas documentais, revisar consistência e registrar que compilação e testes da aplicação não se aplicam.

## Responsabilidades

- Vincular requisitos e critérios de aceite a evidências automatizadas.
- Separar testes determinísticos de avaliações com modelos externos, sem confundir mocks com integração real.
- Validar limites entre serviços, autorização, falhas, qualidade da recuperação e resistência a documentos hostis.

## Passo a passo de implementação

1. **Spec:** Ler `AGENTS.md`, constituição, requisitos, arquitetura, contratos e plano de testes. Identificar comportamentos observáveis, cenários negativos e critérios de aprovação. Não inventar endpoints, campos ou códigos para preencher especificações vazias.
2. **Plan:** Definir testes unitários com JUnit 5 e Mockito e testes de integração adequados ao impacto. Usar Spring AI 2.0.1 e versões de Spring Boot, JUnit, Mockito e Testcontainers compatíveis com a arquitetura. Se o gerenciamento de dependências resolver outra geração de JUnit, registrar e resolver a incompatibilidade antes de afirmar conformidade com JUnit 5.
3. Testar regras e casos de uso sem subir Spring, usando `MockitoExtension` quando houver colaboradores a substituir. Preferir mocks nas interfaces de fronteira, como adaptador MCP, provedor de geração e `EmbeddingModel`, em vez de simular cada elo interno do fluent API de ChatClient. Verificar saídas e efeitos relevantes, incluindo ausência de chamadas indevidas.
4. Para persistência, verificar Docker Desktop com `docker version` e `docker info`. Usar Testcontainers com imagem PostgreSQL contendo pgvector e versão/digest aprovado; configurar compatibilidade da imagem com `PostgreSQLContainer` quando exigida pela versão de Testcontainers. Uma imagem PostgreSQL comum não garante extensão vector. Não substituir por H2 ou banco instalado localmente.
5. Usar portas dinâmicas e configuração de teste isolada, por exemplo `@DynamicPropertySource` conforme o Spring Boot aprovado. Executar migrações e verificar a extensão. Usar embeddings determinísticos de dimensão válida para testar `PgVectorStore` real, filtros, similaridade, reingestão e versões sem depender de credenciais OpenAI. Limpar apenas dados e recursos pertencentes ao teste.
6. Cobrir a API REST com testes HTTP: MockMvc para testes de camada MVC, ou WebTestClient conforme a stack. Adicionar testes de servidor em porta aleatória para os fluxos que exigem rede real. Validar autenticação, parâmetros, status e schemas OpenAPI aprovados; um teste de controller isolado não comprova integração completa.
7. Testar contrato MCP com servidor e cliente reais compatíveis com Spring AI 2.0.1 por Streamable HTTP. Verificar inicialização, capacidades, `tools/list`, `tools/call`, esquemas, argumentos inválidos, autorização, timeout e erros. Distinguir resultado vazio, erro da tool e erro de protocolo. Mocks do cliente não substituem essa verificação de transporte.
8. Simular falhas de OpenAI, MCP e banco: conexão recusada, timeout, 401/403, 429, 5xx, resposta malformada, recusa e saída inválida quando aplicáveis. Verificar limite de tentativas e prazo total; comprovar que falha técnica não vira sucesso ou ausência fictícia de conhecimento.
9. Montar corpus sintético com fontes conhecidas, restritas, irrelevantes, vencidas e conflitantes. Inserir documentos que ordenem ignorar regras, inventar preços, revelar segredos ou chamar ferramentas não permitidas. Verificar que permissões e ferramentas permanecem limitadas e que nenhuma informação sem evidência é liberada. Usar apenas segredos fictícios como marcadores de detecção de vazamento.
10. Separar avaliações reais com OpenAI em execução explícita, com orçamento e variáveis de ambiente. Medir recuperação e suporte factual com conjunto de referência e limiares definidos antes do teste. Não usar igualdade literal de texto como único oráculo para respostas generativas, nem avaliação por outro modelo como única barreira de segurança.
11. **Test:** Executar os comandos de compilação e testes dos wrappers/builds efetivamente existentes. Garantir descoberta de testes unitários e de integração: um comando que executa zero testes não valida comportamento. Registrar contagens, falhas e testes ignorados; se Docker ou credenciais faltarem, reportar o bloqueio e a suíte não executada.
12. **Review:** Conferir cobertura dos critérios de aceite, independência de `api-atendimentos` de PostgreSQL/pgvector, busca exclusivamente MCP, rastreabilidade e ausência de dados sensíveis nos logs. Reportar comandos e resultados reais conforme `AGENTS.md`.

## Padrões de codificação

- Organizar testes por comportamento, com Arrange–Act–Assert, dados sintéticos e nomes que descrevam cenário e resultado.
- Preferir records para fixtures/DTOs apropriados e injeção por construtor no código testado.
- Utilizar relógio controlável, portas dinâmicas e isolamento entre testes; evitar sleeps fixos e dependência da ordem de execução.
- Gerenciar containers pelo ciclo de vida do Testcontainers, sem remover volumes externos ou usar banco de produção.
- Capturar logs nos cenários relevantes e verificar ausência de chaves, conteúdo restrito e perguntas completas por padrão.

## Antipadrões proibidos

- Afirmar que testes passaram sem execução, ocultar falhas ou contar testes ignorados como sucesso.
- Testar pgvector somente com mocks ou substituir Streamable HTTP por chamada direta de método.
- Chamar OpenAI real por padrão nos testes unitários ou versionar credenciais.
- Testes que apenas repetem a implementação, exigem frases generativas idênticas ou dependem de dados comerciais reais mutáveis.
- Repetir testes até obter uma resposta favorável sem investigar a variabilidade.

## Critérios de validação

Cada comportamento alterado deve ter teste adequado e resultado verificável. Integrações de banco e MCP devem possuir execução real quando afetadas. Casos de insuficiência e autorização devem impedir vazamento e resposta sem suporte, inclusive com prompt injection. Falhas obrigatórias bloqueiam entrega; limitações de ambiente devem permanecer explícitas, sem declaração de validação completa.

## Checklist de conclusão

- [ ] Critérios de aceite ligados a cenários positivos e negativos.
- [ ] JUnit 5, Mockito e dependências compatíveis confirmados.
- [ ] Testcontainers/pgvector e contratos MCP testados quando afetados.
- [ ] Falhas de dependências, autorização e prompt injection cobertas.
- [ ] Compilação e testes executados, com contagens e resultados registrados.
- [ ] Testes ignorados, não executados e pendências discriminados.

## Referências

[Spring AI 2.0.1](https://github.com/spring-projects/spring-ai/tree/v2.0.1), [JUnit 5](https://docs.junit.org/5.13.4/user-guide/), [Mockito](https://javadoc.io/doc/org.mockito/mockito-core), [Testcontainers PostgreSQL](https://java.testcontainers.org/modules/databases/postgres/). Conferir APIs nas versões resolvidas pelo build; estas referências não definem versões de dependências do projeto por conta própria.
