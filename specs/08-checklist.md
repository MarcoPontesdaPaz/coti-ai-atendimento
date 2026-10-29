# Checklist de execução — COTI AI Atendimento

## Estado preservado

Atualizado em 28/10/2026 (America/Sao_Paulo). Execução mais recente limitada à **T005**, única tarefa implementada/validada nesta interação. Registros anteriores preservados. Compose, template de ambiente e instruções locais criados; nenhum container, banco, rede ou volume foi iniciado/criado/alterado nesta execução.

| Escopo | Estado | Evidência / próximo passo |
|---|---|---|
| Preparação — T001 | Concluída e validada documentalmente | Registro abaixo; checkbox em 05-tasks.md |
| Preparação — T002 | Concluída e validada documentalmente | Matriz em 07-test-plan.md com RF01–RF13/RNF01–RNF11 e critérios de evidência abaixo; nenhuma suite de aplicação aprovada |
| Fase 1 — T003–T007 | Em andamento; T003–T005 concluídas | Compose validado para 127.0.0.1:5435→5432 e imagem fixada; T006/T007 pendentes, sem bootstrap/banco em execução e gate G1 aberto |
| Fases 2–8 — T008–T060 | Não iniciadas | Respeitar dependências do backlog; nenhum serviço implementado |

Este arquivo registra evidências; a matriz inicial de T002 está em [07-test-plan.md](07-test-plan.md). Não declara o MVP entregue. As validações futuras devem distinguir aprovado, falhou, ignorado, não executado e não aplicável.

## Registro — T005: Compose de dados e variáveis externas

**Resultado:** concluída no escopo de configuração. Dependência T004 previamente concluída. Entregues Compose somente com serviço postgres, imagem exata fixada por digest, publicação loopback 5435→5432, volume nomeado, rede de dados interna exclusiva, rede de integração declarada para uso futuro e healthcheck na porta interna. Três variáveis obrigatórias externas sem credenciais default, template vazio e regras de exclusão de ambiente no Git. Matriz: MN09/MN11 e gate G1, somente configuração/validação estática; execução real permanece pendente.

**Arquivos criados:**

- `infra/compose.yaml`: projeto coti-ai-atendimento, serviço postgres, referência imutável de T004, portas, volume, redes, healthcheck, SCRAM no initdb, limites locais e rotação de logs.
- `infra/.env.example`: POSTGRES_DB/POSTGRES_USER/POSTGRES_PASSWORD vazios, sem segredos nem valores default; instruções de uso local e porta correta.
- `infra/README.md`: uso/validação com arquivo local ou variáveis de processo, limites iniciais, distinção administrador/bootstrap e aplicação, dependências T006/T007 e preservação dos recursos existentes.

**Arquivos modificados:** `.gitignore` (ignora .env/.env.* em qualquer diretório, permite .env.example); `specs/05-tasks.md` (somente T005 marcada nesta execução); `specs/08-checklist.md` (estado/evidências). Não houve alteração de constituição, requisitos, arquitetura, plano, contratos, matriz, decisão de imagem ou skills.

**Implementação e revisão:** banco conectado apenas à rede knowledge-data com internal=true; service-integration é reservada para MCP em T017 e não conecta o banco. Atendimento ainda não existe nem recebe banco/credenciais. Volume postgres-data sem nome global/container_name, prefixado pelo projeto, mount em /var/lib/postgresql/data e PGDATA em pgdata. POSTGRES_USER é administrador inicial; separação das identidades de migração/runtime e instalação das extensões são T006 e não foram antecipadas. README exige concluir T006 antes da primeira subida. Imagem preservada: pgvector/pgvector:0.8.0-pg16@sha256:a132765ec351c65111b5b675928a3a0515a466a40f97277329db8b8209ad8bc9. Configuração de saúde não implica prontidão validada de banco/extensão. Recursos locais iniciais: 1 CPU/1 GiB/128 PIDs/128 MiB shm; capacidade ainda não medida. Logs do container limitados a três arquivos de 10 MiB, sem substituir retenção da telemetria das APIs.

**Comandos executados e resultados:**

| Verificação / comando | Resultado efetivamente observado |
|---|---|
| `Get-Content`, `Get-ChildItem`, `rg --files`, `git status --short` | AGENTS.md, skills pertinentes de RAG/testes, fontes da sessão, T005/antecessores e decisão de imagem consultados. infra estava vazio, sem Compose/POM/wrapper existente; rg de busca por esses artefatos retornou 1 por ausência, não falha de build. |
| `Get-FileHash -Algorithm SHA256` antes/depois | Confere escopo e preservação das fontes/histórico; somente arquivos listados acima criados/modificados. |
| `docker version --format 'Client={{.Client.Version}}; Server={{if .Server}}{{.Server.Version}}{{else}}unavailable{{end}}'` | Código 0, cliente/servidor 29.6.2. Consulta com aprovação fora do sandbox devido à restrição conhecida do pipe. |
| `docker info --format 'OperatingSystem={{.OperatingSystem}}; OSType={{.OSType}}'` e `docker compose version --short` | Código 0, Docker Desktop/Linux; Compose 5.3.1. Verificados antes das demais operações Docker. |
| `docker compose --env-file infra/.env.example -f infra/compose.yaml config --quiet` com três variáveis sintéticas de processo | Código 0; configuração válida. Valores fictícios não persistidos em .env, não impressos e ambiente restaurado em finally. Não houve leitura/exposição de credencial real no resultado. |
| Mesmo `config --quiet`, retirando uma variável de processo por vez, mantendo seu campo vazio no template | Três casos negativos aprovados: POSTGRES_DB, POSTGRES_USER e POSTGRES_PASSWORD ausentes no processo/vazios no template rejeitados com código não zero e diagnóstico do nome esperado. Falha esperada é aprovação do negativo, não falha da configuração válida. |
| `docker compose --env-file infra/.env.example -f infra/compose.yaml config --no-interpolate --format json`, capturado sem emitir o JSON | Modelo sem interpolação inspecionado por PowerShell: projeto/serviço único, digest, 127.0.0.1:5435→5432/tcp, redes, volume/mount, healthcheck/intervalos, recursos, rotação e três expressões obrigatórias `${VAR:?}` aprovados. Não executar config expandido com segredos em saída compartilhada. |
| Conferência dos recursos no modelo sem interpolação | Primeira asserção esperava bytes normalizados para mem_limit e falhou; consulta restrita confirmou que Compose preserva a string `1g` nesse modo. Corrigido o verificador, sem alteração da configuração, e repetida a checagem do modelo com aprovação (código 0). |
| `git check-ignore --quiet --no-index -- <caminho>` | Cinco caminhos locais ignorados (.env/.env.local/infra/.env/infra/.env.local/api-conhecimento/.env.production); infra/.env.example não ignorado (código 1 esperado). Template conferido com três valores vazios. Nenhum arquivo secreto real criado. |
| `git diff --check` e conferência explícita final de whitespace/backlog/hashes/links locais | Aprovados; arquivos não rastreados também inspecionados, sem depender apenas do diff; 60 IDs preservados, T001–T005 concluídas e T006 próxima aberta. |

**Validações e limites:** um caso de configuração válida e três negativos de variáveis obrigatórias executados; checagens de modelo e ignore aprovadas após ajuste do verificador. Nenhum `up`, `create`, `run`, `down`, remoção/limpeza ou alteração de container/rede/volume foi executado nesta tarefa. O teste não abriu a porta 5435 e não iniciou PostgreSQL. Não se trata de integração real, teste de persistência ou prova do healthcheck em runtime. Nenhum segredo foi incorporado à imagem/YAML/template/documentação; valores de validação eram fictícios e existiram somente no processo de teste.

**Compilação/testes de aplicação:** não aplicáveis, sem código Java/POM/wrappers ou serviços implementados nesta tarefa. Verificações de configuração executável aplicáveis ao Compose foram executadas conforme acima. Suites DB/MCP/REST/E2E/AI não executadas, nenhum teste ignorado contado como sucesso. Rejeitar variável vazia não qualifica runtime/migrador, extensões, permissões ou persistência.

**Pendências:** T006 (bootstrap de privilégios/extensões) e T007 (subida/health/volume/permissões reais), mais isolamento efetivo com as APIs em T017. G1 continua aberto; reconferir porta/daemon antes de iniciar o banco. Não iniciar banco antes de T006 e não alterar recursos de outros projetos.

**Status/próxima tarefa:** T005 concluída; Fase 1 em andamento. Próxima **T006 — preparar bootstrap de privilégios/extensões**, não iniciada nesta execução. Encerrar após esta única tarefa.

## Registro — alteração autorizada da porta e T004

**Resultado:** porta local alterada de 5434 para **5435**, após verificação positiva e conforme autorização condicional do solicitante. Porta interna permanece **5432**. T004 retomada com dependência T003 concluída e encerrada após qualificação da imagem prevista. T005 não iniciada. Evidência detalhada: [decisão de infraestrutura local](../docs/decisions/infra-local.md).

**Porta e impacto:** o primeiro bind exclusivo em 127.0.0.1:5435 dentro do sandbox retornou `AccessDenied` (código 1); repetição com aprovação fora do sandbox retornou sucesso (código 0), com encerramento do listener em finally. A disponibilidade é pontual e será revalidada antes de uso. Configuração vigente: publicação loopback 5435→5432; api-conhecimento no host usa 5435, dentro do Compose usa postgres:5432; atendimento continua sem banco. Não há aplicação/infraestrutura deste projeto já implementada para migrar. Contratos REST/MCP, schemas, credenciais e dados não mudam; os futuros arquivos usarão 5435. Registros históricos de T003/T002 com 5434 são preservados como evidência, não como instrução vigente. O impedimento histórico da porta foi resolvido para este projeto pela mudança autorizada, sem liberar a porta de outro projeto.

**Qualificação T004:** manifesto da tag prevista `pgvector/pgvector:0.8.0-pg16` consultado com sucesso; índice OCI `sha256:a132765ec351c65111b5b675928a3a0515a466a40f97277329db8b8209ad8bc9` fixado na arquitetura/decisão. Manifestos Linux/amd64 e Linux/arm64 identificados; Linux/amd64 executado no Docker Desktop Linux/x86_64. Saída do container confirmou PostgreSQL **16.10 (Debian 16.10-1.pgdg12+1)** e vector.control com versão **0.8.0**. Arm64 somente identificado, não executado. Não houve substituição de imagem/versão proposta.

**Arquivos criados:** `docs/decisions/infra-local.md` — porta autorizada, referência imutável, plataformas, versões, comandos e limites da qualificação.

**Arquivos modificados:**

- `specs/03-architecture.md`: porta/diagrama e ADR011, distinção host/rede Docker, impacto e imagem qualificada/digest.
- `specs/04-plans.md`: publicação/critério de aceite/risco atualizados para 5435.
- `specs/05-tasks.md`: T003 preservada com anotação da nova porta; T004 marcada após validação; T005 exige 5435→5432.
- `specs/07-test-plan.md`: isolamento via host e gate G1 passam a usar 5435.
- `specs/08-checklist.md`: estado atual e este registro, sem reescrever evidências históricas.

**Comandos efetivamente executados:**

| Comando / verificação | Resultado |
|---|---|
| `rg -n` para 5434/5435/T003/T004, `Get-Content` e `git status --short` | Localizadas referências vigentes e históricas; AGENTS.md/skill RAG e fontes da sessão consultadas, backlog retomado em T004. |
| Bind `TcpListener` Loopback:5435 exclusivo, Start/Stop em finally | Sandbox: AccessDenied/código 1; fora do sandbox com aprovação: disponível/código 0; sem reserva persistente. |
| `docker version --format ...` e `docker info --format ...` (formatos completos na decisão) | Código 0; cliente/servidor 29.6.2 e Docker Desktop/Linux/x86_64, antes das demais operações Docker. |
| `docker buildx imagetools inspect pgvector/pgvector:0.8.0-pg16` | Código 0; digest de índice e manifestos reais registrados na decisão; nenhuma imagem escolhida por suposição. |
| `docker run --rm --network none --read-only --entrypoint sh <referência fixada> -c 'postgres --version && cat /usr/share/postgresql/16/extension/vector.control'` | Código 0 após download da imagem; versões 16.10/0.8.0; novo container descartado automaticamente. Sem iniciar PostgreSQL, rede, porta publicada ou volume. Comando completo na decisão. |
| `Get-FileHash -Algorithm SHA256` antes/depois e checagem PowerShell de digest, portas e links | Escopo limitado aos documentos listados; requisitos/constituição/contratos/skills preservados. Primeira checagem de links tratou URL HTTPS como arquivo; filtro corrigido para links locais e repetição aprovada. |
| Verificação explícita de whitespace/backlog e `git diff --check` | Revisão documental aprovada; arquivos não rastreados inspecionados além do diff; 60 IDs mantidos, somente T001–T004 concluídas e T005 próxima aberta. |

**Validações:** bind real de 5435 aprovado fora do sandbox; consulta do daemon aprovada; manifesto/plataforma/digest e versões da imagem aprovados para T004; consistência documental revisada. Preservação dos containers existentes decorre do escopo dos comandos: apenas consultas, pull de imagem e novo container isolado com --rm; nenhum stop/rm/recreate, mount de volume existente ou limpeza foi executado. Houve alteração somente do cache de imagens e ciclo de vida do container novo de inspeção.

**Limitações / não executados:** não foi validada ativação SQL da extensão vector, migração, criação de roles, HNSW, healthcheck, persistência ou isolamento das APIs; são critérios futuros, não aprovação implícita de T004. Compilação Java e suites da aplicação não se aplicam a esta tarefa de qualificação; não foram executadas. Revisão de patches/segurança para publicação em rede permanece pendente à T059.

**Status e próxima tarefa:** T004 concluída; Fase 1 em andamento, G1 ainda aberto. Próxima tarefa T005 — criar Compose de dados com imagem fixada e **127.0.0.1:5435→5432**. Não iniciada nesta interação. Não há mais impedimento de porta constatado para este projeto no instante da verificação; revalidar antes da subida. Não alterar containers/dados existentes nas tarefas seguintes.

## Registro — T003: diagnóstico do ambiente Docker Desktop

Nota de continuidade: o registro a seguir é histórico, anterior à autorização de 5435. O conflito de 5434 pertence ao outro projeto e não é impedimento vigente para o novo mapeamento; a configuração atual e a verificação de 5435 constam no registro acima.

**Resultado da tarefa:** concluída conforme seu aceite de diagnóstico (daemon disponível ou impedimento registrado), com **conflito real de porta registrado**. Dependência T002 já concluída antes do início. Não significa que a porta está disponível ou que a infraestrutura foi validada. Escopo de matriz: MN11/RNF11 e gate G1, somente verificação preliminar do ambiente; os demais critérios de MN11 ainda não foram executados.

**Ambiente observado em 28/10/2026:** cliente Docker 29.6.2, servidor Docker 29.6.2, sistema operacional do engine `Docker Desktop`, containers Linux, arquitetura `x86_64`, storage driver `overlayfs`. Consulta ao endpoint local `dockerDesktopLinuxEngine` foi bem-sucedida fora do sandbox. A enumeração inicial `Get-Process -Name 'Docker Desktop'` dentro do sandbox retornou zero processos e a procura nos caminhos usuais não produziu confirmação do executável; isso foi inconclusivo, não evidência de ausência do Desktop. A confirmação efetiva veio de `docker info`, que identificou Docker Desktop e respondeu com código 0.

**Conflito comprovado:** bind TCP exclusivo em `127.0.0.1:5434` falhou com `AddressAlreadyInUse`. Consulta de containers identificou `api-usuarios-postgres`, publicando `127.0.0.1:5434->5432/tcp`. A consulta do host confirmou listener `127.0.0.1:5434`, estado `Listen`, processo `com.docker.backend` (PID 13084 no instante da verificação). Porta ocupada não foi tratada como livre, e o container existente não foi parado, removido ou reconfigurado.

**Arquivos criados:** nenhum. Não foi criado script de infraestrutura, pois a entrega de T003 é o diagnóstico/evidência, e os scripts e Compose pertencem a tarefas posteriores.

**Arquivos modificados:**

- `specs/05-tasks.md`: somente T003 passa a [x], com referência ao diagnóstico e ao impedimento.
- `specs/08-checklist.md`: estado da Fase 1 e este registro; histórico de T001/T002 preservado.

**Comandos executados e resultados (diretório: raiz do repositório):**

| Verificação / comando efetivo | Resultado / código / limite da evidência |
|---|---|
| `Get-Content`, `rg --files specs skills`, `git status --short` | Releitura de AGENTS.md/skills pertinentes, backlog/checklist e trechos de plano/arquitetura/matriz; especificações integralmente lidas anteriormente na sessão conferidas por hashes. T001/T002 concluídas; T003 primeira aberta. Estado não rastreado dos documentos preservado. |
| `Get-FileHash -Algorithm SHA256` antes/depois | Compara AGENTS.md/specs/skills para preservar fontes normativas e limitar alterações aos dois arquivos de progresso. |
| `docker version --format 'Client={{.Client.Version}}; Server={{if .Server}}{{.Server.Version}}{{else}}unavailable{{end}}'` | Dentro do sandbox: código 1, acesso negado ao pipe local, cliente 29.6.2. Repetido com aprovação fora do sandbox: código 0, cliente/servidor 29.6.2. Não confundir bloqueio do sandbox com daemon indisponível. |
| `docker info --format 'ServerVersion={{.ServerVersion}}; OperatingSystem={{.OperatingSystem}}; OSType={{.OSType}}; Architecture={{.Architecture}}; Driver={{.Driver}}'` | Dentro do sandbox: código 1, mesmo acesso negado. Repetido com aprovação fora do sandbox: código 0, Docker Desktop/Linux/x86_64/overlayfs. Consultas version/info repetidas em conjunto completaram em cerca de 1,35 s de execução. Não foram impressos env, proxies, credenciais ou configuração completa. |
| `Get-Process -Name 'Docker Desktop'` e `Test-Path` dos caminhos usuais do executável | Enumeração restrita sem confirmação; resultado inconclusivo, superado pela resposta real do engine. Nenhuma aplicação foi iniciada/encerrada. |
| `TcpListener` em Loopback:5434 com `ExclusiveAddressUse=true`, `Start()` e `Stop()` em finally | Código 1, `AddressAlreadyInUse`; tentativa local concluída em cerca de 1,15 s junto da enumeração. Não reservou a porta nem deixou listener aberto. |
| `docker ps --filter publish=5434 --format 'Name={{.Names}}; Ports={{.Ports}}'` | Com aprovação fora do sandbox, código 0 em cerca de 0,59 s; retornou somente api-usuarios-postgres e o mapeamento ocupado. Consulta de identificação, sem inspecionar imagem/digest (T004 não iniciada). |
| `Get-NetTCPConnection -LocalPort 5434 -State Listen` e `Get-Process -Id` do proprietário | No sandbox: código 1, recurso CIM não disponível. Repetido com aprovação fora do sandbox: código 0 em cerca de 2,43 s, listener/processo descritos acima. Sem alteração do processo. |
| Verificação PowerShell de backlog/whitespace/hashes e `git diff --check` | Conferência documental final: 60 IDs preservados, somente T001/T002/T003 marcadas, T004 próxima aberta; fontes/skills preservadas. Inspeção explícita dos arquivos complementa diff, que não cobre sozinho arquivos não rastreados. |

**Validações realizadas:** disponibilidade real do daemon **aprovada**, confirmação do backend Docker Desktop **aprovada**, disponibilidade da porta **falhou** (ocupada), identificação do conflito **concluída** por duas consultas independentes. Falhas iniciais de acesso ao pipe/CIM no sandbox foram registradas e as mesmas consultas repetidas com aprovação, sem recorrer a banco local alternativo. A falha de porta permanece aberta como impedimento técnico, não como falha ocultada de um teste aprovado.

**Compilação e testes da aplicação:** não aplicáveis a T003, pois não houve alteração de código/configuração executável; nenhuma suite unitária, PostgreSQL/pgvector, HTTP, MCP, E2E ou OpenAI foi executada e nenhuma foi declarada aprovada. Não houve runner de testes com testes ignorados/zero testes contabilizados como sucesso. Imagem/digest, extensão vector, migrações, healthcheck, volume, permissões e isolamento ainda são **não executados**, pertencentes às próximas tarefas.

**Pendências e efeito no gate:** liberar a porta contratada 5434 antes de iniciar o PostgreSQL do projeto ou decidir formalmente uma mudança de porta nas especificações, se necessária. Não mudar a arquitetura/porta silenciosamente nem interromper container de outro projeto sem autorização específica. A T003 aceita o diagnóstico de impedimento e está concluída; **G1/Fase 1 não estão aprovados**, e a subida/validação do banco continua impedida enquanto o conflito existir. Revalidar daemon/porta no momento de uso, pois esta evidência é pontual.

**Próxima tarefa recomendada:** T004 — qualificar e fixar imagem do banco, com daemon disponível e sem precisar publicar 5434 para consultar o manifesto. Não foi iniciada nesta execução. O conflito deverá ser resolvido antes da subida do banco; T005–T007 e Fase 2 permanecem sem aceite. Encerrar após T003, respeitando a instrução de uma única tarefa.

## Critérios obrigatórios de evidência — T002

Aplicar estes critérios a cada tarefa/fase futura, sem sobrescrever registros históricos. A matriz inicial define cenários planejados, não resultados de execução. Cada linha de requisito contém conjuntos de subcasos; vincular os testes implementados ao ID MF/MN/CT e ao critério original. Nenhum cenário pode ser descartado por já haver outro teste do mesmo requisito.

| Campo de registro | Evidência exigida |
|---|---|
| Escopo e origem | Data/fuso, tarefa/fase, requisitos e IDs MF/MN/CT/subcasos, gate de entrada, versão/revisão do artefato e do corpus/prompt/modelo quando aplicável |
| Alterações | Arquivos criados/modificados e finalidade; explicitar quando não houver; preservar trabalho anterior e revisar requisitos/contratos/arquitetura |
| Ambiente | Java/Boot/AI/SDK/JUnit/plugins efetivamente resolvidos quando relevantes; imagem/digest, Docker Desktop/daemon e isolamento; dependências reais e controladas discriminadas |
| Execução | Comando efetivamente executado, diretório, código de saída, duração e caminho do relatório; sanitizar segredo/argumentos sensíveis, nunca copiar tokens ou dumps de ambiente |
| Contagens | Executados, aprovados, falhos, ignorados por suite unitária/integração/contrato/avaliação; testes não descobertos e suite vazia não aprovam comportamento |
| Resultado por cenário | Estado da tabela abaixo, comparação com oráculo e efeitos proibidos (incluindo ausência de chamadas, publicação parcial ou vazamento) |
| Avaliação paga | Autorização/escopo e teto de orçamento da tarefa, chamadas/tentativas/tokens/custo disponível, perfil do modelo/prompt/tokenizer/corpus e qualidade por pergunta; resultado separado de mocks |
| Pendências e continuidade | Falha/impedimento/lacuna, verificações ainda necessárias e gate bloqueado; próxima tarefa em ordem de dependência, sem iniciá-la automaticamente |

| Estado de validação | Quando registrar | Efeito no aceite |
|---|---|---|
| Aprovado | Verificação aplicável executada com resultado esperado e evidência | Conta apenas para o cenário/nível que realmente comprova; DOC não aprova aplicação e mock não aprova integração real |
| Falhou | Comando/teste executado não cumpriu oráculo ou apresentou erro | Corrigir, registrar a falha e reexecutar verificações afetadas; falha obrigatória bloqueia entrega |
| Ignorado | Teste descoberto desabilitado/abortado/ignorado pelo runner | Registrar contagem/motivo; não somar como aprovado, requisito continua pendente sem evidência pertinente equivalente |
| Não executado | Verificação prevista não foi iniciada por dependência ausente, credencial, ambiente ou escopo ainda futuro | Registrar comando tentado quando houver, falha/causa e gate afetado; não declarar integração/provedor validado |
| Não aplicável | Verificação não tem relação com o artefato/escopo alterado | Justificar; exemplo: compilação e Docker em tarefa exclusivamente documental, sem eliminar obrigações futuras |

**Regra para [x]:** tarefa concluída somente após entrega/revisão de seus artefatos e aprovação das verificações aplicáveis ao próprio aceite. Impedimento pode satisfazer uma tarefa cujo resultado explicitamente admite diagnóstico de bloqueio (como T003), mas não aprova o gate técnico nem libera o trabalho que depende do ambiente ausente. T002 pode concluir com suites futuras não executadas porque sua entrega é a matriz documental; não encerra Fases 1–8.

**Fechamento de fase:** confrontar o gate G1–G8 da matriz, executar suites pertinentes reais, registrar resultados individuais e falhas/ignorados/não executados. Sem contagens ou com zero testes não declarar fase validada. OpenAI real requer execução explícita separada; integração MCP exige transporte real e persistência exige pgvector real em Docker, sem H2/banco local alternativo. Não avançar com critério obrigatório incompleto nem publicar em rede sem os controles arquiteturais de TLS/identidade/permissões/limites/segredos/backup.

## Registro — T002: matriz inicial de testes e aceite

**Resultado:** concluída e validada documentalmente. Dependência T001 já concluída antes do início. Foram definidos MF01–MF13 e MN01–MN11, cada um com cenários positivos/negativos, oráculo, nível de execução e gate/tarefas; CT-REST, CT-MCP, CT-TEMPO e CT-CONST detalham controles contratuais e constitucionais. G-PREP/G1–G8 definem evidência e impedimento por fase. Critérios de status/registro nesta seção geral completam a entrega de T002.

**Arquivos criados:** nenhum; os três documentos editados já existiam, e 07-test-plan.md estava vazio.

**Arquivos modificados nesta execução:**

- `specs/07-test-plan.md`: matriz inicial, ambientes, cenários/oráculos, gates por fase, contratos/tempos, estratégia de corpus/AI paga e comandos futuros.
- `specs/05-tasks.md`: somente T002 passa a [x], com referência a esta evidência; T001 preservada e T003–T060 abertas.
- `specs/08-checklist.md`: estado atual, critérios de evidência/status e este registro; histórico T001 preservado.

**Comandos executados e resultados:**

1. `Get-Content -Raw` para AGENTS.md, todas as especificações e skills locais de testes/Spring AI/MCP/RAG; complemento com `Get-Content ... | Select-Object -Skip 50` para a parte final do backlog. `rg --files specs skills` e `git status --short` conferiram inventário e estado; confirmado que T002 era a primeira tarefa aberta e T001 estava marcada.
2. `Get-FileHash -Algorithm SHA256` antes/depois para AGENTS.md, specs e skills; comparação verifica que somente os três arquivos acima mudaram. Fontes normativas, plano, skills e decisões técnicas foram preservados.
3. Verificação estrutural em PowerShell (`Get-Content`, `Select-Object`, `Where-Object`, `Compare-Object`, `Test-Path` e regex): cobertura exata dos 24 requisitos extraídos da fonte, 24 linhas com cenários/oráculos/níveis/gates, quatro controles transversais e nove gates, links locais existentes e ausência de whitespace final. Primeira tentativa falhou por expectativa incorreta do verificador (seis colunas em vez das cinco da tabela); corrigida a verificação para cinco colunas, a repetição foi **aprovada**. A falha foi da checagem documental, não de teste de aplicação; não foi ocultada.
4. `git diff --check`: **aprovado**. Como specs permanece não rastreado no estado Git existente, conteúdo/referências/whitespace também foram inspecionados explicitamente; diff isolado não é evidência suficiente.
5. Conferência final: manter 60 IDs de tarefa únicos/sequenciais, somente T001/T002 marcadas e T003 como próxima aberta; verificar preservação do histórico de T001 e escopo por hashes.

**Revisão de conteúdo:** cada conjunto da matriz cobre os cenários de aceite de RF01–RF13/RNF01–RNF11; explicita ingestão íntegra/idempotência/overlap, MCP determinístico antes do chat, filtros de autorização/publicação/vigência, isolamento de identidade/banco, fontes montadas pela aplicação, tratamento de conflitos/insuficiência/falhas, orçamento de contexto/prazos, limites HTTP/MCP e telemetria sem conteúdo. Contrato, requisitos e arquitetura não foram alterados. Suites DB/MCP reais, testes controlados e AI paga têm evidências distintas. Nenhum limiar semântico arbitrário foi definido para antecipar T031/T033/T047.

**Testes realizados:** revisão e verificações documentais acima, aprovadas após correção do verificador. Nenhuma suite U/H/DB/MCP/OPS/E2E/AI foi implementada ou executada nesta tarefa. Compilação Maven, execução Docker e testes da aplicação são **não aplicáveis à alteração documental**. Os cenários futuros permanecem **não executados**; aprovação da matriz não aprova o software. Não houve suite de aplicação ignorada ou falha de aplicação testada, pois não foram executadas.

**Pendências:** nenhuma impeditiva para concluir T002. Permanecem gates futuros: ambiente/imagens/digests, dependências/JUnit/SDK, emissor/delegação, corpus/limiares de recuperação, perfil de chat e validador semântico. Os arquivos normativos podem conservar referências históricas a T002 como futura; o estado atual é este checklist e o backlog, sem reescrever fontes fora do escopo.

**Status da preparação:** concluída documentalmente (T001/T002). **Próxima fase:** Fase 1 — infraestrutura Docker/PostgreSQL/pgvector. **Próxima tarefa:** T003 — verificar Docker Desktop/daemon e porta 5434. Não executada nesta interação; encerrar após T002, sem iniciar outra tarefa.

## Registro — T001: reconciliação documental

**Resultado:** concluída. Referências operacionais usam os arquivos existentes, sem renomear ou duplicar especificações. Requisitos RF11–RF13 registram listagem, detalhe e diagnóstico já previstos na seção 4 do contrato; nove cenários Given/When/Then especificam aceite positivo/negativo. Plano/backlog relacionam essas capacidades a T023/T038 e à futura matriz T002.

**Necessidade e impacto:** correção do caminho de requisitos no AGENTS.md e do caminho de contratos no artigo 16 da constituição; atualização de referências e pendências obsoletas. A alteração dos requisitos é necessária à entrega explícita de T001 e reconcilia operações existentes, sem antecipar novas funcionalidades. A arquitetura recebeu apenas ajustes de referências, rastreabilidade e estado documental; nenhuma decisão de stack, rede, segurança, recuperação ou geração foi alterada. No contrato, somente o parágrafo introdutório de reconciliação foi atualizado; rotas, schemas, exemplos, permissões e semântica permanecem iguais. Não há migração de consumidores/dados ou reindexação.

**Arquivos criados:** nenhum.

**Arquivos modificados:**

| Caminho | Alteração |
|---|---|
| AGENTS.md | Leitura obrigatória aponta para specs/02-requiriments.md |
| specs/01-constitution.md | Artigo 16 aponta para specs/06-api-contracts.md, preservando sua regra |
| specs/02-requiriments.md | Referências canônicas, ator operacional, RF11–RF13, critérios de aceite, escopo e rastreabilidade |
| specs/03-architecture.md | Referências/estado documental e gate T001 atualizado, sem decisões técnicas novas |
| specs/04-plans.md | Referências e rastreabilidade administrativa em fases 3/5/8; matriz ainda pendente à T002 |
| specs/05-tasks.md | Somente T001 marcada; cobertura RF11–RF13 explicitada nas tarefas existentes, sem renumerar/criar tarefas |
| specs/06-api-contracts.md | Nota introdutória registra reconciliação e ausência de impacto no contrato v1 |
| specs/08-checklist.md | Estado da execução, resultados, limites e próxima tarefa |

**Comandos e verificações efetivamente executados:**

- `Get-Content -Raw` para AGENTS.md, todas as especificações e quatro skills locais; `rg --files specs skills` e `git status --short` para inventário/estado. A matriz 07-test-plan.md foi encontrada vazia; não foi preenchida nesta TASK.
- `rg -n` para localizar referências incorretas e notas de reconciliação/arquivos vazios antes das edições. Divergências estavam em AGENTS.md, constituição, requisitos, arquitetura e plano e foram corrigidas.
- `Get-FileHash -Algorithm SHA256` para arquivos de AGENTS.md/specs/skills antes e depois, comparando o conjunto alterado. Finalidade: preservar demais documentos/skills e confirmar escopo desta execução.
- Verificação PowerShell com `Select-String`, `Test-Path` e expressões regulares: ausência dos nomes incorretos, caminhos canônicos existentes, 13 RF únicos/sequenciais, RF11–RF13 com campos obrigatórios e ao menos três cenários cada; 60 tarefas preservadas. **Aprovada** antes da marcação de T001; nova conferência final verifica somente T001 marcada e T002 como próxima pendente.
- Inspeção explícita de whitespace final nos arquivos documentais e `git diff --check`: **aprovadas**. Os documentos estão não rastreados no estado Git existente; portanto diff isolado não cobre seu conteúdo e não foi usado como única evidência.

**Revisão de conteúdo:** listagem/detalhe respeitam versão ativa/visibilidade e filtros antes da contagem, sem introduzir filtro de vigência administrativo não contratado; diagnóstico reutiliza RF04 sob permissões do operador e rede administrativa. Identidade de atendimento continua sem acesso REST administrativo; toda recuperação do atendimento permanece exclusivamente MCP e anterior à geração. RF01–RF10/RNF01–RNF11 mantêm identificadores e obrigações. Nenhum endpoint ou código foi implementado.

**Testes e limitações:** validações estruturais e revisão documental executadas. Compilação Maven, unitários, integração PostgreSQL/MCP, avaliações OpenAI e Docker são **não aplicáveis à alteração exclusivamente documental** e não foram executados; não há evidência de aprovação dessas suites. Os cenários novos são critérios futuros, não testes de aplicação aprovados. Nenhuma validação documental falhou no resultado final.

**Pendências:** nenhuma impeditiva para encerrar T001. T002 continua pendente, assim como os gates técnicos já previstos: Docker/imagens/digests, compatibilidade/JUnit/SDK MCP, emissor/delegação, perfil de modelo e calibração RAG. Não foram qualificados nesta interação e bloqueiam os respectivos trabalhos dependentes.

**Próxima tarefa recomendada:** T002 — definir matriz inicial de testes/aceite. Depois de sua validação, a próxima fase técnica é Fase 1, começando por T003. Nenhuma tarefa seguinte foi executada automaticamente.
