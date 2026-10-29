# Infraestrutura local — configuração T005

O Compose contém somente PostgreSQL 16/pgvector, com a [imagem fixada em T004](../docs/decisions/infra-local.md), volume nomeado e rede de dados interna. Publicação: **127.0.0.1:5435 → 5432**. O serviço de conhecimento na rede Docker usará `postgres:5432`; atendimento não terá conexão/credenciais nem acesso à rede de dados.

## Configuração e validação

Na raiz do repositório, copie `infra/.env.example` para `infra/.env` e preencha `POSTGRES_DB`, `POSTGRES_USER` e `POSTGRES_PASSWORD` somente no arquivo local ou no ambiente do processo. O template não é utilizável como credencial: campos ausentes/vazios devem falhar na validação. O arquivo preenchido é ignorado pelo Git. Variáveis do processo prevalecem sobre o arquivo; não imprimir seus valores.

Antes de operações Docker, confira Docker Desktop/daemon com `docker version` e `docker info`. Para validar a configuração sem iniciar serviços e sem imprimir segredos:

```powershell
docker compose --env-file infra/.env -f infra/compose.yaml config --quiet
```

Também é possível usar `docker compose -f infra/compose.yaml config --quiet` com as três variáveis já fornecidas no ambiente do processo. Não executar `config` sem `--quiet` em saídas compartilhadas: a configuração expandida contém credenciais. Não colocar valores reais na linha de comando, em imagens ou arquivos versionados.

`POSTGRES_USER`/`POSTGRES_PASSWORD` são o administrador de bootstrap. **Não iniciar o banco antes da entrega/revisão de T006**, que definirá identidades separadas de runtime/migração e extensões. Inicialização e validações reais de persistência/health/permissões ficam para T007; sintaxe aprovada em T005 não comprova essas capacidades. A autenticação do initdb está configurada para SCRAM, sem trust. `pg_isready` verifica prontidão do servidor, não autenticação, extensão ou permissão de aplicação.

## Recursos e isolamento planejado

- Projeto Compose `coti-ai-atendimento`, sem container_name ou nomes de volume/rede globais: recursos recebem o prefixo do projeto. Não reutilizar volumes externos.
- Volume `postgres-data` montado em `/var/lib/postgresql/data`, com PGDATA no subdiretório `pgdata`; não apagar volumes para reaplicar bootstrap.
- PostgreSQL participa somente de `knowledge-data` (bridge interna). `service-integration` fica declarada para as APIs/MCP em T017; PostgreSQL não participa dela. Isolamento real, inclusive via host publicado, ainda será testado nas tarefas correspondentes.
- Limites locais iniciais: 1 CPU, 1 GiB de memória, 128 processos, 128 MiB de shared memory; logs do container rotacionados em até três arquivos de 10 MiB. São limites de recursos do container, não metas comprovadas de capacidade nem a política de retenção da telemetria das APIs. Ajustes exigem medição nas tarefas de capacidade.
- Healthcheck a cada 10 s, timeout 3 s, cinco tentativas e início de 30 s; usa a porta interna 5432, sem chamadas pagas.

Não parar, recriar, remover ou reconfigurar containers/dados de outros projetos. Não usar a porta 5434. Não executar `down -v`, `prune` ou limpeza de volume persistente sem autorização específica. T005 não cria containers, redes ou volumes: a configuração só será aplicada em tarefa posterior.
