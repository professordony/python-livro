# Capítulo 11 — Analisador de Logs

**Objetivos deste capítulo:**
- Entender a importância dos logs na segurança
- Construir um analisador de logs SSH
- Detectar padrões suspeitos (ataques de força bruta)
- Gerar relatórios de atividade por IP
- Implementar threshold para alertas

---

## O valor dos logs

Todo sistema gera logs. SSH registra cada tentativa de login. Firewalls registram cada conexão bloqueada. Servidores web registram cada requisição. Esses logs contam a história do que aconteceu no sistema — e é neles que você encontra evidências de ataques.

O problema? Logs são **volume**. Um servidor pode gerar milhões de linhas por dia. Um ser humano não consegue ler tudo. Um analisador automatizado, sim.

---

## Analisando logs de SSH

Logs de autenticação no Linux ficam em `/var/log/auth.log` (Ubuntu/Debian) ou `/var/log/secure` (RHEL/CentOS). As linhas típicas são:

```
Aug  9 14:30:22 server sshd[1234]: Failed password for root from 10.0.0.5 port 22 ssh2
Aug  9 14:30:23 server sshd[1235]: Failed password for admin from 10.0.0.5 port 22 ssh2
Aug  9 14:30:25 server sshd[1236]: Accepted password for admin from 192.168.1.100 port 22 ssh2
Aug  9 14:31:01 server sshd[1237]: Failed password for nobody from 10.0.0.5 port 5555 ssh2
```

### Passo 1 — Lendo e filtrando

```python
import re
from collections import defaultdict
from datetime import datetime

def parsear_log_ssh(caminho_log):
    """Analisa um log SSH e extrai eventos de autenticação."""
    tentativas = []

    with open(caminho_log, "r") as f:
        for linha in f:
            # Pular linhas que não são de autenticação SSH
            if "sshd" not in linha:
                continue

            # Tentativa de senha (Failed password)
            match = re.search(
                r'(\w+\s+\d+\s+\d+:\d+:\d+).*Failed password for (\w+) from (\d+\.\d+\.\d+\.\d+)',
                linha
            )
            if match:
                tentativas.append({
                    "timestamp": match.group(1),
                    "tipo": "FALHA",
                    "usuario": match.group(2),
                    "ip": match.group(3)
                })
                continue

            # Sucesso (Accepted password)
            match = re.search(
                r'(\w+\s+\d+\s+\d+:\d+:\d+).*Accepted password for (\w+) from (\d+\.\d+\.\d+\.\d+)',
                linha
            )
            if match:
                tentativas.append({
                    "timestamp": match.group(1),
                    "tipo": "SUCESSO",
                    "usuario": match.group(2),
                    "ip": match.group(3)
                })

    return tentativas
```

### Passo 2 — Estatísticas por IP

```python
def estatisticas_por_ip(tentativas):
    """Agrupa tentativas por IP e calcula estatísticas."""
    stats = defaultdict(lambda: {"falhas": 0, "sucessos": 0, "usuarios": set()})

    for t in tentativas:
        ip = t["ip"]
        stats[ip]["usuarios"].add(t["usuario"])
        if t["tipo"] == "FALHA":
            stats[ip]["falhas"] += 1
        else:
            stats[ip]["sucessos"] += 1

    return stats

def gerar_relatorio(tentativas, stats):
    """Gera relatório formatado."""
    print("=" * 60)
    print("RELATÓRIO DE ANÁLISE DE LOG SSH")
    print(f"Total de eventos analisados: {len(tentativas)}")
    print("=" * 60)

    # Top IPs com mais falhas
    print(f"\n-- TOP 10 IPs com mais falhas --")
    ips_ordenados = sorted(stats.items(), key=lambda x: x[1]["falhas"], reverse=True)
    for ip, dados in ips_ordenados[:10]:
        if dados["falhas"] > 0:
            usuarios = ", ".join(sorted(dados["usuarios"])[:5])
            print(f"  {ip:20s} → {dados['falhas']:4d} falhas, "
                  f"{dados['sucessos']} sucessos | usuarios: {usuarios}")

    # Detectar ataques de força bruta
    print(f"\n-- ALERTAS: Possíveis ataques de força bruta --")
    alertas = 0
    for ip, dados in ips_ordenados:
        if dados["falhas"] >= 10:  # Threshold: 10+ falhas
            print(f"  ⚠ ALERTA: {ip} — {dados['falhas']} tentativas falhas!")
            alertas += 1

    if alertas == 0:
        print(f"  Nenhum ataque detectado.")

    print(f"\n-- Estatísticas gerais --")
    total_falhas = sum(d["falhas"] for d in stats.values())
    total_sucessos = sum(d["sucessos"] for d in stats.values())
    ips_unicos = len(stats)
    print(f"  IPs únicos:      {ips_unicos}")
    print(f"  Total falhas:    {total_falhas}")
    print(f"  Total sucessos:  {total_sucessos}")
    print("=" * 60)

# Uso
tentativas = parsear_log_ssh("/var/log/auth.log")
stats = estatisticas_por_ip(tentativas)
gerar_relatorio(tentativas, stats)
```

---

## Código completo do analisador

```python
#!/usr/bin/env python3
"""
Analisador de Logs SSH.
Uso: python3 log_analyzer.py /var/log/auth.log
"""

import re
import sys
from collections import defaultdict

def analisar(arquivo):
    eventos = []
    with open(arquivo, "r") as f:
        for linha in f:
            if "Failed password" in linha:
                m = re.search(r'Failed password for (\w+) from (\d+\.\d+\.\d+\.\d+)', linha)
                if m:
                    eventos.append(("FALHA", m.group(1), m.group(2)))
            elif "Accepted password" in linha:
                m = re.search(r'Accepted password for (\w+) from (\d+\.\d+\.\d+\.\d+)', linha)
                if m:
                    eventos.append(("SUCESSO", m.group(1), m.group(2)))
    return eventos

def reportar(eventos):
    stats = defaultdict(lambda: {"falhas": 0, "sucessos": 0})

    for tipo, usuario, ip in eventos:
        stats[ip][tipo] = stats[ip].get(tipo, 0) + 1

    print(f"{'='*60}")
    print(f"ANÁLISE DE LOG — {len(eventos)} eventos")
    print(f"{'='*60}")

    for ip, dados in sorted(stats.items(), key=lambda x: x[1]["FALHA"], reverse=True):
        f = dados.get("FALHA", 0)
        s = dados.get("SUCESSO", 0)
        alerta = " ⚠ BRUTE FORCE" if f >= 10 else ""
        print(f"  {ip:20s}  falhas={f:4d}  sucessos={s:2d}{alerta}")

    print(f"{'='*60}")

if __name__ == "__main__":
    arquivo = sys.argv[1] if len(sys.argv) > 1 else "/var/log/auth.log"
    eventos = analisar(arquivo)
    reportar(eventos)
```

---

## Gerando logs de teste

Para testar o analisador sem precisar de um servidor real, crie um arquivo de log falso:

```python
#!/usr/bin/env python3
"""Gera um log SSH falso para testes."""

import random
from datetime import datetime, timedelta

ips_maliciosos = [
    "10.0.0.5", "10.0.0.50", "10.0.0.99",
    "185.220.101.1", "45.33.32.156", "91.121.87.34"
]
ips_legitimos = ["192.168.1.100", "192.168.1.50", "10.0.0.10"]
usuarios = ["root", "admin", "nobody", "user", "teste", "joao", "maria"]

agora = datetime.now()
with open("/tmp/teste_auth.log", "w") as f:
    for i in range(200):
        timestamp = (agora - timedelta(seconds=random.randint(0, 3600))).strftime("%b %d %H:%M:%S")

        if random.random() < 0.7:  # 70% ataques
            ip = random.choice(ips_maliciosos)
            usuario = random.choice(usuarios)
            f.write(f"{timestamp} server sshd[{random.randint(1000,9999)}]: "
                    f"Failed password for {usuario} from {ip} port 22 ssh2\n")
        else:  # 30% acessos legítimos
            ip = random.choice(ips_legitimos)
            usuario = random.choice(["joao", "maria", "admin"])
            if random.random() < 0.8:
                f.write(f"{timestamp} server sshd[{random.randint(1000,9999)}]: "
                        f"Accepted password for {usuario} from {ip} port 22 ssh2\n")
            else:
                f.write(f"{timestamp} server sshd[{random.randint(1000,9999)}]: "
                        f"Failed password for {usuario} from {ip} port 22 ssh2\n")

print("Log de teste gerado: /tmp/teste_auth.log")
```

Execute o `log_analyzer.py` contra este log de teste.

---

## Exercícios do Capítulo

### Exercício 1 — Analisador de logs HTTP

Adapte o analisador para logs de servidor web (Apache/nginx). O formato típico:

```
192.168.1.1 - - [09/Aug/2026:14:30:22 -0300] "GET /admin HTTP/1.1" 404 1234
192.168.1.5 - - [09/Aug/2026:14:31:01 -0300] "POST /wp-login.php HTTP/1.1" 200 5678
```

Extraia: IP, método (GET/POST), URL, status code. Detecte:
- Múltiplos 404 do mesmo IP (scanner de diretórios)
- Múltiplos POST no mesmo formulário (força bruta)

### Exercício 2 — Timeline

Gere uma timeline de eventos: quantas tentativas (falhas e sucessos) por hora. Use um dicionário onde a chave é a hora (ex: "14:00", "15:00").

### Exercício 3 — Detector de scanner de portas

Além de logs SSH, analise logs de firewall (`/var/log/syslog` ou `/var/log/kern.log`) procurando por:
- Conexões para múltiplas portas do mesmo IP em curto período
- Conexões para portas sequenciais (1, 2, 3...)

---

## Resumo do Capítulo

- **Logs** registram tudo que acontece no sistema — são a principal fonte de evidência
- **Regex** extrai informações estruturadas de logs não-estruturados
- **Threshold** (limiar) define o que é considerado um ataque (ex: 10+ falhas)
- **IPs com muitas falhas** são prováveis ataques de força bruta
- **Logs sintéticos** permitem testar analisadores sem ambiente real

No próximo capítulo, vamos construir um **sniffer de pacotes** com Scapy para capturar e analisar tráfego de rede.