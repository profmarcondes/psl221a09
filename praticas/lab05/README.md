# Laboratório 05: <br> Inicialização do Sistema Linux (Boot)

<!-- **PSL221A09 — Programação em Sistemas Linux**

| | |
|---|---|
| **Duração estimada** | 80–90 minutos (parte prática) |
| **Formato** | Duplas ou trios, no LSC/LSI |
| **Pré-requisito** | Uma VM Linux com `systemd` (Ubuntu/Debian) e acesso a `sudo` |
| **Entregável** | Pasta `laboratorio05-nomes/` com os 3 exercícios |

---
-->

Este guia consolida o processo de boot do Linux: o que a firmware, o bootloader e o kernel fazem antes do seu primeiro login, e como o **systemd** organiza tudo que roda depois disso. Siga os três exercícios em ordem. Sempre que houver uma seção de **saída esperada**, compare com o que apareceu no seu terminal antes de seguir em frente.

> **Importante — use uma VM com *snapshot*.** O Exercício 3 mexe na configuração de boot (GRUB). Isso é seguro em uma VM com um *snapshot* recente, mas **não é** o tipo de coisa para testar em uma máquina de produção ou sem backup. Se você não tiver certeza de como tirar um snapshot na sua VM, pergunte ao professor antes de começar o Exercício 3.

## Preparação do ambiente

1. Confirme que está em um sistema com `systemd` como init (não em um contêiner):

   ```bash
   $ ps -p 1 -o comm=
   ```

   A saída deve ser `systemd`. Se for outra coisa (ex.: rodando dentro de um contêiner Docker), os Exercícios 2 e 3 não vão funcionar como esperado — use a VM do laboratório.

2. Crie a pasta de trabalho:

   ```bash
   $ mkdir -p working_dir/lab05-nome/{ex1,ex2,ex3}
   $ cd working_dir/lab05-nome
   ```

---

## Exercício 1 — Investigando o boot com `journalctl` (20 min)

> **Objetivo:** Usar `journalctl` para ler o log do boot atual, filtrar por serviço e por prioridade, e identificar quanto tempo cada fase do boot levou.

```bash
$ cd ex1
$ journalctl -b > boot_atual.log            # salva o log do boot atual num arquivo
$ wc -l boot_atual.log
$ journalctl -b -p err                       # so mensagens de erro (ou mais graves)
$ journalctl -u ssh.service --no-pager        # so as mensagens de um servico especifico
$ systemd-analyze                             # tempo total de boot, dividido por fase
$ systemd-analyze blame | head -10            # quais servicos demoraram mais para subir
```

**Responda no arquivo `ex1/respostas.txt`:**

- [ ] **Q1.** Quantas linhas tem o log do boot atual (`wc -l boot_atual.log`)?
- [ ] **Q2.** `journalctl -b -p err` mostrou alguma mensagem? Se sim, cole uma delas e explique, em uma frase, o que parece ter acontecido.
- [ ] **Q3.** Cole a saída do `systemd-analyze`. Quanto tempo o firmware, o carregamento do kernel e o espaço de usuário levaram, cada um?
- [ ] **Q4.** Segundo o `systemd-analyze blame`, qual foi o serviço que mais demorou para subir? Isso é esperado, dado o que ele faz?

> **Dica:** Se você tiver mais de um boot registrado (a VM já reiniciou antes), experimente `journalctl --list-boots` para ver todos, e `journalctl -b -1` para o boot anterior ao atual.

---

## Exercício 2 — Criando e gerenciando um serviço systemd (30–35 min)

> **Objetivo:** Escrever um `.service` do zero, instalá-lo, e usar o ciclo `daemon-reload` → `start` → `status` → `enable`.

### Parte A — O script que o serviço vai rodar

Crie `ex2/monitor-disco.sh`:

```bash
#!/bin/bash
set -euo pipefail

LOG="/var/log/monitor-disco.log"
USO=$(df -h / | tail -1 | awk '{print $5}')

echo "$(date '+%Y-%m-%d %H:%M:%S') - Uso do disco em /: $USO" >> "$LOG"
```

Instale o script e dê permissão de execução:

```bash
$ sudo cp ex2/monitor-disco.sh /usr/local/bin/monitor-disco.sh
$ sudo chmod +x /usr/local/bin/monitor-disco.sh
$ sudo /usr/local/bin/monitor-disco.sh    # teste manual antes de virar servico
$ cat /var/log/monitor-disco.log
```

### Parte B — A unit `.service`

Crie `ex2/monitor-disco.service`:

```ini
[Unit]
Description=Monitor de espaco em disco (PSL221A09)
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/monitor-disco.sh

[Install]
WantedBy=multi-user.target
```

Antes de instalar, valide a sintaxe da unit:

```bash
$ systemd-analyze verify ex2/monitor-disco.service
```

**Saída esperada:** nenhuma saída (silêncio = sem erros de sintaxe). Se aparecer um erro sobre o `ExecStart` não ser executável, confirme que você já rodou o `chmod +x` da Parte A.

Instale e rode o serviço:

```bash
$ sudo cp ex2/monitor-disco.service /etc/systemd/system/
$ sudo systemctl daemon-reload
$ sudo systemctl start monitor-disco.service
$ systemctl status monitor-disco.service
```

**Saída esperada (resumida):**

```
● monitor-disco.service - Monitor de espaco em disco (PSL221A09)
     Loaded: loaded (/etc/systemd/system/monitor-disco.service; disabled; ...)
     Active: inactive (dead) since ...; ...
    Process: ... ExecStart=/usr/local/bin/monitor-disco.sh (code=exited, status=0/SUCCESS)
```

> Como o serviço é `Type=oneshot`, ele roda e termina — `inactive (dead)` com `status=0/SUCCESS` é o resultado **correto**, não um erro.

**Responda no arquivo `ex2/respostas.txt`:**

- [ ] **Q5.** O `systemctl status` mostrou `status=0/SUCCESS`? Se não, o que a saída indicou de errado, e como você corrigiu?
- [ ] **Q6.** Rode `sudo systemctl enable monitor-disco.service`. O que esse comando faz — ele **executa** o serviço agora? Qual a diferença entre `start` e `enable`?
- [ ] **Q7.** Rode `sudo systemctl start monitor-disco.service` mais uma vez e depois `cat /var/log/monitor-disco.log`. Quantas linhas o arquivo tem agora? Isso mostra que tipo de comportamento — o serviço acumula histórico ou sobrescreve?

---

## Exercício 3 — Editando um parâmetro de kernel via GRUB (20–25 min)

> **Objetivo:** Adicionar um parâmetro de kernel de forma temporária (só naquele boot) e depois de forma permanente, e entender a diferença.

> **Antes de começar:** confirme com o professor/monitor que sua VM tem um *snapshot* recente. Este exercício é sobre a configuração de **boot** — o risco de deixar a VM sem iniciar, embora baixo, existe se um parâmetro digitado errado for salvo permanentemente.

### Parte A — Mudança temporária (só neste boot)

1. Reinicie a VM: `sudo reboot`
2. No menu do GRUB (segure `Shift` durante o boot se ele não aparecer automaticamente), selecione a entrada do Linux e pressione `e` para editar
3. Localize a linha que começa com `linux` (ou `linux16`/`linuxefi`) — ela deve conter algo como `quiet splash`
4. Remova `quiet splash` dessa linha (deixe o resto como está)
5. Pressione `Ctrl+X` (ou `F10`) para iniciar o boot com essa mudança

**Em `ex3/respostas.txt`:**

- [ ] **Q8.** O que mudou visualmente durante o boot, comparado a antes? Você conseguiu ler as mensagens do kernel passando na tela?
- [ ] **Q9.** Depois que o sistema terminou de iniciar normalmente, reinicie de novo **sem** editar o GRUB. O comportamento (com ou sem `quiet splash`) voltou ao original? Por que isso confirma que a mudança foi só temporária?

### Parte B — Mudança permanente

```bash
$ cat /etc/default/grub | grep GRUB_CMDLINE_LINUX_DEFAULT
$ sudo cp /etc/default/grub /etc/default/grub.bak    # backup antes de editar!
$ sudo nano /etc/default/grub
```

Edite a linha `GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"` para incluir um parâmetro extra, por exemplo:

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash psl221a09.turma=2026"
```

(esse parâmetro específico não faz nada de funcional — é só para vocês verem que ele aparece depois do boot)

Aplique a mudança e reinicie:

```bash
$ sudo update-grub
$ sudo reboot
```

Depois do boot, confirme que o parâmetro foi de fato usado pelo kernel:

```bash
$ cat /proc/cmdline
```

**Finalize em `ex3/respostas.txt`:**

- [ ] **Q10.** O parâmetro extra apareceu na saída de `cat /proc/cmdline`? Cole a linha completa.
- [ ] **Q11.** Qual comando você rodaria para **reverter** essa mudança, caso precisasse (dica: você tem um backup do arquivo original)?
- [ ] **Q12.** Compare com a Parte A: por que a mudança da Parte B sobreviveu a um `reboot` e a da Parte A não?

---

## Entrega <!-- e critérios de avaliação -->

### O que entregar

Uma pasta `lab05-nome/` contendo:

- [ ] `ex1/boot_atual.log`, `ex1/respostas.txt`
- [ ] `ex2/monitor-disco.sh`, `ex2/monitor-disco.service`, `ex2/respostas.txt`
- [ ] `ex3/respostas.txt`

<!--
### Rubrica (10 pontos)

| Critério | Pontos | O que é avaliado |
|---|---|---|
| Exercício 1 — `journalctl` | 3,0 | Filtros usados corretamente; interpretação correta de `systemd-analyze`/`blame` |
| Exercício 2 — serviço systemd | 4,0 | `.service` válido (`systemd-analyze verify` sem erros); ciclo start/status/enable demonstrado |
| Exercício 3 — GRUB | 3,0 | Diferença entre edição temporária e permanente demonstrada e explicada corretamente |

> **Dica:** No Exercício 2, "funcionou na primeira tentativa" não é o mais importante — o que conta é vocês saberem **diagnosticar** quando não funciona (por isso o `systemd-analyze verify` e o `systemctl status` fazem parte da rubrica).

---

## Anexo A — Referência rápida

```bash
# journald
journalctl -b                    # log do boot atual
journalctl -b -1                 # log do boot anterior
journalctl -u nome.service       # so um servico
journalctl -p err                # so erros (ou mais graves)
journalctl -f                    # acompanha em tempo real

# systemd-analyze
systemd-analyze                  # tempo total de boot
systemd-analyze blame            # servicos mais lentos
systemd-analyze verify arquivo.service   # valida sintaxe de uma unit

# systemctl
systemctl status [unit]
systemctl start unit
systemctl stop unit
systemctl enable unit            # roda automatico no proximo boot
systemctl disable unit
sudo systemctl daemon-reload     # depois de criar/editar um .service

# GRUB
cat /proc/cmdline                # parametros de kernel usados no boot atual
sudo nano /etc/default/grub      # editar permanentemente
sudo update-grub                 # aplicar (Debian/Ubuntu)
```

## Anexo B — `monitor-disco.service` (referência)

```ini
[Unit]
Description=Monitor de espaco em disco (PSL221A09)
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/monitor-disco.sh

[Install]
WantedBy=multi-user.target
```

**Testado e validado:** `systemd-analyze verify` não reporta erros de sintaxe quando `/usr/local/bin/monitor-disco.sh` existe e é executável; `systemctl start` seguido de `systemctl status` mostra `status=0/SUCCESS`.

-->