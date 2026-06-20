# Solução de Problemas do Sistema Linux

Os logs no Linux funcionam como um registro detalhado de eventos e ações que ocorrem no sistema. Por meio desses registros, os administradores podem rastrear atividades, detectar anomalias, solucionar problemas e manter a integridade do sistema. As ferramentas `journalctl` e `dmesg` são essenciais para interagir com esses logs, permitindo não apenas o acesso a informações passadas, mas também o monitoramento em tempo real da atividade do sistema. A seguir, um **passo a passo** profissional como usar essas ferramentas para analisar os logs do sistema juntamente com informações sobre consumo de recursos e eventos de segurança.

## 1. Descubra quando ocorreu o problema
Primeiro, descubra aproximadamente o horário em que ocorreu o problema.
Verifique o horário do último boot:

```bash
who -b
```
Ou os boots anteriores:
```bash
last reboot
```
Lista dos boots armazenados pelo journal:
```bash
journalctl --list-boots
```
Exemplo:
```
-2  8c1234... Mon Jun 16 09:30
-1  a7b234... Tue Jun 17 14:20
 0  b8c345... Wed Jun 18 08:15
```
Você pode investigar um boot específico:
```bash
journalctl -b -1
```
---

# 2. Verifique erros críticos
Filtre mensagens importantes:
```bash
journalctl -p err -b
```
ou
```bash
journalctl -p warning -b
```
Níveis:
| Nível | Significado |
| ----- | ----------- |
| 0     | Emergency   |
| 1     | Alert       |
| 2     | Critical    |
| 3     | Error       |
| 4     | Warning     |
Para erros e acima:
```bash
journalctl -p 3 -xb
```
---

# 3. Procure mensagens do Kernel
Muitos problemas de hardware aparecem aqui.
```bash
dmesg -T
```
Procure:
```
error
fail
segfault
I/O
oom
panic
BUG
reset
```
Exemplo:
```bash
dmesg -T | grep -Ei "error|fail|oom|segfault|panic"
```
---

# 4. Verifique se houve falta de memória (OOM Killer)
Uma causa muito comum de lentidão.
```bash
journalctl -k | grep -i oom
```
ou
```bash
dmesg -T | grep -i "Killed process"
```
Exemplo:
```
Out of memory: Killed process 4567 firefox
```
Isso significa que o kernel matou um processo.

---

# 5. Verifique falhas de disco
```bash
journalctl | grep -Ei "I/O|ext4|nvme|sda|ata|error"
```
ou
```bash
dmesg -T | grep -Ei "I/O|error|nvme|ssd|ata"
```
Procure mensagens como:
```
Buffer I/O error
EXT4-fs error
NVMe timeout
```
---

# 6. Verifique travamentos de programas
Procure:
```bash
journalctl | grep -i segfault
```
ou
```bash
journalctl | grep -i crash
```
---

# 7. Verifique reinicializações inesperadas
```bash
last -x
```
Mostra:
```
shutdown
reboot
runlevel
```
Se houver reboot sem shutdown, pode indicar:
* kernel panic;
* queda de energia;
* travamento.
---

# 8. Veja o consumo de CPU

Se o problema ainda estiver ocorrendo:
```
top
```
ou
```
htop
```
Instalar:
```bash
sudo apt install htop
```
Ou:
```bash
iotop
```
para uso do disco.

---

# 9. Verifique logs de autenticação
Caso suspeite de atividade indevida:
```bash
journalctl -u ssh
```
ou
```bash
grep "Failed password" /var/log/auth.log
```
No Kali moderno:
```bash
journalctl _COMM=sshd
```
Veja também:
```bash
last
```
e
```bash
lastb
```
---

# 10. Verifique serviços que falharam
```bash
systemctl --failed
```
Para detalhes:
```bash
systemctl status nome_do_servico
```
---

# 11. Verifique eventos de hardware
USB:
```bash
journalctl | grep USB
```
Rede:
```bash
journalctl | grep NetworkManager
```
Wi-Fi:
```bash
journalctl | grep wlan
```
GPU:
```bash
journalctl | grep -Ei "drm|nvidia|amdgpu|i915"
```
---

# 12. Procure mensagens suspeitas
Uma investigação rápida:
```bash
journalctl -b | grep -Ei \
"error|fail|warning|critical|panic|segfault|oom|timeout|denied|attack"
```
---

# 13. Estatísticas do boot
Veja quais serviços demoraram para iniciar:
```bash
systemd-analyze blame
```
E o tempo total:
```bash
systemd-analyze
```
Dependências críticas:
```bash
systemd-analyze critical-chain
```
---

# Procedimento profissional de diagnóstico
Quando um sistema apresenta lentidão ou comportamento estranho, uma sequência eficiente é:
```bash
# Qual boot?
journalctl --list-boots
# Erros do boot atual
journalctl -p err -b
# Kernel
dmesg -T
# OOM
journalctl -k | grep -i oom
# Disco
dmesg -T | grep -Ei "error|nvme|ata|I/O"
# Serviços com falha
systemctl --failed
# Reinicializações
last -x
# Boot lento
systemd-analyze blame
# Travamentos
journalctl | grep -Ei "segfault|crash"
```

## Se você suspeita de comprometimento de segurança
Além dos passos acima, é útil verificar:
* processos executados recentemente (`ps auxf`);
* tarefas agendadas (`crontab -l`, `/etc/cron*`);
* logins e tentativas de autenticação (`last`, `lastb`, `journalctl _COMM=sshd`);
* novos serviços habilitados (`systemctl list-unit-files --state=enabled`);
* conexões de rede ativas (`ss -tulpn`);
* alterações recentes em arquivos críticos (`find /etc -mtime -7`).

## Solicite ajuda da Inteligencia Artificial
Execute a bash aaixo:
```bash
# Redireciona a saida de cada linha para o arquivo diag.txt separando por seções.
{
  echo "== journalctl (err, boot) =="; journalctl -p err -b
  echo "== dmesg -T =="; dmesg -T
  echo "== systemctl --failed =="; systemctl --failed
  echo "== systemd-analyze blame =="; systemd-analyze blame
  echo "== last -x =="; last -x
} > diag.txt 2>&1
```
Envie o arquivo `diag.txt` para uma AI (ChatGPT, Claude, Gemini,...), explique, se hover, o comportamento anômalo do seu SO e solicite ajuda para interpretar e identificar a causa provável baseado no arquivo enviado.

### Se desejar incluir timestamps e o hostname no arquivo, execute a seguinte bash:
```bash
{
  echo "== journalctl (err, boot) | $(hostname) | $(date -Is) ==";
  journalctl -p err -b
  echo "== dmesg -T | $(hostname) | $(date -Is) ==";
  dmesg -T
  echo "== systemctl --failed | $(hostname) | $(date -Is) ==";
  systemctl --failed
  echo "== systemd-analyze blame | $(hostname) | $(date -Is) ==";
  systemd-analyze blame
  echo "== last -x | $(hostname) | $(date -Is) ==";
  last -x
} > diag.txt 2>&1
```
