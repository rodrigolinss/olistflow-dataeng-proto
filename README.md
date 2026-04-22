# OlistFlow — Protótipo de Ciclo de Vida de Engenharia de Dados

> **Projeto acadêmico — 1ª avaliação da disciplina de Engenharia de Dados**
> **Parte 1 de 2:** planejamento e desenho arquitetural
> **Apresentação:** 30/04/2026

---

## Sobre este repositório

Este repositório contém o **planejamento** de um protótipo de ciclo de vida completo de engenharia de dados aplicado a um cenário de **e-commerce brasileiro**. O trabalho está dividido em duas partes: nesta Parte 1 definimos **o que** e **como** será feito (arquitetura, domínios, tecnologias); na Parte 2 (próxima avaliação) faremos a implementação prática do pipeline.

O cenário escolhido simula um marketplace no estilo **Olist / Mercado Livre**, onde múltiplos vendedores comercializam produtos a consumidores finais. Usamos o [Olist Brazilian E-Commerce Public Dataset](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce) como fonte operacional de referência e complementamos com um simulador de eventos em tempo real (clickstream) para ilustrar o caminho de streaming.

## Índice

| # | Documento | Conteúdo |
|---|-----------|----------|
| 4.1 | [01-projeto.md](docs/01-projeto.md) | Nome, contexto, problema, objetivos e stakeholders |
| 4.2 | [02-dados.md](docs/02-dados.md) | Fontes, formatos, volumes, classificação batch × streaming |
| 4.3 | [03-dominios-servicos.md](docs/03-dominios-servicos.md) | Domínios de negócio, serviços e diagrama DDD |
| 4.4 | [04-arquitetura.md](docs/04-arquitetura.md) | Arquitetura Medalhão + Lambda light, fluxo ponta-a-ponta, tradeoffs |
| 4.5 | [05-tecnologias.md](docs/05-tecnologias.md) | Stack por etapa do ciclo de vida com justificativas |
| 4.6 | [06-consideracoes.md](docs/06-consideracoes.md) | Riscos, próximos passos para Parte 2, referências |
| — | [apresentacao/roteiro.md](apresentacao/roteiro.md) | Divisão de fala da dupla e tempo por tópico |
| — | [apresentacao/index.html](apresentacao/index.html) | **Slides HTML interativos** (reveal.js + Mermaid) |

## Apresentação

Os slides estão em `apresentacao/index.html` — um arquivo HTML auto-contido que roda em qualquer navegador. Três formas de visualizar:

1. **Localmente:** duplo-clique em `apresentacao/index.html` (abre no navegador padrão).
2. **GitHub Pages:** após ativar Pages no repositório, os slides ficam em `https://<usuario>.github.io/<repo>/apresentacao/`.
3. **Exportar para PDF:** com a apresentação aberta no Chrome/Edge, `Ctrl+P` (Win/Linux) ou `Cmd+P` (macOS) → "Salvar como PDF". Use orientação paisagem.

Durante a apresentação, atalhos úteis do reveal.js: `→` avança, `←` volta, `S` abre speaker notes em janela separada, `F` tela cheia, `ESC` visão geral.

## Equipe

- Integrante 1 — *preencher nome e matrícula*
- Integrante 2 — *preencher nome e matrícula*

## Como ler este relatório

Recomendamos a leitura na ordem numérica dos documentos. Cada seção foi desenhada para ser auto-contida, mas a arquitetura (4.4) e as tecnologias (4.5) fazem referência aos domínios descritos em 4.3 e à classificação dos dados em 4.2.

Todos os diagramas são renderizados diretamente pelo GitHub usando **Mermaid** — basta abrir os arquivos `.md` no navegador.

## Escopo da Parte 2 (implementação futura)

A Parte 2 consistirá em materializar o pipeline descrito aqui: ingerir o dataset Olist, popular um PostgreSQL simulando um sistema transacional, simular eventos de clickstream, implementar as transformações em dbt + DuckDB, orquestrar com Airflow local e servir dashboards em Metabase. Todo o stack foi deliberadamente escolhido para ser **executável em máquina modesta** (≈ 8 GB de RAM), com containers Docker.

## Licença

Trabalho acadêmico. Uso livre para fins educacionais.
