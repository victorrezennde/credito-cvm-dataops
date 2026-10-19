# Changelog - Crédito CVM DataOps

Todas as alterações notáveis e marcos do projeto serão documentados neste arquivo.

---

## [0.3.0] - 2026-10-18

### 🧪 Prototipagem dos 5 Filtros, Mapeamento Master/FIC & Diagnóstico Quantum Axis

#### Commits Relacionados:
- `feat: implementa pipeline de screening posicional em 5 etapas sequenciais`
- `feat: adiciona mapeamento relacional e regra de heranca de elegibilidade master para fic`
- `refactor: inclui tratamento de fics na etapa 1 e isolamento de pl sensivel para evitar double counting`
- `test: isola carteira divulgada mais recente por fundo respeitando janela de sigilo cvm (d-15 a d-90)`
- `docs: mapeia divergencias metodologicas de amostragem entre motor cvm e relatorio quantum axis`
- `docs: consolida questionario executivo de negocios para definicao de requisitos da camada silver`

#### Detalhamento das Implementações:

1. **Desenvolvimento do Pipeline de Screening Posicional (5 Etapas):**
   * **Etapa 0 (Base Bruta):** Filtro de integridade isolando fundos ativos com Patrimônio Líquido ($PL > 0$).
   * **Etapa 1 (Crédito Privado ≥ 10% - Masters + FICs):** Ajuste do algoritmo para considerar tanto alocação direta em papéis de dívida corporativa/bancária quanto alocação indireta em cotas (`BLC_7` $\ge 50\%$), preservando os FICs de distribuição.
   * **Etapa 2 (Exclusão no Exterior / IE):** Exclusão de fundos com exposição a ativos offshore e legislação internacional.
   * **Etapa 3 (Teto de FIDC ≤ 30%):** Restrição de exposição a FIDCs para isolar o crédito corporativo pulverizado tradicional.
   * **Etapa 4 (Exclusão de Puramente Soberanos):** Refinamento da regra para filtrar apenas fundos $100\%$ Soberanos/DI Simples ($> 95\%$ LFTs), permitindo a manutenção de caixa para liquidez.
   * **Etapa 5 (Teto de Ações ≤ 3% do PL):** Tolerância marginal para pequenas posições de conversão de dívida (*debt-to-equity swap*).

2. **Captura da Posição Mais Recente por Fundo (`D-15` a `D-90`):**
   * Implementação da varredura por `.max()` na data de competência (`DT_COMPTC` / `DT_COMPET`) para cada CNPJ individualmente.
   * Garantia de amostragem atualizada e sem duplicação histórica, respeitando a janela de sigilo regulatório de até 90 dias da CVM.

3. **Mapeamento Anti-Dupla Contagem de PL (Master vs. FIC):**
   * Definição da regra de consolidação de métricas: medem-se o **PL e os ativos de ponta nos Fundos Masters**, enquanto mede-se o **fluxo de captação e resgate nos Fundos FICs**.
   * Modelagem da coluna de controle `VL_PL_SENSIVEL_MERCADO` para zerar o PL de FICs em somatórias agregadas de mercado.

4. **Diagnóstico do Relatório Quantum Axis (`captacao_quantum`):**
   * Identificação da divergência volumétrica entre os **6.071 fundos do Quantum** e os **~2.500 a 3.200 fundos do motor CVM**.
   * Mapeamento das causas de inflação no Quantum: abordagem cadastral *Top-Down*, presença de duplicidade de PL entre Master e FICs, desmembramento de subclasses da Resolução CVM 175 (`CIC` / `FIF`) e inclusão de fundos Exclusivos/Family Offices/Previdência.

5. **Estruturação do Roteiro de Validação com o Negócio:**
   * Formulação das perguntas executivas sobre consolidação de AUM, escopo de público-alvo (Varejo vs. Exclusivos) e enquadramento de Letras Financeiras, FI-Infra e Previdência.

---

### 🎯 Próximos Passos Programados
- [ ] **Validação de Negócio com Gestores (Buy-Side):**
  - [ ] Apresentar o roteiro de perguntas para definição final do perímetro da amostragem.
  - [ ] Travar as decisões relativas a fundos Exclusivos vs. Abertos e subclasses da CVM 175.
- [ ] **Construção da Camada Silver (Parquet & Data Lakehouse):**
  - [ ] Implementar as transformações finais com de-duplicação de PL e chaves relacionais (`CNPJ_MASTER_DIRETO`).
  - [ ] Criar as tabelas derivadas para o cálculo de fluxo de captação e resgate na Camada Gold.
- [ ] **Automação do Informe Diário (E-mail / Gold):**
  - [ ] Desenvolver a geração do relatório em e-mail HTML (*Top Captação*, *Top Resgate* e *Top PL*).

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