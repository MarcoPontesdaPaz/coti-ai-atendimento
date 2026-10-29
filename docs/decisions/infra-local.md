# Decisão de infraestrutura local — T004

## Estado e fontes

Qualificação da imagem concluída em 28/10/2026, com dependência T003 concluída. Fontes: [arquitetura](../../specs/03-architecture.md), [plano](../../specs/04-plans.md), [tarefas](../../specs/05-tasks.md) e [checklist de execução](../../specs/08-checklist.md). Esta entrega não cria Compose nem inicia banco de dados do projeto; T005 é a próxima tarefa. Não modifica containers ou dados existentes.

## Porta local autorizada

O solicitante autorizou 5435 caso estivesse disponível. O teste de bind TCP exclusivo em `127.0.0.1:5435` fora do sandbox retornou sucesso e o listener de teste foi encerrado em finally. A tentativa inicial no sandbox retornou AccessDenied; o reteste distinguiu restrição do ambiente de ocupação real. A disponibilidade é pontual, sem reserva persistente da porta.

Configuração obrigatória a partir desta decisão:

| Contexto | Endereço / regra |
|---|---|
| Publicação no host local | `127.0.0.1:5435` → porta `5432` do container PostgreSQL |
| api-conhecimento dentro do Compose | Host de serviço `postgres`, porta `5432` |
| api-conhecimento executada no host | `127.0.0.1:5435`, credenciais externas |
| api-atendimentos | Sem driver, conexão, credencial ou acesso ao banco |
| Porta 5434 | Pertence a outro projeto; não utilizar, liberar ou reconfigurar por este trabalho |

Futuros `infra/compose.yaml`, `infra/.env.example`, scripts e configurações da aplicação devem respeitar esses valores. Manter redes, autorização e restrições de acesso da arquitetura. Nenhum arquivo executável existente deste projeto requer migração, pois ainda não foi implementado; contratos REST/MCP e dados permanecem iguais. Registros anteriores que mencionam 5434 são evidências históricas e não a configuração vigente.

## Referência de imagem fixada

Utilizar no futuro Compose:

```text
pgvector/pgvector:0.8.0-pg16@sha256:a132765ec351c65111b5b675928a3a0515a466a40f97277329db8b8209ad8bc9
```

| Dado verificado | Resultado |
|---|---|
| Registry/repositório/tag | docker.io/pgvector/pgvector:0.8.0-pg16 |
| Media type do índice | application/vnd.oci.image.index.v1+json |
| Digest do índice OCI | sha256:a132765ec351c65111b5b675928a3a0515a466a40f97277329db8b8209ad8bc9 |
| Manifesto linux/amd64 | sha256:c72df104202409108a1660ecccd2b7ffac6b17e44c31b2636d10e983f410f8eb |
| Manifesto linux/arm64 | sha256:bf8a1d42f6d35d34d3bdc3bf64045bf66521404c4e3db68a149f2ff9e4ae7711 |
| Plataforma efetivamente executada | Docker Desktop Linux/x86_64 (amd64) |
| PostgreSQL na imagem | 16.10 (Debian 16.10-1.pgdg12+1) |
| pgvector presente na imagem | vector.control com default_version = '0.8.0', module_pathname = '$libdir/vector' e relocatable = true |

O digest fixa o índice multi-plataforma; o engine escolhe seu manifesto compatível. As entradas unknown/unknown exibidas no índice são attestations, não plataformas de execução. Linux/arm64 foi identificado no manifesto, mas **não executado nem qualificado em runtime** nesta tarefa. Nenhuma tag ou versão foi substituída em relação à imagem proposta pela arquitetura. Não confundir PostgreSQL principal 16 com promessa de atualização automática do patch: atualizações exigem novo digest, revisão e validações pertinentes.

## Comandos efetivamente executados e evidências

Os comandos Docker foram realizados com aprovação fora do sandbox, cujo pipe local teve acesso negado nas verificações anteriores. Saídas limitadas, sem dump de ambiente, credenciais, proxies ou configuração completa.

```powershell
docker version --format 'Client={{.Client.Version}}; Server={{if .Server}}{{.Server.Version}}{{else}}unavailable{{end}}'
docker info --format 'OperatingSystem={{.OperatingSystem}}; OSType={{.OSType}}; Architecture={{.Architecture}}'
docker buildx imagetools inspect pgvector/pgvector:0.8.0-pg16
docker run --rm --network none --read-only --entrypoint sh pgvector/pgvector:0.8.0-pg16@sha256:a132765ec351c65111b5b675928a3a0515a466a40f97277329db8b8209ad8bc9 -c 'postgres --version && cat /usr/share/postgresql/16/extension/vector.control'
```

Resultados: version/info código 0 (cliente/servidor 29.6.2, Docker Desktop/Linux/x86_64); manifesto código 0 e digest/plataformas acima; run código 0, PostgreSQL 16.10 e vector.control 0.8.0. A imagem foi baixada no cache local; somente um container de verificação descartável foi criado, sem rede, porta publicada ou volume, com filesystem somente leitura e entrypoint substituído para não inicializar PostgreSQL. `--rm` removeu apenas esse novo container ao terminar. Nenhum container preexistente foi parado, recriado, removido ou reconfigurado e nenhum volume/dado preexistente foi montado ou alterado.

## Limites e próximos gates

T004 comprova manifesto, plataforma compatível e versões presentes na imagem. Não comprova `CREATE EXTENSION vector`, conectividade SQL, criação de roles, migrações, HNSW, healthcheck, persistência, backup ou isolamento entre APIs. Essas evidências pertencem a T006/T007/T017/T018 e integrações futuras. Compilação Java e suites da aplicação não se aplicam a esta qualificação. Revisão de patches/segurança antes de publicação em rede permanece obrigatória em T059; esta tarefa não a substitui.

Próximo passo: T005 — criar Compose de dados com a referência imutável e publicação **127.0.0.1:5435 → 5432**, revalidando daemon/porta antes de uso. Não executar essa tarefa nesta interação, respeitando a sequência de uma tarefa de implementação por execução.
