# Laboratório 07 <br> Dispositivos, /proc e sysfs

<!-- **PSL221A09 — Programação em Sistemas Linux**

| | |
|---|---|
| **Duração estimada** | 45–60 min (conduzido em conjunto com a turma) |
| **Formato** | Individual, em dupla ou trio, no LSC/LSI, no mesmo encontro da teoria |
| **Pré-requisito** | GCC (Laboratório 01), conceitos de dispositivos de caractere/bloco, `/proc` e `/sys` vistos hoje |
| **Entregável** | Pasta `working_dir/lab07-nome/` com os 3 exercícios (`ex1` a `ex3`) |

---
-->

Este laboratório é **curto e guiado**: ao contrário dos demais, pois ele é
conduzido logo após a teoria, **no mesmo encontro**. Os três exercícios são mais
dirigidos que o normal; siga o roteiro passo a passo e pergunte ao professor
sempre que uma saída não bater com o esperado.

## Preparação do ambiente

```bash
$ gcc --version
$ mkdir -p working_dir/lab07-nome/{ex1,ex2,ex3}
$ cd working_dir/lab07-nome
```

---

## Exercício 1 — Dispositivos em `/dev`: caractere, bloco, major/minor (15 min)

> **Objetivo:** Reconhecer, na saída de `ls -l`, se um dispositivo é de
> caractere ou de bloco, e o que os números de *major*/*minor* significam.

Rode, em ordem:

```bash
$ ls -l /dev/sda /dev/null /dev/tty1 /dev/zero 2>/dev/null
$ ls -l /dev | grep -E '^[bc]' | head -15
```

**Saída esperada (nomes de disco podem variar: `sda`, `vda`, `nvme0n1`...):**

```
crw-rw-rw- 1 root root      1,   3 ago 20 10:00 /dev/null
brw-rw---- 1 root disk      8,   0 ago 20 10:00 /dev/sda
crw--w---- 1 root tty       4,   1 ago 20 10:00 /dev/tty1
crw-rw-rw- 1 root root      1,   5 ago 20 10:00 /dev/zero
```

Escolha **três** dispositivos diferentes da listagem completa (`ls -l /dev`)
e, para cada um, anote: tipo (`c` ou `b`), *major*, *minor*, e o que você
acha que esse dispositivo faz.

**Responda no arquivo `ex1/respostas.txt`:**

- [ ] **Q1.** Para `/dev/null`, `/dev/sda` (ou o disco da sua máquina) e
  `/dev/tty1`: qual é o tipo (`c`/`b`), o *major* e o *minor* de cada um?
- [ ] **Q2.** Dos três dispositivos que você escolheu por conta própria na
  listagem completa, qual é de **bloco** e qual(is) são de **caractere**? Como
  você decidiu, olhando só a saída do `ls -l`?
- [ ] **Q3.** Rode `ls -l /dev/sda /dev/sda1` (ajuste para o nome do disco da
  sua máquina, se houver partição). O *major* dos dois é igual ou diferente? E o
  *minor*? O que isso indica sobre a relação entre um disco inteiro e suas
  partições?

---

## Exercício 2 — Consultando `/proc` e `/sys` (15–20 min)

> **Objetivo:** Usar `/proc` para ler estado de processos/kernel, e `/sys`
> para ler atributos de um dispositivo real, e comparar os dois.

Rode:

```bash
$ cat /proc/cpuinfo | grep "model name" | head -1
$ cat /proc/meminfo | head -3
$ cat /proc/uptime
$ cat /proc/$$/status | head -5          # $$ = PID do seu proprio shell

$ ls /sys/class/net/
$ cat /sys/class/net/*/operstate          # estado de cada interface de rede
$ cat /sys/class/net/*/address            # endereco MAC de cada interface
```

**Saída esperada (valores variam por máquina):**

```
model name      : Intel(R) Core(TM) i5-8250U CPU @ 1.60GHz
MemTotal:        8127960 kB
MemFree:         2103456 kB
MemAvailable:    5219884 kB
12345.67 8901.23
```

**Responda no arquivo `ex2/respostas.txt`:**

- [ ] **Q4.** Qual foi o `model name` da sua CPU e o `MemTotal` da sua
  máquina/VM?
- [ ] **Q5.** No `/proc/$$/status`, qual é o `Pid` e o `PPid` (processo pai)
  mostrados? Rode `echo $$` antes, para confirmar que bate com o `Pid`.
- [ ] **Q6.** Escolha uma interface de rede da saída de `ls /sys/class/net/`
  (ex.: `eth0`, `wlan0` ou `lo`). Qual é o `operstate` e o `address` (MAC) dela?
  Se você tivesse essa mesma informação em `/proc` em vez de `/sys`, você
  esperaria encontrá-la organizada por **arquivo separado** (um valor por
  arquivo, como em `/sys`) ou **misturada em um único arquivo** (como
  `/proc/cpuinfo`)? Justifique com o que foi discutido na teoria.

---

## Exercício 3 — Um programa em C com `ioctl()` (15–25 min)

> **Objetivo:** Compilar e rodar um programa que usa `ioctl()` para obter
> uma informação que `read()`/`write()` não fornecem: o tamanho do
> terminal em linhas e colunas.

Crie `ex3/tamanho_terminal.c`:

```c
#include <sys/ioctl.h>
#include <stdio.h>
#include <unistd.h>

int main(void) {
    struct winsize ws;

    if (ioctl(STDOUT_FILENO, TIOCGWINSZ, &ws) == -1) {
        perror("ioctl");
        return 1;
    }

    printf("Linhas: %d  Colunas: %d\n", ws.ws_row, ws.ws_col);
    return 0;
}
```

Compile e rode:

```bash
$ cd ex3
$ gcc -Wall -Wextra -o tamanho_terminal tamanho_terminal.c
$ ./tamanho_terminal
```

**Saída esperada (os valores dependem do tamanho da sua janela de terminal):**

```
Linhas: 24  Colunas: 80
```

Agora **redimensione** a janela do terminal (arraste a borda para deixá-la
maior ou menor) e rode `./tamanho_terminal` de novo, sem recompilar.

**Responda no arquivo `ex3/respostas.txt`:**

- [ ] **Q7.** Os valores de `Linhas`/`Colunas` mudaram depois de redimensionar a
  janela e rodar de novo? Isso é esperado, já que `ioctl()` consulta o estado do
  terminal **no momento da chamada**?
- [ ] **Q8.** No código, o `request` passado para `ioctl()` é `TIOCGWINSZ`. O
  terceiro argumento é `&ws`, um ponteiro para `struct winsize`. O que
  aconteceria (na sua opinião, sem precisar testar) se você passasse um ponteiro
  para um `int` no lugar de `&ws`? Por que o compilador não consegue avisar
  sobre esse erro, diferente de um erro de tipo em uma função normal?
- [ ] **Q9.** Modifique o programa para também imprimir `ws.ws_xpixel` e
  `ws.ws_ypixel` (largura/altura do terminal em pixels, quando disponível). Cole
  a saída do seu terminal. Em muitos emuladores de terminal modernos, esses dois
  valores aparecem como `0` — por que você acha que isso acontece (dica: pense
  em terminais que não sabem o tamanho do pixel do caractere, como uma conexão
  SSH em modo texto puro)?

---

## Entrega <!-- e critérios de avaliação -->

### O que entregar

Uma pasta `lab07-nome/` contendo:

- [ ] `ex1/respostas.txt`
- [ ] `ex2/respostas.txt`
- [ ] `ex3/tamanho_terminal.c` (versão final, com `ws_xpixel`/`ws_ypixel`), `ex3/respostas.txt`

<!--
### Rubrica (10 pontos)

| Critério | Pontos | O que é avaliado |
|---|---|---|
| Ex.1 — Dispositivos em `/dev` | 3,0 | Tipo, *major* e *minor* identificados corretamente; relação disco/partição explicada |
| Ex.2 — `/proc` e `/sys` | 3,0 | Valores lidos corretamente; comparação `/proc` vs. `/sys` bem justificada |
| Ex.3 — `ioctl()` em C | 4,0 | Programa compila e roda; modificação com `ws_xpixel`/`ws_ypixel` funcional; respostas conceituais corretas |

> **Dica:** Por ser um laboratório guiado, o professor acompanha a turma
> exercício a exercício — não há problema em perguntar no meio do caminho
> se uma saída não bateu com o esperado. O importante é sair daqui sabendo
> **onde procurar** essas informações no dia a dia (`ls -l /dev`, `cat
> /proc/...`, `cat /sys/.../...`), não decorar números de *major*.

---

## Anexo A — Referência rápida de código

```c
/* ioctl(): assinatura geral */
#include <sys/ioctl.h>
int ioctl(int fd, unsigned long request, ...);

/* Tamanho do terminal */
#include <sys/ioctl.h>
struct winsize ws;
ioctl(STDOUT_FILENO, TIOCGWINSZ, &ws);
/* ws.ws_row, ws.ws_col, ws.ws_xpixel, ws.ws_ypixel */

/* Lendo /proc em C (sem API especial: fopen/fgets normais) */
#include <stdio.h>
FILE *f = fopen("/proc/meminfo", "r");
char linha[256];
while (fgets(linha, sizeof(linha), f)) { /* ... */ }
fclose(f);
```

```bash
# Referencia rapida de comandos
ls -l /dev/<nome>                  # tipo (c/b), major, minor
sudo mknod /dev/<nome> c <maj> <min>  # cria um no de dispositivo (requer root)
cat /proc/cpuinfo                  # info da CPU
cat /proc/meminfo                  # memoria
cat /proc/<pid>/status             # estado de um processo
ls /sys/class/net/                 # interfaces de rede
cat /sys/class/net/<if>/address    # MAC de uma interface
cat /sys/block/<disco>/size        # tamanho em setores de 512 bytes
```
-->