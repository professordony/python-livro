# Capítulo 18 — Kit de Ferramentas CLI

**Objetivos deste capítulo:**
- Unificar todas as ferramentas do livro num único CLI
- Usar argparse para argumentos de linha de comando
- Criar menu interativo
- Integrar scanner, verificador, sniffer e analisador

---

## Unificando as ferramentas

Ao longo do livro, construímos ferramentas independentes: scanner de portas, verificador de integridade, brute force SSH, sniffer, detector de varredura, monitor de arquivos, analisador de logs, brute force web, dashboard de hardening.

Agora vamos unificar tudo num **kit de ferramentas CLI** — um único comando que acessa qualquer ferramenta.

---

## Estrutura do projeto

```
kit-seguranca/
├── kit.py              # CLI principal (argparse)
├── modules/
│   ├── __init__.py
│   ├── scanner.py      # Scanner de portas
│   ├── integridade.py  # Verificador de integridade
│   ├── ssh_brute.py    # Força bruta SSH
│   ├── log_analyzer.py # Analisador de logs
│   ├── sniffer.py      # Sniffer de pacotes
│   ├── scan_detect.py  # Detector de varredura
│   ├── monitor.py      # Monitor de arquivos
│   ├── web_brute.py    # Força bruta web
│   └── hardening.py    # Dashboard de hardening
├── output/             # Relatórios gerados
└── requirements.txt    # Dependências
```

---

## CLI principal com argparse

```python
#!/usr/bin/env python3
"""
Kit de Ferramentas de Segurança - CLI
Uso: python3 kit.py scan 192.168.1.1 --portas 1-1024
     python3 kit.py integridade /etc --criar
     python3 kit.py ssh usuario@192.168.1.1 --senhas rockyou.txt
     python3 kit.py logs /var/log/auth.log
     python3 kit.py monitor /etc
     python3 kit.py hardening
     python3 kit.py menu              # Modo interativo
"""

import argparse
import sys
import os

# Importa módulos
from modules import (
    scanner,
    integridade,
    ssh_brute,
    log_analyzer,
    monitor,
    hardening
)

def cmd_scan(args):
    """Comando: scan — scanner de portas."""
    porta_range = args.portas
    if "-" in porta_range:
        inicio, fim = map(int, porta_range.split("-"))
    else:
        inicio, fim = 1, int(porta_range)
    scanner.escanear(args.host, inicio, fim, args.threads)

def cmd_integridade(args):
    """Comando: integridade — verificar ou criar baseline."""
    if args.criar:
        integridade.criar_baseline(args.diretorio)
    else:
        integridade.verificar(args.diretorio)

def cmd_ssh(args):
    """Comando: ssh — força bruta SSH."""
    if not args.senhas:
        print("Erro: informe --senhas com o arquivo de senhas.")
        return
    usuario, host = args.alvo.split("@")
    with open(args.senhas) as f:
        senhas = [l.strip() for l in f if l.strip()]
    ssh_brute.brute(host, usuario, senhas, delay=args.delay)

def cmd_logs(args):
    """Comando: logs — analisar logs."""
    log_analyzer.analisar(args.arquivo)

def cmd_monitor(args):
    """Comando: monitor — monitorar diretório."""
    monitor.monitorar(args.diretorio)

def cmd_hardening(args):
    """Comando: hardening — verificar segurança."""
    hc = hardening.HardeningCheck()
    hc.executar_todas()

def cmd_menu():
    """Modo interativo com menu."""
    while True:
        print("\n" + "=" * 50)
        print("KIT DE FERRAMENTAS DE SEGURANÇA")
        print("=" * 50)
        print("1.  Scanner de portas")
        print("2.  Verificador de integridade")
        print("3.  Força bruta SSH")
        print("4.  Analisador de logs")
        print("5.  Monitor de arquivos")
        print("6.  Dashboard de hardening")
        print("0.  Sair")
        print("-" * 50)

        opcao = input("Escolha: ").strip()

        if opcao == "0":
            break
        elif opcao == "1":
            host = input("Host: ")
            scanner.escanear(host, 1, 1024, 100)
        elif opcao == "2":
            dir = input("Diretório: ")
            modo = input("Criar baseline (c) ou verificar (v)? ")
            if modo.lower() == "c":
                integridade.criar_baseline(dir)
            else:
                integridade.verificar(dir)
        elif opcao == "3":
            alvo = input("Alvo (usuario@host): ")
            arq = input("Arquivo de senhas: ")
            usuario, host = alvo.split("@")
            with open(arq) as f:
                senhas = [l.strip() for l in f if l.strip()]
            ssh_brute.brute(host, usuario, senhas)
        elif opcao == "4":
            arq = input("Arquivo de log: ")
            log_analyzer.analisar(arq)
        elif opcao == "5":
            dir = input("Diretório: ")
            monitor.monitorar(dir)
        elif opcao == "6":
            hc = hardening.HardeningCheck()
            hc.executar_todas()

def main():
    parser = argparse.ArgumentParser(
        description="Kit de Ferramentas de Segurança",
        formatter_class=argparse.RawDescriptionHelpFormatter,
        epilog="""
Exemplos:
  kit.py scan 192.168.1.1 --portas 1-1000
  kit.py integridade /etc --criar
  kit.py ssh root@192.168.1.1 --senhas senhas.txt
  kit.py logs /var/log/auth.log
  kit.py hardening
        """
    )

    subparsers = parser.add_subparsers(dest="comando")

    # Scan
    p_scan = subparsers.add_parser("scan", help="Scanner de portas")
    p_scan.add_argument("host", help="Host alvo")
    p_scan.add_argument("--portas", default="1-1024", help="Range de portas")
    p_scan.add_argument("--threads", type=int, default=100, help="Nº de threads")
    p_scan.set_defaults(func=cmd_scan)

    # Integridade
    p_int = subparsers.add_parser("integridade", help="Verificador de integridade")
    p_int.add_argument("diretorio", help="Diretório a verificar")
    p_int.add_argument("--criar", action="store_true", help="Criar baseline")
    p_int.set_defaults(func=cmd_integridade)

    # SSH
    p_ssh = subparsers.add_parser("ssh", help="Força bruta SSH")
    p_ssh.add_argument("alvo", help="usuário@host")
    p_ssh.add_argument("--senhas", help="Arquivo de senhas")
    p_ssh.add_argument("--delay", type=float, default=0.5, help="Delay entre tentativas")
    p_ssh.set_defaults(func=cmd_ssh)

    # Logs
    p_log = subparsers.add_parser("logs", help="Analisador de logs")
    p_log.add_argument("arquivo", help="Arquivo de log")
    p_log.set_defaults(func=cmd_logs)

    # Monitor
    p_mon = subparsers.add_parser("monitor", help="Monitor de diretório")
    p_mon.add_argument("diretorio", help="Diretório a monitorar")
    p_mon.set_defaults(func=cmd_monitor)

    # Hardening
    p_hard = subparsers.add_parser("hardening", help="Dashboard de hardening")
    p_hard.set_defaults(func=cmd_hardening)

    # Menu interativo
    p_menu = subparsers.add_parser("menu", help="Modo interativo")
    p_menu.set_defaults(func=cmd_menu)

    args = parser.parse_args()

    if args.comando:
        args.func(args)
    else:
        parser.print_help()

if __name__ == "__main__":
    main()
```

---

## Arquivo requirements.txt

```txt
paramiko>=3.0
scapy>=2.5
requests>=2.28
watchdog>=3.0
psutil>=5.9
pynput>=1.7
```

---

## Usando o kit

```bash
# Scanner
python3 kit.py scan 192.168.1.1 --portas 1-1000 --threads 200

# Verificador de integridade
python3 kit.py integridade /etc --criar
python3 kit.py integridade /etc

# SSH brute force
python3 kit.py ssh root@192.168.1.100 --senhas rockyou.txt

# Analisar logs
python3 kit.py logs /var/log/auth.log

# Monitor
python3 kit.py monitor /etc

# Hardening
python3 kit.py hardening

# Menu interativo
python3 kit.py menu
```

---

## Exercícios

### Exercício 1 — Adicione o sniffer ao CLI

Integre o sniffer de pacotes (Cap 12) como um comando do kit.

### Exercício 2 — Adicione o brute force web

Integre o brute force web (Cap 16) como comando.

### Exercício 3 — Log de uso

Adicione logging para registrar cada comando executado, com timestamp e parâmetros. Útil para auditoria.

---

## Resumo

- **argparse** cria CLIs profissionais com subcomandos
- A organização em módulos (`modules/`) mantém o código limpo
- O menu interativo facilita o uso para iniciantes
- Um kit unificado é mais útil que scripts isolados

No último capítulo, vamos montar um **laboratório final** que integra ataque e defesa.