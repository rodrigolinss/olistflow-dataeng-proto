# 4.6 — Considerações Finais

## Principais riscos e limitações previstos

### Riscos técnicos

| Risco | Impacto | Probabilidade | Mitigação planejada |
|-------|---------|---------------|---------------------|
| **Hardware da sala aquém do esperado** (< 8 GB RAM) | Airflow + Postgres + Metabase + DuckDB não sobem juntos | Média | Fallback: subir apenas Postgres + dbt/DuckDB + Metabase (sem Airflow, usando cron); streaming sem Redis (arquivos JSONL diretos) |
| **Porta já em uso** (5432 do Postgres, 3000 do Metabase, 8080 do Airflow) | Containers não sobem | Alta | Redirecionar portas no `docker-compose.yml`; documentar no README |
| **Docker não instalado ou versão antiga** | Nada sobe | Média | Instruções no README + validação prévia nas máquinas |
| **Versões incompatíveis** (dbt 1.x vs DuckDB 1.x) | Quebras sutis | Baixa | Pinar versões exatas em `requirements.txt` e `packages.yml` |
| **Dataset Olist indisponível** (Kaggle offline) | Falta dado-fonte | Baixa | Baixar uma vez e versionar em `./data/raw/` (arquivos ≤ 50 MB, toleráveis em Git LFS se preciso) |
| **Simulador gera dados inconsistentes** com o dataset Olist | Joins vazios no funil | Média | Simulador referencia `order_id`s reais; validação automática no dbt |

### Limitações conceituais

1. **Ausência de volume industrial.** Não testamos o pipeline em GBs de dados. As decisões de desenho (partição por data, dbt incremental) preparam para escalar, mas não comprovam escala.
2. **Streaming didático.** Redis Streams ilustra a semântica; não oferece as garantias operacionais que Kafka/Kinesis entregariam em produção (replicação, compactação, retenção longa, consumer groups distribuídos).
3. **Ausência de ML.** O Gold está desenhado para ser consumível por notebooks de Data Science, mas nenhum modelo é treinado nesta Parte nem na Parte 2 planejada.
4. **Sem governança corporativa formal.** O catálogo é local (dbt docs); não há integração com Data Catalog corporativo, nem política de classificação de dados sensíveis além do que a Olist já anonimiza.
5. **Sem multi-tenant.** O Metabase roda como usuário único — um e-commerce real teria múltiplos perfis (executivo, analista, operacional) com RBAC.
6. **Single-node.** Todo o pipeline roda em uma máquina. Falha dessa máquina = todo o serviço fora. Aceitável para protótipo acadêmico.

### Limitações de escopo

- Não cobrimos **aquisição de dados externos** além dos disponíveis (ex: integração com APIs de clima, câmbio, feriados).
- Não implementamos **reverse ETL** (devolver dados enriquecidos para sistemas operacionais).
- Não há **experimentação A/B** integrada ao pipeline.

## Próximos passos — Parte 2 (implementação)

A implementação seguirá cinco ondas, cada uma produzindo um entregável testável:

### Onda 1 — Fundação (semana 1)
- Subir `docker-compose` com Postgres + Airflow + Redis + Metabase.
- Carregar dataset Olist no Postgres via script Python.
- Configurar projeto dbt com conexão DuckDB.
- Commit inicial da estrutura do repositório de código.

### Onda 2 — Camada Bronze (semana 2)
- Implementar DAG `ingest_olist` (batch diário).
- Implementar simulador de clickstream (`simulator.py`).
- Implementar DAG `ingest_events` (micro-batch de 5 min).
- Validar que Bronze está sendo populada em Parquet particionado.

### Onda 3 — Camada Silver (semana 3)
- Modelos dbt `stg_*` para todas as tabelas Olist.
- Modelo dbt `stg_events` com schema validado.
- Modelos intermediários (`int_*`) de lógica compartilhada.
- dbt tests básicos (`not_null`, `unique`, `relationships`).

### Onda 4 — Camada Gold e qualidade (semana 4)
- Modelagem dimensional: `dim_*` e `fct_*`.
- Dimensão de tempo pré-materializada.
- Snapshots dbt para SCD Tipo 2.
- Suite Great Expectations para testes analíticos mais ricos.

### Onda 5 — Consumo e finalização (semana 5)
- Dashboards Metabase (Executivo, Vendas, Logística, Funil, Sellers).
- (Opcional) Endpoints FastAPI de métricas.
- dbt docs publicado como GitHub Pages.
- CI via GitHub Actions.
- Slides finais e ensaio de apresentação.

### Critérios de pronto da Parte 2

- [ ] `make up` sobe todo o stack em < 5 min em máquina limpa.
- [ ] `dbt build` roda do zero e passa em 100% dos testes.
- [ ] Cinco dashboards do Metabase carregam em < 3 s cada.
- [ ] README documenta como executar, testar, derrubar.
- [ ] Apresentação prática: qualquer pessoa da sala consegue reproduzir em seu notebook em < 30 min.

---

## Referências

### Livros-base da disciplina

- **Reis, Joe; Housley, Matt.** *Fundamentals of Data Engineering.* O'Reilly, 2022. — Referência principal para conceitos de ciclo de vida, correntes e princípios de arquitetura.
- **Kimball, Ralph; Ross, Margy.** *The Data Warehouse Toolkit,* 3ª ed. Wiley, 2013. — Modelagem dimensional aplicada à camada Gold.
- **Kleppmann, Martin.** *Designing Data-Intensive Applications.* O'Reilly, 2017. — Fundamentos de sistemas distribuídos e tradeoffs de arquitetura.

### Artigos e documentação técnica

- **Databricks.** *What is the Medallion Architecture?* Disponível em: https://www.databricks.com/glossary/medallion-architecture
- **Inmon, Bill.** *Building the Data Lakehouse.* Technics Publications, 2021.
- **dbt Labs.** *Analytics Engineering Guide.* Disponível em: https://www.getdbt.com/analytics-engineering/
- **Apache Airflow.** *Best Practices.* Disponível em: https://airflow.apache.org/docs/apache-airflow/stable/best-practices.html
- **Zaharia, Matei et al.** *Delta Lake: High-Performance ACID Table Storage over Cloud Object Stores.* VLDB, 2020.

### Dataset e ferramentas

- **Olist.** *Brazilian E-Commerce Public Dataset by Olist.* Kaggle. Disponível em: https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce
- **DuckDB.** Documentação oficial. https://duckdb.org/docs/
- **Great Expectations.** Documentação oficial. https://docs.greatexpectations.io/
- **Metabase.** Documentação oficial. https://www.metabase.com/docs/

### Referências contextuais sobre o mercado brasileiro

- **ABComm — Associação Brasileira de Comércio Eletrônico.** *Relatório anual de e-commerce brasileiro.*
- **Ebit/Nielsen.** *Webshoppers — Relatório sobre comércio eletrônico no Brasil.*

---

## Palavras finais

Este relatório é a materialização de uma decisão central: **planejar com ambição conceitual e humildade operacional**. Escolhemos uma arquitetura que conversa com a literatura moderna de engenharia de dados (Medalhão, Lakehouse, Lambda) mas nos mantivemos disciplinados quanto à viabilidade de execução — cada tecnologia escolhida cabe em um notebook modesto, tem comunidade ativa, e produz artefatos visuais fortes para defender a proposta em sala.

Se bem-sucedida, a Parte 2 entregará um pipeline funcional, reprodutível e defensável — que poderá servir como base para evoluções reais (migração para cloud, adição de ML, integração com sistemas de negócio) sem redesenho da topologia.
