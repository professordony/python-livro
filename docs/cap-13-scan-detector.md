# Capítulo 13 — Detector de Varredura

**Objetivos deste capítulo:**
- Entender como detectar um scan de rede
- Usar o sniffer para identificar padrões de varredura
- Implementar threshold de conexões por IP
- Gerar alertas quando um scan for detectado

---

## Como identificar um scan

Um scanner de portas (como o que construímos no Capítulo 8) tenta conectar em várias portas de um alvo. Esse comportamento é **anormal** para um usuário legítimo e pode ser detectado.

Características típicas:
- Múltiplas conexões do mesmo IP para portas **diferentes** no mesmo alvo
- Conexões para portas **sequenciais** (1, 2, 3...)
- Conexões para portas **fechadas** (alvo recusa)
- Intervalo curto entre tentativas

---

## Detector simples

```python
from scapy.all import sniff, IP, TCP
from collections import defaultdict
import time

# Estrutura: { (ip_origem, ip_destino): [(timestamp, porta), ...] }
conexoes = defaultdict(list)
ALERT_THRESHOLD = 10  # Número de portas diferentes antes do alerta
JANELA_TEMPO = 10     # Janela de tempo em segundos

def detectar_scan(pacote):
    if TCP in pacote and IP in pacote:
        ip_origem = pacote[IP].src
        ip_destino = pacote[IP].dst
        porta = pacote[TCP].dport
        agora = time.time()
        chave = (ip_origem, ip_destino)

        # Adiciona a conexão
        conexoes[chave].append((agora, porta))

        # Remove conexões antigas (fora da janela)
        conexoes[chave] = [
            (t, p) for t, p in conexoes[chave]
            if agora - t < JANELA_TEMPO
        ]

        # Verifica se atingiu o threshold
        if len(conexoes[chave]) >= ALERT_THRESHOLD:
            portas = sorted(set(p for _, p in conexoes[chave]))
            print(f"\n⚠ ALERTA: Possível scan detectado!")
            print(f"   Origem:  {ip_origem}")
            print(f"   Alvo:    {ip_destino}")
            print(f"   Portas:  {portas[:20]}{'...' if len(portas) > 20 else ''}")
            print(f"   Conexões: {len(conexoes[chave])} em {JANELA_TEMPO}s")
            # Limpa após alerta para evitar repetição
            conexoes[chave] = []

print("Detector de varredura iniciado. Ctrl+C para parar.")
sniff(prn=detectar_scan, store=False)
```

---

## Detector avançado com relatório

```python
#!/usr/bin/env python3
"""
Detector de Varredura de Rede.
Uso: sudo python3 scan_detector.py
"""

from scapy.all import sniff, IP, TCP
from collections import defaultdict
from datetime import datetime
import time
import json

db = defaultdict(lambda: {
    "primeira_vista": 0,
    "ultima_vista": 0,
    "portas": set(),
    "total_pacotes": 0,
    "flag_syn": 0,
    "flag_fin": 0,
    "alerta_emitido": False
})

ALERTA_PORTAS = 15       # Portas diferentes = alerta
ALERTA_SYN = 50          # SYN puros = alerta (SYN scan)
JANELA_SEGUNDOS = 30     # Janela de análise

def analisar(ip_origem, ip_destino, porta, flags_tcp):
    """Analisa padrões de tráfego."""
    info = db[ip_origem]
    agora = time.time()

    # Inicializa
    if info["primeira_vista"] == 0:
        info["primeira_vista"] = agora

    info["ultima_vista"] = agora
    info["portas"].add(porta)
    info["total_pacotes"] += 1

    # Conta flags
    if flags_tcp == 2:    # SYN puro (início de conexão)
        info["flag_syn"] += 1
    elif flags_tcp == 1:  # FIN (fechamento)
        info["flag_fin"] += 1

    # Verifica se está na janela
    if agora - info["primeira_vista"] > JANELA_SEGUNDOS:
        return  # Ainda dentro da janela, acumulando

    # Gera alerta se necessário
    if info["alerta_emitido"]:
        return

    motivos = []

    if len(info["portas"]) >= ALERTA_PORTAS:
        motivos.append(f"{len(info['portas'])} portas diferentes")

    if info["flag_syn"] >= ALERTA_SYN:
        motivos.append(f"{info['flag_syn']} pacotes SYN puros (SYN scan)")

    if motivos:
        info["alerta_emitido"] = True
        portas_list = sorted(info["portas"])
        print(f"\n⚠ SCAN DETECTADO — {datetime.now().strftime('%H:%M:%S')}")
        print(f"   Origem:   {ip_origem}")
        print(f"   Motivo:   {'; '.join(motivos)}")
        print(f"   Período:  {JANELA_SEGUNDOS}s")
        print(f"   Portas:   {portas_list[:30]}{'...' if len(portas_list) > 30 else ''}")
        print(f"   SYN/FIN:  {info['flag_syn']}/{info['flag_fin']}")

def processar(pacote):
    if TCP in pacote and IP in pacote:
        flags = pacote[TCP].flags
        analisar(
            ip_origem=pacote[IP].src,
            ip_destino=pacote[IP].dst,
            porta=pacote[TCP].dport,
            flags_tcp=int(flags)
        )

print(f"Detector de varredura iniciado (threshold: {ALERTA_PORTAS} portas / "
      f"{ALERTA_SYN} SYN em {JANELA_SEGUNDOS}s)")
sniff(prn=processar, store=False)
```

---

## Testando o detector

Em outro terminal, execute o scanner de portas do Capítulo 8:

```bash
python3 scanner.py 127.0.0.1 1 1000 50
```

O detector deve alertar sobre o scan em segundos.

---

## Exercícios

### Exercício 1 — Detector de scan vertical vs horizontal

- **Scan vertical**: um IP escaneia várias portas do mesmo alvo
- **Scan horizontal**: um IP testa a mesma porta em vários alvos

Adapte o detector para identificar ambos os tipos.

### Exercício 2 — Whitelist de IPs confiáveis

Adicione uma lista de IPs confiáveis (ex: seu monitoramento interno) que não disparam alertas.

### Exercício 3 — Log de alertas

Salve todos os alertas num arquivo JSON com timestamp, IP origem, portas envolvidas.

---

## Resumo

- **Scans de rede** têm padrões detectáveis (muitas portas, SYN puros, intervalos curtos)
- **Threshold** define o que é comportamento normal vs suspeito
- **Janela de tempo** agrupa eventos para análise contextual
- Scanners comuns (nmap, masscan) usam técnicas que deixam rastros detectáveis

No próximo capítulo, construímos um **keylogger** (e seu detector) — ferramenta que captura teclas digitadas.