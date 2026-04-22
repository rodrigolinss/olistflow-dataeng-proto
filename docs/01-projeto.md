# 4.1 — Descrição do Projeto

## Nome do projeto

**OlistFlow — Plataforma Analítica para E-commerce Marketplace**

O nome combina a inspiração no dataset público *Olist* (fonte de dados operacional de referência para a Parte 2) com a ideia de *Flow*, que remete ao fluxo contínuo de dados ao longo das camadas de ingestão, refinamento e consumo do pipeline.

## Contexto de negócio

O e-commerce brasileiro movimentou mais de R$ 185 bilhões em 2024 (fonte: ABComm) e segue crescendo a dois dígitos ao ano. Neste cenário, **marketplaces multivendedor** — onde lojistas independentes vendem por meio de uma plataforma comum — representam a fatia majoritária do mercado e concentram a maior complexidade de dados: cada pedido envolve múltiplos atores (consumidor, vendedor, operador logístico, meio de pagamento), em múltiplos estados brasileiros, com ciclos de vida longos (da compra à entrega e avaliação).

O **OlistFlow** simula o núcleo analítico de um marketplace desse tipo, a partir de dados reais e publicamente disponíveis da Olist (uma das maiores plataformas do país), enriquecidos com eventos sintéticos de navegação do usuário (clickstream) para compor um cenário híbrido realista de dados operacionais e dados de comportamento.

## Problema que o projeto pretende resolver

Em marketplaces, os dados nascem dispersos em sistemas desacoplados: o ERP registra pedidos, o gateway de pagamentos registra transações, o CRM guarda avaliações, o sistema de logística controla entregas, e o front-end produz um fluxo massivo de eventos (visitas, buscas, cliques, abandonos de carrinho) que nunca chega à base transacional. Sem um pipeline que **integre, padronize e sirva** esses dados, a empresa enfrenta três dores recorrentes:

1. **Visão de negócio fragmentada.** Analistas puxam relatórios conflitantes porque cada equipe consulta uma fonte diferente — não há "uma única versão da verdade".
2. **Decisões lentas.** Sem camada analítica pronta, responder "qual categoria cresceu na semana?" exige dias de trabalho manual em Excel.
3. **Perda de sinais de comportamento.** Eventos de navegação que antecedem a compra (ou seu abandono) raramente são cruzados com dados pós-compra (entrega, avaliação), impedindo análises de jornada completa.

O OlistFlow endereça essas dores projetando um **pipeline de engenharia de dados end-to-end** que unifica dados operacionais (batch) e comportamentais (streaming), os organiza em camadas de qualidade crescente e os expõe a times de negócio em dashboards e APIs.

## Objetivos principais

1. **Integrar** fontes heterogêneas (banco transacional + eventos de navegação) em um repositório analítico único.
2. **Padronizar e modelar** os dados em uma estrutura dimensional que responda a perguntas recorrentes de negócio com baixa latência de consulta.
3. **Garantir qualidade** por meio de testes automatizados em cada camada, tornando regressões silenciosas visíveis antes de chegarem ao consumo.
4. **Servir** os dados a três tipos de consumidor: analistas (Metabase), aplicações (API) e, futuramente, times de Data Science (camada Gold consumível em notebooks).
5. **Documentar** todo o ciclo de vida — fontes, transformações, métricas — tornando o conhecimento reutilizável.

## Stakeholders / usuários finais dos dados

| Stakeholder | Papel | Uso dos dados |
|-------------|-------|---------------|
| **Liderança executiva** (C-level) | Patrocinadores do produto | Dashboards de KPIs: GMV, ticket médio, NPS, share por região |
| **Analistas de negócio** | Usuários diários | Exploração ad-hoc, relatórios recorrentes por categoria, vendedor, região |
| **Time de Produto** | Geram hipóteses | Funil de conversão, análise de abandono de carrinho, AB testing |
| **Time de Logística** | Operação | SLA de entrega, gargalos por rota, reincidência de atrasos |
| **Time de Marketing** | Decisões de campanha | Segmentação de clientes, performance de canal, CAC/LTV (futuro) |
| **Time de Data Science** | Modelos preditivos | Features já limpas na camada Gold para previsão de demanda e recomendação |
| **Engenheiros de Dados** | Mantenedores | Ownership do pipeline, monitoramento, DataOps |

## Escopo

**Dentro do escopo (Parte 1 + Parte 2):**
- Ingestão de um snapshot do dataset Olist simulando carga batch diária de um ERP.
- Simulação de eventos de clickstream via script Python (streaming leve).
- Camadas Bronze → Silver → Gold (Arquitetura Medalhão).
- Modelagem dimensional de Vendas e Logística na camada Gold.
- Testes de qualidade de dados em Silver e Gold.
- Dashboard exemplo em Metabase.
- Orquestração em Airflow local.

**Fora do escopo (deliberadamente):**
- Infraestrutura de nuvem gerenciada (Parte 2 rodará em máquina local).
- Streaming com throughput real (volumes industriais). O caminho de streaming aqui é didático.
- Features de Machine Learning — ficam como trabalho futuro sobre a camada Gold.
- Autenticação e autorização de usuários finais no Metabase além do padrão.
