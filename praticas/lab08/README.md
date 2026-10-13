# Laboratório 08 <br> Processos e Daemons

<!-- **PSL221A09 — Programação em Sistemas Linux**

| | |
|---|---|
| **Duração estimada** | 2h — Parte A (presencial, ~90 min) + Parte B (extraclasse) |
| **Formato** | Individual, em dupla ou trio |
| **Pré-requisito** | GCC, `man` local, um segundo terminal (ou `tmux`/`screen`) para observar processos "de fora"; para os Exercícios 6 e 7, acesso a `sudo` e, de preferência, uma VM com `systemd` |
| **Entregável** | Pasta `working_dir/lab08-nome/` com os exercícios da Parte A; Parte B entregue até o encontro seguinte |

---
-->

Este guia percorre o ciclo de vida completo de um processo no Linux: como `fork()` cria uma cópia do processo atual, como `exec()` substitui a imagem por um programa diferente, o que acontece quando um processo termina antes ou depois do seu pai, e como um processo comum se transforma num *daemon* — desvinculado do terminal, controlado por PID file e registrado como serviço.

**Como usar este guia.** A **Parte A** (Exercícios 1 a 4) é feita durante o encontro de laboratório, com o professor disponível. A **Parte B** (Exercícios 5 a 7) é atividade extraclasse: mesmo formato e mesmo nível de detalhe, para consolidar sozinho o que não coube no tempo de aula. Sempre que houver uma seção de **saída esperada**, compare com o que apareceu no seu terminal antes de seguir em frente.

> **Dica geral:** sempre que o guia pedir para rodar `ps` "em outro terminal", é porque o processo sob observação está dormindo (`sleep()`) de propósito, para dar tempo de olhar o estado dele antes que termine.

> **Atenção à ordem.** Os Exercícios 6 e 7 partem do daemon construído no Exercício 4. Se não terminar o Exercício 4 em aula, conclua-o antes de começar a Parte B.

## Preparação do ambiente

```bash
$ mkdir -p working_dir/lab08-nome/{ex1,ex2,ex3,ex4,ex5,ex6,ex7}
$ cd working_dir/lab08-nome
```

Substitua `nome` pelo seu nome (ou pelos nomes da dupla/trio). Cada exercício tem sua própria pasta (`ex1/` a `ex7/`) e seu próprio `respostas.txt`.

Os programas deste laboratório usam apenas cabeçalhos do sistema — nenhuma biblioteca externa é necessária. Compile sempre com `-Wall`:

```bash
$ gcc -Wall -o programa programa.c
```

---

# Parte A — presencial (~90 min)

## Exercício 1 — `fork()` e a pegadinha do buffer do `printf` (20 min)

> **Objetivo:** Observar que `fork()` duplica o processo (incluindo o que ainda está no buffer de `stdout`), identificar o bug e corrigi-lo.

### Etapa 1 — Rodando o programa

Crie `ex1/fork_demo.c`:

```c
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/wait.h>

int main(void) {
    printf("Antes do fork: PID=%d\n", getpid());

    pid_t pid = fork();

    if (pid < 0) {
        perror("fork");
        return 1;
    } else if (pid == 0) {
        printf("[FILHO]  meu PID=%d, PID do pai=%d\n", getpid(), getppid());
    } else {
        printf("[PAI]    meu PID=%d, PID do filho=%d\n", getpid(), pid);
        int status;
        waitpid(pid, &status, 0);
        printf("[PAI]    filho terminou com status=%d\n", WEXITSTATUS(status));
    }

    return 0;
}
```

Compile e rode **direto no terminal**, várias vezes seguidas:

```bash
$ gcc -Wall -o fork_demo fork_demo.c
$ ./fork_demo
$ ./fork_demo
```

**Saída real capturada (o seu PID vai ser diferente):**

```
Antes do fork: PID=834
[PAI]    meu PID=834, PID do filho=835
[FILHO]  meu PID=835, PID do pai=834
[PAI]    filho terminou com status=0
```

Tudo normal: quatro linhas, uma para cada `printf`. Guarde essa saída.

### Etapa 2 — A mesma execução, com a saída redirecionada

Agora rode **exatamente o mesmo binário**, mas mandando a saída para um `pipe` e depois para um arquivo:

```bash
$ ./fork_demo | cat
$ ./fork_demo > saida.txt && cat saida.txt
```

**Saída real capturada (nos dois casos):**

```
Antes do fork: PID=836
[FILHO]  meu PID=838, PID do pai=836
Antes do fork: PID=836
[PAI]    meu PID=836, PID do filho=838
[PAI]    filho terminou com status=0
```

Repare: agora a linha `Antes do fork: ...` **aparece duas vezes**, mesmo só existindo um `printf` para ela no código — e sem nenhuma mudança no programa. **O mesmo executável se comporta de forma diferente dependendo de para onde a saída vai.**

**Em `ex1/respostas.txt`:**

- [ ] **Q1.** Quantas vezes a linha `Antes do fork: ...` apareceu quando você rodou direto no terminal? E quando rodou com `| cat` ou com `> saida.txt`?
- [ ] **Q2.** `printf` escreve para `stdout`, que é **bufferizado** — o texto fica numa área de memória até ser "descarregado". A diferença é *quando* esse descarregamento acontece: quando `stdout` é um **terminal**, a libc usa buffer **de linha** (cada `\n` já descarrega); quando é um **arquivo ou pipe**, ela usa buffer **cheio** (só descarrega quando enche ou quando o processo termina). Sabendo disso, formule uma hipótese: por que o `fork()` faz a linha aparecer duas vezes só no segundo caso?

### Etapa 3 — Corrigindo o problema

Você já viu esse comportamento de buffer antes (Aula 5, quando comparamos `printf` com `write()` via `strace`). A causa aqui é a mesma: `fork()` faz uma cópia completa da memória do processo — **incluindo o buffer de `stdout` que ainda não foi escrito**. Com buffer cheio (saída em arquivo ou pipe), o texto ainda está no buffer quando o `fork()` acontece, os dois processos carregam uma cópia dele, e cada um vai imprimi-lo quando o seu próprio buffer for descarregado — por isso a linha dobra. No terminal, o `\n` já tinha esvaziado o buffer antes do `fork()`, e nada dobra.

> **Por que isso importa na prática.** Um programa que "funciona" quando você testa no terminal pode duplicar saída assim que alguém o chamar dentro de um `pipeline`, de um `cron`, ou redirecionando para um log. Não dá para confiar no teste interativo.

Crie `ex1/fork_demo_fixed.c` a partir do arquivo anterior, adicionando **uma única linha** logo após o primeiro `printf`, para forçar o buffer a ser descarregado antes do `fork()`:

```bash
$ cp fork_demo.c fork_demo_fixed.c
$ nano fork_demo_fixed.c   # adicione a linha que falta
```

Compile e rode de novo — desta vez testando **o caso que falhava**, com a saída redirecionada:

```bash
$ gcc -Wall -o fork_demo_fixed fork_demo_fixed.c
$ ./fork_demo_fixed | cat
```

**Saída real capturada (com a correção, mesmo redirecionado):**

```
Antes do fork: PID=842
[FILHO]  meu PID=844, PID do pai=842
[PAI]    meu PID=842, PID do filho=844
[PAI]    filho terminou com status=0
```

**Em `ex1/respostas.txt`:**

- [ ] **Q3.** Qual foi a linha exata que você adicionou, e por que ela resolve o problema nos **dois** casos (terminal e redirecionado)?
- [ ] **Q4.** Rode `./fork_demo_fixed` algumas vezes: a ordem entre a linha `[FILHO] ...` e a linha `[PAI] ...` foi sempre a mesma? O que isso indica sobre a ordem de execução entre pai e filho após o `fork()`?

---

## Exercício 2 — `exec()`: trocando o programa em execução (15–20 min)

> **Objetivo:** Diferenciar `fork()` (cria uma cópia) de `exec()` (substitui a imagem do processo por outro programa, mantendo o mesmo PID).

Crie `ex2/exec_demo.c`:

```c
#include <stdio.h>
#include <unistd.h>
#include <sys/wait.h>

int main(void) {
    pid_t pid = fork();

    if (pid == 0) {
        printf("[FILHO] PID=%d antes do exec, vou virar 'ls'\n", getpid());
        fflush(stdout);
        execlp("ls", "ls", "-l", NULL);
        /* so chega aqui se o exec falhar */
        perror("execlp");
        _exit(1);
    } else {
        waitpid(pid, NULL, 0);
        printf("[PAI] filho (que virou 'ls') terminou\n");
    }

    return 0;
}
```

```bash
$ gcc -Wall -o exec_demo exec_demo.c
$ ./exec_demo
```

**Saída real capturada (a listagem varia conforme os arquivos da sua pasta):**

```
[FILHO] PID=27278 antes do exec, vou virar 'ls'
total 124
-rwxr-xr-x 1 aluno aluno 16352 Oct  7 14:17 exec_demo
-rw-r--r-- 1 aluno aluno   479 Oct  7 14:17 exec_demo.c
...
[PAI] filho (que virou 'ls') terminou
```

Repare que **não existe** uma linha `[FILHO]` depois do `ls -l` — porque depois do `execlp`, o processo filho deixou de ser o programa `exec_demo` e passou a ser o programa `ls`. O código que vem depois do `execlp` (o `perror`/`_exit`) só roda se o `exec` **falhar**.

**Em `ex2/respostas.txt`:**

- [ ] **Q5.** Rode `echo $?` logo após `./exec_demo`. Esse número é o código de saída de **qual** processo, e vem do `exec_demo.c` ou do `ls`? E se o pai imprimisse `WEXITSTATUS(status)` do filho, de onde viria *esse* outro número? *(Dica: o ramo do filho em `exec_demo.c` nunca chega a um `return` próprio — pense no porquê.)*
- [ ] **Q6.** Modifique `execlp("ls", "ls", "-l", NULL);` para `execlp("ls", "ls", "-l", "/caminho/que/nao/existe", NULL);` e rode de novo. O `perror("execlp")` chegou a executar? Por quê (ou por que não)? *(Dica: pense sobre o que faz o `exec` "falhar" — é o `execlp` não encontrar o comando `ls`, ou é o `ls` sozinho reclamar do argumento?)*
- [ ] **Q7.** Em uma frase, explique a diferença entre `fork()` e `exec()`: o que cada um faz com o PID e com o código do processo?

---

## Exercício 3 — Zumbis e órfãos: observando com `ps` (20–25 min)

> **Objetivo:** Provocar, de propósito, um processo zumbi e um processo órfão, e observar os dois em `ps` enquanto eles existem.

### 3.1 — Processo zumbi

Um processo **zumbi** é um filho que já terminou, mas cujo pai ainda não chamou `wait()`/`waitpid()` para "colher" o status de saída dele. Enquanto isso não acontece, o kernel mantém uma entrada mínima na tabela de processos (PID, status de saída) — é essa entrada que aparece com `STAT=Z` no `ps`.

Crie `ex3/zumbi_demo.c`:

```c
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>

int main(void) {
    pid_t pid = fork();

    if (pid == 0) {
        printf("[FILHO] PID=%d vai terminar imediatamente\n", getpid());
        return 0;
    } else {
        printf("[PAI] PID=%d NAO vou chamar wait() por 5 segundos...\n", getpid());
        printf("[PAI] rode em outro terminal: ps -o pid,ppid,stat,cmd -p %d\n", pid);
        sleep(5);
        printf("[PAI] agora terminando (o zumbi devia ter sumido apos o pai morrer)\n");
    }

    return 0;
}
```

```bash
$ gcc -Wall -o zumbi_demo zumbi_demo.c
$ ./zumbi_demo
```

Enquanto ele estiver dormindo (você tem 5 segundos), **em outro terminal**, rode:

```bash
$ ps -eo pid,ppid,stat,cmd | grep zumbi_demo
```

**Saída real capturada durante os 5 segundos de espera:**

```
27652 27289 S    ./zumbi_demo
27654 27652 Z    [zumbi_demo] <defunct>
```

Note os dois processos: o pai (`STAT=S`, dormindo) e o filho (`STAT=Z`, zumbi, marcado como `<defunct>` — o kernel já nem mantém mais o nome do programa, só o PID e o status).

**Em `ex3/respostas.txt`:**

- [ ] **Q8.** Cole a linha do `ps` que mostrou o processo com `STAT=Z`.
- [ ] **Q9.** Depois que o pai imprime `"agora terminando..."` e o processo inteiro acaba, rode `ps -eo pid,ppid,stat,cmd | grep zumbi_demo` de novo. O zumbi ainda aparece? Por quê?
- [ ] **Q10.** Um zumbi consome memória ou CPU como um processo normal (por exemplo, como um processo em loop infinito)? O que ele realmente "ocupa" na tabela de processos do kernel?

### 3.2 — Processo órfão

Um processo **órfão** é o oposto: o **pai** termina antes do filho. Quando isso acontece, o kernel "reparenta" o filho — o novo `getppid()` dele passa a ser o processo `init`/`systemd` (PID 1, ou um subreaper equivalente), que existe justamente para adotar processos órfãos e colher o status deles quando terminarem (evitando que virem zumbis permanentes).

Crie `ex3/orfao_demo.c`:

```c
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>

int main(void) {
    pid_t pid = fork();

    if (pid == 0) {
        printf("[FILHO] PID=%d, PID do pai logo apos fork=%d\n", getpid(), getppid());
        sleep(2);
        printf("[FILHO] PID=%d, PID do pai APOS pai morrer=%d\n", getpid(), getppid());
    } else {
        printf("[PAI] PID=%d vou terminar ja, sem esperar o filho\n", getpid());
    }

    return 0;
}
```

```bash
$ gcc -Wall -o orfao_demo orfao_demo.c
$ ./orfao_demo
```

**Saída real capturada (as duas linhas do filho aparecem com um intervalo de ~2s, depois que o pai já terminou):**

```
[PAI] PID=28021 vou terminar ja, sem esperar o filho
[FILHO] PID=28023, PID do pai logo apos fork=28021
[FILHO] PID=28023, PID do pai APOS pai morrer=1
```

Repare que o `PID do pai` do filho muda de `28021` (o processo `orfao_demo` original) para `1` — o filho foi adotado.

Se você rodar em outro terminal, **durante** os 2 segundos de `sleep`, verá algo como:

```bash
$ ps -eo pid,ppid,stat,cmd | grep orfao_demo
28023     1 S    ./orfao_demo
```

Note que o `PPID` já é `1`, mesmo antes do filho imprimir a segunda linha — a reparentação acontece assim que o pai original termina, não quando o filho "percebe".

**Em `ex3/respostas.txt`:**

- [ ] **Q11.** Qual foi o `PPID` do filho antes e depois do pai terminar, na sua execução?
- [ ] **Q12.** Compare 3.1 e 3.2: no zumbi, quem "sobra" (o filho, esperando ser colhido). No órfão, quem "sobra" (o filho, sem pai original). O que evita que um processo órfão vire um zumbi permanente quando ele finalmente terminar?
- [ ] **Q13.** Em uma frase: por que um zumbi é considerado um "vazamento" que o programador deveria evitar (chamando `wait`/`waitpid`), enquanto um órfão é um comportamento **normal e tratado pelo próprio sistema operacional**?

---

## Exercício 4 — Um daemon completo em C (30 min)

> **Objetivo:** Implementar o padrão *double fork* do zero e confirmar, com `ps`, que o processo final está corretamente desvinculado do terminal.

> **Nota sobre `syslog()` no ambiente de testes.** Se você estiver testando num contêiner ou ambiente sem um daemon de log ativo (sem `/dev/log`), as chamadas `syslog()` não falham nem travam — só não há onde a mensagem apareça. Numa VM normal com `systemd`/`rsyslog`, use `journalctl -t meudaemon` para ver as mensagens.

Crie `ex4/meudaemon.c`:

```c
#define _DEFAULT_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <signal.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <syslog.h>
#include <fcntl.h>
#include <string.h>
#include <errno.h>

#define PIDFILE "/tmp/meudaemon.pid"

static void escreve_pidfile(void) {
    FILE *f = fopen(PIDFILE, "w");
    if (!f) { syslog(LOG_ERR, "nao consegui abrir pidfile: %s", strerror(errno)); exit(1); }
    fprintf(f, "%d\n", getpid());
    fclose(f);
}

int main(void) {
    /* 1o fork: garante que o filho nao e' lider de grupo de processos */
    pid_t pid = fork();
    if (pid < 0) { perror("fork 1"); exit(1); }
    if (pid > 0) { exit(0); }               /* pai original termina */

    if (setsid() < 0) { exit(1); }          /* nova sessao, sem terminal */
    signal(SIGHUP, SIG_IGN);

    /* 2o fork: garante que o daemon nunca readquire um terminal de controle */
    pid = fork();
    if (pid < 0) { exit(1); }
    if (pid > 0) { exit(0); }               /* filho intermediario termina */

    /* a partir daqui, estamos no NETO -- o daemon de fato */
    umask(0027);
    chdir("/");

    close(STDIN_FILENO);
    close(STDOUT_FILENO);
    close(STDERR_FILENO);
    open("/dev/null", O_RDONLY);
    open("/dev/null", O_WRONLY);
    open("/dev/null", O_WRONLY);

    openlog("meudaemon", LOG_PID, LOG_DAEMON);
    syslog(LOG_INFO, "meudaemon iniciado, PID=%d", getpid());
    escreve_pidfile();

    int contador = 0;
    while (1) {
        syslog(LOG_INFO, "ainda vivo, iteracao=%d", contador++);
        sleep(2);
        if (contador >= 5) break;   /* soh para este teste nao rodar pra sempre */
    }

    syslog(LOG_INFO, "meudaemon encerrando");
    closelog();
    unlink(PIDFILE);
    return 0;
}
```

Compile e rode:

```bash
$ cd ex4
$ gcc -Wall -o meudaemon meudaemon.c
$ rm -f /tmp/meudaemon.pid
$ ./meudaemon
$ echo "o terminal voltou imediatamente? isso ja e um bom sinal"
```

Enquanto ele roda (tem ~10 segundos, por causa do `contador >= 5`), observe com `ps`:

```bash
$ ps -eo pid,ppid,sid,tty,stat,cmd | grep meudaemon
```

**Saída real capturada:**

```
 2905     1  2905 ?        Zs   [meudaemon] <defunct>
 2906     1  2905 ?        S    ./meudaemon
```

**Em `ex4/respostas.txt`:**

- [ ] **Q14.** Qual processo (PID) é o daemon "de fato" — o que sobrou depois dos dois `fork()`? Como você identificou isso pela coluna `CMD`?
- [ ] **Q15.** A coluna `TTY` mostra `?` para os dois processos. O que isso confirma?
- [ ] **Q16.** O processo com `STAT=Zs` é o filho intermediário do **segundo** `fork()` — ele devia ter sumido rapidamente. Numa VM com `systemd` real como `init`, ele some quase instantaneamente (o `init` recolhe órfãos automaticamente). Rode `ps` de novo depois de alguns segundos: ele ainda aparece? Isso é esperado no ambiente onde você testou?
- [ ] **Q17.** Depois que `meudaemon` termina sozinho (10s), confirme com `cat /tmp/meudaemon.pid` que o PID file foi removido. Isso acontece por causa de qual linha do código?

---

# Parte B — atividade extraclasse

Os três exercícios abaixo não cabem no encontro de laboratório, mas cobrem conteúdo cobrado na Avaliação 2. Faça-os antes do encontro seguinte e entregue junto com a Parte A.

## Exercício 5 — `fork()` + `exec()` + `wait()`: tarefas em paralelo (25 min)

> **Objetivo:** Combinar os três na prática — um processo pai que cria vários filhos, cada um executando uma tarefa diferente via `exec()`, e espera todos terminarem com `waitpid()`.

Crie `ex5/monitor_paralelo.c`:

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/wait.h>

void tarefa_monitorar_disco(void) {
    execlp("df", "df", "-h", "/", NULL);
    perror("execlp df");
    _exit(1);
}

void tarefa_monitorar_memoria(void) {
    execlp("free", "free", "-h", NULL);
    perror("execlp free");
    _exit(1);
}

int main(void) {
    const int N_TAREFAS = 2;
    pid_t filhos[N_TAREFAS];

    printf("[PAI %d] Criando %d processos filhos para tarefas paralelas...\n", getpid(), N_TAREFAS);
    fflush(stdout);

    filhos[0] = fork();
    if (filhos[0] == 0) {
        tarefa_monitorar_disco();
    }

    filhos[1] = fork();
    if (filhos[1] == 0) {
        tarefa_monitorar_memoria();
    }

    for (int i = 0; i < N_TAREFAS; i++) {
        int status;
        pid_t terminado = waitpid(filhos[i], &status, 0);
        printf("[PAI] filho PID=%d terminou (status=%d)\n", terminado, WEXITSTATUS(status));
    }

    printf("[PAI] todas as tarefas terminaram.\n");
    return 0;
}
```

```bash
$ gcc -Wall -o monitor_paralelo monitor_paralelo.c
$ ./monitor_paralelo
```

**Saída real capturada:**

```
[PAI 28395] Criando 2 processos filhos para tarefas paralelas...
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda        252G   12G   30G  29% /
               total        used        free      shared  buff/cache   available
Mem:           7.8Gi       608Mi       6.7Gi       4.8Mi       751Mi       7.2Gi
Swap:             0B          0B          0B
[PAI] filho PID=28396 terminou (status=0)
[PAI] filho PID=28397 terminou (status=0)
[PAI] todas as tarefas terminaram.
```

> **Por que `fflush(stdout)` aparece de novo aqui?** Mesma razão do Exercício 1: sem ele, a linha `[PAI ...] Criando...` poderia ficar no buffer do pai e ser duplicada quando os `fork()` seguintes copiarem a memória do processo.

### Sua tarefa: adicione uma terceira tarefa

Adicione uma função `tarefa_monitorar_uptime(void)` que executa `uptime` via `execlp`, crie um terceiro `fork()` para ela, e ajuste `N_TAREFAS` para `3`. Compile e rode.

**Em `ex5/respostas.txt`:**

- [ ] **Q18.** Cole a saída completa do seu programa com as 3 tarefas.
- [ ] **Q19.** No código, os três `fork()` são chamados um após o outro, **antes** do laço de `waitpid()`. Isso significa que as três tarefas rodam em paralelo, ou uma espera a outra terminar para começar? Justifique olhando o código (não a saída).
- [ ] **Q20.** O que aconteceria se você trocasse a ordem — chamando `waitpid(filhos[0], ...)` **logo depois** do primeiro `fork()`, antes de criar o segundo filho? As tarefas ainda rodariam em paralelo? *(Pode testar para confirmar.)*

---

## Exercício 6 — Script de controle `start`/`stop`/`status` (25–30 min)

> **Objetivo:** Escrever um script de controle baseado no PID file, e ajustar o daemon do Exercício 4 para encerrar de forma limpa ao receber `SIGTERM`.

### 6.1 — Tratando `SIGTERM` corretamente

O `meudaemon.c` do Exercício 4 só reage a `SIGTERM` da forma **padrão** do sistema (termina na hora, sem rodar o `unlink(PIDFILE)` final). Para um daemon controlado por `start`/`stop`, isso deixa um PID file "sujo" para trás. Copie `ex4/meudaemon.c` para `ex6/meudaemon.c` e adicione um tratamento de sinal:

```c
/* adicione perto do topo, antes do main: */
#include <signal.h>   /* provavelmente ja esta incluso */

static volatile sig_atomic_t deve_parar = 0;

static void trata_term(int sig) {
    (void)sig;
    deve_parar = 1;
}
```

No `main()`, registre o handler **depois** do `setsid()` (junto com o `signal(SIGHUP, SIG_IGN)`):

```c
signal(SIGTERM, trata_term);
```

E troque o laço `while (1) { ... if (contador >= 5) break; }` por um laço que roda **até receber o sinal**:

```c
while (!deve_parar) {
    syslog(LOG_INFO, "ainda vivo, iteracao=%d", contador++);
    sleep(1);
}
syslog(LOG_INFO, "meudaemon recebeu SIGTERM, encerrando");
```

Compile e instale:

```bash
$ cd ex6
$ gcc -Wall -o meudaemon meudaemon.c
$ sudo cp meudaemon /usr/local/bin/meudaemon
```

### 6.2 — O script de controle

Crie `ex6/meudaemon-ctl.sh`:

```bash
#!/bin/bash
set -euo pipefail

DAEMON_BIN="/usr/local/bin/meudaemon"
PIDFILE="/tmp/meudaemon.pid"

status() {
    if [ -f "$PIDFILE" ] && kill -0 "$(cat "$PIDFILE")" 2>/dev/null; then
        echo "meudaemon esta rodando (PID $(cat "$PIDFILE"))"
        return 0
    else
        echo "meudaemon NAO esta rodando"
        return 1
    fi
}

start() {
    if status > /dev/null 2>&1; then
        echo "ja esta rodando, nada a fazer"
        exit 0
    fi
    "$DAEMON_BIN"
    sleep 0.3
    status
}

stop() {
    if ! status > /dev/null 2>&1; then
        echo "ja estava parado"
        exit 0
    fi
    kill -TERM "$(cat "$PIDFILE")"
    echo "sinal TERM enviado"
}

case "${1:-}" in
    start)  start ;;
    stop)   stop ;;
    status) status ;;
    *) echo "Uso: $0 {start|stop|status}"; exit 1 ;;
esac
```

Teste o ciclo completo:

```bash
$ chmod +x meudaemon-ctl.sh
$ rm -f /tmp/meudaemon.pid
$ ./meudaemon-ctl.sh status
$ ./meudaemon-ctl.sh start
$ ./meudaemon-ctl.sh status
$ ./meudaemon-ctl.sh stop
$ ./meudaemon-ctl.sh status
```

**Saída real capturada (ciclo completo):**

```
meudaemon NAO esta rodando
meudaemon esta rodando (PID 7686)
meudaemon esta rodando (PID 7686)
sinal TERM enviado
meudaemon NAO esta rodando
```

E o PID file some de fato:

```bash
$ ls /tmp/meudaemon.pid
ls: cannot access '/tmp/meudaemon.pid': No such file or directory
```

**Em `ex6/respostas.txt`:**

- [ ] **Q21.** Por que a função `status()` usa `kill -0` em vez de, por exemplo, `kill -9`? O que `kill -0` faz de especial (dica: `man 2 kill`, sinal número 0)?
- [ ] **Q22.** Rode `./meudaemon-ctl.sh start` duas vezes seguidas. O que aconteceu na segunda vez? Qual parte do script evita rodar duas instâncias?
- [ ] **Q23.** O que aconteceria se você tentasse `./meudaemon-ctl.sh stop` **sem** ter adicionado o `signal(SIGTERM, trata_term)` (ou seja, usando o `meudaemon.c` original do Exercício 4)? O processo ainda terminaria? O PID file seria removido do mesmo jeito?

---

## Exercício 7 — Registrando como serviço `systemd` (20–25 min)

> **Objetivo:** Escrever uma unit `.service` do tipo `Type=forking` para o daemon, validar a sintaxe, e (numa VM com `systemd` de verdade) instalar e testar o ciclo completo.

Crie `ex7/meudaemon.service`:

```ini
[Unit]
Description=Meu daemon de exemplo (PSL221A09)
After=network.target

[Service]
Type=forking
ExecStart=/usr/local/bin/meudaemon
PIDFile=/tmp/meudaemon.pid
ExecStop=/bin/kill -TERM $MAINPID

[Install]
WantedBy=multi-user.target
```

Valide a sintaxe (funciona mesmo sem `systemd` rodando como PID 1 — é só uma checagem estática):

```bash
$ systemd-analyze verify ex7/meudaemon.service
```

**Saída esperada:** nenhuma saída (silêncio = sem erros de sintaxe).

> **Se você estiver numa VM com `systemd` de verdade, continue:**

```bash
$ sudo cp ex7/meudaemon.service /etc/systemd/system/
$ sudo systemctl daemon-reload
$ sudo systemctl start meudaemon.service
$ systemctl status meudaemon.service
$ sudo systemctl stop meudaemon.service
$ journalctl -t meudaemon --no-pager | tail -20
```

> **Importante — sobre `Type=forking` e `PIDFile=`.** Com `Type=forking`, o `systemd` espera que o processo que ele iniciou (`ExecStart`) **termine** depois de fazer o *double fork* — é exatamente o comportamento do `meudaemon` (o pai original e o intermediário terminam com `exit(0)`). O `systemd` então lê o `PIDFile=` para descobrir **qual** processo é o daemon de verdade (o neto). Se o `PIDFile` não existir no momento certo, o `systemd` reporta falha de inicialização — por isso a ordem das operações em `meudaemon.c` importa (o `escreve_pidfile()` roda **depois** do segundo `fork()`, no processo certo).

**Em `ex7/respostas.txt`:**

- [ ] **Q24.** O `systemd-analyze verify` reportou algum erro na sua unit? Se sim, o que você corrigiu?
- [ ] **Q25.** Se você tem acesso a uma VM com `systemd`: cole a saída de `systemctl status meudaemon.service` logo após o `start`. O `Active:` mostrou `active (running)`?
- [ ] **Q26.** Compare `Type=forking` (usado aqui) com `Type=simple`. Por que um programa escrito para `Type=simple` **não deveria** fazer `fork()`/`setsid()` sozinho?

---

## Entrega <!-- e critérios de avaliação -->

### O que entregar

Uma pasta `lab08-nome/` contendo:

**Parte A (entrega ao final do encontro):**

- [ ] `ex1/fork_demo.c`, `ex1/fork_demo_fixed.c`, `ex1/respostas.txt`
- [ ] `ex2/exec_demo.c`, `ex2/respostas.txt`
- [ ] `ex3/zumbi_demo.c`, `ex3/orfao_demo.c`, `ex3/respostas.txt`
- [ ] `ex4/meudaemon.c`, `ex4/respostas.txt`

**Parte B (entrega até o encontro seguinte):**

- [ ] `ex5/monitor_paralelo.c` (com a terceira tarefa adicionada), `ex5/respostas.txt`
- [ ] `ex6/meudaemon.c` (com tratamento de `SIGTERM`), `ex6/meudaemon-ctl.sh`, `ex6/respostas.txt`
- [ ] `ex7/meudaemon.service`, `ex7/respostas.txt`


<!--

### Rubrica (10 pontos)

| Critério | Pontos | O que é avaliado |
|---|---|---|
| Ex. 1 — `fork()` e buffer | 1,5 | Diferença entre terminal e saída redirecionada reproduzida e explicada (buffer de linha vs. buffer cheio); correção com `fflush()` funcional |
| Ex. 2 — `exec()` | 1,5 | Diferença fork/exec explicada corretamente; comportamento do `exec` que falha investigado |
| Ex. 3 — Zumbis e órfãos | 1,5 | Estados `Z` e reparentação para PID 1 observados via `ps`; diferença entre os dois casos bem explicada |
| Ex. 4 — Daemon (*double fork*) | 2,5 | Padrão implementado corretamente; `ps` confirma `TTY=?` e `PPID=1`; PID file criado e removido |
| Ex. 5 — fork+exec+wait | 1,0 | Terceira tarefa adicionada e funcionando; entendimento de paralelismo vs. sequencial demonstrado |
| Ex. 6 — Script de controle | 1,5 | `start`/`stop`/`status` funcionam; `SIGTERM` tratado de forma limpa; PID file consistente com o estado real |
| Ex. 7 — `systemd` | 0,5 | Unit válida (`systemd-analyze verify` sem erros); entendimento de `Type=forking` + `PIDFile=` demonstrado |

> **Dica 1:** Nos Exercícios 1 e 5, o `fflush(stdout)` é pequeno no código, mas é exatamente o tipo de detalhe que a rubrica valoriza — mostra que você entendeu **por que** ele é necessário, não só que ele "resolve".

> **Dica 2:** No Exercício 4, "meu daemon roda" não é suficiente — o professor vai olhar a saída do `ps` (colunas `TTY`, `PPID`, `SID`) para confirmar que ele realmente se desvinculou do terminal.

---

## Anexo A — Referência rápida: processos

```c
#include <unistd.h>     // fork(), exec*(), getpid(), getppid()
#include <sys/types.h>  // pid_t
#include <sys/wait.h>   // wait(), waitpid()

pid_t fork(void);
// retorno < 0  -> erro
// retorno == 0 -> voce esta no processo FILHO
// retorno > 0  -> voce esta no processo PAI; retorno = PID do filho

int execlp(const char *arquivo, const char *arg0, ..., NULL);
// substitui a imagem do processo atual pelo programa "arquivo"
// so retorna se der ERRO (senao, o processo "virou" outro programa)

pid_t waitpid(pid_t pid, int *status, int options);
// espera o filho "pid" terminar e coleta seu status de saida
// WEXITSTATUS(status) extrai o codigo de saida (0-255)
```

```bash
# observando processos
ps -eo pid,ppid,stat,cmd      # todos os processos, com PID/PPID/estado/comando
ps -eo pid,ppid,stat,cmd | grep NOME
ps -eo pid,ppid,sid,tty,stat,cmd | grep NOME   # inclui sessao e terminal
pstree -p                     # arvore de processos com PIDs

# codigos de STAT mais comuns
# R = rodando/pronto para rodar     S = dormindo (esperando algo)
# Z = zumbi (terminou, nao colhido) T = parado (ex: Ctrl+Z)
# sufixo s = lider de sessao        sufixo + = em primeiro plano
```

## Anexo B — Referência rápida: daemons

```c
#include <unistd.h>    // fork(), setsid(), chdir(), close()
#include <sys/stat.h>  // umask()
#include <syslog.h>    // openlog(), syslog(), closelog()

pid_t setsid(void);
// cria nova sessao, processo chamador vira lider (sem terminal de controle)
// falha se o chamador ja for lider de grupo de processos

void syslog(int prioridade, const char *formato, ...);
// LOG_ERR, LOG_WARNING, LOG_INFO, LOG_DEBUG
```

```bash
# controlando um daemon pelo PID file
kill -0 PID         # nao manda sinal nenhum -- so testa se o processo existe
kill -TERM PID      # pede para terminar (pode ser tratado/ignorado)
kill -9 PID         # forca o encerramento (NAO pode ser tratado)

# systemd
systemd-analyze verify arquivo.service   # valida sintaxe sem precisar rodar
journalctl -t NOME_DO_PROGRAMA           # log filtrado por "tag" do syslog
```

## Anexo C — Tabela-resumo: quem é o quê

| Situação | O que acontece | Como observar |
|---|---|---|
| `fork()` bem-sucedido | Processo é duplicado; PID novo para o filho | `getpid()`/`getppid()` diferentes em cada ramo do `if` |
| `exec*()` bem-sucedido | Imagem do processo é substituída; PID **não muda** | Nada depois do `exec*()` no código original executa |
| `exec*()` falhou | Processo continua sendo o programa original | Código depois do `exec*()` executa; `perror()` mostra o motivo |
| Filho termina, pai não colheu | **Zumbi** — entrada mínima na tabela de processos | `ps` mostra `STAT=Z` e `<defunct>` |
| Pai termina antes do filho | **Órfão** — filho reparentado para PID 1 (init/systemd) | `getppid()` do filho passa a valer `1` |
| Pai chama `waitpid()` | Zumbi é removido da tabela de processos | `ps` não mostra mais o PID do filho |
| *Double fork* + `setsid()` | Processo vira daemon, sem terminal de controle | `ps` mostra `TTY=?` e `PPID=1` |

## Anexo D — `meudaemon.service` (referência)

```ini
[Unit]
Description=Meu daemon de exemplo (PSL221A09)
After=network.target

[Service]
Type=forking
ExecStart=/usr/local/bin/meudaemon
PIDFile=/tmp/meudaemon.pid
ExecStop=/bin/kill -TERM $MAINPID

[Install]
WantedBy=multi-user.target
```

**Testado e validado em sandbox (2026-10-07):** todos os programas em C deste guia (`fork_demo.c`, `fork_demo_fixed.c`, `exec_demo.c`, `zumbi_demo.c`, `orfao_demo.c`, `meudaemon.c` e `monitor_paralelo.c`, incluindo a variante com 3 tarefas e a variante com `SIGTERM` do Exercício 6) foram compilados com `gcc -Wall` sem warnings e executados; as saídas mostradas são reais, não simuladas. O Exercício 1 foi verificado nos **dois** modos de bufferização — rodando num terminal de verdade (via pseudo-terminal) e com a saída redirecionada —, confirmando que a duplicação da linha só ocorre no segundo caso. O daemon do Exercício 4 foi observado em `ps` com `TTY=?`, `PPID=1` e PID file criado e removido; o ciclo `status`/`start`/`start`/`stop`/`status` do script de controle produz exatamente a saída do Exercício 6, incluindo a remoção do PID file após o `SIGTERM`; e `systemd-analyze verify` não reporta erros na unit do Exercício 7.

-->