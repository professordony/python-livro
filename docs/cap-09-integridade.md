# Capítulo 9 — Verificador de Integridade

**Objetivos deste capítulo:**
- Entender o conceito de integridade de arquivos
- Construir um verificador que cria baseline e detecta mudanças
- Usar hashlib para gerar impressões digitais
- Percorrer diretórios recursivamente
- Gerar alertas quando arquivos são alterados, criados ou removidos

---

## O que é integridade de arquivos?

Integridade significa que um arquivo **não foi alterado** desde a última vez que foi verificado.

Imagine que você configurou um servidor Linux. O arquivo `/etc/passwd` contém os usuários do sistema, `/etc/shadow` contém as senhas, e `/etc/ssh/sshd_config` contém as configurações do SSH.

Se um invasor conseguir acesso ao servidor, a primeira coisa que ele vai fazer é **alterar arquivos** — criar um usuário, abrir uma porta, plantar um backdoor. Um verificador de integridade detecta exatamente isso: ele calcula o hash de cada arquivo importante e compara com o hash da última verificação.

Se o hash mudou, o arquivo foi alterado. Simples e poderoso.

---

## Passo 1 — Calculando hash de arquivos

Vamos começar com a função que calcula o hash de um único arquivo:

```python
import hashlib
import os

def hash_arquivo(caminho):
    """Calcula o SHA-256 de um arquivo. Retorna None em caso de erro."""
    try:
        h = hashlib.sha256()
        with open(caminho, "rb") as f:
            # Lê em blocos de 64KB para não consumir muita memória
            for bloco in iter(lambda: f.read(65536), b""):
                h.update(bloco)
        return h.hexdigest()
    except (PermissionError, FileNotFoundError):
        return None
```

Por que SHA-256 e não MD5? MD5 é mais rápido, mas não é mais considerado seguro para verificação de integridade contra adversários intencionais. SHA-256 é o padrão atual.

---

## Passo 2 — Percorrendo diretórios

Precisamos percorrer todos os arquivos dentro de um diretório:

```python
def listar_arquivos(diretorio):
    """Retorna todos os arquivos dentro de um diretório (recursivo)."""
    arquivos = []
    for raiz, dirs, arquivos_atuais in os.walk(diretorio):
        for nome in arquivos_atuais:
            caminho_completo = os.path.join(raiz, nome)
            # Pula links simbólicos por segurança
            if not os.path.islink(caminho_completo):
                arquivos.append(caminho_completo)
    return sorted(arquivos)
```

`os.walk()` percorre a árvore de diretórios automaticamente, entrando em subpastas. É o `PARA` para diretórios.

---

## Passo 3 — Criando a baseline

A **baseline** é o estado original — uma "fotografia" dos hashes de todos os arquivos quando o sistema estava íntegro.

```python
import json
from datetime import datetime

def criar_baseline(diretorio, arquivo_saida="baseline.json"):
    """Cria um arquivo JSON com os hashes de todos os arquivos do diretório."""
    print(f"Criando baseline de: {diretorio}")
    print(f"Isso pode levar alguns segundos...")

    baseline = {
        "criado_em": datetime.now().isoformat(),
        "diretorio": os.path.abspath(diretorio),
        "arquivos": {}
    }

    arquivos = listar_arquivos(diretorio)
    total = len(arquivos)

    for i, caminho in enumerate(arquivos, 1):
        # Mostra progresso
        if i % 100 == 0:
            print(f"  {i}/{total} arquivos processados...")

        hash_valor = hash_arquivo(caminho)
        if hash_valor:
            # Usa caminho relativo ao diretório base
            relativo = os.path.relpath(caminho, diretorio)
            info = os.stat(caminho)
            baseline["arquivos"][relativo] = {
                "hash": hash_valor,
                "tamanho": info.st_size,
                "modificado": datetime.fromtimestamp(info.st_mtime).isoformat()
            }

    with open(arquivo_saida, "w") as f:
        json.dump(baseline, f, indent=2)

    print(f"\nBaseline criada: {arquivo_saida}")
    print(f"Total de arquivos: {total}")
    return arquivo_saida
```

---

## Passo 4 — Verificando contra a baseline

Agora a parte mais importante: comparar o estado atual com a baseline e detectar mudanças.

```python
def verificar_integridade(diretorio, arquivo_baseline="baseline.json"):
    """Compara o estado atual com a baseline e reporta mudanças."""
    # Carrega a baseline
    with open(arquivo_baseline, "r") as f:
        baseline = json.load(f)

    print(f"\nVerificando integridade de: {diretorio}")
    print(f"Baseline criada em: {baseline['criado_em']}")
    print("-" * 60)

    arquivos_atuais = listar_arquivos(diretorio)
    mudancas = {
        "modificados": [],
        "novos": [],
        "removidos": [],
        "erros": []
    }

    # Converte lista para conjunto para busca rápida
    atuais_rel = set(os.path.relpath(p, diretorio) for p in arquivos_atuais)
    baseline_rel = set(baseline["arquivos"].keys())

    # Arquivos que estavam na baseline mas não existem mais (removidos)
    for rel in baseline_rel - atuais_rel:
        mudancas["removidos"].append(rel)

    # Arquivos que existem agora mas não estavam na baseline (novos)
    for rel in atuais_rel - baseline_rel:
        mudancas["novos"].append(rel)

    # Arquivos que existem em ambos — verificar hash (modificados)
    for rel in baseline_rel & atuais_rel:
        caminho_completo = os.path.join(diretorio, rel)
        hash_atual = hash_arquivo(caminho_completo)
        hash_original = baseline["arquivos"][rel]["hash"]

        if hash_atual is None:
            mudancas["erros"].append(rel)
        elif hash_atual != hash_original:
            mudancas["modificados"].append(rel)

    # Relatório
    if any(mudancas.values()):
        for rel in mudancas["modificados"]:
            print(f"  ⚠ MODIFICADO: {rel}")
            info = baseline["arquivos"][rel]
            print(f"    → Hash original:   {info['hash'][:16]}...")
            novo_hash = hash_arquivo(os.path.join(diretorio, rel))
            print(f"    → Hash atual:      {novo_hash[:16]}...")

        for rel in mudancas["novos"]:
            print(f"  ✚ NOVO: {rel}")

        for rel in mudancas["removidos"]:
            print(f"  ✖ REMOVIDO: {rel}")

        for rel in mudancas["erros"]:
            print(f"  ! ERRO AO LER: {rel}")
    else:
        print("  ✅ Nenhuma alteração detectada.")

    print("-" * 60)
    total = len(mudancas["modificados"]) + len(mudancas["novos"]) + len(mudancas["removidos"])
    print(f"Total de mudanças: {total}")
    print(f"  Modificados: {len(mudancas['modificados'])}")
    print(f"  Novos:       {len(mudancas['novos'])}")
    print(f"  Removidos:   {len(mudancas['removidos'])}")

    return mudancas
```

---

## Código completo do verificador

```python
#!/usr/bin/env python3
"""
Verificador de Integridade de Arquivos.
Uso:
  python3 verificador.py --criar /caminho/diretorio    # Cria baseline
  python3 verificador.py --verificar /caminho/diretorio # Verifica mudanças
"""

import hashlib
import os
import json
import sys
from datetime import datetime

def hash_arquivo(caminho):
    """Calcula SHA-256 de um arquivo."""
    try:
        h = hashlib.sha256()
        with open(caminho, "rb") as f:
            for bloco in iter(lambda: f.read(65536), b""):
                h.update(bloco)
        return h.hexdigest()
    except (PermissionError, FileNotFoundError):
        return None

def listar_arquivos(diretorio):
    """Lista todos os arquivos recursivamente."""
    arquivos = []
    for raiz, _, arquivos_atuais in os.walk(diretorio):
        for nome in arquivos_atuais:
            caminho = os.path.join(raiz, nome)
            if not os.path.islink(caminho):
                arquivos.append(caminho)
    return sorted(arquivos)

def criar_baseline(diretorio, saida="baseline.json"):
    """Cria baseline com hashes de todos os arquivos."""
    print(f"Criando baseline: {diretorio}")
    baseline = {
        "criado_em": datetime.now().isoformat(),
        "diretorio": os.path.abspath(diretorio),
        "arquivos": {}
    }
    arquivos = listar_arquivos(diretorio)
    for i, caminho in enumerate(arquivos, 1):
        if i % 200 == 0:
            print(f"  {i}/{len(arquivos)}...")
        h = hash_arquivo(caminho)
        if h:
            rel = os.path.relpath(caminho, diretorio)
            info = os.stat(caminho)
            baseline["arquivos"][rel] = {
                "hash": h,
                "tamanho": info.st_size,
                "modificado": datetime.fromtimestamp(info.st_mtime).isoformat()
            }
    with open(saida, "w") as f:
        json.dump(baseline, f, indent=2)
    print(f"OK: {len(arquivos)} arquivos em {saida}")

def verificar(diretorio, baseline_arquivo="baseline.json"):
    """Verifica integridade contra a baseline."""
    with open(baseline_arquivo) as f:
        baseline = json.load(f)

    atuais = set(os.path.relpath(p, diretorio) for p in listar_arquivos(diretorio))
    originais = set(baseline["arquivos"].keys())

    print(f"Verificando integridade...")
    mudancas = {"modificados": [], "novos": [], "removidos": []}

    for rel in originais - atuais:
        mudancas["removidos"].append(rel)
    for rel in atuais - originais:
        mudancas["novos"].append(rel)
    for rel in originais & atuais:
        hash_atual = hash_arquivo(os.path.join(diretorio, rel))
        if hash_atual and hash_atual != baseline["arquivos"][rel]["hash"]:
            mudancas["modificados"].append(rel)

    total = sum(len(v) for v in mudancas.values())
    if total == 0:
        print("  ✅ Tudo íntegro — nenhuma alteração.")
    else:
        for r in mudancas["modificados"]:
            print(f"  ⚠ MODIFICADO: {r}")
        for r in mudancas["novos"]:
            print(f"  ✚ NOVO: {r}")
        for r in mudancas["removidos"]:
            print(f"  ✖ REMOVIDO: {r}")
        print(f"  Total: {total} alteração(ões)")

if __name__ == "__main__":
    if len(sys.argv) < 3:
        print("Uso: verificador.py --criar|--verificar <diretorio>")
        sys.exit(1)

    modo, diretorio = sys.argv[1], sys.argv[2]

    if not os.path.isdir(diretorio):
        print(f"Erro: '{diretorio}' não é um diretório válido.")
        sys.exit(1)

    if modo == "--criar":
        criar_baseline(diretorio)
    elif modo == "--verificar":
        verificar(diretorio)
    else:
        print("Modo inválido. Use --criar ou --verificar.")
```

---

## Testando o verificador

Vamos testar com um diretório de exemplo:

```bash
# Cria um diretório de teste
mkdir -p /tmp/teste_integridade
echo "Arquivo importante" > /tmp/teste_integridade/dados.txt
echo "Configuracao" > /tmp/teste_integridade/config.ini

# Cria a baseline
python3 verificador.py --criar /tmp/teste_integridade

# Altera um arquivo (simulando um invasor)
echo "dados adulterados" >> /tmp/teste_integridade/dados.txt

# Cria um arquivo novo (invasor plantou backdoor)
echo "#!/bin/bash" > /tmp/teste_integridade/backdoor.sh

# Verifica
python3 verificador.py --verificar /tmp/teste_integridade
```

A saída deve mostrar:
- `dados.txt` como MODIFICADO
- `backdoor.sh` como NOVO

---

## Exercícios do Capítulo

### Exercício 1 — Monitore diretórios críticos

Crie uma baseline para o diretório `/etc` do seu Linux (ou `C:\Windows\System32\drivers\etc` no Windows). Depois altere um arquivo (adicione um comentário no hosts) e veja se o verificador detecta.

### Exercício 2 — Ignorando diretórios

Adicione uma lista de diretórios para ignorar (ex: `.git`, `__pycache__`, `node_modules`). Modifique a função `listar_arquivos` para pular esses diretórios.

### Exercício 3 — Modo daemon

Faça o verificador rodar em loop a cada N minutos, como um serviço de monitoramento:

```bash
python3 verificador.py --monitorar /diretorio --intervalo 300
```

Dica: use `time.sleep(300)` dentro de um `while True`.

### Exercício 4 — Alerta por e-mail

Quando o verificador detectar mudanças, envie um alerta. Você pode usar a biblioteca `smtplib` para enviar e-mail, ou simplesmente escrever um arquivo de log com timestamp.

---

## Resumo do Capítulo

- **Integridade** garante que arquivos não foram alterados sem autorização
- **SHA-256** é o padrão atual para hashes de verificação
- **Baseline** é o estado original — a "fotografia" dos hashes
- O verificador detecta **modificações**, **novos arquivos** e **arquivos removidos**
- `os.walk()` percorre diretórios recursivamente
- A combinação de hashlib + JSON + os.walk() é a base de ferramentas profissionais como Tripwire e OSSEC

No próximo capítulo, voltamos ao ataque: vamos construir uma **ferramenta de força bruta SSH**.