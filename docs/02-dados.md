# 4.2 — Definição e Classificação dos Dados

Este documento descreve **todas** as fontes de dados que o OlistFlow irá consumir, classificando-as pela natureza do processamento (batch ou streaming) e detalhando origem, formato, volume, periodicidade e latência esperada.

## Visão geral

O projeto consome dois tipos de dados:

1. **Dados operacionais (batch)** — registros transacionais persistidos em um sistema de origem (ERP / banco operacional). Representam o "estado do negócio" e são processados em lotes periódicos.
2. **Dados de streaming** — eventos de comportamento do usuário no front-end (clickstream), produzidos continuamente e consumidos em janelas de segundos ou minutos.

Essa dupla classificação é intencional: permite exercitar em um único projeto os dois paradigmas clássicos de engenharia de dados, justificando a escolha de uma arquitetura **Lambda light** (detalhada em 04-arquitetura.md).

---

## Dados operacionais (batch)

**Origem:** [Olist Brazilian E-Commerce Public Dataset](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce) — conjunto de 9 arquivos CSV publicado no Kaggle pela Olist Store, cobrindo ~100.000 pedidos reais realizados entre 2016 e 2018 em múltiplos marketplaces brasileiros. No OlistFlow, o dataset será carregado em um **PostgreSQL** que simula o sistema transacional (OLTP) do marketplace; a ingestão batch lê do Postgres, não diretamente dos CSVs — replicando o padrão realista em que um ERP é a fonte autoritativa.

**Natureza:** histórico, estruturado, relacional, chave-estrangeira entre tabelas.

**Frequência de ingestão:** diária (D-1). Em produção seria horária ou via CDC; aqui, diária simula um ciclo completo de batch noturno.

**Latência esperada:** dados disponíveis no dashboard até 6 horas após o fechamento do dia operacional.

### Tabelas (fontes individuais)

| Tabela | Descrição | Linhas (~) | Chave | Colunas-chave |
|--------|-----------|-----------|-------|---------------|
| `orders` | Pedidos realizados | 99.441 | `order_id` | status, datas de aprovação, envio, entrega |
| `order_items` | Itens de cada pedido | 112.650 | (`order_id`, `order_item_id`) | produto, vendedor, preço, frete |
| `customers` | Cadastro de clientes | 99.441 | `customer_id` | cidade, estado, CEP |
| `products` | Catálogo de produtos | 32.951 | `product_id` | categoria, peso, dimensões |
| `sellers` | Vendedores do marketplace | 3.095 | `seller_id` | cidade, estado, CEP |
| `order_payments` | Transações de pagamento | 103.886 | (`order_id`, `payment_sequential`) | tipo (cartão, boleto, voucher), valor, parcelas |
| `order_reviews` | Avaliações pós-entrega | 99.224 | `review_id` | nota (1-5), comentário |
| `geolocation` | Pontos geográficos por CEP | ~1.000.000 | `zip_code_prefix` | lat, lon, cidade, estado |
| `product_category_name_translation` | Tradução pt→en de categorias | 71 | `product_category_name` | nome em inglês |

**Volume total estimado:** ~1,5 milhão de registros, ≈ 120 MB em CSV puro, ≈ 45 MB em Parquet com compressão Snappy.

**Formato de origem:** CSV (UTF-8, vírgula como separador, primeira linha com header).

**Formato após ingestão:** Parquet particionado por data de ingestão, armazenado na camada Bronze do data lake local (diretório `./data/bronze/olist/<tabela>/dt_ingest=YYYY-MM-DD/`).

### Desafios conhecidos destas fontes

- **Datas nulas em `orders`**: pedidos cancelados têm `order_delivered_customer_date` nulo. A Silver deve tratar isso explicitamente em vez de deixar vazar para a Gold.
- **Texto livre em `order_reviews.review_comment_message`**: pode conter emojis, quebras de linha, acentuação inconsistente — exige tratamento de encoding.
- **CEPs multi-registro em `geolocation`**: o mesmo prefixo de CEP tem múltiplas coordenadas (uma por esquina). A Silver deve consolidar por centroide.
- **Volume de `geolocation`**: 1 milhão de linhas é a maior tabela — não carregar em pandas sem cuidado; ideal processar com DuckDB direto do Parquet.
- **Delay entre eventos**: `order_purchase_timestamp` antecede `order_approved_at` por minutos, que antecede `order_delivered_carrier_date` por dias. Qualquer análise de SLA precisa normalizar esses fusos e ordens.

---

## Dados de streaming

**Origem:** gerador sintético em Python (`simulator.py`) que emite eventos de navegação do usuário no site do marketplace. Em um cenário de produção esses eventos viriam de um rastreador no front-end (tipo Snowplow, Segment ou um coletor próprio) publicando em Kafka/Kinesis. **Como o projeto roda em máquina local, usamos Redis Streams** (leve, cabe em container Docker com <50 MB de RAM) como broker de eventos. Alternativamente, o simulador pode escrever direto em JSONL append-only no disco — modo "sem broker" para ambientes ainda mais restritos.

**Natureza:** append-only, semiestruturado, particionado no tempo, sem chave estrangeira forte (usa `session_id` e `user_id` como correlação).

**Frequência:** contínua. Simulador gera ~50.000 eventos/dia distribuídos conforme curva realista (picos à noite e fins de semana).

**Latência esperada do pipeline:** janelas de 5 minutos entre produção do evento e disponibilidade na camada Silver (latência "near real-time" suficiente para o propósito didático).

### Tipos de evento

| Evento | Disparo | Campos principais |
|--------|---------|-------------------|
| `page_view` | Usuário abre uma página | `session_id`, `user_id`, `url`, `product_id?`, `referrer`, `timestamp`, `user_agent` |
| `search` | Usuário executa uma busca | `session_id`, `query`, `results_count`, `timestamp` |
| `add_to_cart` | Produto adicionado ao carrinho | `session_id`, `user_id`, `product_id`, `quantity`, `unit_price`, `timestamp` |
| `remove_from_cart` | Produto removido do carrinho | `session_id`, `user_id`, `product_id`, `timestamp` |
| `checkout_start` | Usuário inicia checkout | `session_id`, `user_id`, `cart_value`, `items_count`, `timestamp` |
| `checkout_abandon` | Usuário sai sem concluir | `session_id`, `user_id`, `last_step`, `timestamp` |
| `purchase` | Compra concluída | `session_id`, `user_id`, `order_id`, `total`, `timestamp` |

**Formato:** JSON Lines (JSONL) — um evento por linha, cada linha é um objeto JSON válido. Codificação UTF-8. Esse formato é nativo ao Kafka/Redis Streams e trivial de processar com pandas/polars/DuckDB.

**Volume estimado (diário):** ~50.000 eventos/dia × ~400 bytes/evento ≈ **20 MB/dia** em JSONL bruto, ≈ 5 MB em Parquet comprimido.

**Armazenamento após ingestão:** Parquet particionado por data e hora em `./data/bronze/events/dt=YYYY-MM-DD/hr=HH/`, permitindo consulta eficiente por janela temporal em qualquer ferramenta SQL (DuckDB, Spark, dbt).

### Por que chamamos de streaming e não de micro-batch?

Do ponto de vista conceitual, qualquer processamento em janela pequena é um "micro-batch" — inclusive a implementação do Spark Streaming. O que caracteriza o caminho de streaming no OlistFlow é:

1. A **fonte é contínua** e não-limitada (não existe "fim do dia" para eventos).
2. O **consumidor espera janelas curtas** (minutos, não horas).
3. O **esquema é evolutivo e tolerante** (novos campos em eventos não quebram o pipeline).
4. A **ordem importa** para algumas análises (ex: funil de abandono depende da sequência de eventos).

Essas quatro características são suficientes para justificar o tratamento separado em relação ao batch operacional.

---

## Tabela consolidada (visão rápida)

| Fonte | Tipo | Formato origem | Formato após ingestão | Volume | Periodicidade | Latência |
|-------|------|----------------|----------------------|--------|---------------|----------|
| Olist ERP (Postgres) | **Batch** | Tabelas relacionais | Parquet | ~1,5M linhas, 45 MB | Diária (D-1) | até 6h |
| Geolocation | **Batch** | CSV | Parquet | 1M linhas, 30 MB | Carga única + mensal | semanal |
| Clickstream | **Streaming** | JSONL via Redis Streams | Parquet particionado | ~50k/dia, 5 MB/dia | Contínua | minutos (≤5 min) |

## Considerações de qualidade e privacidade

- **PII no dataset Olist:** nomes, CPFs e dados pessoais **já foram anonimizados** pela Olist antes da publicação pública (identificadores hasheados, geolocalização limitada a prefixo de CEP). Não há ação adicional necessária, mas o pipeline assume postura defensiva: logs de transformação nunca imprimem valores de colunas `customer_*` ou `seller_*`.
- **Eventos sintéticos** não contêm dado pessoal real — são gerados localmente.
- **Retenção:** Bronze guarda dados brutos indefinidamente (reprodutibilidade). Silver e Gold são reconstruíveis a partir de Bronze, então podem ser recalculados sem perda.
