---
name: code-review
description: Revisa as mudanças de código deste repositório (HTML de arquivo único, sem dependências) comparando o branch atual com a main. Use SEMPRE que o usuário disser que vai abrir um Pull Request, pedir para revisar seu próprio código antes de commitar/dar push, ou pedir para revisar o PR de um colega — mesmo que ele não use a palavra "revisão" explicitamente (ex: "dá uma olhada no que eu mudei", "confere esse PR", "isso tá pronto pra subir?"). Não use para revisar arquivos que não fazem parte de um diff de branch, nem para tarefas de estilo/formatação.
---

# Code Review

Revisão de diffs de branch, focada em **achados acionáveis e verificados** — não em opinião de estilo.

## Regra de ouro

Todo achado precisa responder três perguntas. Se não conseguir responder as três, o achado não entra no relatório:

1. **Onde** — arquivo e linha exatos.
2. **Como reproduzir** — passo a passo (ação do usuário, input, comando) que expõe o problema.
3. **Quebra ou é preferência?** — nunca misturar as duas coisas no mesmo item.

## O que esta skill NUNCA faz

- Reclamar de formatação, indentação, espaçamento, ponto e vírgula, ordem de imports, etc.
- Reportar um problema sem apontar arquivo + linha.
- Sugerir refatoração sem dizer explicitamente **qual problema concreto** ela resolve (bug, comportamento errado, ou risco de quebra — não "fica mais limpo").
- Revisar arquivos que não mudaram no diff.
- Afirmar que "testou" algo que na verdade só leu.

Se um achado não se encaixa nas categorias "quebra o programa" ou "não é bloqueante mas vale mencionar com justificativa concreta" (ex: um bug latente que não quebra ainda, mas vai quebrar em um caso de uso real), **descarte o achado**.

## Passo a passo

### 1. Descobrir o escopo do diff

```bash
# descobrir o branch default (main ou master)
git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null || git rev-parse --abbrev-ref HEAD

# branch atual
git rev-parse --abbrev-ref HEAD

# arquivos alterados em relação à main
git diff main...HEAD --name-only
# se "main" não existir, tentar "master"

# diff completo com números de linha
git diff main...HEAD
```

Revise **apenas** os arquivos e trechos que aparecem nesse diff. Não comente código que não mudou, mesmo que perceba algo estranho nele — não é o escopo da revisão (pode mencionar de forma separada e claramente rotulada como "fora do escopo", só se for algo grave, mas nunca misturado com os achados do diff).

Se estiver revisando o PR de um colega, o processo é o mesmo: substitua `HEAD` pelo branch do colega (`git diff main...nome-do-branch`).

### 2. Analisar cada trecho alterado

Como é um projeto **HTML de arquivo único, sem dependências**, preste atenção especial a:

- HTML/CSS/JS misturados no mesmo arquivo — variáveis JS vazando pro escopo global, seletores CSS colidindo, IDs duplicados.
- Erros de sintaxe em `<script>` (chave/parêntese não fechado, `;` faltando em ponto crítico, etc.).
- Referências a elementos do DOM que podem não existir no momento da execução (`document.getElementById` retornando `null`, script rodando antes do elemento existir).
- Lógica quebrada: condições invertidas, off-by-one, comparação com tipo errado (`==` vs `===` quando isso importa), variável reatribuída sem querer.
- Qualquer coisa que só vai quebrar em um cenário específico de uso — nesse caso, o "como reproduzir" é obrigatório e precisa ser um passo real (ex: "abrir o arquivo no navegador, clicar em X sem preencher Y, ver o erro Z no console").

### 3. Verificar de fato o que der para verificar

Não é permitido dizer "isso deveria funcionar" sem checar. Faça, nessa ordem:

**a) Validar sintaxe do arquivo alterado:**
```bash
# extrai o conteúdo de cada <script> do HTML e valida a sintaxe JS
node --check <(sed -n '/<script/,/<\/script>/p' arquivo.html | sed '1d;$d')
```
(Adapte o comando ao arquivo real; se houver mais de uma tag `<script>`, valide cada bloco separadamente.)

**b) Rodar build/compilação, se existir uma no repositório:**
```bash
# procurar por script de build já existente no repo (não instalar nada novo)
cat package.json 2>/dev/null | grep -A5 '"scripts"'
ls Makefile 2>/dev/null
```
Se existir um comando de build já configurado, rode-o e reporte o resultado. Se **não existir nenhum build configurado** (comum nesse tipo de projeto), diga isso claramente no relatório final — não invente um passo de build que não existe.

**c) Rodar testes automatizados, se existirem:**
```bash
ls test* tests* spec* 2>/dev/null
cat package.json 2>/dev/null | grep -i test
```
Se houver testes cobrindo a área alterada, rode-os. Se não houver teste automatizado para aquele trecho, diga isso — não é para simular um teste manualmente e reportar como se fosse execução automatizada.

**d) O que não dá para rodar, você só lê.** Isso é esperado (ex: comportamento visual, interação de usuário que exige navegador real) — só não pode ser **apresentado** como algo que foi executado.

### 4. Escrever o relatório

Use exatamente esta estrutura:

```markdown
## Revisão: <branch-atual> vs main

### Escopo revisado
- Arquivos alterados: <lista>
- Comando: `git diff main...HEAD`

### 🔴 Quebra o programa
1. **arquivo.html:42** — <descrição objetiva do problema>
   - Como reproduzir: <passo a passo>

(Se não houver nenhum: "Nenhum problema que quebra o programa foi encontrado no diff.")

### 🟡 Não bloqueante (preferência / risco menor, com justificativa)
1. **arquivo.html:18** — <observação> — <por que isso importa, concretamente>

(Se não houver nenhum, omita a seção ou diga "Nenhuma observação não bloqueante.")

### O que foi verificado
- ✅ Rodei: `node --check ...` → <resultado>
- ✅ Rodei: <testes existentes, se houver> → <resultado>
- 📖 Só li (não dá pra automatizar aqui): <o quê e por quê>
- ⚠️ Sem build/testes configurados no repositório para: <o quê>
```

Nunca omita a seção "O que foi verificado" — ela é o que diferencia essa skill de uma revisão genérica: o usuário precisa saber exatamente o que foi confirmado por execução e o que foi só leitura de código.

## Lembretes finais

- Diff vazio ou sem mudanças reais (ex: só espaço em branco) → diga isso diretamente, não force achados.
- Se `main` não existir localmente, avise e pergunte se deve usar `master` ou outro branch de referência, em vez de assumir.
- Nunca liste o mesmo achado duas vezes em categorias diferentes.
