# Capítulo 12 — Sniffer de Pacotes

**Objetivos deste capítulo:**
- Entender como capturar tráfego de rede
- Usar Scapy para snifar pacotes
- Filtrar protocolos específicos (HTTP, DNS)
- Extrair informações como IPs, portas e payloads

---

## O que é um sniffer?

Um sniffer (ou analisador de pacotes) captura o tráfego que passa pela interface de rede. Em modo **promíscuo**, a placa de rede captura **todos** os pacotes que passam por ela, não apenas os destinados ao seu computador.

Ferramentas como Wireshark e tcpdump fazem isso. Agora vamos fazer o nosso em Python.

**Aviso:** Sniffar tráfego de rede que não é seu pode ser crime (Lei 12.737/2012). Teste apenas em redes que você controla ou tem autorização.

---

## Instalando o Scapy

```bash
pip install scapy
```

No Linux, o Scapy precisa de privilégios administrativos para capturar pacotes. Execute com `sudo`.

---

## Passo 1 — Primeiro sniffer

```python
from scapy.all import sniff

def processar_pacote(pacote):
    """Função chamada para cada pacote capturado."""
    print(f"Pacote capturado: {pacote.summary()}")

# Captura 5 pacotes
print("Capturando 5 pacotes...")
sniff(prn=processar_pacote, count=5)
```

Execute com `sudo python3 sniffer.py`. Ele capturará 5 pacotes e exibirá um resumo de cada um.

---

## Passo 2 — Extraindo informações detalhadas

```python
from scapy.all import sniff, IP, TCP, UDP, Raw

def analisar_pacote(pacote):
    """Extrai informações detalhadas de cada pacote."""
    if IP in pacote:
        ip_origem = pacote[IP].src
        ip_destino = pacote[IP].dst

        if TCP in pacote:
            porta_origem = pacote[TCP].sport
            porta_destino = pacote[TCP].dport
            flags = pacote[TCP].flags

            print(f"[TCP] {ip_origem}:{porta_origem} → "
                  f"{ip_destino}:{porta_destino} [Flags: {flags}]")

            # Verifica se tem payload (dados)
            if Raw in pacote:
                payload = pacote[Raw].load.decode(errors="ignore")
                # Exibe apenas payload HTTP (texto legível)
                if len(payload) < 200:  # Limita tamanho
                    print(f"       Payload: {payload[:100]}")

        elif UDP in pacote:
            porta_origem = pacote[UDP].sport
            porta_destino = pacote[UDP].dport
            print(f"[UDP] {ip_origem}:{porta_origem} → "
                  f"{ip_destino}:{porta_destino}")

# Captura 20 pacotes com análise detalhada
print("Analisando 20 pacotes...")
sniff(prn=analisar_pacote, count=20, store=False)
```

---

## Passo 3 — Filtros por protocolo

O Scapy permite capturar apenas tipos específicos de tráfego:

```python
from scapy.all import sniff

# Capturar apenas pacotes HTTP (porta 80)
def http_filter(pacote):
    from scapy.all import TCP, Raw
    if TCP in pacote and (pacote[TCP].dport == 80 or pacote[TCP].sport == 80):
        if Raw in pacote:
            return pacote[Raw].load.decode(errors="ignore")[:200]
    return None

# Capturar apenas DNS (porta 53)
def dns_filter(pacote):
    from scapy.all import UDP, DNS
    if UDP in pacote and pacote[UDP].dport == 53:
        if DNS in pacote:
            try:
                nome = pacote[DNS].qd.qname.decode()
                return f"Consulta DNS: {nome}"
            except:
                return "Consulta DNS detectada"
    return None

# Capturar apenas SSH (porta 22)
def ssh_filter(pacote):
    from scapy.all import TCP
    if TCP in pacote and (pacote[TCP].dport == 22 or pacote[TCP].sport == 22):
        return f"Tráfego SSH detectado"
    return None
```

---

## Passo 4 — Sniffer de credenciais HTTP

HTTP é um protocolo não criptografado. Isso significa que senhas trafegam em texto puro. Vamos construir um sniffer que captura credenciais HTTP:

```python
#!/usr/bin/env python3
"""
Sniffer HTTP — captura credenciais em texto plano.
Uso: sudo python3 http_sniffer.py
"""

from scapy.all import sniff, TCP, Raw

def extrair_credenciais(payload):
    """Tenta extrair usuário e senha de um payload HTTP."""
    credenciais = []

    # POST de formulário
    if "POST" in payload and "login" in payload.lower():
        linhas = payload.split("\n")
        corpo = None
        for i, linha in enumerate(linhas):
            if linha.strip() == "":
                # Linha em branco separa header do body
                corpo = "\n".join(linhas[i+1:])
                break
        if corpo:
            # user=joao&pass=123456
            if "user" in corpo or "login" in corpo or "email" in corpo:
                credenciais.append(f"POST: {corpo[:200]}")

    # Basic Auth
    if "Authorization: Basic" in payload:
        import base64
        match = __import__('re').search(r'Authorization: Basic (\S+)', payload)
        if match:
            try:
                decoded = base64.b64decode(match.group(1)).decode(errors="ignore")
                credenciais.append(f"BASIC AUTH: {decoded}")
            except:
                pass

    return credenciais

def processar_pacote(pacote):
    if TCP in pacote and pacote[TCP].dport == 80 and Raw in pacote:
        payload = pacote[Raw].load.decode(errors="ignore")

        # Filtra apenas requisições HTTP
        if payload.startswith(("GET", "POST", "PUT")):
            host = "desconhecido"
            for linha in payload.split("\n"):
                if linha.startswith("Host:"):
                    host = linha.split(":")[1].strip()
                    break

            creds = extrair_credenciais(payload)
            if creds:
                print(f"\n[!] CREDENCIAIS CAPTURADAS em {host}")
                for c in creds:
                    print(f"    {c}")

print("Sniffer HTTP iniciado. Pressione Ctrl+C para parar.")
sniff(prn=processar_pacote, store=False)
```

---

## Analisando pacotes capturados

Se você tiver um arquivo PCAP (capturado com tcpdump ou Wireshark), o Scapy pode lê-lo:

```python
from scapy.all import rdpcap

pacotes = rdpcap("captura.pcap")

print(f"Arquivo: {len(pacotes)} pacotes")

for i, pacote in enumerate(pacotes[:10]):  # Primeiros 10
    print(f"{i+1}. {pacote.summary()}")

# Filtrar por IP
ips_alvo = [p for p in pacotes if IP in p and p[IP].dst == "192.168.1.100"]
print(f"\nPacotes destinados a 192.168.1.100: {len(ips_alvo)}")
```

---

## Exercícios do Capítulo

### Exercício 1 — Monitor de DNS

Crie um sniffer que exiba todas as consultas DNS feitas na rede. Cada vez que um dispositivo pergunta "qual o IP de exemplo.com?", seu programa deve mostrar.

Dica: DNS usa porta 53 UDP. O pacote DNS tem o campo `qd.qname`.

### Exercício 2 — Contador de protocolos

Crie um sniffer que conte quantos pacotes TCP, UDP e ICMP passam pela interface em 1 minuto. Exiba um resumo a cada 10 segundos.

### Exercício 3 — Sniffer de arquivos

Modifique o sniffer HTTP para salvar arquivos transferidos via HTTP (imagens, PDFs).

Dica: procure por `Content-Type: image/jpeg` no header e extraia os bytes após o corpo.

### Exercício 4 — Alerta de conexões suspeitas

Crie um sniffer que dispare um alerta quando detectar conexões para portas conhecidas de malware (ex: 4444, 31337, 6667).

---

## Resumo do Capítulo

- **Scapy** é a principal biblioteca Python para manipulação de pacotes
- **sniff()** captura pacotes em tempo real; **rdpcap()** lê arquivos PCAP
- Pacotes podem ser filtrados por protocolo (TCP, UDP, DNS)
- **HTTP não é criptografado** — senhas e dados trafegam em texto puro
- O modo promíscuo da placa de rede permite capturar todo o tráfego do segmento

No próximo capítulo, usamos o sniffer para **detectar varreduras** — identificando quando alguém está escaneando nossa rede.