# Capítulo 5 — Instalação e Primeiro Script

**Objetivos deste capítulo:**
- Instalar o Python 3 no Windows e Linux
- Configurar o VS Code para programar em Python
- Escrever o primeiro script: entrada, saída, comentários
- Traduzir Portugol para Python
- Entender e resolver erros comuns

---

## Chegou a hora do código

Até agora, tudo foi lógica pura. Você aprendeu algoritmos, fluxogramas, Portugol — sem precisar de computador. Isso foi proposital.

Agora vamos dar o salto: tudo que você aprendeu vai virar **código de verdade**.

A transição é mais simples do que parece. Veja:

| Conceito | Portugol | Python |
|:---------|:---------|:-------|
| Escrever na tela | `ESCREVER "Olá"` | `print("Olá")` |
| Ler do teclado | `LER variavel` | `variavel = input()` |
| Guardar valor | `idade = 18` | `idade = 18` |
| Decisão | `SE idade >= 18 ENTAO` | `if idade >= 18:` |
| Senão | `SENAO` | `else:` |
| Enquanto | `ENQUANTO cond FACA` | `while cond:` |
| Para | `PARA i DE 1 ATE 10 FACA` | `for i in range(1, 11):` |
| Função | `FUNCAO nome(x)` | `def nome(x):` |
| Retornar | `RETORNAR valor` | `return valor` |
| Verdadeiro/Falso | `verdadeiro / falso` | `True / False` |
| E lógico | `E` | `and` |
| Ou lógico | `OU` | `or` |
| Não lógico | `NAO` | `not` |
| Comentário | `// comentário` | `# comentário` |

Viu como é próximo? Você já sabe 80% do Python. Agora é aprender a sintaxe.

---

## Instalação

### No Windows

1. Acesse https://python.org e clique em **Downloads**
2. Baixe a versão mais recente do Python 3
3. Execute o instalador
4. **ATENÇÃO:** Marque a opção **"Add Python to PATH"** — isso é fundamental
5. Clique em "Install Now"
6. Abra o **Prompt de Comando** (cmd) e digite:

```
python --version
```

Se aparecer algo como `Python 3.12.x`, a instalação funcionou.

### No Linux (Ubuntu/Debian/Kali)

No Linux, o Python já vem instalado na maioria das distribuições. Verifique:

```bash
python3 --version
```

Se não estiver instalado:

```bash
sudo apt update
sudo apt install python3 python3-pip -y
```

### Verificando a instalação

Abra o terminal (Linux) ou prompt de comando (Windows) e digite:

```bash
python3 --version
```

Você deve ver algo como:

```
Python 3.12.3
```

> Se no Windows o comando `python` não funcionar, tente `py` ou `python3`.

---

## VS Code — seu ambiente de programação

Você pode programar Python no bloco de notas se quiser. Mas um bom editor facilita muito a vida. Recomendo o **VS Code** (Visual Studio Code).

### Instalação do VS Code

**Windows:** https://code.visualstudio.com — baixar e instalar
**Linux:** `sudo apt install code -y`

### Extensões essenciais

Abra o VS Code e instale estas extensões (ícone de quadradinhos na barra lateral):

1. **Python** (Microsoft) — suporte a Python, realce de sintaxe, depuração
2. **Python Debugger** (Microsoft) — para depurar passo a passo

### Seu primeiro arquivo

1. Crie uma pasta: `Documentos/livro-python/`
2. No VS Code: **File → Open Folder** e selecione essa pasta
3. Crie um novo arquivo: **File → New File** e salve como `hello.py`
4. Digite:

```python
print("Olá, mundo!")
```

5. Execute: clique no ícone de play (▶) no canto superior direito

Pronto. Você acabou de executar seu primeiro programa em Python.

---

## Primeiros passos em Python

### print() — escrever na tela

```python
print("Olá, mundo!")           # Texto entre aspas
print(42)                       # Número direto
print("A resposta é", 42)      # Múltiplos valores
```

### input() — ler do teclado

```python
nome = input("Digite seu nome: ")
print("Olá,", nome)
```

O `input()` sempre retorna **texto** (string). Se você digitar um número, ele ainda será texto — precisa converter.

### Convertendo tipos

```python
idade_texto = input("Digite sua idade: ")  # "18"
idade = int(idade_texto)                    # 18 (agora é número)

# Ou em uma linha só:
altura = float(input("Digite sua altura: "))
```

- `int()` converte para número inteiro
- `float()` converte para número decimal
- `str()` converte para texto

### Comentários

```python
# Isto é um comentário — o computador ignora
print("Olá")  # Comentários também podem ficar no fim da linha
```

Use comentários para explicar **por que** o código faz algo, não **o que** ele faz. O "o que" deve ser óbvio pelo código.

---

## Traduzindo Portugol para Python

### Exemplo 1 — Decisão (SE/ENTÃO/SENÃO)

**Portugol:**
```pseudocode
INICIO
    idade = 18
    SE idade >= 18 ENTAO
        ESCREVER "Maior de idade"
    SENAO
        ESCREVER "Menor de idade"
    FIMSE
FIM
```

**Python:**
```python
idade = 18
if idade >= 18:
    print("Maior de idade")
else:
    print("Menor de idade")
```

Diferenças:
- `SE` vira `if` (minúsculo)
- `ENTAO` vira `:` (dois pontos)
- `SENAO` vira `else:`
- O bloco dentro do `if`/`else` é **indentado** com 4 espaços
- Não tem `FIMSE` — a indentação define onde termina

### Exemplo 2 — Loop (ENQUANTO)

**Portugol:**
```pseudocode
contador = 10
ENQUANTO contador >= 0 FACA
    ESCREVER contador
    contador = contador - 1
FIMENQUANTO
```

**Python:**
```python
contador = 10
while contador >= 0:
    print(contador)
    contador = contador - 1
```

### Exemplo 3 — Loop (PARA)

**Portugol:**
```pseudocode
PARA n DE 1 ATE 5 FACA
    ESCREVER n
FIMPARA
```

**Python:**
```python
for n in range(1, 6):   # range(1, 6) = 1, 2, 3, 4, 5
    print(n)
```

> `range(inicio, fim)` vai do `inicio` até **fim-1**. Por isso `range(1, 6)` vai de 1 a 5.

### Exemplo 4 — Função

**Portugol:**
```pseudocode
FUNCAO somar(a, b)
    resultado = a + b
    RETORNAR resultado
FIMFUNCAO
```

**Python:**
```python
def somar(a, b):
    resultado = a + b
    return resultado
```

---

## Erros comuns (e como ler as mensagens)

Erros assustam quem está começando. Mas as mensagens de erro do Python são suas amigas — elas dizem exatamente o que deu errado.

### SyntaxError — você escreveu algo que o Python não entendeu

```python
print("Olá"   # Faltou fechar o parêntese
```

Mensagem de erro:
```
SyntaxError: '(' was never closed
```

Tradução: você abriu um parêntese e não fechou.

### NameError — a variável não existe

```python
print(idade)   # Você nunca definiu 'idade'
```

Mensagem de erro:
```
NameError: name 'idade' is not defined
```

Tradução: você tentou usar uma variável que não foi criada.

### TypeError — você misturou tipos incompatíveis

```python
idade = "18"
print(idade + 2)   # String + número não dá
```

Mensagem de erro:
```
TypeError: can only concatenate str (not "int") to str
```

Tradução: você tentou juntar texto com número. Converta com `int()` primeiro.

### IndentationError — indentação errada

```python
if True:
print("Olá")   # Faltou indentar
```

Mensagem de erro:
```
IndentationError: expected an indented block
```

Tradução: depois de `if`, `else`, `for`, `while`, `def`, o bloco precisa estar indentado (4 espaços).

---

## Exercícios do Capítulo

### Exercício 1 — Conversão de Portugol para Python

Converta os algoritmos abaixo de Portugol para Python:

**A)**
```pseudocode
INICIO
    nome = "Ana"
    idade = 25
    ESCREVER "Nome: ", nome
    ESCREVER "Idade: ", idade
FIM
```

**B)**
```pseudocode
INICIO
    LER numero
    SE numero > 0 ENTAO
        ESCREVER "Positivo"
    SENAO
        SE numero < 0 ENTAO
            ESCREVER "Negativo"
        SENAO
            ESCREVER "Zero"
        FIMSE
    FIMSE
FIM
```

**C)**
```pseudocode
INICIO
    soma = 0
    PARA i DE 1 ATE 100 FACA
        soma = soma + i
    FIMPARA
    ESCREVER "Soma: ", soma
FIM
```

### Exercício 2 — Calculadora de idade

Escreva um programa Python que:
1. Pergunte o ano de nascimento do usuário
2. Pergunte o ano atual
3. Calcule e exiba a idade
4. Se a idade for maior ou igual a 18, exiba "Maior de idade"
5. Senão, exiba "Menor de idade"

### Exercício 3 — Tabuada

Escreva um programa que exiba a tabuada de um número digitado pelo usuário.

Exemplo de saída para o número 7:
```
7 x 1 = 7
7 x 2 = 14
7 x 3 = 21
...
7 x 10 = 70
```

### Exercício 4 — Média de 4 notas

Escreva um programa que:
1. Peça 4 notas ao usuário (use `float()`)
2. Calcule e exiba a média
3. Se a média for >= 7, exiba "Aprovado"
4. Se for >= 5 e < 7, exiba "Recuperação"
5. Se for < 5, exiba "Reprovado"

---

## Resumo do Capítulo

- Python 3 se instala em Windows e Linux com um comando
- VS Code com extensão Python é o ambiente recomendado
- `print()` exibe valores na tela; `input()` lê do teclado
- Os blocos em Python são definidos por **indentação** (4 espaços), não por palavras-chave como FIMSE ou FIMPARA
- As mensagens de erro do Python são descritivas — aprenda a lê-las
- `range(inicio, fim)` gera números de `inicio` até `fim-1`
- `int()` e `float()` convertem texto para números

No próximo capítulo vamos mergulhar nos recursos essenciais do Python para segurança: manipulação de strings, leitura de arquivos, uso de bibliotecas, e tratamento de erros.