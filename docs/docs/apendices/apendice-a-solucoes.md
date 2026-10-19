# Apêndice A — Soluções dos Exercícios

Este apêndice contém soluções comentadas para os exercícios selecionados do livro. Use para conferir suas respostas, mas **tente resolver antes de olhar a solução**.

> Aprendizado real acontece quando você tenta, erra, e descobre o erro. O gabarito é o último recurso.

## Capítulo 1 — Pensamento Computacional

### Exercício 1 — Algoritmo do dia a dia (exemplo: escovar os dentes)

1. Pegar a escova de dentes
2. Colocar pasta na escova
3. Abrir a torneira
4. Molhar a escova
5. Escovar os dentes por 2 minutos
6. Cuspir a espuma
7. Enxaguar a boca
8. Enxaguar a escova
9. Fechar a torneira
10. Guardar a escova

### Exercício 4 — Identificando tipos

1. Nome do host → string
2. Endereço IP → string (tem pontos, não é número)
3. RAM em GB → int (16, 32, 64)
4. Temperatura CPU → float (68.5)
5. Status do serviço → bool (True/False)
6. Versão SO → string ("Ubuntu 22.04")

### Exercício 7 — Que tipo é cada um?

| Valor | Tipo |
|:------|:----:|
| 443 | int |
| 3.14 | float |
| "192.168.1.1" | string |
| verdadeiro | bool |
| -5 | int |
| "bloqueado" | string |
| 0.001 | float |
| 1024 | int |

---

## Capítulo 2 — Decisões e Condições

### Exercício 3 — Combinando condições

1. `SE admin OU permissao_especial ENTAO`
2. `SE disco_cheio E NAO backup_rodou ENTAO`
3. `SE NAO conectado ENTAO`
4. `SE porta == 22 OU porta == 443 ENTAO`
5. `SE tamanho_senha >= 8 ENTAO` (não precisa de E/OU, é uma condição só)

### Exercício 7 — Tabela verdade

| A | B | C | A E B | NÃO C | (A E B) OU (NÃO C) |
|:-:|:-:|:-:|:-----:|:-----:|:------------------:|
| F | F | F | F | V | V |
| F | F | V | F | F | F |
| F | V | F | F | V | V |
| F | V | V | F | F | F |
| V | F | F | F | V | V |
| V | F | V | F | F | F |
| V | V | F | V | V | V |
| V | V | V | V | F | V |

---

## Capítulo 3 — Laços e Repetições

### Exercício 1 — Enquanto houver senha fraca

```pseudocode
INICIO
    senha_valida = falso
    ENQUANTO NAO senha_valida FACA
        ESCREVER "Digite uma senha (min. 8 caracteres):"
        LER senha
        SE COMPRIMENTO(senha) >= 8 ENTAO
            senha_valida = verdadeiro
        SENAO
            ESCREVER "Senha muito curta. Tente novamente."
        FIMSE
    FIMENQUANTO
    ESCREVER "Senha aceita!"
FIM
```

---

## Capítulo 4 — Vetores, Listas e Funções

### Exercício 5 — Funções de segurança (porta_conhecida)

```pseudocode
FUNCAO porta_conhecida(porta)
    portas = [22, 80, 443, 3306, 8080]
    PARA i DE 1 ATE COMPRIMENTO(portas) FACA
        SE portas[i] == porta ENTAO
            RETORNAR verdadeiro
        FIMSE
    FIMPARA
    RETORNAR falso
FIMFUNCAO
```

---

## Capítulo 5 — Instalação e Primeiro Script

### Exercício 2 — Calculadora de idade

```python
ano_nasc = int(input("Ano de nascimento: "))
ano_atual = int(input("Ano atual: "))
idade = ano_atual - ano_nasc
print(f"Idade: {idade}")
if idade >= 18:
    print("Maior de idade")
else:
    print("Menor de idade")
```

---

## Capítulo 7 — Programação Defensiva

### Exercício 1 — Validador de senha completo

```python
def validar_senha(senha):
    problemas = []
    if len(senha) < 8:
        problemas.append("Senha muito curta (min 8)")
    if not any(c.isupper() for c in senha):
        problemas.append("Falta letra maiúscula")
    if not any(c.islower() for c in senha):
        problemas.append("Falta letra minúscula")
    if not any(c.isdigit() for c in senha):
        problemas.append("Falta número")
    if not any(c in "!@#$%&*" for c in senha):
        problemas.append("Falta caractere especial")
    if " " in senha:
        problemas.append("Não pode conter espaços")
    return problemas
```

---

## Capítulo 8 — Scanner de Portas

### Exercício 2 — Detecção de serviço

```python
import socket
# Em vez da tabela fixa, use:
try:
    servico = socket.getservbyport(porta, "tcp")
    print(f"Porta {porta}: {servico}")
except:
    print(f"Porta {porta}: Desconhecido")
```

---

## Capítulo 10 — Força Bruta SSH

### Exercício 4 — Defesa com fail2ban

```python
# Simulação básica de detecção de brute force
def detectar_brute_force(arquivo_log, limite=5, janela=60):
    import time
    tentativas = {}
    with open(arquivo_log) as f:
        for linha in f:
            if "Failed password" in linha:
                ip = linha.split("from ")[1].split()[0]
                agora = time.time()
                if ip not in tentativas:
                    tentativas[ip] = []
                tentativas[ip].append(agora)
                # Remove entradas antigas
                tentativas[ip] = [t for t in tentativas[ip] if agora - t < janela]
                if len(tentativas[ip]) >= limite:
                    print(f"BLOQUEAR {ip} — {len(tentativas[ip])} tentativas")
                    # Aqui executaria: os.system(f"iptables -A INPUT -s {ip} -j DROP")
```

---

## Capítulo 17 — Dashboard de Hardening

### Exercício 1 — Verificação de SUID

```python
import os
def verificar_suid(diretorio="/usr"):
    suid = []
    for raiz, _, arquivos in os.walk(diretorio):
        for nome in arquivos:
            caminho = os.path.join(raiz, nome)
            if os.path.isfile(caminho) and os.access(caminho, os.X_OK):
                try:
                    perms = oct(os.stat(caminho).st_mode)
                    if "4000" in perms or "6000" in perms:
                        suid.append(caminho)
                except:
                    pass
    return suid
```