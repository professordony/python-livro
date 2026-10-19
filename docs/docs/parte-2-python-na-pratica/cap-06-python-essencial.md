# Capítulo 6 — Python Essencial para Segurança

**Objetivos deste capítulo:**
- Manipular strings como um profissional (split, join, strip)
- Ler e escrever arquivos (logs, relatórios)
- Importar e usar bibliotecas
- Tratar erros com try/except — não deixar o programa quebrar
- Usar expressões regulares para encontrar padrões
- Calcular hashes com hashlib

---

## Strings — o tipo mais versátil

Em segurança, você vai lidar com textos o tempo todo: endereços IP, nomes de host, logs, senhas, cabeçalhos HTTP, pacotes de rede. Saber manipular strings é essencial.

### Dividindo strings com split()

O método `split()` divide uma string em partes baseado em um separador:

```python
ip = "192.168.1.1"
partes = ip.split(".")
print(partes)  # ['192', '168', '1', '1']

# Muito útil para extrair informações
log = "2026-08-09 14:30:22 - Falha de login - IP: 10.0.0.5"
campos = log.split(" - ")
print(campos)
# ['2026-08-09 14:30:22', 'Falha de login', 'IP: 10.0.0.5']

data = campos[0]
evento = campos[1]
ip_atacante = campos[2].replace("IP: ", "")
print(f"Data: {data}, Evento: {evento}, IP: {ip_atacante}")
```

### Juntando strings com join()

O inverso de `split()`:

```python
partes = ["192", "168", "1", "1"]
ip = ".".join(partes)
print(ip)  # '192.168.1.1'

# Juntar uma lista de portas como string
portas = ["22", "80", "443"]
portas_str = ", ".join(portas)
print(f"Portas liberadas: {portas_str}")  # Portas liberadas: 22, 80, 443
```

### Limpando strings com strip()

`strip()` remove espaços em branco (ou outros caracteres) do início e fim:

```python
entrada = "  admin  "
print(entrada.strip())  # 'admin' — espaços removidos

# strip() evita erros bobos de comparação
senha_digitada = " segredo "  # usuário digitou com espaço sem querer
if senha_digitada.strip() == "segredo":
    print("Senha correta!")
```

### Case (maiúscula/minúscula) com lower() e upper()

```python
usuario = "Admin"
if usuario.lower() == "admin":  # converte para minúsculo antes de comparar
    print("Usuário encontrado")

# Comparação case-insensitive é essencial em:
# - Login (usuário digitou "Admin" mas o banco tem "admin")
# - DNS (exemplo.com = Exemplo.COM = EXEMPLO.com)
# - Headers HTTP
```

### Verificando conteúdo com in

```python
log = "192.168.1.10 - Falha de login - SSH"
if "Falha" in log:
    print("Alerta: falha detectada!")

# in funciona em qualquer sequência
portas_malignas = [22, 23, 135, 445, 1433, 3389]
porta = 3389
if porta in portas_malignas:
    print(f"Porta {porta} é comumente visada por ataques")
```

---

## Trabalhando com arquivos

Ler arquivos de log é uma das tarefas mais comuns na segurança. Python torna isso trivial.

### Lendo um arquivo inteiro

```python
with open("servidores.txt", "r") as arquivo:
    conteudo = arquivo.read()
    print(conteudo)
```

O `with` garante que o arquivo será fechado automaticamente, mesmo se ocorrer um erro.

### Lendo linha por linha

```python
with open("access.log", "r") as log:
    for linha in log:
        if "Falha" in linha or "ERRO" in linha:
            print(linha.strip())
```

Isso lê um arquivo de log linha por linha e exibe apenas as que contêm "Falha" ou "ERRO". Para arquivos com milhões de linhas, essa abordagem é eficiente — nunca carrega tudo na memória de uma vez.

### Escrevendo em arquivos

```python
with open("relatorio.txt", "w") as saida:
    saida.write("Relatório de varredura\n")
    saida.write("=" * 30 + "\n")
    saida.write("Porta 22: ABERTA\n")
    saida.write("Porta 80: ABERTA\n")
    saida.write("Porta 443: FILTRADA\n")
```

Modos de abertura:
- `"r"` — leitura (padrão)
- `"w"` — escrita (sobrescreve)
- `"a"` — anexar (adiciona ao final)
- `"r+"` — leitura e escrita

### Exemplo prático — Analisando um log

```python
# Suponha que temos um arquivo auth.log com linhas como:
# Aug  9 14:30:22 server sshd[1234]: Failed password for root from 10.0.0.5 port 22
# Aug  9 14:30:25 server sshd[1235]: Accepted password for admin from 10.0.0.10 port 22

with open("auth.log", "r") as log:
    for linha in log:
        if "Failed password" in linha:
            # Extrai o IP do atacante
            partes = linha.split("from ")
            if len(partes) > 1:
                ip = partes[1].split(" port")[0]
                print(f"Tentativa de invasão de: {ip}")
```

---

## Bibliotecas — o superpoder do Python

Uma das maiores vantagens do Python é seu ecossistema de bibliotecas. Quase tudo que você precisa fazer já foi feito por alguém e está disponível para instalar.

### Instalando bibliotecas com pip

```bash
pip install requests   # Para fazer requisições HTTP
pip install paramiko   # Para SSH
pip install scapy      # Para manipular pacotes de rede
pip install hashlib    # Já vem instalado (não precisa pip)
```

### Importando e usando

```python
import hashlib       # Biblioteca inteira
from datetime import datetime  # Apenas uma função específica
import sys            # Para interagir com o sistema
```

### Exemplo — datetime

```python
from datetime import datetime

agora = datetime.now()
print(f"Data e hora: {agora}")
print(f"Ano: {agora.year}, Mês: {agora.month}, Dia: {agora.day}")
print(f"Hora: {agora.hour}:{agora.minute}:{agora.second}")

# Timestamp Unix (útil para logs)
timestamp = agora.timestamp()
print(f"Timestamp: {timestamp}")
```

---

## Tratamento de erros — try/except

Programas de segurança precisam ser robustos. Uma falha de rede não pode derrubar seu scanner. Um arquivo corrompido não pode quebrar seu analisador.

### Estrutura básica

```python
try:
    # Código que PODE falhar
    with open("confidenciais.txt", "r") as f:
        conteudo = f.read()
except FileNotFoundError:
    print("Arquivo não encontrado. Usando configuração padrão.")
except PermissionError:
    print("Sem permissão para ler o arquivo.")
except Exception as e:
    print(f"Erro inesperado: {e}")
```

### Capturando o erro sem parar o programa

```python
def testar_porta(host, porta):
    """Tenta conectar em uma porta e retorna True se estiver aberta."""
    import socket
    try:
        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        sock.settimeout(2)
        resultado = sock.connect_ex((host, porta))
        sock.close()
        return resultado == 0
    except Exception as e:
        print(f"Erro ao testar porta {porta}: {e}")
        return False

# Mesmo que uma porta dê erro, as outras continuam sendo testadas
host = "8.8.8.8"
for porta in [22, 80, 443, 8080]:
    if testar_porta(host, porta):
        print(f"Porta {porta}: ABERTA")
    else:
        print(f"Porta {porta}: FECHADA/ERRO")
```

---

## Expressões Regulares (re)

Expressões regulares (regex) são padrões de busca para texto. Elas permitem encontrar **padrões** e não apenas palavras fixas.

Por exemplo: "encontrar todos os IPs num log" é impossível com `in`, mas trivial com regex.

```python
import re

log = """
Aug 9 14:30:22 sshd[1234]: Failed password for root from 10.0.0.5 port 22
Aug 9 14:30:25 sshd[1235]: Accepted password for admin from 192.168.1.100 port 22
Aug 9 14:31:01 sshd[1236]: Failed password for nobody from 172.16.0.50 port 5555
"""

# Encontrar todos os IPs no texto
ips = re.findall(r'\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}', log)
print("IPs encontrados:", ips)
# ['10.0.0.5', '192.168.1.100', '172.16.0.50']
```

### Métodos principais do módulo re

```python
import re

texto = "Falha de login no IP 10.0.0.5 às 14:30"

# re.search() — encontra a primeira ocorrência
match = re.search(r'\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}', texto)
if match:
    print("IP encontrado:", match.group())  # 10.0.0.5

# re.findall() — encontra todas as ocorrências (já vimos acima)

# re.sub() — substitui padrões
anonimizado = re.sub(r'\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}', 'IP_OCULTO', texto)
print(anonimizado)  # Falha de login no IP_OCULTO às 14:30

# Capturando grupos específicos
log = "192.168.1.10 - admin - Acesso CONCEDIDO"
padrao = r'(\d+\.\d+\.\d+\.\d+) - (.+?) - (.+)'
match = re.match(padrao, log)
if match:
    ip = match.group(1)
    usuario = match.group(2)
    resultado = match.group(3)
    print(f"IP: {ip}, Usuário: {usuario}, Resultado: {resultado}")
```

### Caracteres especiais comuns em regex

| Padrão | Significado | Exemplo |
|:-------|:------------|:--------|
| `\d` | Um dígito (0-9) | `\d{3}` = exatamente 3 dígitos |
| `\w` | Uma letra, número ou _ | `\w+` = uma palavra |
| `\s` | Um espaço em branco | |
| `.` | Qualquer caractere | |
| `*` | Zero ou mais vezes | `\d*` = zero ou mais dígitos |
| `+` | Uma ou mais vezes | `\d+` = pelo menos um dígito |
| `{n}` | Exatamente n vezes | `\d{4}` = 4 dígitos |
| `{n,m}` | Entre n e m vezes | `\d{1,3}` = 1 a 3 dígitos |
| `[abc]` | Um dos caracteres listados | `[Pp]ython` = "Python" ou "python" |
| `^` | Início da linha | `^192\.168` = linhas começando com 192.168 |
| `$` | Fim da linha | `erro$` = linhas terminando com "erro" |

---

## Hash com hashlib — a base da integridade

Hash é uma função matemática que transforma qualquer dado numa impressão digital de tamanho fixo. É usada para:

- **Verificar integridade** de arquivos (o arquivo foi alterado?)
- **Armazenar senhas** com segurança (nunca guarde senhas em texto puro!)
- **Identificar arquivos** duplicados

```python
import hashlib

# Hash de uma string
senha = "minha_senha_secreta"
hash_senha = hashlib.sha256(senha.encode()).hexdigest()
print(f"SHA256: {hash_senha}")
# Exemplo: 5e884898da28047151d0e56f8dc6292773603d0d6aabbdd62a11ef721d1542d8

# MD5 (mais rápido, mas não seguro para senhas)
hash_md5 = hashlib.md5(senha.encode()).hexdigest()
print(f"MD5: {hash_md5}")
```

### Verificando integridade de arquivos

```python
import hashlib

def calcular_hash(arquivo):
    """Calcula o SHA256 de um arquivo."""
    h = hashlib.sha256()
    with open(arquivo, "rb") as f:  # "rb" = modo binário
        for bloco in iter(lambda: f.read(4096), b""):
            h.update(bloco)
    return h.hexdigest()

# Uso
hash_original = "abc123def456..."
hash_atual = calcular_hash("servidor_backup.zip")

if hash_atual == hash_original:
    print("Arquivo íntegro — não foi alterado")
else:
    print("ALERTA: Arquivo foi modificado!")
```

Este é o mesmo princípio usado por ferramentas de verificação de integridade como Tripwire, AIDE e OSSEC.

---

## Exercícios do Capítulo

### Exercício 1 — Analisador de log de SSH

Dado o arquivo `auth.log` com linhas como:

```
Aug  9 14:30:22 server sshd[1234]: Failed password for root from 10.0.0.5 port 22
Aug  9 14:30:25 server sshd[1235]: Accepted password for admin from 192.168.1.100 port 22
Aug  9 14:31:01 server sshd[1236]: Failed password for nobody from 10.0.0.5 port 5555
Aug  9 14:32:10 server sshd[1237]: Failed password for root from 10.0.0.5 port 22
```

Escreva um programa que:
1. Leia o arquivo linha por linha
2. Para cada linha com "Failed password", extraia o IP do atacante
3. Conte quantas tentativas cada IP fez
4. Se um IP tiver mais de 2 tentativas, exiba "ALERTA: IP {x} com {n} tentativas"

Dica: use um dicionário para contar (ip -> número de tentativas).

### Exercício 2 — Verificador de senha forte

Usando `hashlib`, crie um programa que:
1. Leia uma lista de senhas comuns de um arquivo (uma por linha)
2. Calcule o SHA256 de cada senha
3. Exiba a senha e seu hash
4. Pergunte ao usuário uma senha e verifique se ela está na lista (calculando o hash)

### Exercício 3 — Extrator de e-mails

Usando regex, escreva um programa que extraia todos os endereços de e-mail de um texto.

Dica: um e-mail simples segue o padrão `\w+@\w+\.\w+`.

Teste com:
```
texto = "Contato: admin@empresa.com ou suporte@sec.com. Email invalido: teste@"
```

### Exercício 4 — Relatório de varredura

Crie um programa que:
1. Pergunte ao usuário um host para escanear
2. Para cada porta em [22, 80, 443, 3306, 8080], teste a conexão (use socket)
3. Escreva os resultados num arquivo `relatorio.txt` no formato:
```
Varredura em 192.168.1.1 em 2026-08-09 15:00
Porta 22: ABERTA
Porta 80: FILTRADA
...
```

---

## Resumo do Capítulo

- **split()** e **join()** são as ferramentas mais úteis para manipular strings
- **strip()** remove espaços extras — sempre use em entradas do usuário
- **with open()** é a forma segura de trabalhar com arquivos
- **try/except** impede que erros derrubem seu programa
- **re** (expressões regulares) encontra padrões complexos em textos
- **hashlib** calcula hashes para verificar integridade e armazenar senhas
- Bibliotecas externas instaladas com `pip` expandem o Python para qualquer necessidade

No próximo capítulo vamos aprender **programação defensiva** — validação de entrada, logging e boas práticas para escrever código seguro.