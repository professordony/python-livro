# Capítulo 15 — Monitor de Integridade em Tempo Real

**Objetivos deste capítulo:**
- Monitorar alterações em diretórios em tempo real
- Usar a biblioteca watchdog
- Detectar criações, modificações e exclusões instantaneamente
- Combinar com hashes para maior precisão

---

## Monitoramento em tempo real

O verificador do Capítulo 9 é **reativo**: você precisa executá-lo para detectar mudanças. Mas ataques acontecem em tempo real. E se pudéssemos detectar a alteração **no momento em que ela acontece**?

A biblioteca `watchdog` faz exatamente isso: monitora diretórios e dispara eventos quando arquivos são criados, modificados ou excluídos.

```bash
pip install watchdog
```

---

## Monitor simples

```python
#!/usr/bin/env python3
"""
Monitor de diretório em tempo real.
Uso: python3 watcher.py /caminho/para/monitorar
"""

import time
import sys
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler

class MonitorHandler(FileSystemEventHandler):
    """Handler personalizado para eventos do sistema de arquivos."""

    def on_modified(self, event):
        """Arquivo modificado."""
        if not event.is_directory:
            print(f"[MODIFICADO] {event.src_path}")

    def on_created(self, event):
        """Arquivo criado."""
        if not event.is_directory:
            print(f"[CRIADO]     {event.src_path}")

    def on_deleted(self, event):
        """Arquivo excluído."""
        if not event.is_directory:
            print(f"[EXCLUÍDO]   {event.src_path}")

    def on_moved(self, event):
        """Arquivo movido/renomeado."""
        if not event.is_directory:
            print(f"[MOVIDO]     {event.src_path} → {event.dest_path}")

if __name__ == "__main__":
    caminho = sys.argv[1] if len(sys.argv) > 1 else "."

    if not os.path.exists(caminho):
        print(f"Erro: {caminho} não existe.")
        sys.exit(1)

    event_handler = MonitorHandler()
    observer = Observer()
    observer.schedule(event_handler, caminho, recursive=True)
    observer.start()

    print(f"Monitorando: {os.path.abspath(caminho)}")
    print("Pressione Ctrl+C para parar.")

    try:
        while True:
            time.sleep(1)
    except KeyboardInterrupt:
        print("\nMonitor encerrado.")
        observer.stop()
    observer.join()
```

---

## Monitor com alerta de hash

O monitor simples acima pode gerar **falsos positivos** (arquivos que são salvos mas sem conteúdo alterado). Vamos combinar com hash para confirmar:

```python
#!/usr/bin/env python3
"""
Monitor de integridade com verificação de hash.
"""

import hashlib
import os
import sys
import time
import json
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler

ARQUIVO_BASELINE = "baseline_watch.json"

def calcular_hash(caminho):
    """Calcula SHA-256 de um arquivo."""
    try:
        h = hashlib.sha256()
        with open(caminho, "rb") as f:
            for bloco in iter(lambda: f.read(65536), b""):
                h.update(bloco)
        return h.hexdigest()
    except:
        return None

class IntegridadeHandler(FileSystemEventHandler):
    def __init__(self, baseline):
        self.baseline = baseline

    def on_modified(self, event):
        if event.is_directory or not os.path.exists(event.src_path):
            return

        # Calcula o hash atual
        hash_atual = calcular_hash(event.src_path)
        if not hash_atual:
            return

        # Compara com a baseline
        rel = os.path.relpath(event.src_path)
        hash_original = self.baseline.get(rel)

        if hash_original and hash_atual != hash_original:
            print(f"\n⚠ ALERTA DE INTEGRIDADE!")
            print(f"   Arquivo: {event.src_path}")
            print(f"   Hash original: {hash_original[:16]}...")
            print(f"   Hash atual:    {hash_atual[:16]}...")
            # Atualiza baseline
            self.baseline[rel] = hash_atual

    def on_created(self, event):
        if not event.is_directory:
            hash_valor = calcular_hash(event.src_path)
            if hash_valor:
                rel = os.path.relpath(event.src_path)
                self.baseline[rel] = hash_valor
                print(f"[NOVO] {event.src_path} (adicionado ao baseline)")

    def on_deleted(self, event):
        if not event.is_directory:
            rel = os.path.relpath(event.src_path)
            if rel in self.baseline:
                del self.baseline[rel]
                print(f"[EXCLUÍDO] {event.src_path} (removido do baseline)")

def criar_baseline(diretorio):
    """Cria baseline inicial de hashes."""
    baseline = {}
    for raiz, _, arquivos in os.walk(diretorio):
        for nome in arquivos:
            caminho = os.path.join(raiz, nome)
            h = calcular_hash(caminho)
            if h:
                rel = os.path.relpath(caminho)
                baseline[rel] = h
    with open(ARQUIVO_BASELINE, "w") as f:
        json.dump(baseline, f, indent=2)
    print(f"Baseline criada: {len(baseline)} arquivos")
    return baseline

if __name__ == "__main__":
    caminho = sys.argv[1] if len(sys.argv) > 1 else "."

    # Carrega ou cria baseline
    if os.path.exists(ARQUIVO_BASELINE):
        with open(ARQUIVO_BASELINE) as f:
            baseline = json.load(f)
        print(f"Baseline carregada: {len(baseline)} arquivos")
    else:
        baseline = criar_baseline(caminho)

    handler = IntegridadeHandler(baseline)
    observer = Observer()
    observer.schedule(handler, caminho, recursive=True)
    observer.start()

    print(f"Monitorando: {os.path.abspath(caminho)} [Ctrl+C para parar]")

    try:
        while True:
            time.sleep(1)
    except KeyboardInterrupt:
        # Salva baseline atualizada
        with open(ARQUIVO_BASELINE, "w") as f:
            json.dump(baseline, f, indent=2)
        print(f"\nBaseline atualizada salva.")
        observer.stop()
    observer.join()
```

---

## Exercícios

### Exercício 1 — Monitore /etc

Execute o monitor contra `/etc` do Linux. Altere um arquivo (adicione um comentário) e veja se o alerta dispara.

### Exercício 2 — Filtro por extensão

Modifique o monitor para ignorar certas extensões (`.tmp`, `.swp`, `.pyc`).

### Exercício 3 — Relatório diário

Adicione uma função que gere um relatório ao final do dia: quantos arquivos foram criados, modificados e excluídos.

---

## Resumo

- **watchdog** monitora diretórios em tempo real
- Eventos: `on_created`, `on_modified`, `on_deleted`, `on_moved`
- Combinar watchdog + hash = monitor de integridade profissional
- Útil para detectar alterações não autorizadas em tempo real

No próximo capítulo, construímos uma **ferramenta de força bruta web** (Red Team).