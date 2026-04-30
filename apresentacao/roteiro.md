# Roteiro da Apresentação — OlistFlow

**Duração-alvo:** 15 minutos de exposição + 5 minutos de perguntas = 20 min total (dentro da janela 10–20 min do enunciado).
**Formato:** dupla, alternando fala entre as seções. Ambos os integrantes devem apresentar alguma parte (requisito explícito do enunciado).

**Integrantes:**
- **Rodrigo César Lins**
- **Vitor Nascimento Franco**

---

## Divisão proposta (ajustem conforme força de cada um)

| Seção | Duração | Quem fala | Foco |
|-------|---------|-----------|------|
| 1. Abertura + Contexto de negócio | 1,5 min | **Rodrigo** | Engajar a sala com o problema real |
| 2. Dados e classificação (batch vs streaming) | 2,5 min | **Rodrigo** | Fontes e por que dois caminhos |
| 3. Domínios e serviços | 2,5 min | **Vitor** | DDD aplicado; diagrama de domínios |
| 4. Arquitetura e fluxo | 3,5 min | **Vitor** | Diagrama principal; por que Medalhão |
| 5. Tecnologias e justificativas | 3,5 min | **Rodrigo** | Stack com justificativa por etapa |
| 6. Tradeoffs + Próximos passos | 1,5 min | **Vitor** | Honestidade sobre limites + roadmap |
| 7. Encerramento | 0,5 min | **Rodrigo** | Convite a perguntas |
| 8. Perguntas | 5 min | **Ambos** | Dividir por afinidade temática |

Alternativa: se preferirem dividir por "blocos" em vez de alternar, Rodrigo pega seções 1–3 (contexto, dados, domínios) e Vitor pega 4–7 (arquitetura, tecnologias, tradeoffs). Escolham o que sair mais fluido no ensaio.

---

## Roteiro detalhado — fala a fala

### Slide 1 — Abertura (30 s)

> "Oi pessoal. A gente vai apresentar o OlistFlow, nosso protótipo de ciclo de vida de engenharia de dados para um e-commerce marketplace. Antes de mergulhar na arquitetura, deixa a gente te convencer de que o problema vale a pena."

Mostrar logo/nome, cenário, dupla.

### Slide 2 — Por que um marketplace? (1 min)

> "E-commerce brasileiro movimenta mais de 185 bilhões por ano. O formato marketplace — Mercado Livre, Shopee, Olist — concentra a parte mais complexa: cada pedido envolve consumidor, vendedor, meio de pagamento, operador logístico, review pós-entrega, em todo o Brasil. Os dados nascem dispersos em sistemas que raramente se falam."

Ponto-chave: problema é **integração + visão unificada**, não só "construir um banco".

### Slide 3 — Os dois tipos de dados (2,5 min)

Mostrar tabela de 02-dados.md (batch vs streaming, lado a lado).

> "A gente tem dois tipos de dados muito diferentes. Dados operacionais — pedidos, produtos, pagamentos, avaliações — são batch, vêm do ERP, são estruturados, processados uma vez por dia. Do outro lado temos dados de comportamento — eventos de navegação do usuário: visitas, buscas, carrinho, abandono — que são contínuos, semiestruturados, e a gente quer consumir em janelas de minutos."

Frase-âncora: **"tratamos os dois porque eles contam histórias diferentes da mesma jornada."**

### Slide 4 — Domínios de negócio (2,5 min)

Mostrar diagrama de domínios de 03-dominios-servicos.md.

> "A gente organizou o pipeline em 6 domínios que espelham como um marketplace real se estrutura por times: Vendas, Catálogo, Clientes, Logística, Marketing e Sellers. Cada um tem seus próprios dados, seus próprios fatos e dimensões — mas compartilham serviços centrais como o data lake, o orquestrador, os testes de qualidade."

Destacar: **conformed dimensions** (uma `dim_clientes` consumida por 3 domínios) como exemplo de reuso sem duplicação.

### Slide 5 — Arquitetura: o coração da apresentação (3,5 min)

Mostrar o diagrama grande de 04-arquitetura.md (Mermaid completo).

> "A gente escolheu Arquitetura Medalhão — Bronze, Silver, Gold — em cima de um Lakehouse conceitual, com um caminho Lambda light. Em uma frase: **Bronze captura o que chegou, Silver limpa e uniformiza, Gold modela para o negócio consumir**. Cada camada tem um contrato claro."

Justificar a escolha em três bullets rápidos:
1. Cabe no hardware (tudo roda em um notebook);
2. Preserva o raw (reversibilidade total);
3. É migrável (mesma topologia roda em S3/BigQuery sem redesenho).

> "A gente considerou alternativas — Data Warehouse puro, Data Lake puro, Lambda industrial, Kappa, Data Mesh — e desenhou uma tabela comparativa no relatório explicando por que cada uma ficou de fora. O resumo é: Medalhão é o equilíbrio entre o que a literatura recomenda hoje e o que a gente consegue executar na sala de aula."

### Slide 6 — Stack (3,5 min)

Mostrar a tabela consolidada de 05-tecnologias.md.

> "Cada tecnologia foi escolhida com alternativa considerada e justificativa por que essa ganhou. Vou destacar três decisões não-triviais:"

- **DuckDB** no lugar de Spark: "pra 45 MB de dados, Spark seria overkill de ordem de grandeza. DuckDB roda SQL ANSI in-process, direto em Parquet, e conversa com dbt."
- **Redis Streams** no lugar de Kafka: "mesma semântica de log imutável, em um container de 50 MB. Se nem Redis rodar, a gente cai num fallback ainda mais simples com JSONL em disco."
- **dbt** como espinha dorsal da transformação: "versiona SQL, gera linhagem automática, testes viram parte do pipeline. É o que a indústria usa hoje."

Ponto-final: **"cada escolha tem um plano B documentado no relatório."**

### Slide 7 — Tradeoffs + Próximos passos (1,5 min)

Mostrar a tabela de tradeoffs de 04-arquitetura.md.

> "A gente é honesto sobre os limites: não tem alta disponibilidade real, o streaming é didático, é single-node. Cedemos nessas dimensões de propósito. Em contrapartida, nosso Bronze imutável + dbt versionado dá reversibilidade quase total — qualquer erro pode ser reprocessado com um comando."

Transição para próximos passos:

> "Pra Parte 2, a gente dividiu a implementação em 5 ondas semanais — fundação, Bronze, Silver, Gold, consumo. O critério de pronto é: qualquer pessoa dessa sala consegue subir o pipeline inteiro em menos de 5 minutos no próprio notebook."

### Slide 8 — Encerramento (30 s)

> "Resumindo: OlistFlow é um pipeline realista pra um marketplace brasileiro, com batch e streaming, arquitetura Medalhão, stack que cabe num notebook. Todo o material está no repositório público do GitHub — link no relatório. Bora pras perguntas."

---

## Estratégia para perguntas (5 min)

### Perguntas prováveis e respostas curtas

**"Por que não usaram Kafka?"**
> "Kafka exigiria 2+ GB de RAM só em overhead. Redis Streams tem a mesma abstração conceitual (log imutável, consumer groups, replay) em 50 MB. Em produção seria Kafka; em protótipo didático, preservamos a semântica sem a complexidade."

**"Por que Medalhão e não Data Mesh?"**
> "Data Mesh pressupõe maturidade organizacional — múltiplos times com ownership de data products. A gente é uma dupla em um protótipo. Mas a separação por domínios que a gente desenhou na seção 3 já prepara uma evolução pra Data Mesh sem redesenho."

**"DuckDB escala?"**
> "DuckDB é single-node, escala vertical. Pro nosso volume (dezenas de MB), é mais rápido que qualquer engine distribuída. Em produção com TBs, a gente trocaria por BigQuery ou Athena — mas o dbt que a gente escreveu continuaria o mesmo, só mudaria o perfil de conexão."

**"Como vocês garantem qualidade dos dados?"**
> "Duas camadas: dbt tests rodam junto com cada modelo (`not_null`, `unique`, `relationships`, `accepted_values`) e bloqueiam o pipeline se falharem em nível crítico. Great Expectations complementa com testes de distribuição e integridade cross-table."

**"O que muda se vocês forem pra cloud?"**
> "Três coisas: Parquet local vira S3, DuckDB vira Athena ou BigQuery, Airflow local vira MWAA ou Cloud Composer. Os modelos dbt, as DAGs, os testes — tudo continua igual. É essa a virtude da arquitetura Medalhão bem construída: ela é portátil."

**"Qual é o diferencial do projeto de vocês?"**
> "Três coisas: (1) planejamento explícito de tradeoffs — a gente documenta o que cedemos e por quê; (2) stack inteiramente open-source e reprodutível em notebook modesto — qualquer colega consegue rodar; (3) arquitetura pensada pra evoluir sem redesenho, não pra impressionar agora."

### Dica geral

Se receberem uma pergunta técnica muito específica que não dominam: **remeter ao relatório**. "Isso a gente cobre em detalhe na seção 4.5 do relatório — se quiser posso mostrar agora ou depois da apresentação."

---

## Checklist de ensaio (fazer 2 vezes antes do dia 30/04)

- [ ] Cronometrar cada seção — estouros > 30 s exigem corte
- [ ] Confirmar que ambos apresentam alguma parte (requisito de avaliação)
- [ ] Testar projeção dos diagramas Mermaid (renderizam bem em tela grande?)
- [ ] Preparar backup em PDF caso a internet falhe
- [ ] Revisar 3 perguntas mais prováveis acima — responder sem olhar
- [ ] Combinar quem responde primeiro cada tipo de pergunta

---

## Sugestão de ferramenta para slides

**Opção 1 (recomendada): [Marp](https://marp.app/)** — gera slides a partir de Markdown. Fluxo:
1. Instalar extensão Marp for VS Code
2. Criar `apresentacao/slides.md`
3. Usar headings e `---` como separadores de slide
4. Exportar para PDF na véspera

**Opção 2: Google Slides** — clássico e confortável, dá pra colar os diagramas Mermaid como imagens (exportar cada um via https://mermaid.live).

**Opção 3: Canva** — visualmente polido com baixo esforço.

Regra para todas: **no máximo 8–10 slides**. Os 15 minutos não comportam mais.
