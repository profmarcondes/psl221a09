# Laboratório 11 <br> Sinais e Gerenciamento de Tempo

<!-- **PSL221A09 — Programação em Sistemas Linux**

| | |
|---|---|
| **Duração estimada** | 2h — Parte A (presencial, ~90 min) + Parte B (extraclasse) |
| **Formato** | Individual, em dupla ou trio |
| **Pré-requisito** | GCC, `man` local; todos os cenários usam `fork()` internamente (pai e filho no mesmo programa) — nenhum terminal extra é necessário |
| **Entregável** | Pasta `working_dir/lab11-nome/` com os exercícios da Parte A; Parte B entregue até o encontro seguinte |

---
-->

Este guia percorre os dois temas da Aula 12: **sinais** (capturar com `sigaction()`,
bloquear com `sigprocmask()`, reagir a filhos com `SIGCHLD`) e **gerenciamento de
tempo** (`alarm()`, `setitimer()`, `timerfd_create()`). Os dois temas se encontram
no fim: um temporizador pode ser entregue como sinal **ou** como evento de E/S —
e essa escolha é a ponte para a Aula 13 (E/S multiplexada).

**Como usar este guia.** A **Parte A** (Exercícios 1 a 3) é feita durante o
encontro de laboratório, com o professor disponível. A **Parte B** (Exercícios 4
a 6) é atividade extraclasse: mesmo formato e mesmo nível de detalhe, para
consolidar sozinho o que não coube no tempo de aula. Sempre que houver uma seção
de **saída esperada**, compare com o que apareceu no seu terminal antes de
seguir em frente.

> **Atenção com os tempos de espera.** Vários exercícios deste guia usam `sleep()`
> para criar uma janela de tempo em que um sinal ou temporizador dispara — é
> esperado que o terminal fique "parado" por 1 a 5 segundos em alguns comandos.
> Isso não é travamento (exceto quando o próprio guia disser o contrário, como
> no Exercício 1). Os tempos impressos na sua máquina podem variar alguns
> milissegundos em relação à saída capturada aqui; o que importa é a ordem dos
> eventos e a ordem de grandeza dos valores.

## Preparação do ambiente

```bash
$ mkdir -p working_dir/lab11-nome/{ex1,ex2,ex3,ex4,ex5,ex6}
$ cd working_dir/lab11-nome
```

Substitua `nome` pelo seu nome (ou pelos nomes da dupla/trio). Cada exercício tem
sua própria pasta (`ex1/` a `ex6/`) e seu próprio `respostas.txt`.

Todos os programas deste laboratório usam apenas cabeçalhos do sistema (sem
`-lrt`, diferente do Laboratório 10):

```bash
$ gcc -Wall -Wextra -o programa programa.c
```

---

# Parte A — presencial (~90 min)

## Exercício 1 — Capturando `SIGINT` com `sigaction()` (25 min)

> **Objetivo:** Comparar a ação default de `SIGINT` (Ctrl+C) com um sinal
> tratado por um *handler*, e entender as regras de segurança de um *handler*.

### Etapa 1 — Ação default

Crie `ex1/sigint_default.c`:

```c
#include <stdio.h>
#include <unistd.h>

int main(void) {
    printf("[PID %d] rodando... pressione Ctrl+C\n", getpid());
    fflush(stdout);
    for (int i = 0; i < 60; i++) {
        printf("tick %d\n", i);
        fflush(stdout);
        sleep(1);
    }
    printf("fim\n");
    return 0;
}
```

```bash
$ cd ex1
$ gcc -Wall -Wextra -o sigint_default sigint_default.c
$ ./sigint_default
```

Deixe rodar por ~2 segundos e pressione **Ctrl+C**. Depois confira o código de
saída com `echo $?`.

**Saída real capturada** (sinal enviado via `kill -INT`, equivalente a Ctrl+C):

```
[PID 232] rodando... pressione Ctrl+C
tick 0
tick 1
$ echo $?
130
```

**Em `ex1/respostas.txt`:**

- [ ] **Q1.** O código de saída foi `130`. Sabendo que `SIGINT` é o sinal número
  `2`, que relação existe entre `130` e `2`? Essa é a convenção que o shell usa
  para todo processo terminado por sinal.

### Etapa 2 — Tratando o sinal

Crie `ex1/sigint_sigaction.c`:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <signal.h>
#include <unistd.h>

static volatile sig_atomic_t pedidos_de_saida = 0;

static void handler_sigint(int signo) {
    (void)signo;
    pedidos_de_saida++;
    const char msg[] = "\n[handler] SIGINT recebido, terminando com calma...\n";
    write(STDOUT_FILENO, msg, sizeof(msg) - 1);
}

int main(void) {
    struct sigaction sa;
    memset(&sa, 0, sizeof(sa));
    sa.sa_handler = handler_sigint;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = 0;

    if (sigaction(SIGINT, &sa, NULL) < 0) {
        perror("sigaction");
        return 1;
    }

    printf("[PID %d] rodando com SIGINT tratado... pressione Ctrl+C\n", getpid());
    fflush(stdout);

    int i = 0;
    while (!pedidos_de_saida) {
        printf("tick %d\n", i++);
        fflush(stdout);
        sleep(1);
    }

    printf("saindo de forma controlada (pedidos_de_saida=%d)\n", pedidos_de_saida);
    return 0;
}
```

```bash
$ gcc -Wall -Wextra -o sigint_sigaction sigint_sigaction.c
$ ./sigint_sigaction
```

Deixe rodar por ~3 segundos e pressione Ctrl+C.

**Saída real capturada:**

```
[PID 234] rodando com SIGINT tratado... pressione Ctrl+C
tick 0
tick 1
tick 2

[handler] SIGINT recebido, terminando com calma...
saindo de forma controlada (pedidos_de_saida=1)
$ echo $?
0
```

**Em `ex1/respostas.txt`:**

- [ ] **Q2.** Desta vez o código de saída foi `0`, não `130`. O que o programa
  fez de diferente entre receber o `SIGINT` e efetivamente terminar?
- [ ] **Q3.** A variável `pedidos_de_saida` é declarada `volatile sig_atomic_t`
  em vez de um `int` comum. Explique o papel de cada uma das duas palavras
  (`volatile` e `sig_atomic_t`) separadamente.
- [ ] **Q4.** O *handler* chama `write(STDOUT_FILENO, ...)` em vez de
  `printf(...)`. Por que `printf()` é arriscado dentro de um *handler* de
  sinal, mesmo que "funcione na maioria das vezes" em teste?

---

## Exercício 2 — `EINTR` e `SA_RESTART` (20 min)

> **Objetivo:** Observar uma chamada de sistema bloqueante sendo interrompida
> por um sinal, e comparar com o comportamento quando `SA_RESTART` está ativo.

Crie `ex2/eintr_demo.c`:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <signal.h>
#include <unistd.h>
#include <errno.h>
#include <sys/types.h>

static volatile sig_atomic_t sinais_recebidos = 0;

static void handler(int signo) {
    (void)signo;
    sinais_recebidos++;
}

int main(void) {
    int fd[2];
    if (pipe(fd) < 0) { perror("pipe"); return 1; }

    pid_t filho = fork();
    if (filho < 0) { perror("fork"); return 1; }

    if (filho == 0) {
        close(fd[0]);
        sleep(1);
        kill(getppid(), SIGUSR1);
        sleep(1);
        write(fd[1], "dados-finais", 12);
        close(fd[1]);
        _exit(0);
    }

    close(fd[1]);

    struct sigaction sa;
    memset(&sa, 0, sizeof(sa));
    sa.sa_handler = handler;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = 0; /* SEM SA_RESTART: read() sera interrompida */
    sigaction(SIGUSR1, &sa, NULL);

    char buf[64];
    printf("[pai] bloqueado em read(), aguardando dados ou sinal...\n");
    fflush(stdout);

    ssize_t n = read(fd[0], buf, sizeof(buf) - 1);
    if (n < 0) {
        printf("[pai] read() retornou -1, errno=%d (%s) -- interrompida por sinal!\n",
               errno, strerror(errno));
        printf("[pai] sinais_recebidos=%d -- tentando read() de novo...\n", sinais_recebidos);
        fflush(stdout);
        n = read(fd[0], buf, sizeof(buf) - 1); /* retry manual */
    }

    if (n > 0) {
        buf[n] = '\0';
        printf("[pai] leu %zd bytes: \"%s\"\n", n, buf);
    }

    close(fd[0]);
    return 0;
}
```

```bash
$ cd ex2
$ gcc -Wall -Wextra -o eintr_demo eintr_demo.c
$ ./eintr_demo
```

**Saída real capturada:**

```
[pai] bloqueado em read(), aguardando dados ou sinal...
[pai] read() retornou -1, errno=4 (Interrupted system call) -- interrompida por sinal!
[pai] sinais_recebidos=1 -- tentando read() de novo...
[pai] leu 12 bytes: "dados-finais"
```

**Em `ex2/respostas.txt`:**

- [ ] **Q5.** `errno` veio `4`, `"Interrupted system call"` (`EINTR`). Em que
  momento exatamente o filho mandou o `SIGUSR1` em relação ao `read()` do pai?

Agora crie `ex2/restart_demo.c` — o **mesmo cenário** (mesmo filho, mesmo
`SIGUSR1` no meio da espera), mas com `sa.sa_flags = SA_RESTART;` e uma única
chamada a `read()`, sem tratamento de `EINTR`:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <signal.h>
#include <unistd.h>
#include <errno.h>
#include <sys/types.h>

static volatile sig_atomic_t sinais_recebidos = 0;

static void handler(int signo) {
    (void)signo;
    sinais_recebidos++;
}

int main(void) {
    int fd[2];
    if (pipe(fd) < 0) { perror("pipe"); return 1; }

    pid_t filho = fork();
    if (filho < 0) { perror("fork"); return 1; }

    if (filho == 0) {
        close(fd[0]);
        sleep(1);
        kill(getppid(), SIGUSR1);
        sleep(1);
        write(fd[1], "dados-finais", 12);
        close(fd[1]);
        _exit(0);
    }

    close(fd[1]);

    struct sigaction sa;
    memset(&sa, 0, sizeof(sa));
    sa.sa_handler = handler;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = SA_RESTART; /* read() e reiniciada automaticamente */
    sigaction(SIGUSR1, &sa, NULL);

    char buf[64];
    printf("[pai] bloqueado em read() com SA_RESTART, aguardando dados...\n");
    fflush(stdout);

    ssize_t n = read(fd[0], buf, sizeof(buf) - 1); /* uma unica chamada */

    if (n > 0) {
        buf[n] = '\0';
        printf("[pai] leu %zd bytes: \"%s\" (sem nunca ver EINTR)\n", n, buf);
    } else {
        printf("[pai] read() retornou %zd, errno=%d\n", n, errno);
    }
    printf("[pai] sinais_recebidos durante a espera: %d\n", sinais_recebidos);

    close(fd[0]);
    return 0;
}
```

```bash
$ gcc -Wall -Wextra -o restart_demo restart_demo.c
$ ./restart_demo
```

**Saída real capturada:**

```
[pai] bloqueado em read() com SA_RESTART, aguardando dados...
[pai] leu 12 bytes: "dados-finais" (sem nunca ver EINTR)
[pai] sinais_recebidos durante a espera: 1
```

**Em `ex2/respostas.txt`:**

- [ ] **Q6.** A `read()` de `restart_demo.c` nunca retornou `-1`, mesmo com o
  mesmo `SIGUSR1` chegando no meio da espera. O que `SA_RESTART` faz o kernel
  fazer de diferente?
- [ ] **Q7.** O `eintr_demo.c` precisou de um `if (n < 0)` com uma segunda
  chamada manual a `read()`. O `restart_demo.c` não tem esse tratamento. Isso é
  um descuido ou é seguro? Por quê?
- [ ] **Q8.** Dê um exemplo de situação em que você **não** gostaria de usar
  `SA_RESTART` — ou seja, em que a interrupção da chamada de sistema pelo sinal
  é justamente o comportamento desejado. *(Dica: pense no Exercício 4.)*

---

## Exercício 3 — Bloqueio de sinais e `SIGCHLD` não bloqueante (30–35 min)

> **Objetivo:** Usar `sigprocmask()`/`sigpending()` para proteger uma seção
> crítica, e tratar `SIGCHLD` de forma correta mesmo quando vários filhos
> terminam quase ao mesmo tempo.

### 3.1 — Bloqueando e verificando sinais pendentes

Crie `ex3/sigprocmask_demo.c`:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <signal.h>
#include <unistd.h>
#include <sys/types.h>

static volatile sig_atomic_t recebidos = 0;

static void handler(int signo) {
    (void)signo;
    recebidos++;
    const char msg[] = "[handler] SIGUSR1 entregue!\n";
    write(STDOUT_FILENO, msg, sizeof(msg) - 1);
}

int main(void) {
    struct sigaction sa;
    memset(&sa, 0, sizeof(sa));
    sa.sa_handler = handler;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = 0;
    sigaction(SIGUSR1, &sa, NULL);

    pid_t filho = fork();
    if (filho < 0) { perror("fork"); return 1; }

    if (filho == 0) {
        sleep(1);
        kill(getppid(), SIGUSR1);
        _exit(0);
    }

    sigset_t bloqueado, anterior;
    sigemptyset(&bloqueado);
    sigaddset(&bloqueado, SIGUSR1);
    sigprocmask(SIG_BLOCK, &bloqueado, &anterior);

    printf("[pai] SIGUSR1 bloqueado; entrando na secao critica (2s)...\n");
    fflush(stdout);
    sleep(2); /* o filho manda o sinal durante este sleep */

    sigset_t pendentes;
    sigpending(&pendentes);
    int esta_pendente = sigismember(&pendentes, SIGUSR1);
    printf("[pai] saindo da secao critica. recebidos=%d, SIGUSR1 pendente? %s\n",
           recebidos, esta_pendente ? "SIM" : "nao");
    fflush(stdout);

    printf("[pai] desbloqueando SIGUSR1...\n");
    fflush(stdout);
    sigprocmask(SIG_SETMASK, &anterior, NULL);

    printf("[pai] depois de desbloquear: recebidos=%d\n", recebidos);
    return 0;
}
```

```bash
$ cd ex3
$ gcc -Wall -Wextra -o sigprocmask_demo sigprocmask_demo.c
$ ./sigprocmask_demo
```

**Saída real capturada:**

```
[pai] SIGUSR1 bloqueado; entrando na secao critica (2s)...
[pai] saindo da secao critica. recebidos=0, SIGUSR1 pendente? SIM
[pai] desbloqueando SIGUSR1...
[handler] SIGUSR1 entregue!
[pai] depois de desbloquear: recebidos=1
```

**Em `ex3/respostas.txt`:**

- [ ] **Q9.** O filho mandou o `SIGUSR1` durante a seção crítica (enquanto o pai
  estava com o sinal bloqueado), mas `recebidos` continuou `0` até depois do
  `sigprocmask(SIG_SETMASK, ...)`. Isso significa que o sinal foi perdido?
  Justifique com base no que `sigpending()` mostrou.
- [ ] **Q10.** A mensagem `"[handler] SIGUSR1 entregue!"` apareceu **antes** da
  linha `"[pai] depois de desbloquear..."`, mesmo essa última vindo depois no
  código-fonte. Por quê?
- [ ] **Q11.** Por que bloquear sinais durante uma seção crítica é útil? Dê um
  exemplo de um tipo de dado (estrutura com mais de um campo relacionado, por
  exemplo) que ficaria inconsistente se um *handler* rodasse no meio de uma
  atualização.

### 3.2 — `SIGCHLD` sem perder nenhum filho

Crie `ex3/sigchld_demo.c`:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <signal.h>
#include <unistd.h>
#include <sys/wait.h>
#include <sys/types.h>

static volatile sig_atomic_t total_colhidos = 0;

static void handler_sigchld(int signo) {
    (void)signo;
    int status;
    pid_t pid;
    while ((pid = waitpid(-1, &status, WNOHANG)) > 0) {
        total_colhidos++;
        char msg[96];
        int len;
        if (WIFEXITED(status)) {
            len = snprintf(msg, sizeof(msg),
                "[handler] filho %d terminou, exit=%d\n", pid, WEXITSTATUS(status));
        } else {
            len = snprintf(msg, sizeof(msg),
                "[handler] filho %d terminou de forma anormal\n", pid);
        }
        write(STDOUT_FILENO, msg, (size_t)len);
    }
}

int main(void) {
    struct sigaction sa;
    memset(&sa, 0, sizeof(sa));
    sa.sa_handler = handler_sigchld;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = SA_RESTART;
    sigaction(SIGCHLD, &sa, NULL);

    int tempos[3] = {1, 1, 2}; /* dois filhos terminam juntos em 1s */
    int codigos[3] = {10, 20, 30};

    for (int i = 0; i < 3; i++) {
        pid_t filho = fork();
        if (filho < 0) { perror("fork"); return 1; }
        if (filho == 0) {
            sleep(tempos[i]);
            _exit(codigos[i]);
        }
    }

    printf("[pai] 3 filhos criados; fazendo outro trabalho util enquanto isso...\n");
    fflush(stdout);
    for (int i = 0; i < 3; i++) {
        printf("[pai] trabalho util, passo %d (total_colhidos=%d)\n", i, total_colhidos);
        fflush(stdout);
        sleep(1);
    }

    printf("[pai] fim do trabalho util. total_colhidos=%d\n", total_colhidos);
    return 0;
}
```

```bash
$ gcc -Wall -Wextra -o sigchld_demo sigchld_demo.c
$ ./sigchld_demo
```

**Saída real capturada** (os filhos 1 e 2 terminam depois de 1s, o filho 3
depois de 2s):

```
[pai] 3 filhos criados; fazendo outro trabalho util enquanto isso...
[pai] trabalho util, passo 0 (total_colhidos=0)
[handler] filho 260 terminou, exit=10
[handler] filho 261 terminou, exit=20
[pai] trabalho util, passo 1 (total_colhidos=2)
[handler] filho 262 terminou, exit=30
[pai] trabalho util, passo 2 (total_colhidos=3)
[pai] fim do trabalho util. total_colhidos=3
```

**Em `ex3/respostas.txt`:**

- [ ] **Q12.** Os dois primeiros filhos terminaram quase ao mesmo tempo (ambos
  dormiam 1s), e o *handler* colheu os dois mesmo assim. Agora faça o teste
  contrário: copie o programa para `ex3/sigchld_if.c`, troque o `while` do
  *handler* por `if` (uma única chamada a `waitpid()` por sinal recebido),
  recompile e rode **5 vezes**, anotando o `total_colhidos` final de cada
  execução. Ele foi sempre `3`? O que isso revela sobre quantos `SIGCHLD` o
  processo recebe quando dois filhos terminam juntos — e por que o `while` é
  obrigatório?

```bash
$ cp sigchld_demo.c sigchld_if.c
$ nano sigchld_if.c     # while ((pid = waitpid(...)) > 0)  ->  if ((pid = waitpid(...)) > 0)
$ gcc -Wall -Wextra -o sigchld_if sigchld_if.c
$ for i in 1 2 3 4 5; do ./sigchld_if | tail -1; done
```

**Saída real capturada** (varia de execução para execução — é justamente o
ponto):

```
[pai] fim do trabalho util. total_colhidos=3
[pai] fim do trabalho util. total_colhidos=2
[pai] fim do trabalho util. total_colhidos=3
[pai] fim do trabalho util. total_colhidos=2
[pai] fim do trabalho util. total_colhidos=2
```
- [ ] **Q13.** Relacione com o Laboratório 08 (Aula 9): lá, zumbis apareciam
  quando ninguém chamava `wait()`/`waitpid()` depois que um filho terminava. O
  que o padrão `SIGCHLD` + `waitpid(WNOHANG)` em loop resolve, em comparação
  com simplesmente chamar `wait()` uma vez logo após os `fork()`?

---

# Parte B — atividade extraclasse

Os três exercícios abaixo não cabem no encontro de laboratório, mas cobrem
conteúdo cobrado na Avaliação 2. Faça-os antes do encontro seguinte e entregue
junto com a Parte A.

## Exercício 4 — Timeout com `alarm()` (20 min)

> **Objetivo:** Usar `alarm()`/`SIGALRM` para limitar o tempo de espera de uma
> operação bloqueante que não tem timeout próprio.

Crie `ex4/alarm_timeout.c`:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <signal.h>
#include <unistd.h>
#include <errno.h>

static volatile sig_atomic_t tempo_esgotado = 0;

static void handler_alarm(int signo) {
    (void)signo;
    tempo_esgotado = 1;
}

int main(void) {
    int fd[2];
    if (pipe(fd) < 0) { perror("pipe"); return 1; }

    pid_t filho = fork();
    if (filho < 0) { perror("fork"); return 1; }

    if (filho == 0) {
        /* filho "lento": demora 5s para escrever -- mais que o timeout de 2s */
        close(fd[0]);
        sleep(5);
        write(fd[1], "resposta-tardia", 15);
        close(fd[1]);
        _exit(0);
    }

    close(fd[1]);

    struct sigaction sa;
    memset(&sa, 0, sizeof(sa));
    sa.sa_handler = handler_alarm;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = 0; /* sem SA_RESTART: queremos que a read() seja interrompida */
    sigaction(SIGALRM, &sa, NULL);

    printf("[pai] aguardando resposta do filho com timeout de 2s...\n");
    fflush(stdout);
    alarm(2);

    char buf[64];
    ssize_t n = read(fd[0], buf, sizeof(buf) - 1);
    alarm(0); /* cancela o alarme pendente (se a read() tiver retornado antes) */

    if (n < 0 && tempo_esgotado) {
        printf("[pai] TIMEOUT: nenhuma resposta em 2s (errno=%d, %s)\n", errno, strerror(errno));
    } else if (n > 0) {
        buf[n] = '\0';
        printf("[pai] resposta recebida a tempo: \"%s\"\n", buf);
    }

    close(fd[0]);
    kill(filho, SIGKILL); /* nao precisamos mais esperar o filho lento */
    return 0;
}
```

```bash
$ cd ex4
$ gcc -Wall -Wextra -o alarm_timeout alarm_timeout.c
$ time ./alarm_timeout
```

**Saída real capturada:**

```
[pai] aguardando resposta do filho com timeout de 2s...
[pai] TIMEOUT: nenhuma resposta em 2s (errno=4, Interrupted system call)

real    0m2.002s
```

**Em `ex4/respostas.txt`:**

- [ ] **Q14.** O `time` mostrou que o programa levou ~2,0s, não os 5s que o
  filho esperaria antes de responder. Explique com suas palavras como o
  `alarm(2)` fez o `read()` desistir mais cedo.
- [ ] **Q15.** Esse exercício **não** usa `SA_RESTART` no `sigaction()` do
  `SIGALRM`. O que aconteceria com o timeout se `SA_RESTART` estivesse ativo?
- [ ] **Q16.** Mude `sleep(5)` do filho para `sleep(1)` (mais rápido que o
  timeout de 2s), recompile e rode de novo. Cole a saída obtida e explique por
  que o ramo do `TIMEOUT` não foi executado desta vez.

---

## Exercício 5 — `setitimer()` periódico x `sleep()` em loop (25 min)

> **Objetivo:** Medir, na prática, o *drift* (atraso acumulado) de um loop
> ingênuo com `sleep()` e comparar com um temporizador periódico de verdade.

Crie `ex5/sleep_loop_drift.c`:

```c
#include <stdio.h>
#include <time.h>
#include <unistd.h>

static double ts_to_s(struct timespec *t) {
    return (double)t->tv_sec + (double)t->tv_nsec / 1e9;
}

int main(void) {
    struct timespec inicio, fim;
    clock_gettime(CLOCK_MONOTONIC, &inicio);

    const int N = 10;
    for (int i = 0; i < N; i++) {
        usleep(50000); /* 50ms de "trabalho" por iteracao */
        sleep(1);      /* so comeca a contar depois do trabalho acima */
    }

    clock_gettime(CLOCK_MONOTONIC, &fim);
    double total = ts_to_s(&fim) - ts_to_s(&inicio);
    double esperado = (double)N * 1.0;

    printf("sleep-loop: %d iteracoes, esperado=%.2fs, real=%.3fs, drift=%.3fs\n",
           N, esperado, total, total - esperado);
    return 0;
}
```

E `ex5/setitimer_periodic.c`:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <signal.h>
#include <sys/time.h>
#include <time.h>
#include <unistd.h>

static volatile sig_atomic_t ticks = 0;

static void handler_alarm(int signo) {
    (void)signo;
    ticks++;
}

static double ts_to_s(struct timespec *t) {
    return (double)t->tv_sec + (double)t->tv_nsec / 1e9;
}

int main(void) {
    struct sigaction sa;
    memset(&sa, 0, sizeof(sa));
    sa.sa_handler = handler_alarm;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = SA_RESTART;
    sigaction(SIGALRM, &sa, NULL);

    struct itimerval it;
    it.it_value.tv_sec = 1;
    it.it_value.tv_usec = 0;
    it.it_interval.tv_sec = 1;
    it.it_interval.tv_usec = 0;

    struct timespec inicio, fim;
    clock_gettime(CLOCK_MONOTONIC, &inicio);
    setitimer(ITIMER_REAL, &it, NULL);

    const int N = 10;
    int alvo = 0;
    while (alvo < N) {
        pause(); /* dorme ate QUALQUER sinal chegar */
        while (ticks > alvo) {
            alvo++;
            usleep(50000); /* mesmos 50ms de "trabalho" por tick */
        }
    }

    clock_gettime(CLOCK_MONOTONIC, &fim);
    double total = ts_to_s(&fim) - ts_to_s(&inicio);
    double esperado = (double)N * 1.0;

    printf("setitimer: %d ticks, esperado=%.2fs, real=%.3fs, drift=%.3fs\n",
           N, esperado, total, total - esperado);
    return 0;
}
```

```bash
$ cd ex5
$ gcc -Wall -Wextra -o sleep_loop_drift sleep_loop_drift.c
$ gcc -Wall -Wextra -o setitimer_periodic setitimer_periodic.c
$ ./sleep_loop_drift
$ ./setitimer_periodic
```

**Saída real capturada:**

```
sleep-loop: 10 iteracoes, esperado=10.00s, real=10.503s, drift=0.503s
setitimer: 10 ticks, esperado=10.00s, real=10.050s, drift=0.050s
```

**Em `ex5/respostas.txt`:**

- [ ] **Q17.** O *drift* do `sleep_loop_drift.c` foi de ~0,5s em 10 iterações —
  bem próximo de `10 × 50ms`. Explique por que o `sleep(1)` não compensa o
  tempo gasto no `usleep(50000)` anterior.
- [ ] **Q18.** O `setitimer_periodic.c` faz o **mesmo** trabalho de 50ms por
  tick, mas terminou com um *drift* dez vezes menor. O que o `setitimer()`
  agenda de diferente em relação ao `sleep()` em loop?
- [ ] **Q19.** Imagine um daemon de monitoramento que deveria coletar uma
  métrica a cada 1 minuto, rodando por 30 dias contínuos. Se ele usasse a
  abordagem ingênua de `sleep_loop_drift.c` com um processamento de ~200ms por
  coleta, aproximadamente quantos minutos de *drift* ele acumularia ao fim de
  30 dias? Mostre a conta.
- [ ] **Q20.** O loop principal de `setitimer_periodic.c` usa `pause()` para
  esperar o próximo sinal. O que aconteceria com o uso de CPU do programa se,
  em vez de `pause()`, ele usasse um laço `while (ticks <= alvo) { }` vazio
  (*busy-waiting*) para esperar o próximo tick?

---
<!-- 
## Exercício 6 — `timerfd_create()` e escolhendo a ferramenta certa (20–25 min)

> **Objetivo:** Usar um temporizador como descritor de arquivo, integrado com
> `select()`, e fechar o laboratório decidindo qual mecanismo de tempo usar em
> cenários diferentes.

Crie `ex6/timerfd_select.c`:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/timerfd.h>
#include <sys/select.h>
#include <sys/types.h>
#include <stdint.h>

int main(void) {
    int tfd = timerfd_create(CLOCK_MONOTONIC, 0);
    if (tfd < 0) { perror("timerfd_create"); return 1; }

    struct itimerspec its;
    its.it_value.tv_sec = 1;
    its.it_value.tv_nsec = 0;
    its.it_interval.tv_sec = 1;
    its.it_interval.tv_nsec = 0;
    if (timerfd_settime(tfd, 0, &its, NULL) < 0) { perror("timerfd_settime"); return 1; }

    int fd[2];
    if (pipe(fd) < 0) { perror("pipe"); return 1; }

    pid_t filho = fork();
    if (filho < 0) { perror("fork"); return 1; }
    if (filho == 0) {
        close(fd[0]);
        sleep(3); /* a mensagem chega entre dois ticks do timer */
        write(fd[1], "mensagem-assincrona", 19);
        close(fd[1]);
        _exit(0);
    }
    close(fd[1]);

    int ticks = 0;
    int mensagem_recebida = 0;
    while (!mensagem_recebida && ticks < 6) {
        fd_set rfds;
        FD_ZERO(&rfds);
        FD_SET(tfd, &rfds);
        FD_SET(fd[0], &rfds);
        int maxfd = (tfd > fd[0]) ? tfd : fd[0];

        int pronto = select(maxfd + 1, &rfds, NULL, NULL, NULL);
        if (pronto < 0) { perror("select"); break; }

        if (FD_ISSET(tfd, &rfds)) {
            uint64_t expiracoes;
            read(tfd, &expiracoes, sizeof(expiracoes));
            ticks++;
            printf("[select] tick do timerfd (expiracoes=%lu, total ticks=%d)\n",
                   (unsigned long)expiracoes, ticks);
            fflush(stdout);
        }
        if (FD_ISSET(fd[0], &rfds)) {
            char buf[64];
            ssize_t n = read(fd[0], buf, sizeof(buf) - 1);
            if (n > 0) {
                buf[n] = '\0';
                printf("[select] pipe pronto: \"%s\"\n", buf);
                mensagem_recebida = 1;
            }
        }
    }

    close(tfd);
    close(fd[0]);
    printf("fim: %d ticks observados, mensagem_recebida=%d\n", ticks, mensagem_recebida);
    return 0;
}
```

```bash
$ cd ex6
$ gcc -Wall -Wextra -o timerfd_select timerfd_select.c
$ ./timerfd_select
```

**Saída real capturada:**

```
[select] tick do timerfd (expiracoes=1, total ticks=1)
[select] tick do timerfd (expiracoes=1, total ticks=2)
[select] tick do timerfd (expiracoes=1, total ticks=3)
[select] pipe pronto: "mensagem-assincrona"
fim: 3 ticks observados, mensagem_recebida=1
```

**Em `ex6/respostas.txt`:**

- [ ] **Q21.** O filho demora 3s para escrever no pipe, e o timer dispara a
  cada 1s. O programa observou exatamente 3 ticks antes da mensagem chegar.
  Essa conta bate com os dois parâmetros (3s do filho, 1s do timer)? Explique.
- [ ] **Q22.** Se este programa precisasse esperar, ao mesmo tempo, um
  temporizador **e** dados de um socket de rede, qual seria a vantagem prática
  de `timerfd_create()` sobre `setitimer()`+`SIGALRM`? *(Dica: pense em como
  `select()` só sabe esperar descritores de arquivo, não sinais.)*
- [ ] **Q23.** O `tfd` retornado por `timerfd_create()` é passado para
  `FD_SET()`/`select()` exatamente como o `fd[0]` do pipe. Cite outras duas
  funções de E/S comuns (além de `read()`) que funcionam normalmente sobre um
  `timerfd`.
- [ ] **Q24.** Para cada cenário abaixo, indique a ferramenta de tempo mais
  adequada (`nanosleep`, `alarm`, `setitimer`, `clock_gettime`,
  `timerfd_create`, ou `time()`+`strftime()`) e justifique em uma frase:
  - a) Um script que deve esperar exatos 500ms entre tentativas de reconexão,
    sem fazer mais nada enquanto espera.
  - b) Um servidor que atende conexões de rede e também precisa enviar um
    "heartbeat" periódico, tudo no mesmo laço de eventos.
  - c) Medir quanto tempo uma função levou para rodar, para um relatório de
    desempenho.
  - d) Imprimir "Relatório gerado em 14/10/2026 17:30" no topo de um arquivo
    de log.
  - e) Garantir que uma chamada de rede não trave por mais de 5 segundos.

---
-->

## Entrega <!-- e critérios de avaliação -->

### O que entregar

Uma pasta `lab11-nome/` contendo:

**Parte A (entrega ao final do encontro):**

- [ ] `ex1/sigint_default.c`, `ex1/sigint_sigaction.c`, `ex1/respostas.txt`
- [ ] `ex2/eintr_demo.c`, `ex2/restart_demo.c`, `ex2/respostas.txt`
- [ ] `ex3/sigprocmask_demo.c`, `ex3/sigchld_demo.c`, `ex3/sigchld_if.c`, `ex3/respostas.txt`

**Parte B (entrega até o encontro seguinte):**

- [ ] `ex4/alarm_timeout.c` (com o teste da Q16), `ex4/respostas.txt`
- [ ] `ex5/sleep_loop_drift.c`, `ex5/setitimer_periodic.c`, `ex5/respostas.txt`
  
<!--
- [ ] `ex6/timerfd_select.c`, `ex6/respostas.txt`

### Rubrica (10 pontos)

| Critério | Pontos | O que é avaliado |
|---|---|---|
| Ex. 1 — `sigaction()` e ação default | 1,5 | Comparou corretamente `SIGINT` sem e com tratamento; entende `volatile sig_atomic_t` e `write()` vs. `printf()` no handler |
| Ex. 2 — `EINTR` e `SA_RESTART` | 1,5 | Reproduziu a interrupção de `read()`; entende o efeito de `SA_RESTART` |
| Ex. 3 — Bloqueio de sinais e `SIGCHLD` | 2,0 | `sigprocmask`/`sigpending` usados e explicados corretamente; reproduziu a perda de filhos com `if` e explicou por que o `while` é obrigatório |
| Ex. 4 — Timeout com `alarm()` | 1,0 | Timeout funcional e medido; variante da Q16 testada |
| Ex. 5 — `setitimer()` x *drift* | 1,5 | Os dois programas rodados e comparados; entende por que o *drift* difere |
| Ex. 6 — `timerfd_create()` e síntese | 2,5 | `timerfd` integrado com `select()` corretamente; respostas da Q24 coerentes com as características de cada ferramenta |

> **Dica:** Nos Exercícios 2, 4 e 5, o valor numérico exato (`errno`, tempos,
> *drift*) pode variar um pouco na sua máquina — o professor vai olhar se a
> **ordem de grandeza** e a **explicação** batem com o que o código faz, não um
> número decorado.

---

## Anexo A — Referência rápida: sinais

```c
#include <signal.h>

int sigaction(int signo, const struct sigaction *acao, struct sigaction *anterior);
int kill(pid_t pid, int sig);     // envia sig a outro processo
int raise(int sig);               // envia sig a si mesmo

sigset_t s;
sigemptyset(&s);
sigaddset(&s, SIGUSR1);
sigprocmask(int como, const sigset_t *conjunto, sigset_t *anterior);
// como: SIG_BLOCK, SIG_UNBLOCK ou SIG_SETMASK
sigpending(sigset_t *pendentes);  // quais sinais bloqueados estao esperando

#include <sys/wait.h>
pid_t waitpid(pid_t pid, int *status, int opcoes);
// opcoes = WNOHANG: nunca bloqueia; usar em loop dentro do handler de SIGCHLD
```

Funções seguras dentro de um *handler* (lista parcial, *async-signal-safe*):
`write()`, `read()`, `_exit()`, `kill()`, `signal()`, `waitpid()`. **Nunca**
`printf()`, `malloc()`/`free()`.

## Anexo B — Referência rápida: tempo e temporizadores

```c
#include <time.h>
time_t agora = time(NULL);
struct tm *local = localtime(&agora);
strftime(buf, sizeof(buf), "%Y-%m-%d %H:%M:%S", local);

struct timespec t;
clock_gettime(CLOCK_MONOTONIC, &t);  // nunca retrocede -- usar para medir duracao
clock_gettime(CLOCK_REALTIME, &t);   // tempo de calendario -- pode saltar

#include <unistd.h>
unsigned int alarm(unsigned int segundos);  // dispara SIGALRM uma vez

#include <sys/time.h>
struct itimerval it = {
    .it_value    = {.tv_sec = 1, .tv_usec = 0},
    .it_interval = {.tv_sec = 1, .tv_usec = 0},
};
setitimer(ITIMER_REAL, &it, NULL);          // dispara SIGALRM periodicamente

#include <sys/timerfd.h>
int tfd = timerfd_create(CLOCK_MONOTONIC, 0);
timerfd_settime(tfd, 0, &its, NULL);        // mesma struct itimerspec
read(tfd, &expiracoes, sizeof(expiracoes)); // bloqueia ate o proximo disparo
// tfd pode entrar em select()/poll()/epoll_wait() como qualquer outro fd
```

## Anexo C — Tabela-resumo: qual ferramenta de tempo usar

| Situação | Ferramenta recomendada |
|---|---|
| Pausa simples, sem outros eventos | `nanosleep()` |
| Timeout de uma chamada bloqueante | `alarm()` + `SIGALRM` (sem `SA_RESTART`) |
| Repetir uma tarefa a cada N segundos, sem acumular atraso | `setitimer()` |
| Medir quanto tempo uma operação levou | `clock_gettime(CLOCK_MONOTONIC)` |
| Temporizador junto de outros eventos de E/S (sockets, pipes) | `timerfd_create()` + `select`/`poll`/`epoll` |
| Registrar data/hora para humanos (logs, relatórios) | `time()` + `strftime()` |

**Testado e validado em sandbox (2026-10-14):** todos os programas em C deste
guia (`sigint_default.c`, `sigint_sigaction.c`, `eintr_demo.c`,
`restart_demo.c`, `sigprocmask_demo.c`, `sigchld_demo.c`, `alarm_timeout.c`,
`sleep_loop_drift.c`, `setitimer_periodic.c`, `timerfd_select.c`) foram
compilados com `gcc -Wall -Wextra` sem nenhum warning e executados; as saídas
mostradas são reais, não simuladas. A impossibilidade de capturar `SIGKILL`/
`SIGSTOP` foi confirmada isoladamente (`sigaction()` retorna `-1`,
`errno=22`/`EINVAL` para os dois). O *drift* do Exercício 5 foi medido
comparando o mesmo "trabalho" simulado (50ms por tick) nas duas abordagens,
em duas rodadas (drift de 0,503s e 0,504s no loop com `sleep()`; 0,050s nas
duas com `setitimer()`). A coalescência de `SIGCHLD` foi medida contando as
invocações do *handler*: em 5 de 8 execuções de `sigchld_demo.c` o handler
rodou só 2 vezes para 3 filhos (e colheu os 3 graças ao `while`); a variante
com `if` da Q12 colheu só 2 filhos em 3 de 5 execuções.

-->