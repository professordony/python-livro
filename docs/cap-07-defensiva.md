# Capítulo 7 — Programação Defensiva

**Objetivos deste capítulo:**
- Validar entrada do usuário de forma robusta
- Sanitizar dados para evitar ataques
- Implementar logging profissional
- Conhecer boas práticas de código seguro
- Debug básico e interpretação de erros

---

## O princípio da desconfiança

Na segurança, existe um princípio fundamental: **nunca confie na entrada do usuário**.

Parece exagero? Pense no seguinte: seu programa pergunta o IP do servidor. O usuário digita `192.168.1.1`. Tudo certo. Mas e se ele digitar `"; DROP TABLE servidores; --`? Ou `../../etc/passwd`? Ou `-1`? Ou uma string de 10.000 caracteres?

Um programa bem escrito trata **toda entrada como potencialmente maliciosa**. Isso se chama **programação defensiva** — você escreve o código assumindo que os dados podem estar errados, incompletos ou intencionalmente adulterados.

---

## Validando entradas

### Validar é diferente de verificar

- **Verificar** é conferir se a senha está correta
- **Validar** é garantir que a senha digitada tem pelo menos 8 caracteres, não contém espaços, etc.

A validação vem **antes** da verificação.

### Validando números

```python
def ler_idade():
    """Lê e valida uma idade. Só aceita números entre 0 e 150."""
    while True:
        entrada = input("Digite sua idade: ")
        try:
            idade = int(entrada)
            if 0 <= idade <= 150:
                return idade
            else:
                print("Idade deve estar entre 0 e 150.")
        except ValueError:
            print("Digite apenas números.")

# Uso
idade = ler_idade()
print(f"Idade registrada: {idade}")
```

Por que validar? Se a idade for usada para calcular acesso a sistemas, um valor inválido (`-1`, `999`) poderia liberar acesso indevidamente ou travar o sistema.

### Validando IPs

```python
import re

def ip_valido(ip):
    """Verifica se uma string é um endereço IP válido."""
    padrao = r'^(\d{1,3})\.(\d{1,3})\.(\d{1,3})\.(\d{1,3})$'
    match = re.match(padrao, ip)
    if not match:
        return False
    # Cada octeto deve estar entre 0 e 255
    for grupo in match.groups():
        if int(grupo) > 255:
            return False
    return True

# Testes
print(ip_valido("192.168.1.1"))    # True
print(ip_valido("256.1.1.1"))      # False — 256 > 255
print(ip_valido("192.168.1"))      # False — só 3 octetos
print(ip_valido("abc.def.ghi.jkl")) # False — não são números
```

### Validando portas

```python
def porta_valida(porta):
    """Uma porta TCP/IP válida está entre 1 e 65535."""
    try:
        p = int(porta)
        return 1 <= p <= 65535
    except (ValueError, TypeError):
        return False
```

### Validando caminhos de arquivo

```python
import os

def caminho_seguro(caminho):
    """Verifica se o caminho não tenta escapar do diretório permitido."""
    # Normaliza o caminho (remove ../, etc.)
    caminho_real = os.path.realpath(caminho)
    diretorio_base = os.path.realpath("/home/usuario/logs/")

    # Verifica se o caminho está dentro do diretório permitido
    return caminho_real.startswith(diretorio_base)
```

Isso previne **Path Traversal** — quando um atacante usa `../../etc/passwd` para acessar arquivos fora do diretório permitido.

---

## Sanitização — limpando a entrada

Validar diz "isso é aceitável?" Sanitizar diz "vou limpar isso para ficar seguro."

### Removendo caracteres perigosos

```python
import re

def sanitizar_nome_arquivo(nome):
    """Remove caracteres que podem ser perigosos em nomes de arquivo."""
    # Apenas letras, números, underline e hífen
    return re.sub(r'[^a-zA-Z0-9_\-\.]', '', nome)

print(sanitizar_nome_arquivo("relatório!@#$v2.1"))  # relatóriov2.1
# Perdeu o "!" e "@#$" — mas o "ó" se manteve porque é letra
```

### Escape de strings para shell

```python
import shlex

# NUNCA faça isso:
comando = f"ping -c 1 {ip_do_usuario}"  # Perigoso! Injeção de comando!
os.system(comando)  # Se ip for "; rm -rf /" — bye bye sistema!

# SEMPRE use shlex.quote():
import subprocess
comando_seguro = ["ping", "-c", "1", ip_do_usuario]
subprocess.run(comando_seguro)  # Argumentos separados = seguro
```

### Sanitizando para HTML (prevenir XSS)

```python
def sanitizar_html(texto):
    """Converte caracteres especiais HTML para entidades seguras."""
    trocas = {
        "&": "&amp;",
        "<": "&lt;",
        ">": "&gt;",
        '"': "&quot;",
        "'": "&#x27;",
    }
    for original, seguro in trocas.items():
        texto = texto.replace(original, seguro)
    return texto

print(sanitizar_html("<script>alert('xss')</script>"))
# &lt;script&gt;alert(&#x27;xss&#x27;)&lt;/script&gt;
```

---

## Logging profissional

Quando algo der errado (e vai dar), você precisa saber o que aconteceu. Um `print()` aqui e ali não resolve. Use o módulo `logging` do Python.

```python
import logging

# Configuração básica
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s',
    filename='app.log',  # Salva em arquivo
    filemode='a'         # Append (não sobrescrever)
)

# Níveis de log (do menos ao mais severo):
logging.debug("Diagnóstico detalhado")       # DEBUG: só quando está depurando
logging.info("Scanner iniciado em 10.0.0.1") # INFO: informações gerais
logging.warning("Timeout na porta 443")       # WARNING: algo inesperado mas não crítico
logging.error("Falha ao conectar no banco")  # ERROR: algo quebrou
logging.critical("Sistema sem disco!")        # CRITICAL: sistema vai parar
```

### Exemplo prático — scanner com logging

```python
import logging
import socket

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s',
    filename='scanner.log',
    filemode='a'
)

def testar_porta(host, porta):
    try:
        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        sock.settimeout(1)
        resultado = sock.connect_ex((host, porta))
        sock.close()

        if resultado == 0:
            logging.info(f"Porta {porta} ABERTA em {host}")
            return True
        else:
            logging.debug(f"Porta {porta} FECHADA em {host}")
            return False
    except socket.gaierror as e:
        logging.error(f"Erro de DNS ao resolver {host}: {e}")
        return False
    except Exception as e:
        logging.error(f"Erro inesperado ao testar {host}:{porta}: {e}")
        return False

host = "192.168.1.1"
logging.info(f"Iniciando varredura em {host}")
for porta in [22, 80, 443, 3306]:
    if testar_porta(host, porta):
        print(f"Porta {porta}: ABERTA")
logging.info(f"Varredura concluída em {host}")
```

---

## Debug básico — quando o código não funciona

### 1. Print estratégico

A técnica mais antiga e ainda uma das mais eficazes:

```python
def calcula_hash(arquivo):
    print(f"[DEBUG] Calculando hash de: {arquivo}")  # Print temporário
    h = hashlib.sha256()
    with open(arquivo, "rb") as f:
        for bloco in iter(lambda: f.read(4096), b""):
            print(f"[DEBUG] Lendo bloco de {len(bloco)} bytes")  # Print temporário
            h.update(bloco)
    resultado = h.hexdigest()
    print(f"[DEBUG] Hash final: {resultado}")  # Print temporário
    return resultado
```

Depois que o problema for resolvido, remova ou comente os prints.

### 2. Depurador do VS Code

O VS Code tem um depurador visual que permite:
- Executar linha por linha
- Ver o valor das variáveis
- Parar em pontos específicos (breakpoints)

Para usar: clique à esquerda do número da linha (aparece um ponto vermelho) e pressione F5.

### 3. Leia a mensagem de erro

As mensagens de erro do Python seguem um padrão:

```
Traceback (most recent call last):
  File "scanner.py", line 15, in testar_porta
    resultado = sock.connect_ex((host, porta))
               ^^^^^^
NameError: name 'sock' is not defined
```

A mensagem diz:
- **Onde**: `scanner.py`, linha 15, função `testar_porta`
- **O quê**: `NameError: name 'sock' is not defined`
- **Contexto**: a linha que falhou é `resultado = sock.connect_ex((host, porta))`

A causa? A variável `sock` foi definida antes, mas talvez tenha sido sobrescrita ou o bloco onde foi criada terminou.

---

## Práticas de código seguro

### 1. Nunca hardcode senhas

```python
# ❌ ERRADO
senha_banco = "admin123"
conexao = conectar("root", senha_banco)

# ✅ CERTO — use variáveis de ambiente
import os
senha_banco = os.environ.get("DB_PASSWORD")
if not senha_banco:
    raise ValueError("DB_PASSWORD não configurada!")
```

### 2. Limite recursos (rate limiting)

```python
import time

def tentar_login(usuario, senha):
    """Simula uma tentativa de login com delay."""
    time.sleep(1)  # Força 1 segundo entre tentativas
    return senha == "segredo"

# O delay de 1 segundo torna ataques de força bruta muito mais lentos
```

### 3. Use timeout em conexões

```python
import socket

# SEMPRE defina timeout
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.settimeout(5)  # 5 segundos, no máximo
```

Sem timeout, seu programa pode travar esperando uma conexão que nunca acontece.

### 4. Feche recursos

```python
# Use with — garante fechamento automático
with open("arquivo.txt", "r") as f:
    dados = f.read()
# Arquivo fechado automaticamente aqui

with socket.socket() as s:
    s.connect(("example.com", 80))
    s.send(b"GET / HTTP/1.0\r\n\r\n")
    resposta = s.recv(4096)
# Socket fechado automaticamente aqui
```

### 5. Valide antes de processar

```python
def processar_log(arquivo):
    # Validar antes
    if not arquivo.endswith(".log"):
        raise ValueError("Formato de arquivo inválido")
    if not os.path.exists(arquivo):
        raise FileNotFoundError("Arquivo não encontrado")
    if os.path.getsize(arquivo) == 0:
        raise ValueError("Arquivo vazio")

    # Processar
    with open(arquivo, "r") as f:
        ...
```

### 6. Use tipos específicos

```python
# ❌ ERRADO — usar string para tudo
def verificar_acesso(nivel):
    if nivel == "1":
        print("Acesso básico")

# ✅ CERTO — use inteiros ou enumerações
def verificar_acesso(nivel: int):
    if nivel < 1 or nivel > 3:
        raise ValueError("Nível inválido")
    if nivel >= 1:
        print("Acesso básico")
```

---

## Exercícios do Capítulo

### Exercício 1 — Validador de senha completo

Crie uma função `validar_senha(senha)` que retorne uma lista de problemas encontrados. Regras:

- Mínimo 8 caracteres
- Pelo menos 1 letra maiúscula
- Pelo menos 1 letra minúscula
- Pelo menos 1 número
- Pelo menos 1 caractere especial (!@#$%&*)
- Não pode conter espaços

A função deve retornar uma lista vazia se tudo estiver OK, ou uma lista de mensagens (ex: "Senha muito curta") para cada regra não atendida.

### Exercício 2 — Log analyzer com logging

Reescreva o scanner de portas do capítulo anterior usando `logging` em vez de `print()`. Use níveis apropriados:

- INFO para portas abertas
- WARNING para timeouts
- ERROR para falhas de DNS
- DEBUG para portas fechadas

### Exercício 3 — Menu com validação

Crie um programa que exiba um menu:

```
1. Escanear portas
2. Verificar integridade
3. Analisar log
4. Sair
```

Valide a entrada do usuário: só aceite 1, 2, 3 ou 4. Se digitar outra coisa, exiba "Opção inválida" e mostre o menu novamente. Use um loop `while`.

### Exercício 4 — Caminho seguro

Escreva uma função `ler_arquivo_seguro(caminho, diretorio_base)` que:

1. Verifica se o caminho está dentro do diretório base (use `os.path.realpath`)
2. Se estiver, lê e retorna o conteúdo
3. Se não estiver, levanta uma exceção `PermissionError`

Teste com:
- `caminho = "relatorios/log.txt", base = "/home/usuario/relatorios"` → OK
- `caminho = "../../etc/passwd", base = "/home/usuario/relatorios"` → Erro

---

## Resumo do Capítulo

- **Programação defensiva** assume que toda entrada pode ser maliciosa
- **Validar** é verificar se o dado é aceitável; **sanitizar** é limpar o dado
- Use `try/except` para capturar erros sem quebrar o programa
- **logging** substitui `print()` para registro profissional de eventos
- **Timeout** em conexões evita que o programa trave
- **Recursos** (arquivos, sockets) devem ser fechados com `with`
- **Senhas e credenciais** nunca devem estar hardcoded no código
- Valide **antes** de processar, não depois

Com isso, encerramos a **Parte 2**. Você agora tem a base de Python necessária para construir ferramentas de segurança.

Na **Parte 3**, vamos começar a construir ferramentas reais de ataque e defesa — começando com um scanner de portas completo em Python.