# Roteiro — OlistFlow

Apresentação de ~15 min, sem perguntas no final. Cada slide tem **o que falar** (a fala natural) e **o que entender** (pontos técnicos pra ter na cabeça caso o professor pergunte ou comente).

**Quem fala:** Rodrigo César Lins e Vitor Nascimento Franco — alternem por slide ou peguem blocos inteiros.

---

## Slide 1 — Capa

**O que falar:**
> "Oi pessoal. A gente é Rodrigo e Vitor, e vai apresentar o OlistFlow — um protótipo de pipeline de engenharia de dados pra um marketplace brasileiro de e-commerce."

**O que entender:**
- Parte 1 = planejamento e desenho. Parte 2 (avaliação futura) = implementar o que foi planejado.
- O cenário escolhido (e-commerce/marketplace) é um pretexto pra exercitar batch + streaming, ingestão, modelagem dimensional e orquestração.

---

## Slide 2 — Por que um marketplace?

**O que falar:**
> "E-commerce no Brasil já passou dos 185 bilhões por ano, e marketplace é o formato mais complexo: cada pedido envolve cliente, vendedor, pagamento, transportadora, review pós-entrega. Os dados nascem espalhados em sistemas que mal conversam — gera três dores: visão fragmentada, decisão lenta e sinais de comportamento que somem."

**O que entender:**
- O problema do projeto é **integração de dados**, não construir banco.
- "Visão fragmentada" = cada time tem sua fonte e contradiz os outros.
- "Sinais de comportamento perdidos" = eventos do site nunca cruzam com pedidos pós-compra.

**Frase-âncora:** *"o problema é integração, não armazenamento"*.

---

## Slide 3 — Os dois tipos de dados

**O que falar:**
> "A gente tem dois tipos de dados muito diferentes. Operacionais — pedidos, pagamentos, avaliações — vêm do ERP em batch diário. Comportamentais — navegação, busca, carrinho, abandono — vêm em streaming, com latência de minutos. A tabela mostra origem, formato, volume e cadência de cada um."

**O que entender:**
- **Batch (Olist Public Dataset):** ~1,5 milhão de linhas, 9 tabelas, ~45 MB em Parquet, periodicidade D-1, latência alvo até 6h no dashboard.
- **Streaming (clickstream sintético):** ~50k eventos/dia em JSONL, processado em micro-batch de 5 min via Redis Streams.
- Por que clickstream é sintético: não existe dataset público de comportamento acoplado ao Olist.
- Por que tratamos como streaming mesmo sendo micro-batch: fonte contínua, schema evolutivo, ordem importa pra funil de abandono.

**Frase-âncora:** *"contam histórias diferentes da mesma jornada"*.

---

## Slide 4 — Domínios: quem quer o quê

**O que falar:**
> "A gente identificou seis domínios, cada um com um interessado claro: Vendas serve liderança e analistas — GMV, ticket médio, conversão. Catálogo é referência usada por todos os outros — produto e categoria canônicos. Clientes serve Marketing e Vendas com segmentação RFM. Logística é da operação — SLA de entrega. Marketing serve Produto — funil e abandono. Sellers é gestão de marketplace — scorecard, NPS por vendedor."

Depois aponta pro diagrama:

> "O diagrama mostra que Batch alimenta cinco domínios, Streaming alimenta só Marketing, e todos os seis viram dashboards no Metabase. Catálogo e Clientes são reusados como dimensões — princípio de conformed dimensions do Kimball."

**O que entender:**
- Domínio aqui não é sinônimo de tabela — é um agrupamento de fatos, dimensões e responsáveis com mesmo "público".
- `dim_clientes` é construída uma vez (domínio Clientes) e consumida por Vendas, Logística e Marketing — **conformed dimension**.
- A separação por domínios prepara uma evolução pra Data Mesh, mas não somos Data Mesh ainda (Mesh exige múltiplos times).

**Frase-âncora:** *"domínio é definido pelo consumidor, não pela tabela"*.

---

## Slide 5 — Arquitetura (o coração)

**O que falar:**
> "Escolhemos Arquitetura Medalhão — Bronze, Silver, Gold — em cima de um Lakehouse local, com Lambda light separando batch e streaming na ingestão e juntando na Gold. Bronze captura tudo cru, Silver limpa e padroniza, Gold modela em estrela pro negócio consumir."

Como ler o diagrama (5 colunas, da esquerda pra direita):

> "Fontes alimentam Bronze (raw imutável em Parquet particionado), Bronze vira Silver (modelos `stg_` e `int_` em dbt — cast, dedup, joins), Silver vira Gold (fatos e dimensões Kimball), Gold abastece Metabase. Embaixo: Airflow orquestra os dois ritmos, e dbt tests + Great Expectations bloqueiam dado ruim antes de chegar no dashboard."

**O que entender:**
- **Medallion** é da Databricks, popularizada pelo Lakehouse — três camadas com qualidade crescente.
- **Lambda light** = a gente preserva a ideia de Lambda (caminhos batch e streaming separados) mas sem o overhead de duas stacks completas (Spark + Kafka).
- **Bronze imutável** garante reversibilidade: se errei na Silver/Gold, recomputo do zero.
- **Idempotência:** cada DAG sobrescreve sua partição (`INSERT OVERWRITE PARTITION`) — religar não duplica.
- **Lakehouse aqui** é conceitual: Parquet + DuckDB. Não é Delta Lake/Iceberg (não cabia no escopo).

---

## Slide 5b — Como o Airflow coordena (DAGs)

**O que falar:**
> "São dois ritmos independentes. O batch tem um DAG mestre que roda às 2 da manhã: ingere tudo do Postgres pra Bronze, roda dbt na Silver, roda dbt na Gold, passa pelos testes, passa pelo Great Expectations, e só depois atualiza o Metabase. O streaming roda a cada 5 minutos: lê do Redis com XREADGROUP, escreve em Parquet por hora, roda dbt incremental, valida. Os blocos azuis são gates — se um teste falha em nível crítico, o pipeline para antes do Metabase."

**O que entender:**
- **DAG = Directed Acyclic Graph** — Airflow representa o pipeline como um grafo de tasks.
- **LocalExecutor** (não Celery): tasks rodam no mesmo processo do scheduler — basta pro escopo, sem broker extra.
- **Gates de qualidade** (`dbt_tests`, `ge_checks`) usam `trigger_rule="all_success"` — se falham, downstream não roda.
- **dbt_incremental no streaming** = não recomputa tudo, só insere as novas partições de hora.
- Religar uma DAG não duplica dado porque cada task é idempotente por sobrescrita de partição.

---

## Slide 6 — Por que Medalhão e não outras?

**O que falar:**
> "A gente considerou as alternativas com seriedade. Data Warehouse puro perde o raw — não consegue reprocessar. Data Lake puro sem estrutura vira pântano. Lambda industrial com Kafka + Spark precisa de hardware que a gente não tem. Kappa trata tudo como streaming, mas a maior parte dos nossos dados é batch por natureza. Data Mesh exige múltiplos times com ownership — uma dupla não reproduz isso. Medalhão é o equilíbrio entre o que a literatura recomenda hoje e o que dá pra rodar num notebook."

**O que entender:**
- **DW puro** = ETL clássico, dado já transformado entra. Bom pra negócio estável, ruim pra exploração.
- **DL puro** = todo dado bruto, sem qualidade garantida. "Data swamp".
- **Lakehouse** = Lake + esquema/transação. Medallion é uma maneira de organizar um Lakehouse.
- **Lambda** = batch + streaming separados, unidos no consumo. **Kappa** = só streaming, batch é caso particular.
- **Data Mesh** = domínios são donos do dado e expõem como produto. Pressupõe maturidade organizacional.

---

## Slide 7 — Stack tecnológico

**O que falar:**
> "Cada escolha tem alternativa considerada e justificativa. A tabela mostra o stack inteiro com peso em RAM. Total: 2,6 GB, cabe em notebook de 8 GB. Tudo open-source — zero custo de licença. Cada linha tem uma alternativa cloud equivalente, então a topologia é migrável sem redesenho."

**O que entender:**
- **Python + pyarrow** na ingestão batch: 100 linhas de código vs Airbyte (4+ containers).
- **PostgreSQL** simulando ERP: torna a ingestão realista (JDBC, cast de tipos) em vez de ler CSV direto.
- **Parquet** colunar: 60% menor que CSV, queries 10–50× mais rápidas em DuckDB.
- **DuckDB** in-process: zero servidor, lê Parquet direto, integra com dbt via `dbt-duckdb`.
- **dbt-core** (não Cloud): SQL versionado em Git, testes em `schema.yml`, linhagem em `dbt docs`.
- **Airflow LocalExecutor**: sem Celery/Redis (não confundir: o Redis aqui é pros eventos, não pro Airflow).
- **Metabase** vs Superset: Metabase tem UX mais simples pra analista — é o critério.

---

## Slide 7b — Como roda na máquina (Docker Compose)

**O que falar:**
> "Cinco containers numa rede bridge do Docker: Postgres simulando o ERP, Redis pros eventos, Airflow orquestrando (webserver + scheduler), Metabase pros dashboards e o simulador de eventos. Mais dois volumes: pg_data persiste o Postgres, e ./data é o data lake — onde o scheduler escreve Parquet e o Metabase lê em read-only."

Como ler o diagrama:

> "Setas cinzas são leitura/escrita normal, vermelha é XADD do simulador no Redis, e tracejadas são montagens de volume. Tudo sobe com `make up`."

**O que entender:**
- **DuckDB não é container** — é um arquivo `.duckdb` dentro do volume `./data`. O scheduler abre pra escrever, o Metabase abre em read-only.
- **Volumes vs bind-mounts:** `pg_data` é volume nomeado (gerenciado pelo Docker). `./data` é bind-mount (pasta do host, dá pra editar fora do container).
- **Bridge network** = todos os containers se enxergam pelo nome do serviço (`olist-postgres`, `olist-redis`).
- Por que usuário acessa `localhost:8080` e `localhost:3000`: portas mapeadas no `docker-compose.yml`.

---

## Slide 8 — Três decisões não-triviais

**O que falar:**
> "Três decisões que não são óbvias e a gente quer destacar. Primeira: DuckDB no lugar de Spark — pra 45 MB de dados Spark seria overkill, DuckDB roda in-process e conversa com dbt nativamente. Segunda: Redis Streams no lugar de Kafka — mesma semântica de log imutável e consumer group em 50 MB de RAM. Terceira: dbt como espinha — transforma SQL em código versionável com testes e linhagem automática."

**O que entender:**
- **DuckDB é colunar**, OLAP, lê Parquet sem importar. Postgres é OLTP, otimizado pra transação. Não dá pra usar Postgres como engine analítico.
- **Redis Streams** tem `XADD`, `XREADGROUP`, consumer groups com offset persistente — análogo a Kafka, sem ZooKeeper/KRaft.
- **dbt** materializa modelos como tabelas/views, gera linhagem automática, expõe testes declarativos. Substitui stored procedure + ETL imperativo.
- **Plano B** se Redis não rodar: simulador escreve direto em JSONL append-only no disco. Mesmo pipeline a partir do Bronze.

---

## Slide 9 — Tradeoffs honestos

**O que falar:**
> "Honestidade sobre o que cedemos. Não tem alta disponibilidade real, o streaming é didático, é single-node. Em compensação, Bronze imutável + dbt versionado dá reversibilidade quase total — qualquer erro pode ser reprocessado com um comando, e a topologia inteira é portável pra cloud sem redesenho."

**O que entender (dimensões cobradas no enunciado):**
- **Acoplamento:** baixo — camadas conversam só via Parquet com schema contratado, dbt expressa dependências por `ref()`.
- **Escalabilidade:** vertical no protótipo (single-node). Horizontal exige trocar DuckDB por Athena/BigQuery — tarefa de configuração, não redesenho.
- **Disponibilidade:** baixa — falha do scheduler atrasa o batch (mas não perde, próximo run reprocessa).
- **Confiabilidade:** alta — gates de qualidade impedem dado inválido de avançar.
- **Reversibilidade:** excelente — Bronze imutável + dbt em Git → `dbt run --full-refresh` reconstrói tudo.

---

## Slide 10 — Parte 2 (5 ondas de implementação)

**O que falar:**
> "Pra Parte 2, dividimos em cinco ondas semanais: Fundação (sobe stack), Bronze (ingestão), Silver (limpa), Gold + QA (modelagem dimensional + Great Expectations), e Consumo (dashboards Metabase + CI). O critério de pronto: qualquer pessoa da sala consegue subir o pipeline inteiro em menos de 5 minutos no próprio notebook."

**O que entender:**
- Cada onda gera um entregável testável — não é "tudo no fim".
- A onda 1 sobe `docker-compose` e popula Postgres com o dataset Olist. O resto depende disso.
- O critério "5 minutos no notebook do colega" é o teste real de portabilidade — derruba projetos que só rodam na máquina do dono.

---

## Slide 11 — Encerramento

**O que falar:**
> "Resumindo: OlistFlow é um pipeline realista pra um marketplace brasileiro, com batch e streaming, arquitetura Medalhão e stack que cabe num notebook. Todo o material está no repositório do GitHub. Obrigado."

Agradece e sai. Não abre para perguntas.

---

## Lembretes pro dia

- Começar com calma — a sala demora uns segundos pra silenciar.
- Apontar pro projetor quando mostrar diagrama (não fica olhando só pro notebook).
- Se quiser ampliar um diagrama, **clique nele** — abre em tela cheia. `Esc` fecha.
- Cronometrar nos ensaios — se passar de 16 min, cortar uma frase em cada slide.
- Backup em PDF no pendrive caso o navegador trave.
- Tela cheia: tecla `F` no slide aberto. `→` avança, `←` volta.
