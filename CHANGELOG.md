# Changelog - Crédito CVM DataOps

Todas as alterações notáveis e marcos do projeto serão documentados neste arquivo.

# Changelog - Crédito CVM DataOps

Todas as alterações notáveis e marcos do projeto serão documentados neste arquivo.

---

## [0.2.0] - 2026-10-13

### 🚀 Ingestão da Camada Bronze (CVM & BCB) & Prototipagem de Filtros

#### Commits Relacionados:
- `feat: implementa prototipo de ingestao bronze cvm e bcb com full load 2026`
- `docs: adiciona levantamento de requisitos dos 5 filtros de credito privado`

#### Detalhamento das Implementações:

1. **Estruturação da Camada Bronze no Google Cloud Storage (GCS):**
   * Padronização do armazenamento dos arquivos brutos da CVM e BCB no bucket `cvm-landing-zone-bronze-dev`.
   * Adoção de nomenclatura com particionamento Hive-style (`ano=YYYY/mes=MM/`) para manter a rastreabilidade e permitir leitura seletiva.
   * Implementação de rotinas de upload *in-memory* (streaming via `blob.upload_from_string` com `requests`) para evitar I/O de disco temporário e o acúmulo de arquivos nos containers do Airflow.

2. **Levantamento de Requisitos & Mapeamento dos 5 Filtros de Crédito Privado:**
   * Definição da matriz de elegibilidade de fundos utilizando a Composição de Carteira (`CDA_FI`):
     1. **Crédito Privado (≥ 10%):** Mapeado via blocos `cda_fi_BLC_5` e `BLC_6` (Debêntures, LF, CDB, CRI, CRA, NC, NP, DPGE, etc.).
     2. **Exclusão de Investimento no Exterior:** Filtro por denominação/classificação excluindo termos como `"INVESTIMENTO NO EXTERIOR"` / `"IE"`.
     3. **Teto de FIDCs (≤ 30%):** Posições em cotas de FIDC via `cda_fi_BLC_2`.
     4. **NãoSoberanos:** Garantido pelo piso de 10% em crédito privado + validação da `cda_fi_BLC_1`.
     5. **Teto de Ações (≤ 3% do PL):** Posições em ações ordinárias/preferenciais via `cda_fi_BLC_4`.
   * Definição da regra temporal: Seleção e elegibilidade de carteiras com atualização **mensal** (`CDA_FI`), alinhada ao cálculo de retornos diários (`INF_DIARIO`).

3. **Execução do Full Load CVM (Histórico 2026):**
   * **Informe Diário (`inf_diario_fi`):** Download e upload de todas as cotas e PLs diários do ano de 2026 para `cvm/inf_diario_fi/ano=2026/mes=MM/`.
   * **Cadastro (`cad_fi`):** Carga da foto cadastral atual dos fundos para `cvm/cad_fi/ano=2026/mes=MM/`.
   * **Carteira (`cda_fi`):** Ingestão completa dos blocos de composição de carteira mensais de 2026 para `cvm/cda_fi/ano=2026/mes=MM/`.

4. **Ingestão do Benchmark (API BCB - SGS):**
   * Extração da série histórica do CDI/Selic diário (% a.d., Série SGS 12) via API REST em JSON para o ano de 2026.
   * Armazenamento bruto em `bcb/cdi/ano=2026/cdi_diario_2026.json`.

---

### 🎯 Próximos Passos Programados
- [ ] **Desenvolvimento dos Scripts Incrementais / Lógica de Append:**
  - [ ] Implementar a lógica de *upsert/append* diário no Informe Diário (`INF_DIARIO`).
  - [ ] Implementar a captura diária incremental do Cadastro de Fundos (`CAD_FI`).
  - [ ] Criar a rotina de execução mensal incremental para carga de novas carteiras (`CDA_FI`).
  - [ ] Desenvolver a busca diária incremental (janela móvel) da taxa CDI na API do BCB.
- [ ] **Construção da Camada Silver (Parquet & Data Lakehouse):**
  - [ ] Criar a estrutura e bucket da Camada Silver (`cvm-lakehouse-silver-dev`).
  - [ ] Desenvolver o parser com tipagem forte (`Date`, `Float64`, `String`) e conversão de arquivos brutos para Apache Parquet com compressão Snappy.
  - [ ] Consolidar a matriz mensal de **Fundos de Crédito Privado Elegíveis** aplicando os 5 filtros de carteira no dataset `CDA_FI`.
- [ ] **Camada Gold & Migração para DAGs Airflow:**
  - [ ] Calcular rentabilidade nominal, acumulada e % do CDI (produtório da taxa diária).
  - [ ] Empacotar os scripts do protótipo Jupyter em `PythonOperator` em DAGs modularizadas do Airflow.

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
- [x] Configuração da Conexão GCP (`google_cloud_default`) via Airflow UI / CLI.
- [x] Organização e padronização do diretório `dags/` (arquitetura DataOps).
- [x] Construção do pipeline de ingestão da Camada Bronze (extração de dados da CVM para o Google Cloud Storage).