# O que falar — OlistFlow

Guia rápido pra saber o que dizer em cada slide. Apresentação de ~15 minutos, sem perguntas no final.

**Quem fala:** Rodrigo César Lins e Vitor Nascimento Franco. Os dois apresentam — alternem por slide ou peguem blocos inteiros, o que ficar mais natural.

---

## Slide 1 — Capa

> "Oi pessoal. A gente é Rodrigo e Vitor, e vai apresentar o OlistFlow — nosso protótipo de pipeline de engenharia de dados pra um marketplace brasileiro de e-commerce."

Curto. Não explica nada ainda. Só diz quem fala e o que vem.

---

## Slide 2 — Por que um marketplace?

> "O e-commerce no Brasil já passou dos 185 bilhões por ano, e marketplace é o formato mais complexo: cada pedido envolve cliente, vendedor, meio de pagamento, transportadora, review pós-entrega — em todos os estados. Os dados nascem espalhados em sistemas que mal conversam, e isso gera três dores: visão fragmentada, decisão lenta, e sinais de comportamento que somem."

Frase-âncora pra fixar: **"o problema é integração, não só armazenar dado"**.

---

## Slide 3 — Os dois tipos de dados

> "A gente trabalha com dois tipos de dados muito diferentes. De um lado, dados operacionais — pedidos, pagamentos, avaliações — vindos do ERP, em batch diário. Do outro, dados de comportamento — eventos de navegação, busca, carrinho, abandono — em streaming, com latência de minutos. A tabela mostra origem, formato, volume e cadência de cada um."

Frase-âncora: **"tratamos os dois porque eles contam histórias diferentes da mesma jornada"**.

---

## Slide 4 — Domínios: quem quer o quê

A ideia aqui é mostrar que domínio não é abstração técnica — é **quem usa o dado e o que tira dele**.

> "A gente identificou seis domínios, cada um com um interessado claro. Vendas serve liderança e analistas — eles querem GMV, ticket médio, conversão. Catálogo é referência usada por todos os outros domínios — produto e categoria canônicos. Clientes serve Marketing e Vendas com segmentação RFM. Logística é da operação — SLA de entrega, gargalos por rota. Marketing serve Produto e Marketing — funil de conversão, abandono de carrinho. E Sellers é gestão de marketplace — scorecard, NPS por vendedor."

Depois aponta pro diagrama:

> "O diagrama mostra como esses domínios se conectam: cada um escreve no data lake compartilhado, e dimensões como `dim_clientes` são consumidas por mais de um domínio — reuso sem duplicação."

Frase-âncora: **"domínio é definido pelo consumidor, não pela tabela"**.

---

## Slide 5 — Arquitetura (o coração)

> "A gente escolheu Arquitetura Medalhão — Bronze, Silver, Gold — em cima de um Lakehouse local, com um caminho Lambda light que separa batch de streaming na ingestão e une os dois na Gold. Em uma frase: Bronze captura tudo cru, Silver limpa e padroniza, Gold modela pro negócio consumir."

Três motivos rápidos pra justificar a escolha:
1. Cabe num notebook (sem cluster);
2. Bronze imutável dá reversibilidade total;
3. A topologia é portável — mesma estrutura roda em S3/BigQuery sem redesenhar.

---

## Slide 5b — Como o Airflow coordena

> "Aqui é como o Airflow coordena: um trem diário às 2 da manhã que faz a ingestão batch, roda a Silver, roda a Gold, passa nos testes, e só depois atualiza o Metabase. E em paralelo, um DAG de streaming a cada 5 minutos que processa eventos. Os testes do dbt + Great Expectations são gates: dado ruim nunca chega no dashboard."

---

## Slide 6 — Por que Medalhão e não outras?

> "A gente considerou todas as alternativas — Data Warehouse puro, Data Lake puro, Lambda industrial, Kappa, Data Mesh — e descartou cada uma com motivo. Data Warehouse perde o raw, Data Lake puro vira pântano, Lambda industrial precisa de hardware que a gente não tem, Kappa trata tudo como streaming sem necessidade, Data Mesh precisa de múltiplos times. Medalhão é o equilíbrio entre o que a literatura recomenda hoje e o que a gente consegue executar na sala."

---

## Slide 7 — Stack tecnológico

> "Cada tecnologia foi escolhida com alternativa considerada e justificativa. Vou destacar três decisões que não são óbvias: DuckDB no lugar de Spark — porque pra 45 MB de dados Spark seria overkill; Redis Streams no lugar de Kafka — mesma semântica de log imutável em 50 MB de RAM; e dbt como espinha — versiona SQL, gera linhagem, tem testes embutidos. Total da stack: 2,6 GB, cabe em notebook de 8 GB."

---

## Slide 7b — Como roda na máquina (Docker Compose)

> "Cinco containers na mesma rede Docker: Postgres simulando o ERP, Redis pros eventos, Airflow orquestrando, Metabase pros dashboards, e o simulador de eventos. O DuckDB não é container — é arquivo no disco que o scheduler escreve e o Metabase lê em read-only. Tudo isso sobe com um `make up`."

---

## Slide 8 — Três decisões não-triviais

Reforço dos três pontos do slide de stack, agora com mais detalhe — DuckDB sobre Spark, Redis sobre Kafka, dbt como espinha. Frase-âncora: **"cada escolha tem plano B documentado no relatório"**.

---

## Slide 9 — Tradeoffs honestos

> "A gente é honesto sobre os limites: não tem alta disponibilidade real, o streaming é didático e não industrial, é single-node. Cedemos nessas dimensões de propósito. Em compensação, Bronze imutável + dbt versionado dá reversibilidade quase total — qualquer erro pode ser reprocessado com um comando, e a topologia inteira é portável pra cloud sem redesenho."

---

## Slide 10 — Parte 2 (5 ondas de implementação)

> "Pra Parte 2, a gente dividiu a implementação em cinco ondas semanais: Fundação, Bronze, Silver, Gold + QA, e Consumo. O critério de pronto é simples: qualquer pessoa dessa sala consegue subir o pipeline inteiro em menos de 5 minutos no próprio notebook."

---

## Slide 11 — Encerramento

> "Resumindo: OlistFlow é um pipeline realista pra um marketplace brasileiro, com batch e streaming, arquitetura Medalhão, stack que cabe num notebook. Todo o material está no repositório do GitHub. Obrigado!"

Agradece e sai. Não abre para perguntas.

---

## Lembretes pro dia

- Começar com calma — a sala demora uns segundos pra silenciar.
- Apontar pro projetor quando mostrar diagrama (não fica olhando só pro notebook).
- Cronometrar nos ensaios — se passar de 16 min, cortar uma frase em cada slide.
- Backup em PDF no pendrive caso o navegador trave.
- Tela cheia: tecla `F` no slide aberto.
