# Changelog - Crédito CVM DataOps

Todas as alterações notáveis e marcos do projeto serão documentados neste arquivo.

---

## [0.1.0] - 2026-10-12

### 🚀 Infraestrutura & Ambiente Dev (Docker & Airflow)

#### Commits Relacionados:
* `feat: ajusta docker-compose para airflow 2.8.1 e configura .env`

#### Detalhamento das Implementações:

1. **Estabilização do Apache Airflow (2.8.1):**
   * Configuração do `docker-compose.yaml` alinhado à versão estável `2.8.1` do Airflow.
   * Ajuste dos componentes do cluster local: `airflow-webserver`, `airflow-scheduler`, `airflow-worker` (Celery Executor), `airflow-triggerer` e `airflow-dag-processor`.
   * Integração com serviços de suporte: **PostgreSQL 16** (banco de metadados) e **Redis 7.2** (broker de mensagens).

2. **Otimização de Performance & Gestão de Recursos (WSL 2 / Docker Desktop):**
   * Ajuste dos limites de memória e CPU no ambiente WSL 2 (`.wslconfig` com 6GB RAM / 4 CPUs).
   * Desativação da opção *Resource Saver* no Docker Desktop para conter quedas por OOM (*Out Of Memory*) do container `dag-processor`.

3. **Segurança e Mapeamento GCP:**
   * Estruturação do diretório `./config` para carregamento isolado das credenciais da GCP (`gcp_key.json`).
   * Adição do arquivo `.env` e chaves privadas ao `.gitignore` para prevenir o vazamento acidental de credenciais em repositórios remotos.
   * Injeção de variáveis de ambiente (`GCP_PROJECT_ID`, `GCP_GCS_BUCKET`, `GOOGLE_APPLICATION_CREDENTIALS`) diretamente nos serviços do Airflow via Docker.

4. **Validação do Ambiente:**
   * Sucesso na execução do `airflow-init` e migração do banco de dados.
   * Validação de acesso à interface gráfica (Web UI) no endereço `http://localhost:8080`.

---

### 🎯 Próximos Passos Programados
- [ ] Configuração da Conexão GCP (`google_cloud_default`) via Airflow UI / CLI.
- [ ] Organização e padronização do diretório `dags/` (arquitetura DataOps).
- [ ] Construção do pipeline de ingestão da Camada Bronze (extração de dados da CVM para o Google Cloud Storage).