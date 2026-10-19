# Capítulo 8 — Scanner de Portas

**Objetivos deste capítulo:**
- Entender como funciona um scanner de portas
- Construir um scanner TCP funcional em Python
- Adicionar multithreading para velocidade
- Explicar cada linha do código
- Gerar relatório em arquivo

---

## O que é um scanner de portas?

Um scanner de portas é uma ferramenta que testa quais portas TCP ou UDP estão abertas em um alvo. É usado tanto por profissionais de segurança (para descobrir serviços expostos) quanto por atacantes (para encontrar pontos de entrada).

O princípio é simples: para cada porta, o scanner tenta estabelecer uma conexão TCP. Se a conexão é aceita, a porta está **aberta**. Se é recusada ou o tempo esgota, a porta está **fechada** ou **filtrada**.

É exatamente o loop `PARA` que aprendemos no Capítulo 3, mas agora em Python e com conexões de rede reais.

---

## Versão 1 — Scanner básico (uma porta por vez)

Vamos começar com a versão mais simples: testa portas de 1 a 1024, uma de cada vez.

```python
import socket
import sys

def scan_porta(host, porta):
    """Tenta conectar em uma porta e retorna True se estiver aberta."""
    try:
        # Cria um socket TCP
        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        # Define timeout de 1 segundo
        sock.settimeout(1)
        # Tenta conectar
        resultado = sock.connect_ex((host, porta))
        # Fecha o socket
        sock.close()
        # connect_ex retorna 0 se a conexão foi bem-sucedida
        return resultado == 0
    except socket.gaierror:
        print(f"Erro: host '{host}' não encontrado")
        return False
    except Exception as e:
        print(f"Erro inesperado: {e}")
        return False

def scan_host(host):
    """Escaneia as portas de 1 a 1024 em um host."""
    print(f"Iniciando varredura em {host}")
    print("-" * 50)

    for porta in range(1, 1025):
        if scan_porta(host, porta):
            # Identifica o serviço mais comum para a porta
            servico = servicos_conhecidos.get(porta, "Desconhecido")
            print(f"  Porta {porta:5d} → ABERTA [{servico}]")

    print("-" * 50)
    print("Varredura concluída.")

# Dicionário de serviços conhecidos (porta → nome do serviço)
servicos_conhecidos = {
    21: "FTP", 22: "SSH", 23: "Telnet", 25: "SMTP",
    53: "DNS", 80: "HTTP", 110: "POP3", 111: "RPC",
    135: "RPC", 139: "NetBIOS", 143: "IMAP", 443: "HTTPS",
    445: "SMB", 993: "IMAPS", 995: "POP3S", 1433: "MSSQL",
    1521: "Oracle", 2049: "NFS", 3306: "MySQL", 3389: "RDP",
    5432: "PostgreSQL", 5900: "VNC", 5901: "VNC", 6379: "Redis",
    8080: "HTTP-Alt", 8443: "HTTPS-Alt", 27017: "MongoDB",
}

if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("Uso: python3 scanner.py <IP_ou_hostname>")
        sys.exit(1)

    alvo = sys.argv[1]
    scan_host(alvo)
```

### Explicação linha por linha

Vamos entender o que cada parte faz:

**`import socket`**: Importa o módulo de sockets — a biblioteca que permite criar conexões de rede. É ela que faz o "ping" em cada porta.

**`sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)`**: Cria um socket TCP. `AF_INET` significa IPv4; `SOCK_STREAM` significa TCP.

**`sock.settimeout(1)`**: Define o tempo máximo de espera. Se o host não responder em 1 segundo, a conexão é abortada. Sem isso, seu scanner poderia travar minutos em cada porta.

**`sock.connect_ex((host, porta))`**: Tenta conectar. Retorna 0 se conseguiu (porta aberta) ou um código de erro se não conseguiu.

---

## Versão 2 — Com relatório em arquivo

Vamos adicionar logging profissional:

```python
import socket
import logging
from datetime import datetime

# Configuração de logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s',
    filename='scanner.log',
    filemode='a'
)

def scan_porta(host, porta):
    """Tenta conectar em uma porta."""
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
    except socket.timeout:
        logging.warning(f"Timeout na porta {porta} em {host}")
        return False
    except socket.gaierror:
        logging.error(f"Host {host} não encontrado")
        return False
    except Exception as e:
        logging.error(f"Erro na porta {porta}: {e}")
        return False

def gerar_relatorio(host, portas_abertas):
    """Gera um relatório HTML simples."""
    timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    nome_arquivo = f"relatorio_{host}_{datetime.now().strftime('%Y%m%d_%H%M%S')}.html"

    html = f"""
    <html>
    <head><title>Relatório de Varredura - {host}</title>
    <style>
        body {{ font-family: monospace; background: #0F172A; color: #FFFFFF; padding: 20px; }}
        h1 {{ color: #06B6D4; }}
        .aberta {{ color: #22C55E; }}
        .fechada {{ color: #94A3B8; }}
        table {{ border-collapse: collapse; }}
        td {{ padding: 4px 12px; border-bottom: 1px solid #334155; }}
    </style></head>
    <body>
    <h1>Relatório de Varredura</h1>
    <p>Alvo: {host}</p>
    <p>Data: {timestamp}</p>
    <p>Total de portas abertas: {len(portas_abertas)}</p>
    <table>
        <tr><th>Porta</th><th>Serviço</th></tr>
    """

    for porta in portas_abertas:
        servico = servicos_conhecidos.get(porta, "Desconhecido")
        html += f'<tr><td>{porta}</td><td>{servico}</td></tr>\n'

    html += """
    </table>
    </body></html>
    """

    with open(nome_arquivo, "w") as f:
        f.write(html)

    print(f"Relatório salvo: {nome_arquivo}")
```

Agora o scanner não apenas exibe os resultados na tela, mas também gera um relatório HTML formatado — útil para documentar testes de segurança.

---

## Versão 3 — Multithreading (rápido de verdade)

O maior problema da versão 1 é a **velocidade**. Testar 1024 portas uma por uma, com timeout de 1 segundo cada, leva cerca de 17 minutos. Tempo demais.

A solução é **multithreading**: executar vários testes ao mesmo tempo.

```python
import socket
import threading
from queue import Queue

# Fila de portas a testar
fila = Queue()
portas_abertas = []
lock = threading.Lock()  # Garante que apenas uma thread escreve por vez

def scan_porta(host, porta):
    """Tenta conectar em uma porta."""
    try:
        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        sock.settimeout(1)
        resultado = sock.connect_ex((host, porta))
        sock.close()
        return resultado == 0
    except:
        return False

def worker(host):
    """Função executada por cada thread."""
    while not fila.empty():
        porta = fila.get()
        if scan_porta(host, porta):
            with lock:  # Evita que duas threads escrevam ao mesmo tempo
                portas_abertas.append(porta)
                servico = servicos_conhecidos.get(porta, "Desconhecido")
                print(f"  Porta {porta:5d} → ABERTA [{servico}]")
        fila.task_done()

def scan_rapido(host, portas=range(1, 1025), num_threads=100):
    """Escaneia portas usando múltiplas threads."""
    print(f"Varredura rápida em {host} com {num_threads} threads")

    # Preenche a fila com as portas
    for porta in portas:
        fila.put(porta)

    # Cria e inicia as threads
    threads = []
    for _ in range(num_threads):
        t = threading.Thread(target=worker, args=(host,))
        t.start()
        threads.append(t)

    # Aguarda todas terminarem
    for t in threads:
        t.join()

    portas_abertas.sort()
    return portas_abertas
```

### Como funciona o multithreading

1. **Fila (Queue)**: uma lista compartilhada de portas a testar
2. **Threads**: criamos 100 threads, cada uma pega uma porta da fila, testa, e pega a próxima
3. **Lock**: impede que duas threads escrevam na tela ao mesmo tempo (senão o texto sairia misturado)
4. **Velocidade**: 100 portas simultâneas → 1024 portas em ~10 segundos em vez de 17 minutos

### Exercício — comparando velocidades

Teste o scanner versão 1 e versão 3 no mesmo alvo. Quanto tempo cada um leva para escanear 100 portas?

---

## Código completo do scanner multi-thread

```python
#!/usr/bin/env python3
"""
Scanner de portas TCP multithread.
Uso: python3 scanner.py <host> [porta_inicial] [porta_final] [threads]
"""

import socket
import threading
import sys
from queue import Queue
from datetime import datetime

# Dicionário de serviços
servicos = {
    21: "FTP", 22: "SSH", 23: "Telnet", 25: "SMTP", 53: "DNS",
    80: "HTTP", 110: "POP3", 135: "RPC", 139: "NetBIOS",
    143: "IMAP", 443: "HTTPS", 445: "SMB", 993: "IMAPS",
    995: "POP3S", 1433: "MSSQL", 3306: "MySQL", 3389: "RDP",
    5432: "PostgreSQL", 5900: "VNC", 6379: "Redis", 8080: "HTTP-Alt",
    8443: "HTTPS-Alt", 27017: "MongoDB",
}

fila = Queue()
abertas = []
lock = threading.Lock()

def testar(host, porta):
    try:
        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s.settimeout(0.5)
        r = s.connect_ex((host, porta))
        s.close()
        return r == 0
    except:
        return False

def worker(host):
    while not fila.empty():
        p = fila.get()
        if testar(host, p):
            with lock:
                abertas.append(p)
                nome = servicos.get(p, "?")
                print(f"  PORTA {p:5d} → ABERTA [{nome}]")
        fila.task_done()

def escanear(host, inicio, fim, threads=100):
    print(f"\n{'='*50}")
    print(f"Scanner TCP — {host}")
    print(f"Portas: {inicio} a {fim}  Threads: {threads}")
    print(f"Início: {datetime.now().strftime('%H:%M:%S')}")
    print(f"{'='*50}\n")

    start = datetime.now()
    for p in range(inicio, fim+1):
        fila.put(p)

    for _ in range(threads):
        t = threading.Thread(target=worker, args=(host,))
        t.daemon = True
        t.start()

    fila.join()
    abertas.sort()
    duracao = (datetime.now() - start).total_seconds()

    print(f"\n{'='*50}")
    print(f"Concluído em {duracao:.1f}s")
    print(f"Portas abertas: {len(abertas)}")
    if abertas:
        print(f"Lista: {', '.join(str(p) for p in abertas)}")
    print(f"{'='*50}")

if __name__ == "__main__":
    host = sys.argv[1] if len(sys.argv) > 1 else input("Host: ")
    inicio = int(sys.argv[2]) if len(sys.argv) > 2 else 1
    fim = int(sys.argv[3]) if len(sys.argv) > 3 else 1024
    threads = int(sys.argv[4]) if len(sys.argv) > 4 else 100
    escanear(host, inicio, fim, threads)
```

---

## Exercícios do Capítulo

### Exercício 1 — Escaneie seu próprio computador

Execute o scanner contra `127.0.0.1` (localhost). Quais portas estão abertas? Elas fazem sentido para os serviços que você tem rodando?

### Exercício 2 — Adicione detecção de serviço

Atualmente o scanner identifica serviços por uma tabela fixa. Melhore: use a biblioteca `socket.getservbyport()` para descobrir o nome oficial do serviço.

Dica: `socket.getservbyport(22)` retorna `'ssh'`.

### Exercício 3 — Escaneie um range personalizado

Modifique o programa para aceitar ranges no estilo `22,80,443` ou `1-1000` além do padrão. Exemplo:

```bash
python3 scanner.py 192.168.1.1 22,80,443
python3 scanner.py 192.168.1.1 1-500
```

### Exercício 4 — Detector de sistema operacional

Alguns sistemas operacionais respondem de forma diferente a certas portas. Se as portas 135, 139, 445 estiverem abertas, provavelmente é Windows. Se forem 22, 80, 443, provavelmente é Linux. Adicione uma função `identificar_os(portas)` que tente identificar o sistema baseado nas portas abertas.

### Exercício 5 — Scan silencioso

Adicione uma flag `--quiet` que só exibe portas abertas (sem a lista de portas fechadas no log).

---

## Resumo do Capítulo

- Um **scanner de portas** testa quais portas TCP estão abertas em um host
- `socket.connect_ex()` retorna 0 se a porta está aberta
- **Timeout** é essencial para evitar travamentos
- **Multithreading** com `threading` e `Queue` acelera drasticamente a varredura
- **Lock** evita que threads escrevam na tela simultaneamente
- O scanner completo cabe em ~60 linhas de Python e é uma ferramenta funcional
- Serviços conhecidos podem ser identificados pelo número da porta

No próximo capítulo, vamos mudar de lado — do ataque para a defesa. Vamos construir um **verificador de integridade de arquivos**, uma ferramenta que detecta quando arquivos importantes foram alterados sem autorização.