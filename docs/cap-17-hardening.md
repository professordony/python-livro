# Capítulo 17 — Dashboard de Hardening

**Objetivos deste capítulo:**
- Entender o que é hardening de sistemas
- Construir um verificador de segurança automatizado
- Gerar relatório HTML com pontuação
- Verificar 12+ itens de segurança

---

## O que é hardening?

Hardening é o processo de **endurecer** um sistema — reduzir sua superfície de ataque desabilitando serviços desnecessários, aplicando configurações seguras e corrigindo vulnerabilidades conhecidas.

O CIS Benchmarks (Center for Internet Security) define centenas de regras de hardening. Vamos implementar as mais importantes num script automatizado.

---

## Verificador de hardening Linux

```python
#!/usr/bin/env python3
"""
Dashboard de Hardening — verifica segurança do sistema.
Uso: python3 hardening.py
"""

import os
import socket
import subprocess
import platform
from datetime import datetime

class HardeningCheck:
    """Executa verificações de segurança e gera relatório."""

    def __init__(self):
        self.resultados = []
        self.pontos = 0
        self.total_itens = 0

    def check(self, nome, funcao, peso=1):
        """Executa uma verificação e registra o resultado."""
        self.total_itens += 1
        try:
            ok, detalhe = funcao()
            if ok:
                self.pontos += peso
                status = "✓ OK"
            else:
                status = "✗ FALHA"
            self.resultados.append((nome, status, detalhe, ok))
        except Exception as e:
            self.resultados.append((nome, "⚠ ERRO", str(e), False))

    def portas_abertas(self):
        """Verifica portas listening não autorizadas."""
        resultado = subprocess.run(
            ["ss", "-tlnp"], capture_output=True, text=True, timeout=5
        )
        portas = resultado.stdout
        # Portas comuns que devem estar abertas
        aceitaveis = [22, 80, 443, 53]
        portas_abertas = []
        for linha in portas.split("\n"):
            if "LISTEN" in linha:
                for parte in linha.split():
                    if ":" in parte:
                        try:
                            porta = int(parte.split(":")[-1])
                            if porta not in aceitaveis and porta > 0:
                                portas_abertas.append(porta)
                        except ValueError:
                            pass
        if portas_abertas:
            return False, f"Portas não esperadas: {portas_abertas[:10]}"
        return True, "Apenas portas esperadas"

    def senha_root(self):
        """Verifica se root tem senha (não está lockado)."""
        resultado = subprocess.run(
            ["passwd", "-S", "root"], capture_output=True, text=True, timeout=5
        )
        if " L " in resultado.stdout:  # Locked
            return True, "Conta root com senha bloqueada (login root desabilitado)"
        return False, "Conta root ativa — considere bloquear"

    def ssh_config(self):
        """Verifica configurações básicas de segurança SSH."""
        arquivo = "/etc/ssh/sshd_config"
        if not os.path.exists(arquivo):
            return False, "Arquivo sshd_config não encontrado"

        problemas = []
        with open(arquivo) as f:
            config = f.read()

        if "PermitRootLogin yes" in config:
            problemas.append("PermitRootLogin ativado")
        if "PasswordAuthentication no" not in config:
            problemas.append("PasswordAuthentication não desabilitado")
        if "Port 22" in config and "Port " not in config.replace("Port 22", ""):
            pass  # Porta padrão, não é ideal mas não é falha grave

        if problemas:
            return False, "; ".join(problemas)
        return True, "Configuração SSH adequada"

    def firewall_ativo(self):
        """Verifica se o firewall está ativo."""
        try:
            resultado = subprocess.run(
                ["ufw", "status"], capture_output=True, text=True, timeout=5
            )
            if "active" in resultado.stdout or "Status: active" in resultado.stdout:
                return True, "UFW ativo"
            # Tenta iptables
            resultado = subprocess.run(
                ["iptables", "-L", "-n"], capture_output=True, text=True, timeout=5
            )
            if "Chain INPUT" in resultado.stdout:
                return True, "Iptables com regras"
            return False, "Nenhum firewall detectado"
        except:
            return False, "Não foi possível verificar firewall"

    def atualizacoes(self):
        """Verifica se há atualizações pendentes."""
        try:
            resultado = subprocess.run(
                ["apt-get", "-s", "upgrade"], capture_output=True, text=True, timeout=10
            )
            # Conta pacotes a atualizar
            pacotes = [l for l in resultado.stdout.split("\n") if "upgraded" in l]
            if pacotes:
                num = pacotes[0].strip()
                return False, f"Atualizações pendentes: {num}"
            return True, "Sistema atualizado"
        except:
            return False, "Não foi possível verificar"

    def usuarios_sem_senha(self):
        """Verifica se existem usuários sem senha."""
        try:
            with open("/etc/shadow") as f:
                for linha in f:
                    partes = linha.split(":")
                    if len(partes) >= 2 and partes[1] in ["", "!", "*"]:
                        if partes[0] not in ["root"]:
                            return False, f"Usuário sem senha: {partes[0]}"
            return True, "Todos os usuários têm senha"
        except PermissionError:
            return False, "Sem acesso a /etc/shadow (execute como root)"

    def kernel_atual(self):
        """Verifica versão do kernel."""
        versao = platform.release()
        return True, f"Kernel: {versao}"

    def executar_todas(self):
        """Executa todas as verificações."""
        print("Executando verificações de hardening...\n")

        self.check("Firewall ativo", self.firewall_ativo)
        self.check("Portas listening", self.portas_abertas)
        self.check("Conta root", self.senha_root)
        self.check("Configuração SSH", self.ssh_config)
        self.check("Atualizações pendentes", self.atualizacoes)
        self.check("Usuários sem senha", self.usuarios_sem_senha)
        self.check("Versão do kernel", self.kernel_atual)

        self.gerar_relatorio()

    def gerar_relatorio(self):
        """Gera relatório formatado."""
        timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        score = (self.pontos / self.total_itens) * 100

        print(f"\n{'='*60}")
        print(f"DASHBOARD DE HARDENING")
        print(f"Data: {timestamp}")
        print(f"{'='*60}\n")

        for nome, status, detalhe, ok in self.resultados:
            icone = "✅" if ok else "❌"
            print(f"  {icone} {nome}")
            print(f"     {detalhe}")

        print(f"\n{'='*60}")
        print(f"SCORE: {score:.0f}% ({self.pontos}/{self.total_itens})")
        print(f"{'='*60}")

        if score == 100:
            print("  Parabéns! Sistema bem hardening.")
        elif score >= 70:
            print("  Atenção: Itens pendentes de correção.")
        else:
            print("  Crítico: Múltiplas falhas de segurança.")

        # Relatório HTML
        nome_arquivo = f"hardening_{datetime.now().strftime('%Y%m%d_%H%M%S')}.html"
        with open(nome_arquivo, "w") as f:
            f.write(f"""<html>
<head><title>Hardening Report</title>
<style>
    body {{ font-family: monospace; background: #0f172a; color: #fff; padding: 20px; }}
    .ok {{ color: #22c55e; }} .fail {{ color: #ef4444; }}
    h1 {{ color: #06b6d4; }}
    table {{ border-collapse: collapse; width: 100%; }}
    td, th {{ padding: 8px; border-bottom: 1px solid #334155; }}
    .score {{ font-size: 24px; text-align: center; }}
</style></head>
<body>
<h1>Dashboard de Hardening</h1>
<p>{timestamp}</p>
<table>
<tr><th>Verificação</th><th>Status</th><th>Detalhe</th></tr>
""")
            for nome, status, detalhe, ok in self.resultados:
                cls = "ok" if ok else "fail"
                f.write(f"<tr><td>{nome}</td><td class='{cls}'>{status}</td><td>{detalhe}</td></tr>\n")
            f.write(f"""</table>
<p class='score'>Score: {score:.0f}% ({self.pontos}/{self.total_itens})</p>
</body></html>""")

        print(f"\nRelatório HTML: {nome_arquivo}")

if __name__ == "__main__":
    hc = HardeningCheck()
    hc.executar_todas()
```

---

## Exercícios

### Exercício 1 — Mais verificações

Adicione ao dashboard:
- Verificar se o SELinux ou AppArmor está ativo
- Verificar se o sist.log está rodando
- Verificar se há arquivos com SUID/SGID não autorizados
- Verificar sticky bit em /tmp

### Exercício 2 — Comparativo entre servidores

Modifique o script para gerar um relatório comparativo entre múltiplos servidores via SSH.

### Exercício 3 — Recomendações de correção

Para cada falha, exiba também o comando para corrigi-la. Exemplo:
```
❌ Firewall: UFW inativo
→ Correção: sudo ufw enable
```

---

## Resumo

- **Hardening** reduz a superfície de ataque do sistema
- Verificações comuns: firewall, SSH, portas, atualizações, usuários
- **Score** ajuda a quantificar o nível de segurança
- Relatórios HTML permitem documentação e compartilhamento

No próximo capítulo, vamos **unificar todas as ferramentas** num único CLI.