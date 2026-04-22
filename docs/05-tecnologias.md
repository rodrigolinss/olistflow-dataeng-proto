# 4.5 — Tecnologias (Como será feito)

Este documento detalha **cada tecnologia** escolhida para o OlistFlow, justificando a decisão em função do problema, do contexto (projeto acadêmico em máquina modesta) e das alternativas consideradas. A avaliação pesa 2,0 pontos para esta seção — portanto cada escolha é tratada com a seguinte estrutura:

> **Escolha · Alternativas consideradas · Por que esta ganhou · Como se integra ao restante**

---

## 1. Ingestão

### 1.1 Ingestão batch — Python + pyarrow

- **Alternativas consideradas:** Airbyte, Fivetran, Apache NiFi, Meltano, Singer.
- **Por que Python puro:** todas as alternativas são ferramentas de orquestração/ingestão completas — excelentes em produção, mas pesadas para o escopo. Airbyte exige 4+ containers e ~3 GB de RAM ociosos só para o painel de controle. Fivetran é SaaS pago. Para 10 tabelas estáticas vindas de um Postgres local, **o overhead dessas ferramentas supera o benefício**. Um script Python de ~100 linhas com `psycopg2` + `pyarrow` atende ao caso, é 100% reprodutível, e exige zero infraestrutura adicional.
- **Integração:** o script é chamado como operador `PythonOperator` do Airflow; escreve Parquet diretamente no diretório `bronze/olist/<tabela>/dt_ingest=YYYY-MM-DD/`; idempotente por sobrescrita de partição.

### 1.2 Ingestão streaming — Redis Streams + consumer Python

- **Alternativas consideradas:** Apache Kafka, AWS Kinesis, Google Pub/Sub, RabbitMQ, NATS.
- **Por que Redis Streams:** Kafka é o padrão de mercado mas exige ZooKeeper/KRaft + brokers, consumindo >2 GB de RAM só em overhead — incompatível com notebook modesto. Kinesis e Pub/Sub exigem conta em nuvem. RabbitMQ é orientado a filas (não a streams append-only), perdendo a semântica de log imutável que queremos demonstrar. **Redis Streams** tem a mesma abstração conceitual do Kafka (log ordenado, consumer groups, replay) em um único container de ~50 MB — **didaticamente idêntico**, operacionalmente trivial.
- **Fallback explícito:** se até o Redis for inviável nas máquinas da sala, o simulador escreve diretamente em arquivos JSONL append-only — preservando o fluxo conceitual sem broker.
- **Integração:** consumer Python em DAG Airflow de schedule `*/5 * * * *` (a cada 5 min). Lê eventos desde o último offset persistido, converte para Parquet particionado por hora.

### 1.3 Simulador de eventos — script Python próprio

- **Justificativa:** não existe fonte pública de clickstream acoplada ao dataset Olist. Um gerador sintético controlado nos permite **ensinar o conceito** de streaming sem complexidade externa, e nos dá total previsibilidade para testes.
- **Implementação:** script que modela a jornada do usuário como máquina de estados (landing → navegação → busca → PDP → carrinho → checkout), distribui eventos numa curva diária realista (picos à noite), ocasionalmente referencia `order_id`s reais da Olist para permitir joins posteriores. Configurável via `.env` (taxa de eventos, proporção de abandono, etc.).

---

## 2. Armazenamento

### 2.1 PostgreSQL (OLTP simulado)

- **Alternativas consideradas:** MySQL, SQLite, MS SQL Server, simplesmente ler CSVs diretamente sem "fingir" um ERP.
- **Por que Postgres:** é o banco relacional open-source mais comum em marketplaces reais (a própria Olist usa variantes de SGBD relacional). Simular o ERP com Postgres torna o exercício **realista** — a ingestão precisa lidar com JDBC, cast de tipos, tratamento de deltas — em vez de trivializar o problema lendo CSV. SQLite seria leve demais (não há conexão de rede, sem concorrência real). MySQL funcionaria, mas Postgres tem tipos mais ricos (`jsonb`, arrays) que preparam para evoluções futuras.
- **Configuração:** container Docker oficial `postgres:16-alpine`, ~200 MB de RAM, populado na primeira execução via script de load dos CSVs da Olist.
- **Integração:** ingestão batch consulta via `psycopg2`. Credenciais em `.env`.

### 2.2 Data Lake local — Parquet + diretórios particionados

- **Alternativas consideradas:** AWS S3, MinIO, Azure Blob, Google Cloud Storage, HDFS.
- **Por que "diretórios + Parquet":** a semântica de um data lake é **arquivos objetificados + particionamento por prefixo**. Essa semântica é trivialmente reproduzível no sistema de arquivos local: cada partição é uma pasta, cada arquivo Parquet é um objeto imutável. MinIO (S3-compatível) é a alternativa mais fiel, mas adiciona um container de ~150 MB e um serviço extra para ~45 MB de dados — desproporcional. **A topologia é portável**: quando/se migrarmos para S3, basta trocar o path `./data/bronze/...` por `s3://olistflow/bronze/...` no perfil do dbt.
- **Por que Parquet (não CSV, não JSON):** Parquet é colunar, tipado, comprimido. Para o volume do projeto (~50 MB total), Parquet corta 60% do espaço de CSV e acelera consultas analíticas em 10–50×. É o formato canônico em lakehouses modernos (Databricks, Snowflake external tables, BigQuery external tables, Athena).
- **Particionamento:** Bronze particionada por `dt_ingest` (para permitir reprocessar dias específicos); streaming particionado por `dt` e `hr` (para consultas por janela).

### 2.3 DuckDB (engine OLAP)

- **Alternativas consideradas:** Apache Spark, Trino/Presto, ClickHouse, Polars, pandas direto.
- **Por que DuckDB:** é uma das decisões mais estratégicas do projeto. DuckDB é um engine SQL analítico **in-process** — zero servidor, zero configuração — que lê Parquet/CSV/JSON nativamente, executa SQL ANSI com performance comparável a warehouses para dados de até algumas centenas de GB, e integra-se ao **dbt** via adaptador oficial. Para o escopo do projeto:
  - **vs. Spark:** Spark precisa de JVM, cluster (mesmo que local), é massivamente overkill para 45 MB de dados e torna o ambiente frágil em notebooks modestos.
  - **vs. Trino/Presto:** requer catálogo, coordenador, workers — mesmas objeções do Spark.
  - **vs. ClickHouse:** é um banco com estado persistente, não uma engine sobre arquivos. Mais indicado para ingestão contínua/serving. Para o caminho analítico do OlistFlow, gera sobreposição com a Gold.
  - **vs. pandas direto:** pandas não é SQL. Perdemos a padronização de modelagem com dbt, a linhagem e os testes.
- **Integração:** dbt-duckdb é o adaptador; cada `dbt run` abre um arquivo DuckDB no disco, executa os modelos em ordem topológica, persiste. Metabase conecta via JDBC/ODBC do DuckDB.

---

## 3. Processamento e Transformação

### 3.1 dbt-core (transformação analítica)

- **Alternativas consideradas:** Apache Spark (PySpark), Apache Beam, Flink, SQLMesh, Dataform, scripts SQL soltos, stored procedures no Postgres.
- **Por que dbt:** dbt é hoje o padrão de fato para transformações analíticas em lakehouses/warehouses e **materializa os princípios vistos em aula**:
  - **Versionamento:** todo modelo é um arquivo `.sql` no Git.
  - **Testes como código:** `schema.yml` declara testes de `not_null`, `unique`, `relationships`, `accepted_values` que rodam com o modelo.
  - **Linhagem automática:** `ref('modelo_x')` gera grafo de dependências; `dbt docs` visualiza.
  - **Documentação viva:** descrições de colunas ficam ao lado do SQL, geram site estático.
  - **Paralelização:** dbt resolve o DAG e executa modelos independentes em paralelo.
- **vs. Spark/Beam/Flink:** essas são engines de compute distribuído, não frameworks de modelagem. Usadas em conjunto com dbt em produção; sozinhas, não expressam o modelo declarativo. Também não cabem no hardware.
- **vs. SQLMesh:** excelente ferramenta moderna (suporte nativo a "virtual environments" de dados), mas o ecossistema em português e material didático é muito menor — desvantagem para uma apresentação em sala.
- **vs. stored procedures no Postgres:** inviável porque a transformação está no lakehouse (DuckDB sobre Parquet), não no OLTP.
- **Integração:** adaptador `dbt-duckdb` é a ponte; o projeto dbt fica em `./transform/` no repositório, versionado junto com o código do pipeline.

### 3.2 Python + pandas/polars (transformações pontuais)

- **Uso:** somente onde SQL não é natural — ex: parsing complexo de `user_agent`, hash determinístico de campos de PII, cálculo de centroide de geolocalização. A regra é: **SQL primeiro (dbt), Python só quando necessário**.
- **Polars preferido a pandas** para datasets >100 MB em memória (performance 5–10× superior). Para scripts pequenos e legibilidade, pandas continua sendo a escolha.

---

## 4. Orquestração

### 4.1 Apache Airflow (local, docker-compose)

- **Alternativas consideradas:** Prefect, Dagster, Luigi, Mage, cron simples, GitHub Actions agendadas.
- **Por que Airflow:** é o padrão de mercado absoluto para orquestração de pipelines de dados. Razões concretas para o contexto deste projeto:
  - **Reconhecimento profissional:** apresentação em sala e futura empregabilidade — saber Airflow é obrigatório em vagas de engenharia de dados.
  - **DAG como código Python:** alinhado com o que já temos (ingestores em Python).
  - **UI visual robusta:** grafo de dependências, logs por task, retentativas — ótimo material para a apresentação.
  - **Ecossistema de operadores:** `PostgresOperator`, `BashOperator`, `DbtDagFactory`, `SSHOperator` — cobrem 100% dos nossos casos.
- **vs. Prefect:** mais leve e moderno, melhor DX, mas material didático em PT-BR e presença no mercado brasileiro ainda são menores.
- **vs. Dagster:** tecnicamente superior em muitos aspectos (asset-centric, tipagem forte), mas curva de aprendizado maior; ferramenta mais recente, menos conhecida em sala de aula.
- **vs. cron / GitHub Actions:** insuficientes — não oferecem visualização de dependências, logs unificados, nem retentativa inteligente.
- **Configuração:** docker-compose oficial do Airflow modo `LocalExecutor` (sem Celery/Redis), ~800 MB de RAM. DAGs em `./orchestration/dags/`.

### 4.2 Estrutura de DAGs

- **DAGs por domínio** (`vendas_daily`, `catalogo_daily`, `streaming_5min`) — separação alinhada com 4.3.
- **DAG mestre `olistflow_daily`** — coordena ordem global: ingestão → Silver → Gold → testes → refresh Metabase.
- **Uso de TaskFlow API** (decoradores `@task`) em vez de operadores clássicos onde couber — código mais limpo e Pythonic.

---

## 5. Serving (Consumo)

### 5.1 Metabase (BI / dashboards)

- **Alternativas consideradas:** Apache Superset, Power BI, Looker Studio, Tableau Public, Grafana.
- **Por que Metabase:** open-source, leve (container único de ~800 MB rodando), UI amigável para não-técnicos (critério central — dashboards serão consumidos por stakeholders de negócio no exemplo didático), conector nativo para DuckDB, setup em < 10 minutos.
- **vs. Superset:** funcionalmente mais rico, mas requer maior configuração (meta-database, workers) e tem curva de aprendizado maior para analistas de negócio.
- **vs. Power BI / Looker Studio / Tableau:** SaaS ou desktop-bound, conflitam com escolha de stack 100% reprodutível em máquina local e open-source.
- **vs. Grafana:** otimizado para métricas de observabilidade, não para BI analítico de vendas.
- **Integração:** container Docker conectando ao arquivo DuckDB via JDBC. Dashboards versionados via export JSON no diretório `./serving/metabase/`.

### 5.2 FastAPI (endpoints programáticos — opcional)

- **Uso:** expor agregados da Gold como JSON para consumo por aplicações (simulando um "API de métricas" que um app mobile consumiria).
- **Por que FastAPI:** padrão moderno Python, type hints nativos (gera doc OpenAPI automática), baixo overhead.
- **Status:** stretch goal da Parte 2 — implementado se houver tempo após o core do pipeline.

---

## 6. Correntes (Undercurrents) do Ciclo de Vida

O livro-base da disciplina (Reis & Housley, *Fundamentals of Data Engineering*) destaca seis **correntes** que atravessam todas as etapas do ciclo. Abaixo, como cada uma se manifesta no OlistFlow:

### 6.1 Segurança

- **Credenciais** (Postgres, Redis, Metabase) ficam em `.env` listado em `.gitignore`. Nenhum segredo no repositório público.
- **Princípio de menor privilégio:** usuário Postgres do pipeline tem permissão apenas de `SELECT` nas tabelas da Olist; não pode escrever na base operacional.
- **PII** já vem anonimizada pela Olist (hash em identificadores, CEP truncado). O pipeline reforça: logs nunca imprimem colunas `*_id` diretas de clientes; dashboards usam agregações, não listas com PII.
- **Em produção futura:** migrar para gestor de segredos (Vault, AWS Secrets Manager) e RBAC no Metabase.

### 6.2 Gestão de Dados e Governança

- **Catálogo:** dbt docs gera documentação completa de todas as tabelas, colunas e métricas; roda como site estático local (`dbt docs serve`).
- **Linhagem:** o grafo do dbt (`dbt ls --output json`) é o registro autoritativo de quem depende de quem.
- **Versionamento de schema:** todo modelo dbt é SQL no Git — `git blame` responde "quando esse campo apareceu, por quê e por quem".
- **Política de retenção:** Bronze indefinida; Silver/Gold podem ser truncadas e reconstruídas a qualquer momento.

### 6.3 DataOps

- **CI leve via GitHub Actions:** a cada push, roda `dbt parse` + `dbt compile` + `dbt test --target ci` contra um sample de dados. Evita que modelos quebrados cheguem à main.
- **Pre-commit hooks:** `sqlfluff` valida estilo SQL; `yamllint` valida arquivos de configuração.
- **Código versionado** incluindo DAGs, modelos dbt, scripts de ingestão e dashboards Metabase (exportados em JSON).

### 6.4 Arquitetura de Dados

- **Princípios aplicados:** Medalhão (documentado em 04-arquitetura.md), separação de camadas, idempotência, imutabilidade de Bronze, contratos de schema via dbt.

### 6.5 Orquestração

- **Airflow como espinha dorsal:** nenhum job é disparado manualmente; tudo passa pelo scheduler. Logs, histórico e SLAs ficam centralizados na UI.

### 6.6 Engenharia de Software

- **Estrutura de repositório** padronizada: `ingestion/`, `transform/`, `orchestration/`, `serving/`, `docs/`.
- **Testes automatizados:** dbt tests são parte do pipeline, não atividade separada.
- **Revisão por pares:** PRs no GitHub antes de merge à main (aplicável à dupla).
- **Linguagem única:** Python 3.12 + SQL (dialeto DuckDB). Zero dependência de outras linguagens.

---

## Matriz consolidada de stack

| Etapa do ciclo | Tecnologia | Peso na RAM (aprox.) | Alternativa futura (cloud) |
|----------------|------------|----------------------|----------------------------|
| Ingestão batch | Python + pyarrow | 100 MB | Airbyte, Fivetran, CDC via Debezium |
| Ingestão streaming | Redis Streams + consumer Python | 150 MB | Kafka MSK, Kinesis, Pub/Sub |
| OLTP de origem | PostgreSQL 16 | 200 MB | RDS, Cloud SQL, Aurora |
| Data Lake | Parquet + diretórios locais | 0 (disco) | S3, GCS, Azure Blob |
| Engine analítica | DuckDB | 300 MB (execução) | BigQuery, Snowflake, Athena, Trino |
| Transformação | dbt-core (dbt-duckdb) | 50 MB | dbt Cloud |
| Orquestração | Apache Airflow (LocalExecutor) | 800 MB | MWAA, Cloud Composer, Dagster Cloud |
| Serving BI | Metabase | 800 MB | Looker, Power BI, Tableau |
| Serving API (opcional) | FastAPI | 80 MB | API Gateway + Lambda |
| Qualidade | dbt tests + Great Expectations | 100 MB | Soda, Monte Carlo, Anomalo |
| Observabilidade | Logs Airflow + dbt docs | — | Datadog, New Relic, Elementary |
| **Total estimado** | | **~2,6 GB** | |

O total de ~2,6 GB de RAM para o stack completo deixa margem para o sistema operacional + browser + IDE em máquinas de 8 GB — viabilidade confirmada.
