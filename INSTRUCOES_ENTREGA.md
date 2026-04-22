# Instruções de Entrega — Passo a Passo

Este arquivo é pra você (não faz parte do relatório — não precisa subir pro GitHub, ou pode subir se preferir documentar o processo).

## O que o enunciado pede entregar

1. **Repositório público no GitHub** com todo o conteúdo deste diretório
2. **PDF** contendo:
   - Link do repositório público
   - Print (captura de tela) do repositório público
3. **Apresentação em sala** no dia 30/04/2026 (roteiro em `apresentacao/roteiro.md`)

---

## Passo 1 — Revisar e personalizar antes de publicar

Abra cada arquivo e ajuste o que for específico do grupo:

- [ ] `README.md` — preencher nomes da dupla em "Equipe" (linha "Integrante 1 — preencher nome e matrícula")
- [ ] `docs/01-projeto.md` — opcional: ajustar se quiserem mudar o nome "OlistFlow" por outro
- [ ] Qualquer lugar que tenha `preencher` ou marcação de TODO

Se quiser, rode um `grep` pra achar tudo:

```bash
cd "/Users/rodrigolins/Desktop/Trabalho - engenharia de dados/ecommerce-dataeng-proto"
grep -rn "preencher\|TODO\|FIXME" .
```

---

## Passo 2 — Criar o repositório no GitHub

### Se ainda não tem conta GitHub

1. Entre em https://github.com/signup
2. Crie uma conta (gratuita) e confirme o e-mail

### Criar o repositório

1. Logado no GitHub, clique no `+` no topo direito → **New repository**
2. Configuração:
   - **Repository name:** `olistflow-dataeng-proto` (ou outro nome que prefiram)
   - **Description:** `Protótipo de ciclo de vida de engenharia de dados — 1ª avaliação`
   - **Public** ✅ (o enunciado exige repositório público)
   - **NÃO marque** "Add a README file" (já temos um)
   - **NÃO adicione** .gitignore nem licença agora
3. Clique em **Create repository**

### Subir os arquivos

No terminal:

```bash
cd "/Users/rodrigolins/Desktop/Trabalho - engenharia de dados/ecommerce-dataeng-proto"

# Inicializar o git
git init
git branch -M main

# Adicionar remote — TROQUE 'seu-usuario' pelo seu username do GitHub
git remote add origin https://github.com/seu-usuario/olistflow-dataeng-proto.git

# Adicionar tudo e commitar
git add .
git commit -m "Parte 1 - Planejamento e desenho arquitetural do OlistFlow"

# Subir
git push -u origin main
```

Na primeira vez, o GitHub vai pedir autenticação. A forma mais simples:
- Username: seu usuário do GitHub
- Password: um **Personal Access Token** (PAT)

Como gerar um PAT:
1. GitHub → avatar → Settings → Developer settings → Personal access tokens → Tokens (classic)
2. Generate new token (classic) → selecionar escopo `repo` → gerar
3. Copiar o token (só aparece uma vez)
4. Colar no terminal quando pedir senha

**Alternativa recomendada:** usar [GitHub CLI](https://cli.github.com/) (`gh auth login`) que faz login interativo sem PAT.

---

## Passo 3 — Validar que ficou público e renderizando

1. Abrir o link do repositório em uma **janela anônima** do navegador (sem estar logado)
2. Confirmar que:
   - [ ] README.md aparece na página inicial
   - [ ] Clicando em qualquer `.md` em `docs/`, o Markdown renderiza corretamente
   - [ ] Os **diagramas Mermaid** aparecem como imagens, não como código (o GitHub renderiza Mermaid nativamente, mas confirme — especialmente 03-dominios-servicos.md e 04-arquitetura.md)
3. Se algum Mermaid não renderizar, abrir o arquivo específico e verificar se a marcação ` ```mermaid ` está correta

---

## Passo 4 — Tirar o print do repositório

1. Abra o repositório público no navegador (janela anônima ajuda a confirmar o estado público)
2. Tire um print da página principal mostrando:
   - Nome do repositório
   - Badge "Public"
   - Lista de arquivos (README, docs/, apresentacao/)
   - Preview do README renderizado
3. Salve como imagem (PNG ou JPG)

**No macOS:** `Cmd + Shift + 4` + espaço → clicar na janela do navegador.

---

## Passo 5 — Montar o PDF para a entrega

O PDF precisa conter:
1. Link do repositório
2. O print da página

### Opção A — Google Docs (mais simples)

1. Abrir um Google Docs novo
2. Linha 1: "Projeto: OlistFlow — Parte 1"
3. Linha 2: "Integrantes: [Nome 1] ([matrícula]), [Nome 2] ([matrícula])"
4. Linha 3: "Repositório: https://github.com/seu-usuario/olistflow-dataeng-proto"
5. Inserir → Imagem → Upload do print
6. Arquivo → Download → PDF (.pdf)

### Opção B — Pages (macOS) ou Word

Mesma ideia, exportar PDF ao final.

### Opção C — Direto no Preview (macOS)

1. Abrir o print no Preview
2. Cmd + P → salvar como PDF, adicionando o link como anotação ou legenda

---

## Passo 6 — Submeter no sistema da disciplina

Subir o PDF no lugar indicado pela disciplina (Moodle, Google Classroom, etc.) **antes do prazo** (o enunciado diz "ver na atividade" — confirmar a data limite específica).

---

## Checklist final antes de submeter

- [ ] Nomes e matrículas preenchidos no README
- [ ] Repositório público (confirmado em janela anônima)
- [ ] Todos os 7 arquivos presentes (README + 6 docs + roteiro)
- [ ] Diagramas Mermaid renderizam no GitHub
- [ ] PDF tem link + print
- [ ] PDF submetido antes do prazo
- [ ] Apresentação ensaiada (ver checklist em `apresentacao/roteiro.md`)

---

## Sugestão extra — aumentar pontos com trabalhos bem apresentados

Coisas que **não** são obrigatórias mas impressionam:

1. **Adicionar uma licença** (MIT) ao repositório — sinaliza cuidado profissional
2. **Ativar GitHub Pages** do `docs/` — transforma o repo num mini-site navegável
3. **Adicionar um badge "Acadêmico"** no topo do README
4. **Commits limpos** em vez de um único commit gigante — mostra evolução do pensamento

Nada disso muda a nota diretamente, mas dá **presença** ao trabalho.

Boa entrega. 🎓
