---
name: rag-pgvector
description: Implementar ingestão, chunking, embeddings OpenAI e recuperação autorizada com PgVectorStore no Spring AI 2.0.1, exclusivamente na api-conhecimento do COTI AI Atendimento.
---

# RAG com PostgreSQL e pgvector

## Quando utilizar

Usar em tarefas de ingestão de textos, chunking, indexação, reindexação, busca semântica e avaliação de evidências. A implementação de recuperação pertence somente à `api-conhecimento`; sua exposição utiliza a skill MCP e a geração da resposta utiliza a skill Spring AI.

## Responsabilidades

- Garantir consistência entre documentos, chunks, modelo de embeddings e índice.
- Recuperar somente conteúdo autorizado e vigente, com origem e versão rastreáveis.
- Definir relevância e suficiência por critérios avaliáveis, sem equiparar similaridade a verdade.

## Passo a passo de implementação

1. **Spec:** Ler `AGENTS.md`, constituição, requisitos, arquitetura e contratos. Confirmar fontes permitidas, autorização, vigência, volume, estratégia de atualização, limites e critérios de aceite. Não inventar fontes comerciais, limiares definitivos ou contratos MCP.
2. **Plan:** Usar Java 21 e BOM Spring AI 2.0.1, com `spring-ai-starter-vector-store-pgvector` e integração OpenAI no serviço de conhecimento. Definir modelo de embeddings, dimensão, distância e índice segundo arquitetura. Executar PostgreSQL/pgvector em containers por Docker Desktop, com imagem fixada, volume e healthcheck definidos na infraestrutura.
3. Provisionar extensão, esquema, tabela e índices por migrações controladas. `spring.ai.vectorstore.pgvector.initialize-schema` é opt-in; não presumir criação automática nem permitir DDL indiscriminado em produção. Validar nomes de esquema/tabela e permissões mínimas. Não inserir credenciais em imagens ou arquivos versionados.
4. Normalizar textos sem remover informações necessárias à fonte. Rejeitar conteúdo vazio e tamanhos inválidos. Criar `org.springframework.ai.document.Document` com identificadores estáveis e metadados definidos internamente: documento, versão, fonte, chunk, atualização, vigência e autorização. Os nomes públicos devem seguir o contrato.
5. Dividir com `org.springframework.ai.transformer.splitter.TokenTextSplitter`, configurando os limites existentes em 2.0.1. Preservar metadados e vínculo ao documento pai. Não presumir opção nativa de overlap disponível em versões posteriores; se sobreposição for necessária, especificar e testar estratégia própria. Respeitar limites de tokens do modelo escolhido.
6. Usar `EmbeddingModel` OpenAI e `PgVectorStore` por injeção, preferindo a abstração `VectorStore` no adaptador. `add(List<Document>)` do store gera embeddings usando o modelo configurado; não pré-calcular e recalcular os mesmos vetores. Gerar também o embedding da consulta somente neste serviço.
7. Tornar reingestão e atualização idempotentes com IDs e política de substituição explícitos. Controlar duplicatas, remoção de versões obsoletas e falhas parciais. Mudanças de modelo, dimensão ou chunking exigem plano de reindexação e validação antes de trocar o índice ativo.
8. Construir `SearchRequest.builder()` com `query`, `topK`, `similarityThreshold` e `filterExpression`, e executar `similaritySearch(...)`. Construir filtros de autorização no servidor a partir de identidade confiável, preferindo a DSL de filtros em vez de concatenar entrada livre. Aplicar autorização na consulta, não apenas depois de obter os top-K.
9. Calibrar top-K e limiar com conjunto de perguntas e fontes esperadas; documentar distância e semântica do score. Similaridade alta não comprova suporte factual. Verificar vigência, conflitos, cobertura e orçamento de contexto. Retornar pela tool somente evidências autorizadas, com referências; tratar vazio e insuficiência explicitamente.
10. **Test e Review:** Executar integração com PostgreSQL/pgvector real em container, embeddings controlados e casos de autorização, atualização e relevância. Revisar metadados e rastreabilidade. Confirmar que `api-atendimentos` não contém dependências ou credenciais vetoriais.

## Padrões de codificação

- Separar ingestão, transformação, armazenamento e recuperação em componentes testáveis, com injeção por construtor e records nas fronteiras.
- Configurar tamanho de chunks, lotes e limites sem constantes arbitrárias escondidas. Evitar manter transações de banco abertas durante chamadas externas demoradas.
- Tratar fonte como dado não confiável: remover instruções executáveis do fluxo de controle, não confiar em metadados de autorização enviados pelo documento.
- Rastrear operações por IDs, versão e correlação, sem logar documentos, perguntas completas ou vetores por padrão.

## Antipadrões proibidos

- Banco, pgvector, embeddings ou `PgVectorStore` na `api-atendimentos`.
- Misturar embeddings incompatíveis ou publicar atualização parcial como índice íntegro.
- Usar top-K sem autorização, inventar evidências ou considerar todo resultado similar suficiente.
- Aceitar filtros SQL ou expressões arbitrárias do usuário; utilizar documentos como instruções de sistema.

## Critérios de validação

Testar normalização, limites e preservação de metadados; reingestão sem duplicação, exclusão de versão antiga e recuperação após falha parcial. Comprovar dimensão consistente e extensão disponível. Usar fixtures com relevantes, irrelevantes, vencidos, conflitantes e restritos. Demonstrar que ausência de evidência produz insuficiência e que referências retornadas correspondem aos chunks autorizados persistidos.

## Checklist de conclusão

- [ ] Fonte, identidade, metadados e política de vigência definidos.
- [ ] PostgreSQL/pgvector via Docker Desktop e migrações verificadas.
- [ ] Chunking e embeddings compatíveis com 2.0.1.
- [ ] Ingestão idempotente e reindexação controlada.
- [ ] Busca filtrada, relevância avaliada e fontes preservadas.
- [ ] Testes e compilação executados; limitações reportadas.

## Referências da versão

[PgVectorStore 2.0.1](https://github.com/spring-projects/spring-ai/blob/v2.0.1/spring-ai-docs/src/main/antora/modules/ROOT/pages/api/vectordbs/pgvector.adoc) e [TokenTextSplitter 2.0.1](https://github.com/spring-projects/spring-ai/blob/v2.0.1/spring-ai-commons/src/main/java/org/springframework/ai/transformer/splitter/TokenTextSplitter.java). Conferir assinaturas, dimensão e configurações antes de implementar.
