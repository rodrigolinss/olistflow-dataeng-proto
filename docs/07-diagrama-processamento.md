# 4.7 — Diagramas de Processamento de Dados

Este documento reúne os **diagramas profissionais** do pipeline OlistFlow que será implementado na Parte 2. Cada diagrama corresponde a uma vista distinta do mesmo sistema — o conjunto cobre as quatro perspectivas que um time de engenharia de dados normalmente precisa documentar:

| # | Vista | Responde à pergunta | Público-alvo |
|---|-------|---------------------|--------------|
| D1 | **Fluxo end-to-end** (lógico, com schedules) | Como os dados fluem da fonte ao dashboard? | Arquiteto, squad, stakeholder técnico |
| D2 | **Orquestração Airflow** (DAGs e dependências) | Qual DAG dispara o quê, e quando? | Engenheiro de dados, SRE/plantão |
| D3 | **DFD lógico por domínio** | Quem produz e quem consome cada data product? | Product owner, analista, novo integrante |
| D4 | **Deployment físico (Docker)** | Quais containers rodam, em que rede, com que volumes? | DevOps, quem vai executar na máquina local |

Todos os diagramas são escritos em **Mermaid** (renderizado direto pelo GitHub) e exportados em **PNG + SVG** em `docs/img/` para uso em relatórios/apresentações. Schedules estão anotados nos próprios diagramas para que a cadência do pipeline seja legível sem consultar os DAGs.

---

## D1 — Fluxo de processamento end-to-end

Mostra as camadas do Medalhão com os dois caminhos de ingestão (batch e streaming), os schedules de cada ponto de entrada, e as travessias de qualidade.

```mermaid
flowchart LR
    %% ===== FONTES =====
    subgraph SRC["🌐 Fontes"]
        direction TB
        ERP[("PostgreSQL 16<br/>Olist ERP<br/>~1,5M linhas")]
        SIM["Simulador<br/>Python<br/>~50k eventos/dia"]
    end

    %% ===== INGESTÃO =====
    subgraph ING["📥 Ingestão"]
        direction TB
        IB["Python + pyarrow<br/><b>@02:00 diário</b>"]
        IS["Redis Streams<br/>+ consumer Python<br/><b>*/5 min</b>"]
    end

    %% ===== BRONZE =====
    subgraph BRONZE["🥉 Bronze — Raw imutável"]
        direction TB
        BR1["olist/&lt;tabela&gt;<br/>dt_ingest=YYYY-MM-DD<br/>Parquet + Snappy"]
        BR2["events/<br/>dt=YYYY-MM-DD/hr=HH<br/>Parquet + Snappy"]
    end

    %% ===== SILVER =====
    subgraph SILVER["🥈 Silver — Clean & Conformed"]
        direction TB
        STG1["stg_olist_*<br/>cast · dedup · null-handling"]
        STG2["stg_events<br/>schema validation · sessions"]
        INT["int_*<br/>lógica compartilhada"]
    end

    %% ===== GOLD =====
    subgraph GOLD["🥇 Gold — Business-ready"]
        direction TB
        FCT["fct_pedidos<br/>fct_entregas<br/>fct_funil"]
        DIM["dim_clientes · dim_produtos<br/>dim_vendedores · dim_geo<br/>dim_tempo · dim_sessao"]
    end

    %% ===== SERVING =====
    subgraph SERVE["📊 Serving"]
        direction TB
        MB["Metabase<br/>5 dashboards"]
        API["FastAPI<br/><i>opcional</i>"]
    end

    %% ===== CROSS-CUTTING =====
    subgraph ORCH["⚙️ Orquestração"]
        AF["Apache Airflow<br/>LocalExecutor"]
    end

    subgraph QUAL["🔬 Qualidade & Governança"]
        direction TB
        DBT["dbt tests<br/>schema.yml"]
        GE["Great Expectations<br/>checks ricos"]
        DOC["dbt docs<br/>catálogo + linhagem"]
    end

    %% ===== FLUXOS PRINCIPAIS =====
    ERP -->|"SELECT · JDBC"| IB
    SIM -->|"XADD"| IS
    IB -->|"PUT partição"| BR1
    IS -->|"PUT partição horária"| BR2

    BR1 --> STG1
    BR2 --> STG2
    STG1 --> INT
    STG2 --> INT
    INT --> FCT
    INT --> DIM

    FCT --> MB
    DIM --> MB
    FCT --> API
    DIM --> API

    %% ===== CONTROLE =====
    AF -. "schedule @02:00" .-> IB
    AF -. "schedule */5min" .-> IS
    AF -. "dbt run" .-> STG1
    AF -. "dbt run" .-> FCT

    DBT -. "bloqueia inválidos" .-> STG1
    DBT -. "bloqueia inválidos" .-> FCT
    GE -. "valida distribuições" .-> FCT
    DOC -.-> DIM

    %% ===== ESTILO =====
    classDef srcStyle fill:#5a1e2a,stroke:#ef476f,color:#fff,stroke-width:2px
    classDef ingStyle fill:#5a4720,stroke:#dc8a00,color:#fff,stroke-width:2px
    classDef bronzeStyle fill:#663b18,stroke:#cd7f32,color:#fff,stroke-width:2px
    classDef silverStyle fill:#444,stroke:#aaa,color:#fff,stroke-width:2px
    classDef goldStyle fill:#665020,stroke:#ffd166,color:#fff,stroke-width:2px
    classDef serveStyle fill:#1e4a38,stroke:#06d6a0,color:#fff,stroke-width:2px
    classDef orchStyle fill:#3a2a4a,stroke:#b19cd9,color:#fff,stroke-width:2px
    classDef qualStyle fill:#1e3a4a,stroke:#4aa3df,color:#fff,stroke-width:2px

    class SRC srcStyle
    class ING ingStyle
    class BRONZE bronzeStyle
    class SILVER silverStyle
    class GOLD goldStyle
    class SERVE serveStyle
    class ORCH orchStyle
    class QUAL qualStyle
```

**Como ler este diagrama.** O fluxo avança da esquerda para a direita, cruzando as camadas de qualidade crescente do Medalhão. As linhas sólidas representam **fluxo de dados** (Parquet sendo escrito/lido); as tracejadas representam **controle** (Airflow dispara, dbt/GE validam). Anotações em negrito (`@02:00 diário`, `*/5 min`) são os schedules **reais** dos DAGs descritos em D2.

**Contratos implícitos:**

- Bronze → Silver: mesmo esquema relacional, tipagem canonizada.
- Silver → Gold: modelo dimensional (Kimball); chaves substitutas `md5(natural_key || effective_date)`.
- Gold → Serving: apenas leitura; consumidores nunca escrevem em DuckDB.

---

## D2 — Orquestração Airflow (DAGs e dependências)

Airflow opera dois ritmos: o **trem diário** das 02:00 (batch completo, dependências em cascata) e um **consumidor de streaming** que roda a cada 5 minutos, totalmente desacoplado do trem.

```mermaid
flowchart TB
    %% ===== TRIGGERS =====
    subgraph TRIG["🕒 Triggers"]
        direction LR
        CRON_D["CRON<br/>0 2 * * *<br/><b>@02:00 diário</b>"]
        CRON_S["CRON<br/>*/5 * * * *<br/><b>a cada 5 min</b>"]
    end

    %% ===== DAG MESTRE =====
    subgraph MASTER["DAG mestre — olistflow_daily"]
        direction TB
        M0["start"]
        M1["ingest_all"]
        M2["dbt_silver"]
        M3["dbt_gold"]
        M4["dbt_tests"]
        M5["ge_checks"]
        M6["refresh_metabase"]
        M7["end"]

        M0 --> M1 --> M2 --> M3 --> M4 --> M5 --> M6 --> M7
    end

    %% ===== SUB-DAGs POR DOMÍNIO =====
    subgraph DOMAINS["DAGs por domínio (disparados por ingest_all)"]
        direction TB
        D_VEN["vendas_daily<br/>orders · items · payments"]
        D_CAT["catalogo_daily<br/>products · categories"]
        D_CLI["clientes_daily<br/>customers · reviews"]
        D_LOG["logistica_daily<br/>geolocation"]
        D_SEL["sellers_daily<br/>sellers"]
    end

    %% ===== DAG STREAMING =====
    subgraph STREAM["DAG streaming — marketing_events_5min"]
        direction LR
        S1["poll_redis<br/>XREADGROUP"]
        S2["batch_to_parquet"]
        S3["dbt_incremental<br/>stg_events"]
        S4["ge_realtime_checks"]

        S1 --> S2 --> S3 --> S4
    end

    %% ===== QUALITY GATES =====
    subgraph TESTS["Gates de qualidade"]
        direction LR
        T1["dbt test<br/>not_null · unique"]
        T2["GE<br/>distribuições · refs"]
    end

    %% ===== LIGAÇÕES =====
    CRON_D --> M0
    CRON_S --> S1

    M1 --> D_VEN
    M1 --> D_CAT
    M1 --> D_CLI
    M1 --> D_LOG
    M1 --> D_SEL

    M4 --> T1
    M5 --> T2

    %% ===== FALHAS =====
    T1 -. "falha crítica" .-> M7
    T2 -. "falha crítica" .-> M7
    S4 -. "alerta" .-> M4

    %% ===== ESTILO =====
    classDef trig fill:#3a2a4a,stroke:#b19cd9,color:#fff
    classDef master fill:#1e4a38,stroke:#06d6a0,color:#fff
    classDef dom fill:#1e3a5f,stroke:#4a90e2,color:#fff
    classDef stream fill:#4a3f1f,stroke:#ffd166,color:#fff
    classDef test fill:#5a1e2a,stroke:#ef476f,color:#fff

    class TRIG trig
    class MASTER master
    class DOMAINS dom
    class STREAM stream
    class TESTS test
```

**Como ler.** Dois CRONs independentes alimentam dois fluxos. O **trem diário** é sequencial por design (a Silver depende do Bronze completo; Gold depende da Silver limpa). O **streaming** é fire-and-forget de 5 em 5 minutos, gravando em Bronze/events — o trem diário do dia seguinte consome essas partições normalmente. Gates de qualidade com falha crítica interrompem o pipeline antes do refresh dos dashboards, garantindo que **dado ruim nunca chega ao Metabase**.

**Princípio aplicado:** idempotência por sobrescrita de partição (`INSERT OVERWRITE`) — qualquer DAG pode ser religado para a mesma data sem duplicar registros.

---

## D3 — Fluxo de dados lógico por domínio (DFD)

Visão orientada a negócio: mostra **quem produz** e **quem consome** cada data product na camada Gold, destacando as dimensões conformadas (reusadas em múltiplos domínios).

```mermaid
flowchart LR
    %% ===== FONTES POR DOMÍNIO =====
    subgraph SRC_B["Fontes batch (Olist ERP)"]
        direction TB
        T_ORD[("orders · items<br/>payments")]
        T_PROD[("products<br/>categories")]
        T_CLI[("customers<br/>reviews")]
        T_GEO[("geolocation")]
        T_SEL[("sellers")]
    end

    subgraph SRC_S["Fonte streaming"]
        T_EVT["clickstream<br/>7 tipos de evento"]
    end

    %% ===== DOMÍNIOS =====
    subgraph D_VEN["💰 Vendas"]
        direction TB
        V_FCT["fct_pedidos<br/><i>grão: item × pedido</i>"]
        V_KPI["kpi_gmv · kpi_ticket"]
    end

    subgraph D_CAT["📦 Catálogo"]
        direction TB
        C_DIM["dim_produtos<br/><i>SCD2</i>"]
    end

    subgraph D_CLI["👤 Clientes"]
        direction TB
        U_DIM["dim_clientes<br/><i>enriquecido RFM</i>"]
    end

    subgraph D_LOG["🚚 Logística"]
        direction TB
        L_FCT["fct_entregas"]
        L_DIM["dim_geo<br/><i>centroide CEP</i>"]
    end

    subgraph D_MKT["📣 Marketing"]
        direction TB
        M_FCT["fct_funil<br/><i>grão: sessão</i>"]
        M_DIM["dim_sessao"]
    end

    subgraph D_SEL["🏪 Sellers"]
        direction TB
        S_DIM["dim_vendedores"]
        S_KPI["scorecard_seller"]
    end

    %% ===== CONSUMIDORES =====
    subgraph CONS["📊 Consumidores"]
        direction TB
        DASH_EXEC["Dash Executivo"]
        DASH_VEN["Dash Vendas"]
        DASH_LOG["Dash SLA Logístico"]
        DASH_FUN["Dash Funil"]
        DASH_SEL["Dash Sellers"]
    end

    %% ===== FLUXOS =====
    T_ORD --> V_FCT
    T_PROD --> C_DIM
    T_CLI --> U_DIM
    T_GEO --> L_DIM
    T_ORD --> L_FCT
    T_SEL --> S_DIM
    T_EVT --> M_FCT
    T_EVT --> M_DIM

    %% Conformed dimensions (reuso)
    C_DIM -. "conformed" .-> V_FCT
    U_DIM -. "conformed" .-> V_FCT
    S_DIM -. "conformed" .-> V_FCT
    U_DIM -. "conformed" .-> L_FCT
    U_DIM -. "conformed via order_id" .-> M_FCT
    V_FCT -. "join via order_id" .-> M_FCT

    %% KPIs
    V_FCT --> V_KPI
    S_DIM --> S_KPI
    V_FCT --> S_KPI

    %% Consumo
    V_KPI --> DASH_EXEC
    V_FCT --> DASH_VEN
    L_FCT --> DASH_LOG
    L_DIM --> DASH_LOG
    M_FCT --> DASH_FUN
    M_DIM --> DASH_FUN
    S_KPI --> DASH_SEL

    %% ===== ESTILO =====
    classDef srcStyle fill:#5a1e2a,stroke:#ef476f,color:#fff
    classDef domainStyle fill:#1e3a5f,stroke:#4a90e2,color:#fff
    classDef consStyle fill:#1e4a38,stroke:#06d6a0,color:#fff

    class SRC_B,SRC_S srcStyle
    class D_VEN,D_CAT,D_CLI,D_LOG,D_MKT,D_SEL domainStyle
    class CONS consStyle
```

**Como ler.** Setas sólidas são **produção**; tracejadas são **reuso entre domínios** (conformed dimensions). `dim_clientes` aparece 3× como conformed — é produzida **uma vez** pelo domínio Clientes e consumida por Vendas, Logística e Marketing. Essa é a concretização do princípio de Kimball vista em 03-dominios-servicos.md.

**Correlação batch↔streaming.** A seta `V_FCT ⟶ M_FCT "join via order_id"` mostra o ponto-chave: eventos de clickstream só se conectam ao comportamento transacional **após a compra**, via `order_id` emitido no evento `purchase`. Antes disso, `user_id` de marketing e `customer_id` de vendas são universos separados.

---

## D4 — Deployment físico (Docker Compose)

Como o stack é orquestrado na máquina local. Cada container é uma unidade de deploy independente; volumes persistem dados entre execuções.

```mermaid
flowchart TB
    %% ===== HOST =====
    subgraph HOST["💻 Host (notebook 8 GB)"]
        direction TB

        subgraph NET["🕸️ olistflow_net (bridge)"]
            direction TB

            subgraph OLTP["olist-postgres"]
                PG["PostgreSQL 16-alpine<br/>5432<br/>~200 MB RAM"]
            end

            subgraph BROKER["olist-redis"]
                RD["Redis 7-alpine<br/>6379<br/>~50 MB RAM"]
            end

            subgraph AF["airflow-stack"]
                direction TB
                AF_WEB["airflow-webserver<br/>8080"]
                AF_SCH["airflow-scheduler"]
                AF_WEB -.-> AF_SCH
            end

            subgraph MB["olist-metabase"]
                META["Metabase<br/>3000<br/>~800 MB RAM"]
            end

            subgraph SIM["olist-simulator"]
                SIMPY["Python simulator<br/>~80 MB RAM"]
            end
        end

        subgraph VOLS["📁 Volumes locais"]
            direction LR
            V_DATA["./data/<br/>bronze · duckdb"]
            V_DAGS["./orchestration/<br/>dags"]
            V_DBT["./transform/<br/>dbt project"]
            V_MB["./serving/<br/>metabase-data"]
            V_PG["pg_data"]
        end
    end

    %% ===== USUÁRIO =====
    USR["👤 Usuário<br/>browser"]

    %% ===== CONEXÕES INTERNAS =====
    SIMPY -->|"XADD"| RD
    AF_SCH -->|"SELECT"| PG
    AF_SCH -->|"XREADGROUP */5min"| RD
    AF_SCH -->|"dbt run → DuckDB"| V_DATA
    AF_SCH -->|"write parquet"| V_DATA
    META -->|"JDBC (read-only)"| V_DATA

    %% ===== MOUNTS =====
    AF_SCH -. "mount" .- V_DAGS
    AF_SCH -. "mount" .- V_DBT
    PG -. "mount" .- V_PG
    META -. "mount" .- V_MB

    %% ===== EXPOSIÇÕES =====
    USR -->|"localhost:8080"| AF_WEB
    USR -->|"localhost:3000"| META

    %% ===== ESTILO =====
    classDef db fill:#3a4a1e,stroke:#7cb342,color:#fff
    classDef cache fill:#4a1e1e,stroke:#ef5350,color:#fff
    classDef orch fill:#3a2a4a,stroke:#b19cd9,color:#fff
    classDef bi fill:#1e4a38,stroke:#06d6a0,color:#fff
    classDef sim fill:#4a3f1f,stroke:#ffd166,color:#fff
    classDef vol fill:#2a2a3a,stroke:#888,color:#fff
    classDef usr fill:#1e3a5f,stroke:#4a90e2,color:#fff

    class OLTP db
    class BROKER cache
    class AF orch
    class MB bi
    class SIM sim
    class VOLS vol
    class USR usr
```

**Como ler.** Todos os containers conversam pela rede `olistflow_net`. Portas expostas ao host: `5432` (Postgres para conectar com DBeaver/psql), `8080` (UI Airflow), `3000` (Metabase), `6379` (Redis para debug). O DuckDB **não é um container** — é um arquivo `.duckdb` dentro de `./data/`, aberto pelo Airflow-scheduler durante `dbt run` e pelo Metabase em modo leitura.

**Padrão de volumes.** Código fica na árvore do repositório (`./orchestration/dags`, `./transform`) e é montado nos containers — editar um modelo dbt localmente não exige rebuild. Dados (Bronze em Parquet, DuckDB) ficam em `./data/`, **fora do controle do Git** (`.gitignore`), garantindo que um clone novo comece limpo e o `make rebuild` regenere tudo a partir do Postgres + simulador.

**Total de memória em regime:** ~2,6 GB — deixa ≈5 GB livres no host de 8 GB para OS + browser + IDE, conforme matriz em 05-tecnologias.md.

---

## Anexos (PNG/SVG exportados)

Para uso em relatórios impressos, slides ou ambientes sem render Mermaid, cada diagrama tem versão estática gerada via `mermaid-cli` (`mmdc`):

| Diagrama | PNG | SVG |
|----------|-----|-----|
| D1 — Fluxo end-to-end | [d1-end-to-end.png](img/d1-end-to-end.png) | [d1-end-to-end.svg](img/d1-end-to-end.svg) |
| D2 — Orquestração Airflow | [d2-airflow-dags.png](img/d2-airflow-dags.png) | [d2-airflow-dags.svg](img/d2-airflow-dags.svg) |
| D3 — DFD por domínio | [d3-dfd-dominios.png](img/d3-dfd-dominios.png) | [d3-dfd-dominios.svg](img/d3-dfd-dominios.svg) |
| D4 — Deployment Docker | [d4-deployment-docker.png](img/d4-deployment-docker.png) | [d4-deployment-docker.svg](img/d4-deployment-docker.svg) |

Para regerar os anexos após editar um diagrama:

```bash
# a partir da raiz do repositório
npx -p @mermaid-js/mermaid-cli mmdc \
  -i docs/07-diagrama-processamento.md \
  -o docs/img/diagram.png \
  -t dark -b '#0d0d15' \
  --pdfFit
```

---

## Relação com os outros documentos

- **[04-arquitetura.md](04-arquitetura.md)** — justificativa da Arquitetura Medalhão; D1 é a versão visual "com cadência" daquele texto.
- **[03-dominios-servicos.md](03-dominios-servicos.md)** — D3 é a materialização DFD da divisão por domínios.
- **[05-tecnologias.md](05-tecnologias.md)** — D4 reflete a matriz consolidada de stack com os nomes de containers reais.
- **[02-dados.md](02-dados.md)** — volumes e formatos que sustentam D1.
