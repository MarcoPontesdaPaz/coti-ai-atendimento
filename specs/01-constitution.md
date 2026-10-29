# Constituição do COTI AI Atendimento

Esta constituição estabelece regras obrigatórias para especificação, implementação, revisão e evolução da plataforma COTI AI Atendimento. Cada artigo é um critério de conformidade e deve orientar os requisitos, a arquitetura, os planos, as tarefas e os critérios de aceite antes da implementação, conforme o desenvolvimento orientado por especificações (Spec-Driven Development).

A plataforma é composta por dois serviços independentes: `api-conhecimento`, responsável pela ingestão, chunking, embeddings, armazenamento vetorial e pesquisa semântica; e `api-atendimentos`, responsável pela API REST de perguntas, pela consulta de conhecimento via MCP e pela geração de respostas contextualizadas com OpenAI. Esta constituição não implementa esses serviços nem define contratos de transporte substitutos.

## Artigo 1 — Plataforma Java e compatibilidade de versões

**Regra:** Os dois microserviços devem utilizar Java 21 e uma versão de Spring Boot comprovadamente compatível com Spring AI 2.0.1. As dependências devem ter versões controladas e reproduzíveis, preferencialmente por gerenciamento oficial de dependências. Compatibilidade não pode ser presumida apenas pela compilação.

**Justificativa:** Uma plataforma consistente reduz incompatibilidades entre bibliotecas, diferenças entre ambientes e falhas de execução.

**Consequência arquitetural:** Cada serviço deve possuir configuração de build independente. A escolha da versão de Spring Boot deve registrar evidências da compatibilidade com Spring AI 2.0.1. Se a compatibilidade ou a disponibilidade das versões não puder ser confirmada, a definição da stack fica pendente; nenhuma substituição silenciosa é permitida.

**Como validar:** Conferir Java 21 no build e na execução em CI; inspecionar dependências resolvidas e referências oficiais de compatibilidade; executar testes de inicialização e integração dos componentes Spring AI utilizados.

## Artigo 2 — Spring AI e OpenAI

**Regra:** As integrações de IA devem utilizar Spring AI e modelos OpenAI. A `api-conhecimento` deve encapsular a utilização dos modelos de embeddings; a `api-atendimentos` deve utilizar o modelo de geração de respostas.

**Justificativa:** Padronizar o acesso aos modelos facilita configuração, testes e manutenção, mantendo explícita a responsabilidade de cada serviço.

**Consequência arquitetural:** Chamadas aos modelos devem ficar em adaptadores próprios, desacoplados das regras de aplicação. Identificadores de modelos e parâmetros operacionais devem ser configuráveis e rastreáveis. Mudanças de modelo exigem revalidação do comportamento e, para embeddings, da compatibilidade do índice.

**Como validar:** Inspecionar dependências e adaptadores; testar a configuração dos modelos e o tratamento de erros; verificar que os testes automatizados permitem substituir o acesso ao provedor por implementações controladas.

## Artigo 3 — Modularidade e separação de responsabilidades

**Regra:** Os serviços devem ser independentes em build, configuração, execução e implantação. Em cada serviço, regras de domínio e casos de uso devem estar separados dos adaptadores de entrada, persistência, MCP e provedores de IA.

**Justificativa:** Fronteiras claras permitem evolução independente e evitam que mudanças de infraestrutura contaminem as regras de atendimento.

**Consequência arquitetural:** A `api-conhecimento` concentra o ciclo de vida do conhecimento e sua recuperação. A `api-atendimentos` concentra a orquestração do atendimento. Não são permitidos acesso ao banco do outro serviço, dependência de suas classes internas ou compartilhamento de entidades de persistência.

**Como validar:** Revisar diagramas, módulos, dependências e configurações de implantação; executar verificações de arquitetura e comprovar que cada serviço pode ser construído e iniciado independentemente, usando substitutos controlados para dependências externas quando necessário.

## Artigo 4 — Integração exclusivamente por MCP sobre HTTP

**Regra:** Toda consulta da `api-atendimentos` à `api-conhecimento` deve ocorrer exclusivamente por MCP sobre HTTP. A `api-conhecimento` deve expor uma ferramenta MCP para pesquisa semântica; a `api-atendimentos` deve atuar como cliente MCP.

**Justificativa:** Um único protocolo de integração torna a fronteira entre os serviços explícita, verificável e evolutiva.

**Consequência arquitetural:** O transporte MCP sobre HTTP e sua versão devem ser definidos no contrato. Não pode haver atalho por REST proprietário, SQL, biblioteca interna ou acesso direto ao índice. Autenticação, autorização, timeouts, limites e erros devem ser definidos nessa fronteira. Comunicação em ambientes implantados deve utilizar HTTPS.

**Como validar:** Executar testes de contrato e integração com cliente e servidor MCP reais; verificar transporte HTTP, autenticação, erros e timeouts; inspecionar dependências e tráfego para comprovar a ausência de caminhos alternativos.

## Artigo 5 — PostgreSQL com pgvector exclusivo do conhecimento

**Regra:** PostgreSQL com pgvector deve ser utilizado exclusivamente pela `api-conhecimento` para persistir e consultar os vetores da base de conhecimento. A `api-atendimentos` não pode possuir conexão, credenciais ou acesso ao armazenamento vetorial.

**Justificativa:** A propriedade exclusiva dos dados evita acoplamento por banco e preserva o controle de recuperação e autorização.

**Consequência arquitetural:** Esquemas, migrações, índices e operações sobre vetores pertencem à `api-conhecimento`. Permissões de banco e rede devem impedir acesso pela `api-atendimentos`. Eventual persistência própria de atendimentos exige especificação separada e não pode acessar essa base vetorial.

**Como validar:** Inspecionar configurações, permissões e dependências; executar testes de persistência e pesquisa com PostgreSQL e pgvector; verificar que a identidade da `api-atendimentos` não consegue acessar o banco de conhecimento.

## Artigo 6 — Embeddings encapsulados na API de Conhecimento

**Regra:** Preparação dos documentos, chunking, geração de embeddings de documentos e consultas, indexação e reindexação devem ser responsabilidades exclusivas da `api-conhecimento`.

**Justificativa:** Centralizar essas operações garante consistência entre os vetores indexados e as consultas semânticas.

**Consequência arquitetural:** A ferramenta MCP deve receber a consulta conforme contrato, sem exigir vetores calculados pelo cliente. O conhecimento deve manter metadados de origem e versão e identificar modelo, dimensão e estratégia de processamento. Trocas incompatíveis exigem reindexação controlada.

**Como validar:** Testar o fluxo de ingestão até a pesquisa; verificar dimensões e compatibilidade do índice; conferir rastreabilidade dos chunks; comprovar que a `api-atendimentos` não gera nem recebe embeddings como requisito de consulta.

## Artigo 7 — Credenciais por variáveis de ambiente

**Regra:** Credenciais de OpenAI, banco de dados e autenticação entre serviços devem ser fornecidas por variáveis de ambiente, inclusive quando injetadas por um gerenciador de segredos.

**Justificativa:** Separar segredos da configuração versionada permite isolamento entre ambientes e rotação de credenciais.

**Consequência arquitetural:** Cada serviço deve receber somente os segredos necessários às suas responsabilidades. Credenciais obrigatórias ausentes ou inválidas devem produzir falha explícita de configuração, sem valores secretos padrão.

**Como validar:** Revisar o mapeamento das configurações; testar inicialização sem variáveis obrigatórias; verificar isolamento por ambiente e serviço e demonstrar a substituição das credenciais sem alteração do código.

## Artigo 8 — Proibição de API keys no código-fonte

**Regra:** Nenhuma API key ou outra credencial real pode ser inserida em código-fonte, arquivos versionados, exemplos, fixtures, documentação ou logs. Exemplos devem usar apenas placeholders claramente fictícios.

**Justificativa:** Segredos publicados comprometem os serviços e podem permanecer acessíveis no histórico do repositório.

**Consequência arquitetural:** O repositório deve excluir arquivos locais de segredos e utilizar detecção de credenciais. Qualquer exposição exige revogação ou rotação imediata, além da correção do artefato e avaliação do histórico afetado.

**Como validar:** Executar varredura de segredos no repositório e nas alterações em CI; revisar exemplos e configurações; testar a sanitização de logs e mensagens de erro.

## Artigo 9 — Respostas fundamentadas e autorizadas

**Regra:** Toda afirmação factual sobre a COTI deve ser sustentada exclusivamente por informações recuperadas da base autorizada para o solicitante. Conhecimento prévio do modelo, suposições e conteúdo fornecido pelo usuário não constituem fonte oficial.

**Justificativa:** A qualidade do atendimento depende da procedência e da autorização das informações, além da fluência da resposta.

**Consequência arquitetural:** A recuperação deve aplicar as permissões do solicitante antes de disponibilizar conteúdo ao modelo. A identidade ou o escopo de autorização deve ser transmitido e validado de forma confiável, nunca aceito apenas como texto livre. Resultados devem manter referências de origem e versão para vincular resposta e evidências. Conteúdo não autorizado não pode integrar o contexto, a resposta ou seus metadados expostos.

**Como validar:** Utilizar cenários com fontes conhecidas, usuários de permissões diferentes e documentos restritos; verificar suporte factual de cada resposta e ausência de vazamento; testar tentativas de falsificação de identidade ou escopo na consulta.

## Artigo 10 — Resistência a prompt injection documental

**Regra:** Documentos e trechos recuperados devem ser tratados como dados não confiáveis. Instruções contidas neles não podem alterar regras do sistema, autorizações, ferramentas permitidas ou política de resposta.

**Justificativa:** Um documento pode conter comandos maliciosos capazes de induzir divulgação de dados ou respostas fora do escopo.

**Consequência arquitetural:** O contexto recuperado deve ser delimitado e separado das instruções da aplicação. Ferramentas e permissões devem ser limitadas e verificadas no código; prompts não podem ser a única barreira. Trechos documentais não podem iniciar ações, ampliar acesso ou solicitar segredos.

**Como validar:** Executar testes adversariais com documentos que tentem sobrescrever instruções, revelar credenciais, acessar conteúdo restrito ou acionar ferramentas indevidas; comprovar que o fluxo mantém as mesmas restrições e não executa esses comandos.

## Artigo 11 — Conhecimento insuficiente e falhas explícitas

**Regra:** Na ausência de evidências suficientes, relevantes, autorizadas e sem contradições não resolvidas, o atendimento deve declarar a limitação e não apresentar uma resposta factual conclusiva. Indisponibilidade técnica deve ser distinguida de ausência de conhecimento.

**Justificativa:** Uma resposta transparente sobre limites é mais confiável do que uma conclusão sem sustentação.

**Consequência arquitetural:** Critérios de suficiência e relevância devem ser especificados e avaliados. O fluxo deve distinguir resultados vazios, insuficientes, conflitantes, bloqueados por autorização e falhas de MCP ou OpenAI, sem revelar a existência de documentos restritos. Pode solicitar esclarecimentos ou orientar para um canal humano oficialmente cadastrado.

**Como validar:** Testar consultas sem resultados, ambíguas, com fontes conflitantes e com falhas de dependências; conferir a mensagem de limitação e o resultado previsto no contrato, sem complementação inventada.

## Artigo 12 — Logging, observabilidade e rastreabilidade

**Regra:** Cada atendimento deve ser rastreável desde a entrada REST até a consulta MCP, recuperação e geração, com logs estruturados, métricas e traces correlacionados.

**Justificativa:** É necessário explicar a origem de uma resposta e diagnosticar falhas, latência e consumo de recursos.

**Consequência arquitetural:** Deve haver propagação de identificadores de correlação entre serviços, registro das referências e versões das fontes e identificação dos modelos usados. Métricas devem cobrir latência, erros, timeouts, consultas sem evidência e uso dos modelos. Credenciais e conteúdo sensível devem ser omitidos ou sanitizados; acesso e retenção da telemetria devem ser definidos.

**Como validar:** Acompanhar um atendimento de ponta a ponta; verificar correlação, fontes e indicadores; simular falhas e conferir sua identificação; inspecionar logs e traces para ausência de segredos e conteúdo restrito.

## Artigo 13 — Testes automatizados e critérios de aceite

**Regra:** Toda funcionalidade deve possuir especificação e critérios de aceite objetivos antes da implementação e evidências automatizadas de seu cumprimento antes da entrega.

**Justificativa:** Validações reproduzíveis tornam verificável a aderência às regras funcionais, de integração e de segurança.

**Consequência arquitetural:** A estratégia deve incluir testes unitários das regras, testes de integração das dependências relevantes, testes de contrato REST/MCP e avaliações de recuperação, fundamentação, autorização, insuficiência e prompt injection. Cenários controlados devem reduzir a variabilidade dos modelos; avaliações com provedores reais devem ter execução e custos controlados.

**Como validar:** Executar a suíte em CI e relacionar os resultados aos critérios de aceite e artigos aplicáveis. Falhas em critérios obrigatórios bloqueiam a entrega; aprovação manual isolada não substitui os testes exigidos.

## Artigo 14 — Documentação OpenAPI

**Regra:** Toda interface REST pública da plataforma deve possuir documentação OpenAPI versionada e coerente com sua implementação, incluindo a API de perguntas da `api-atendimentos`.

**Justificativa:** Consumidores precisam conhecer entradas, saídas, autenticação e falhas sem depender de detalhes internos.

**Consequência arquitetural:** OpenAPI deve documentar esquemas, validações, exemplos, códigos de resposta e resultados de insuficiência ou erro aplicáveis. A ferramenta MCP deve ser documentada pelo contrato e seus esquemas próprios; OpenAPI não substitui a definição do protocolo MCP.

**Como validar:** Validar sintaxe e esquemas OpenAPI em CI e confrontá-los com testes das interfaces REST; verificar que exemplos não expõem segredos nem apresentam dados comerciais fictícios como oficiais.

## Artigo 15 — Proibição de inventar informações comerciais

**Regra:** É proibido inventar, estimar ou completar cursos, preços, descontos, datas, horários, turmas ou disponibilidade. Esses dados somente podem ser informados quando explicitamente sustentados por fontes recuperadas e autorizadas, respeitando sua vigência.

**Justificativa:** Informações comerciais incorretas prejudicam alunos e podem criar expectativas incompatíveis com a oferta real.

**Consequência arquitetural:** As fontes devem permitir identificar atualização e vigência quando relevantes. Dados ausentes, vencidos ou conflitantes devem acionar o tratamento de insuficiência. O modelo não pode deduzir preços ou agendas com base em padrões, ofertas anteriores ou solicitações do usuário.

**Como validar:** Avaliar perguntas sobre cursos inexistentes, preços ausentes, datas expiradas e horários não cadastrados, inclusive sob pressão para adivinhar; conferir que somente dados sustentados e vigentes aparecem na resposta.

## Artigo 16 — Respeito aos contratos das APIs

**Regra:** As interfaces REST e MCP devem respeitar os contratos definidos em `specs/06-api-contracts.md`. Nenhuma implementação pode criar ou alterar unilateralmente ferramentas, caminhos, campos, esquemas, autenticação ou semântica de erros.

**Justificativa:** Contratos explícitos sustentam a evolução independente dos serviços e evitam incompatibilidades entre produtor e consumidor.

**Consequência arquitetural:** Mudanças de interface devem começar pela atualização e revisão da especificação, com análise de compatibilidade, versionamento e migração quando necessários. Se `specs/06-api-contracts.md` estiver ausente ou incompleto, a implementação das interfaces afetadas fica pendente até sua definição; esta constituição não preenche essas lacunas por inferência. Contratos também devem obedecer a esta constituição.

**Como validar:** Executar testes de contrato de produtor e consumidor contra a versão aprovada; revisar diferenças de esquema e comportamento; manter rastreabilidade entre contrato, critérios de aceite e implementação. Divergências bloqueiam a entrega até correção ou revisão formal das especificações.
