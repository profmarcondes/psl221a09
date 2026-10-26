# Laboratório 10 <br> Comunicação entre Processos (IPC)

<!-- **PSL221A09 — Programação em Sistemas Linux**

| | |
|---|---|
| **Duração estimada** | 2h — Parte A (presencial, ~90 min) + Parte B (extraclasse) |
| **Formato** | Individual, em dupla ou trio |
| **Pré-requisito** | GCC, `man` local; os Exercícios 2 e 5 precisam de dois terminais (ou `&`/background) para rodar dois processos ao mesmo tempo |
| **Entregável** | Pasta `working_dir/lab10-nome/` com os exercícios da Parte A; Parte B entregue até o encontro seguinte |

---
-->

Este guia percorre os quatro mecanismos de comunicação entre processos (IPC) vistos na Aula 11: **pipes anônimos**, **FIFOs**, **memória compartilhada** POSIX e **filas de mensagens** POSIX. Cada um resolve um problema que os anteriores não resolvem — e cada um tem um jeito diferente (ou nenhum) de garantir que os dois lados não leiam/escrevam fora de hora.

**Como usar este guia.** A **Parte A** (Exercícios 1 a 3) é feita durante o encontro de laboratório, com o professor disponível. A **Parte B** (Exercícios 4 a 6) é atividade extraclasse: mesmo formato e mesmo nível de detalhe, para consolidar sozinho o que não coube no tempo de aula. Sempre que houver uma seção de **saída esperada**, compare com o que apareceu no seu terminal antes de seguir em frente.

> **Atenção ao Exercício 1, Etapa 2.** Ele provoca um programa que **trava de propósito** — é a parte mais importante do exercício, não um erro seu. Use `timeout` (como mostrado) para não precisar matar o processo na mão.

## Preparação do ambiente

```bash
$ mkdir -p working_dir/lab10-nome/{ex1,ex2,ex3,ex4,ex5,ex6}
$ cd working_dir/lab10-nome
```

Substitua `nome` pelo seu nome (ou pelos nomes da dupla/trio). Cada exercício tem sua própria pasta (`ex1/` a `ex6/`) e seu próprio `respostas.txt`.

Os programas deste laboratório usam apenas cabeçalhos do sistema. A memória compartilhada e a fila de mensagens exigem a biblioteca de tempo real na hora de **linkar** (não é preciso instalar nada — ela já faz parte do sistema):

```bash
$ gcc -Wall -o programa programa.c            # pipe/FIFO: nenhuma lib extra
$ gcc -Wall -o programa programa.c -lrt        # shm_open/sem_open/mq_*: -lrt
```

---

# Parte A — presencial (~90 min)

## Exercício 1 — Pipe anônimo e a disciplina do `close()` (25 min)

> **Objetivo:** Criar um pipe entre pai e filho, e entender por que esquecer de fechar a ponta não usada trava o programa.

### Etapa 1 — Um pipe básico

Crie `ex1/pipe_demo.c`:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/wait.h>

int main(void) {
    int fd[2];               /* fd[0] = leitura, fd[1] = escrita */

    if (pipe(fd) == -1) { perror("pipe"); return 1; }

    pid_t pid = fork();
    if (pid < 0) { perror("fork"); return 1; }

    if (pid == 0) {
        /* FILHO: só escreve -- fecha a ponta de leitura */
        close(fd[0]);
        char buf[64];
        snprintf(buf, sizeof(buf), "Ola do filho, PID=%d", getpid());
        write(fd[1], buf, strlen(buf) + 1);
        close(fd[1]);
        return 0;
    } else {
        /* PAI: só le -- fecha a ponta de escrita */
        close(fd[1]);
        char buf[64];
        ssize_t n = read(fd[0], buf, sizeof(buf));
        if (n > 0) printf("[PAI] recebi do pipe (%zd bytes): %s\n", n, buf);
        close(fd[0]);
        wait(NULL);
    }
    return 0;
}
```

```bash
$ cd ex1
$ gcc -Wall -o pipe_demo pipe_demo.c
$ ./pipe_demo
```

**Saída real capturada:**

```
[PAI] recebi do pipe (22 bytes): Ola do filho, PID=305
```

**Em `ex1/respostas.txt`:**

- [ ] **Q1.** `pipe(fd)` preenche **dois** descritores. Qual é o de leitura e qual é o de escrita? E por que o filho chama `close(fd[0])` e o pai chama `close(fd[1])` — o que aconteceria se nenhum dos dois fechasse nada?

### Etapa 2 — Provocando o travamento

Agora um cenário em que o pai lê **em laço**, até ver EOF (`read()` retornando `0`). Crie `ex1/pipe_loop_bug.c`:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/wait.h>

/* BUG DE PROPOSITO: o pai NAO fecha fd[1] */
int main(void) {
    int fd[2];
    if (pipe(fd) == -1) { perror("pipe"); return 1; }

    pid_t pid = fork();
    if (pid < 0) { perror("fork"); return 1; }

    if (pid == 0) {
        close(fd[0]);
        for (int i = 1; i <= 3; i++) {
            char buf[32];
            int len = snprintf(buf, sizeof(buf), "linha %d\n", i);
            write(fd[1], buf, len);
            usleep(200000);
        }
        close(fd[1]);
        return 0;
    } else {
        /* close(fd[1]);   <-- linha que falta de proposito */

        char buf[128];
        ssize_t n;
        printf("[PAI] lendo em loop até EOF (vai travar)...\n");
        fflush(stdout);
        while ((n = read(fd[0], buf, sizeof(buf) - 1)) > 0) {
            buf[n] = '\0';
            printf("[PAI] recebi: %s", buf);
        }
        printf("[PAI] read() retornou %zd -- NUNCA deveria chegar aqui\n", n);
        close(fd[0]);
        wait(NULL);
    }
    return 0;
}
```

```bash
$ gcc -Wall -o pipe_loop_bug pipe_loop_bug.c
$ timeout 3 ./pipe_loop_bug
$ echo $?
```

**Saída real capturada:**

```
[PAI] lendo em loop até EOF (vai travar)...
```
*(o programa trava aqui -- nunca chega a imprimir mais nada)*
```
$ echo $?
124
```

O código de saída `124` é o próprio `timeout` avisando que teve de **matar** o processo — ele nunca terminaria sozinho.

**Em `ex1/respostas.txt`:**

- [ ] **Q2.** O filho escreveu as 3 linhas e chamou `close(fd[1])` normalmente. Mesmo assim o `read()` do pai nunca viu EOF. Por quê? *(Dica: o kernel só sinaliza EOF quando **nenhum processo no sistema** ainda tem a ponta de escrita aberta.)*

### Etapa 3 — Corrigindo

Copie o arquivo para `ex1/pipe_loop.c` e adicione a única linha que falta (`close(fd[1]);`, no ramo do pai, antes do laço de leitura). Recompile e rode:

```bash
$ cp pipe_loop_bug.c pipe_loop.c
$ nano pipe_loop.c     # adicione close(fd[1]); no ramo do pai
$ gcc -Wall -o pipe_loop pipe_loop.c
$ ./pipe_loop
```

**Saída real capturada (com a correção):**

```
[PAI] lendo em loop até EOF...
[PAI] recebi: linha 1
[PAI] recebi: linha 2
[PAI] recebi: linha 3
[PAI] read() retornou 0 (EOF) -- laco terminou
```

**Em `ex1/respostas.txt`:**

- [ ] **Q3.** Qual foi a linha exata que você adicionou, e em qual ramo (pai ou filho)?
- [ ] **Q4.** No Laboratório 08 (Aula 9), um bug parecido — esquecer algo que parece irrelevante — também não dava erro de compilação nem de execução, só um comportamento errado em silêncio (ou, neste caso, travado). Qual foi aquele bug, e o que ele tem em comum com este, em termos do tipo de erro que é (não trava a compilação, só aparece em tempo de execução)?

---

## Exercício 2 — FIFO entre processos independentes (20–25 min)

> **Objetivo:** Comunicar dois processos **sem parentesco** usando uma FIFO, e observar o comportamento bloqueante de `open()`.

Crie `ex2/fifo_writer.c`:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>

#define FIFO_PATH "/tmp/psl_fifo_lab10"

int main(void) {
    printf("[WRITER] PID=%d abrindo %s para escrita (vai bloquear até alguem abrir p/ leitura)...\n",
           getpid(), FIFO_PATH);
    fflush(stdout);

    int fd = open(FIFO_PATH, O_WRONLY);
    if (fd < 0) { perror("open"); return 1; }

    printf("[WRITER] conectado! enviando 3 mensagens...\n");
    for (int i = 1; i <= 3; i++) {
        char buf[64];
        int len = snprintf(buf, sizeof(buf), "mensagem %d do writer (PID=%d)\n", i, getpid());
        write(fd, buf, len);
        printf("[WRITER] enviei: mensagem %d\n", i);
        fflush(stdout);
        sleep(1);
    }

    close(fd);
    printf("[WRITER] fechei o FIFO, encerrando.\n");
    return 0;
}
```

E `ex2/fifo_reader.c`:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>

#define FIFO_PATH "/tmp/psl_fifo_lab10"

int main(void) {
    printf("[READER] PID=%d abrindo %s para leitura (vai bloquear até alguem abrir p/ escrita)...\n",
           getpid(), FIFO_PATH);
    fflush(stdout);

    int fd = open(FIFO_PATH, O_RDONLY);
    if (fd < 0) { perror("open"); return 1; }

    printf("[READER] conectado! lendo até EOF...\n");
    fflush(stdout);

    char buf[128];
    ssize_t n;
    while ((n = read(fd, buf, sizeof(buf) - 1)) > 0) {
        buf[n] = '\0';
        printf("[READER] recebi: %s", buf);
        fflush(stdout);
    }

    printf("[READER] EOF -- writer fechou o FIFO. Encerrando.\n");
    close(fd);
    return 0;
}
```

Crie a FIFO e teste (reader em background, writer em primeiro plano):

```bash
$ cd ex2
$ gcc -Wall -o fifo_writer fifo_writer.c
$ gcc -Wall -o fifo_reader fifo_reader.c
$ rm -f /tmp/psl_fifo_lab10
$ mkfifo /tmp/psl_fifo_lab10
$ ls -l /tmp/psl_fifo_lab10
$ ./fifo_reader &
$ ./fifo_writer
```

**Saída real capturada:**

```
$ ls -l /tmp/psl_fifo_lab10
prw-r--r-- 1 aluno aluno 0 Oct 14 16:38 /tmp/psl_fifo_lab10

$ ./fifo_reader &
[READER] PID=348 abrindo /tmp/psl_fifo_lab10 para leitura (vai bloquear até alguem abrir p/ escrita)...

$ ./fifo_writer
[WRITER] PID=351 abrindo /tmp/psl_fifo_lab10 para escrita (vai bloquear até alguem abrir p/ leitura)...
[WRITER] conectado! enviando 3 mensagens...
[WRITER] enviei: mensagem 1
[READER] conectado! lendo até EOF...
[READER] recebi: mensagem 1 do writer (PID=351)
[WRITER] enviei: mensagem 2
[READER] recebi: mensagem 2 do writer (PID=351)
[WRITER] enviei: mensagem 3
[READER] recebi: mensagem 3 do writer (PID=351)
[WRITER] fechei o FIFO, encerrando.
[READER] EOF -- writer fechou o FIFO. Encerrando.
```

**Em `ex2/respostas.txt`:**

- [ ] **Q5.** O que o `ls -l` mostrou na primeira coluna (tipo do arquivo)? Em que isso difere de um arquivo comum?
- [ ] **Q6.** O `reader`, sozinho, ficou parado na primeira linha até você rodar o `writer`. Por que `open()` bloqueia nesse caso — o que exatamente ele está esperando?
- [ ] **Q7.** O Exercício 1 usava `pipe()` (pai e filho). Este exercício usa `mkfifo()` + dois binários **diferentes**, sem relação de parentesco. O que, na API, permite que `fifo_writer` e `fifo_reader` — dois processos que nunca ouviram falar um do outro — se encontrem?
- [ ] **Q8.** Depois que os dois processos terminam, rode `cat /tmp/psl_fifo_lab10` sozinho (sem mais ninguém escrevendo) e aperte Ctrl+C depois de alguns segundos. Ele ficou esperando, ou retornou na hora? Isso é consistente com a resposta da Q6?

---

## Exercício 3 — Memória compartilhada com semáforo nomeado (30–35 min)

> **Objetivo:** Compartilhar dados entre dois processos via `shm_open`/`mmap`, e sincronizar o acesso com um semáforo **nomeado** (`sem_open`) — diferente do `sem_init()` da Aula 10, que só serve dentro de um processo.

### 3.1 — Produtor e consumidor sincronizados

Crie `ex3/shm_produtor.c`:

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/mman.h>
#include <semaphore.h>

#define SHM_NAME "/psl_shm_lab10"
#define SEM_NAME "/psl_sem_lab10"
#define N_INTS   10

int main(void) {
    int fd = shm_open(SHM_NAME, O_CREAT | O_RDWR, 0666);
    if (fd < 0) { perror("shm_open"); return 1; }

    size_t tamanho = N_INTS * sizeof(int);
    if (ftruncate(fd, (off_t)tamanho) < 0) { perror("ftruncate"); return 1; }

    int *vetor = mmap(NULL, tamanho, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    if (vetor == MAP_FAILED) { perror("mmap"); return 1; }
    close(fd);

    sem_t *sem = sem_open(SEM_NAME, O_CREAT, 0666, 0);
    if (sem == SEM_FAILED) { perror("sem_open"); return 1; }

    printf("[PRODUTOR] escrevendo %d inteiros na memoria compartilhada...\n", N_INTS);
    for (int i = 0; i < N_INTS; i++) vetor[i] = i * i;

    printf("[PRODUTOR] dados prontos, avisando o consumidor (sem_post)\n");
    sem_post(sem);

    sleep(2);   /* da tempo do consumidor terminar antes da limpeza */

    munmap(vetor, tamanho);
    sem_close(sem);
    sem_unlink(SEM_NAME);
    shm_unlink(SHM_NAME);
    printf("[PRODUTOR] limpeza feita (sem_unlink + shm_unlink). Encerrando.\n");
    return 0;
}
```

Crie `ex3/shm_consumidor.c`:

```c
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/mman.h>
#include <semaphore.h>

#define SHM_NAME "/psl_shm_lab10"
#define SEM_NAME "/psl_sem_lab10"
#define N_INTS   10

int main(void) {
    int fd = shm_open(SHM_NAME, O_RDONLY, 0666);
    if (fd < 0) { perror("shm_open"); return 1; }

    size_t tamanho = N_INTS * sizeof(int);
    int *vetor = mmap(NULL, tamanho, PROT_READ, MAP_SHARED, fd, 0);
    if (vetor == MAP_FAILED) { perror("mmap"); return 1; }
    close(fd);

    sem_t *sem = sem_open(SEM_NAME, 0);
    if (sem == SEM_FAILED) { perror("sem_open"); return 1; }

    printf("[CONSUMIDOR] esperando o produtor avisar (sem_wait)...\n");
    fflush(stdout);
    sem_wait(sem);

    printf("[CONSUMIDOR] acordei! lendo a memoria compartilhada:\n");
    long soma = 0;
    for (int i = 0; i < N_INTS; i++) {
        printf("  vetor[%d] = %d\n", i, vetor[i]);
        soma += vetor[i];
    }
    printf("[CONSUMIDOR] soma total = %ld\n", soma);

    munmap(vetor, tamanho);
    sem_close(sem);
    return 0;
}
```

```bash
$ cd ex3
$ gcc -Wall -o shm_produtor shm_produtor.c -lrt
$ gcc -Wall -o shm_consumidor shm_consumidor.c -lrt
$ rm -f /dev/shm/psl_shm_lab10
$ ./shm_produtor &
$ sleep 0.3
$ ./shm_consumidor
```

**Saída real capturada:**

```
[PRODUTOR] escrevendo 10 inteiros na memoria compartilhada...
[PRODUTOR] dados prontos, avisando o consumidor (sem_post)
[CONSUMIDOR] esperando o produtor avisar (sem_wait)...
[CONSUMIDOR] acordei! lendo a memoria compartilhada:
  vetor[0] = 0
  vetor[1] = 1
  vetor[2] = 4
  vetor[3] = 9
  vetor[4] = 16
  vetor[5] = 25
  vetor[6] = 36
  vetor[7] = 49
  vetor[8] = 64
  vetor[9] = 81
[CONSUMIDOR] soma total = 285
[PRODUTOR] limpeza feita (sem_unlink + shm_unlink). Encerrando.
```

**Em `ex3/respostas.txt`:**

- [ ] **Q9.** Três passos são necessários antes de `vetor[i]` virar memória de verdade: `shm_open`, `ftruncate` e `mmap`. O que cada um faz, e o que aconteceria se você pulasse o `ftruncate`?
- [ ] **Q10.** Por que este exercício usa `sem_open()` em vez do `sem_init()` visto na Aula 10? *(Dica: pense em ONDE a variável do semáforo mora em cada caso.)*
- [ ] **Q11.** O produtor cria o semáforo com valor inicial `0` (`sem_open(SEM_NAME, O_CREAT, 0666, 0)`). Se ele tivesse criado com valor inicial `1`, o que aconteceria quando o consumidor chamasse `sem_wait()` logo no início — ele ainda esperaria o produtor?

### 3.2 — O que acontece sem sincronização

Crie `ex3/shm_consumidor_bug.c`, uma versão que **não** espera o semáforo:

```c
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/mman.h>

#define SHM_NAME "/psl_shm_lab10"
#define N_INTS   10

int main(void) {
    int fd = shm_open(SHM_NAME, O_RDONLY, 0666);
    if (fd < 0) { perror("shm_open"); return 1; }

    size_t tamanho = N_INTS * sizeof(int);
    int *vetor = mmap(NULL, tamanho, PROT_READ, MAP_SHARED, fd, 0);
    if (vetor == MAP_FAILED) { perror("mmap"); return 1; }
    close(fd);

    printf("[CONSUMIDOR-BUG] lendo SEM esperar o produtor...\n");
    long soma = 0;
    for (int i = 0; i < N_INTS; i++) soma += vetor[i];
    printf("[CONSUMIDOR-BUG] soma total = %ld (esperado: 285)\n", soma);

    munmap(vetor, tamanho);
    return 0;
}
```

Para reproduzir a janela de tempo em que o shm já existe mas ainda não tem dados, crie também `ex3/shm_criar_vazio.c` (só cria e dimensiona o shm, sem escrever nada):

```c
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/mman.h>

#define SHM_NAME "/psl_shm_lab10"
#define N_INTS   10

int main(void) {
    int fd = shm_open(SHM_NAME, O_CREAT | O_RDWR, 0666);
    if (fd < 0) { perror("shm_open"); return 1; }
    ftruncate(fd, N_INTS * sizeof(int));
    close(fd);
    printf("[SETUP] shm criado e dimensionado, ainda sem dados escritos.\n");
    return 0;
}
```

```bash
$ gcc -Wall -o shm_consumidor_bug shm_consumidor_bug.c -lrt
$ gcc -Wall -o shm_criar_vazio shm_criar_vazio.c -lrt
$ rm -f /dev/shm/psl_shm_lab10
$ ./shm_criar_vazio
$ ./shm_consumidor_bug
$ rm -f /dev/shm/psl_shm_lab10
```

**Saída real capturada:**

```
[SETUP] shm criado e dimensionado, ainda sem dados escritos.
[CONSUMIDOR-BUG] lendo SEM esperar o produtor...
[CONSUMIDOR-BUG] soma total = 0 (esperado: 285)
```

**Em `ex3/respostas.txt`:**

- [ ] **Q12.** O programa **não travou nem deu erro** — só calculou um valor errado. Por que a soma deu exatamente `0`, e não lixo de memória aleatório? *(Dica: o que `ftruncate()` garante sobre o conteúdo inicial de uma região recém-dimensionada?)*
- [ ] **Q13.** Compare este bug com a condição de corrida de threads vista na Aula 10 (contador incrementado sem mutex). O que os dois têm em comum, em termos de **como o erro se manifesta** (ou não se manifesta) para quem está rodando o programa?

---

# Parte B — atividade extraclasse

Os três exercícios abaixo não cabem no encontro de laboratório, mas cobrem conteúdo cobrado na Avaliação 2. Faça-os antes do encontro seguinte e entregue junto com a Parte A.

## Exercício 4 — Pipe bidirecional: requisição e resposta (20 min)

> **Objetivo:** Combinar **dois** pipes unidirecionais para montar um canal de ida e volta entre pai e filho.

Crie `ex4/pipe_bidirecional.c`:

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

int main(void) {
    int pai_para_filho[2];
    int filho_para_pai[2];

    if (pipe(pai_para_filho) == -1 || pipe(filho_para_pai) == -1) {
        perror("pipe");
        return 1;
    }

    pid_t pid = fork();
    if (pid < 0) { perror("fork"); return 1; }

    if (pid == 0) {
        close(pai_para_filho[1]);
        close(filho_para_pai[0]);

        int pedido;
        while (read(pai_para_filho[0], &pedido, sizeof(pedido)) == sizeof(pedido)) {
            int resposta = pedido * pedido;
            write(filho_para_pai[1], &resposta, sizeof(resposta));
        }

        close(pai_para_filho[0]);
        close(filho_para_pai[1]);
        return 0;
    } else {
        close(pai_para_filho[0]);
        close(filho_para_pai[1]);

        for (int i = 1; i <= 5; i++) {
            write(pai_para_filho[1], &i, sizeof(i));
            int resposta;
            read(filho_para_pai[0], &resposta, sizeof(resposta));
            printf("[PAI] pedi o quadrado de %d, filho respondeu %d\n", i, resposta);
        }

        close(pai_para_filho[1]);
        close(filho_para_pai[0]);
        wait(NULL);
    }
    return 0;
}
```

```bash
$ cd ex4
$ gcc -Wall -o pipe_bidirecional pipe_bidirecional.c
$ ./pipe_bidirecional
```

**Saída real capturada:**

```
[PAI] pedi o quadrado de 1, filho respondeu 1
[PAI] pedi o quadrado de 2, filho respondeu 4
[PAI] pedi o quadrado de 3, filho respondeu 9
[PAI] pedi o quadrado de 4, filho respondeu 16
[PAI] pedi o quadrado de 5, filho respondeu 25
```

**Em `ex4/respostas.txt`:**

- [ ] **Q14.** Por que são necessários **dois** pipes, e não um só, para esse requisição/resposta funcionar?
- [ ] **Q15.** Repare que o pai fecha `pai_para_filho[1]` (sua própria ponta de escrita) **depois** do laço `for`, antes do `wait()`. O que isso sinaliza ao filho, e por que o laço `while` do filho depende disso para terminar?
- [ ] **Q16.** Modifique o programa para, em vez de pedir o quadrado, o filho devolver o **cubo** do número (`pedido * pedido * pedido`). Rode de novo e cole a saída.

---

## Exercício 5 — Fila de mensagens com prioridade (20 min)

> **Objetivo:** Enviar mensagens com prioridades diferentes e confirmar que a fila de mensagens POSIX entrega sempre a de maior prioridade primeiro, não em ordem de chegada.

Crie `ex5/mq_prioridade.c`:

```c
#include <stdio.h>
#include <string.h>
#include <mqueue.h>
#include <fcntl.h>

#define MQ_NAME "/psl_mq_lab10"

int main(void) {
    mq_unlink(MQ_NAME);   /* garante que nao sobrou fila de execucao anterior */

    struct mq_attr attr = { .mq_flags = 0, .mq_maxmsg = 10, .mq_msgsize = 128, .mq_curmsgs = 0 };

    mqd_t mq = mq_open(MQ_NAME, O_CREAT | O_RDWR, 0666, &attr);
    if (mq == (mqd_t)-1) { perror("mq_open"); return 1; }

    mq_send(mq, "tarefa normal (prioridade 1)",  30, 1);
    printf("[ENVIO] enviei 'tarefa normal' com prioridade 1\n");

    mq_send(mq, "tarefa de fundo (prioridade 0)", 31, 0);
    printf("[ENVIO] enviei 'tarefa de fundo' com prioridade 0\n");

    mq_send(mq, "ALERTA URGENTE (prioridade 9)",  29, 9);
    printf("[ENVIO] enviei 'ALERTA URGENTE' com prioridade 9\n");

    printf("\n[RECEBIMENTO] ordem de ENVIO foi: normal(1), fundo(0), urgente(9)\n");
    printf("[RECEBIMENTO] lendo a fila agora, em ordem de PRIORIDADE:\n");

    char buf[128];
    unsigned int prio;
    struct mq_attr cur;
    for (;;) {
        ssize_t n = mq_receive(mq, buf, sizeof(buf), &prio);
        if (n < 0) break;
        buf[n] = '\0';
        printf("  prioridade=%u -> \"%s\"\n", prio, buf);
        mq_getattr(mq, &cur);
        if (cur.mq_curmsgs == 0) break;
    }

    mq_close(mq);
    mq_unlink(MQ_NAME);
    return 0;
}
```

```bash
$ cd ex5
$ gcc -Wall -o mq_prioridade mq_prioridade.c -lrt
$ ./mq_prioridade
```

**Saída real capturada:**

```
[ENVIO] enviei 'tarefa normal' com prioridade 1
[ENVIO] enviei 'tarefa de fundo' com prioridade 0
[ENVIO] enviei 'ALERTA URGENTE' com prioridade 9

[RECEBIMENTO] ordem de ENVIO foi: normal(1), fundo(0), urgente(9)
[RECEBIMENTO] lendo a fila agora, em ordem de PRIORIDADE:
  prioridade=9 -> "ALERTA URGENTE (prioridade 9)"
  prioridade=1 -> "tarefa normal (prioridade 1)"
  prioridade=0 -> "tarefa de fundo (prioridade 0)"
```

> **Nota sobre limites do sistema.** `attr.mq_maxmsg` e `attr.mq_msgsize` não podem passar dos limites do kernel (`/proc/sys/fs/mqueue/msg_default` e `msgsize_default`, tipicamente 10 e 8192 no Linux). Se `mq_open` falhar com `EINVAL`, confira esses valores com `cat /proc/sys/fs/mqueue/msg_default`.

**Em `ex5/respostas.txt`:**

- [ ] **Q17.** A mensagem `"ALERTA URGENTE"` foi a **última** enviada. Em qual posição ela foi recebida? O que isso prova sobre como a fila ordena as mensagens?
- [ ] **Q18.** Troque a prioridade de `"tarefa de fundo"` de `0` para `1` (igual à `"tarefa normal"`) e rode de novo. Entre duas mensagens de **mesma** prioridade, qual chega primeiro? Isso é consistente com a ordem de **envio**?
- [ ] **Q19.** Diferente do pipe e da FIFO, a fila de mensagens entrega **unidades inteiras** (uma `mq_receive()` nunca devolve metade de uma mensagem e metade de outra). Por que essa diferença importa para um programa que processa, por exemplo, pedidos JSON completos vindos de outro processo?
- [ ] **Q20.** Dê um exemplo (diferente do alerta deste exercício) de um cenário real em que a prioridade de uma fila de mensagens seria útil — alguma situação em que uma mensagem deveria "furar a fila".

---

## Exercício 6 — Escolhendo o mecanismo certo (20–25 min)

> **Objetivo:** Comparar o custo real de copiar dados por pipe contra memória compartilhada, e praticar a decisão de qual mecanismo de IPC usar em cenários diferentes.

### 6.1 — Benchmark: pipe vs. memória compartilhada

Crie `ex6/bench_pipe.c`:

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>
#include <time.h>

#define TOTAL (20 * 1024 * 1024)   /* 20 MB */
#define CHUNK 65536

int main(void) {
    int fd[2];
    if (pipe(fd) == -1) { perror("pipe"); return 1; }

    pid_t pid = fork();
    if (pid == 0) {
        close(fd[0]);
        char *buf = malloc(CHUNK);
        for (size_t enviado = 0; enviado < TOTAL; enviado += CHUNK) {
            if (write(fd[1], buf, CHUNK) < 0) { perror("write"); break; }
        }
        close(fd[1]);
        free(buf);
        return 0;
    } else {
        close(fd[1]);
        char *buf = malloc(CHUNK);
        size_t total_lido = 0;
        struct timespec t0, t1;
        clock_gettime(CLOCK_MONOTONIC, &t0);

        ssize_t n;
        while ((n = read(fd[0], buf, CHUNK)) > 0) total_lido += (size_t)n;

        clock_gettime(CLOCK_MONOTONIC, &t1);
        double ms = (t1.tv_sec - t0.tv_sec) * 1000.0 + (t1.tv_nsec - t0.tv_nsec) / 1e6;
        printf("[PIPE] transferidos %zu bytes em %.2f ms\n", total_lido, ms);

        close(fd[0]);
        free(buf);
        wait(NULL);
    }
    return 0;
}
```

Crie `ex6/bench_shm.c`:

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/mman.h>
#include <semaphore.h>
#include <sys/wait.h>
#include <time.h>
#include <string.h>

#define SHM_NAME "/psl_bench_shm_lab10"
#define SEM_NAME "/psl_bench_sem_lab10"
#define TOTAL    (20 * 1024 * 1024)   /* 20 MB */

int main(void) {
    shm_unlink(SHM_NAME);
    sem_unlink(SEM_NAME);

    int fd = shm_open(SHM_NAME, O_CREAT | O_RDWR, 0666);
    if (fd < 0) { perror("shm_open"); return 1; }
    if (ftruncate(fd, TOTAL) < 0) { perror("ftruncate"); return 1; }
    char *regiao = mmap(NULL, TOTAL, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    close(fd);

    sem_t *sem = sem_open(SEM_NAME, O_CREAT, 0666, 0);

    pid_t pid = fork();
    if (pid == 0) {
        memset(regiao, 0x42, TOTAL);
        sem_post(sem);
        return 0;
    } else {
        struct timespec t0, t1;
        sem_wait(sem);
        clock_gettime(CLOCK_MONOTONIC, &t0);

        volatile long soma = 0;
        for (size_t i = 0; i < TOTAL; i += 4096) soma += regiao[i];

        clock_gettime(CLOCK_MONOTONIC, &t1);
        double ms = (t1.tv_sec - t0.tv_sec) * 1000.0 + (t1.tv_nsec - t0.tv_nsec) / 1e6;
        printf("[SHM]  %d bytes acessiveis diretamente, percorridos em %.2f ms (soma=%ld)\n",
               TOTAL, ms, soma);

        wait(NULL);
        munmap(regiao, TOTAL);
        sem_close(sem);
        sem_unlink(SEM_NAME);
        shm_unlink(SHM_NAME);
    }
    return 0;
}
```

```bash
$ cd ex6
$ gcc -O2 -Wall -o bench_pipe bench_pipe.c
$ gcc -O2 -Wall -o bench_shm bench_shm.c -lrt
$ ./bench_pipe
$ ./bench_shm
```

**Saída real capturada (20 MB transferidos de formas equivalentes):**

```
$ ./bench_pipe
[PIPE] transferidos 20971520 bytes em 10.88 ms

$ ./bench_shm
[SHM]  20971520 bytes acessiveis diretamente, percorridos em 1.06 ms (soma=337920)
```

**Em `ex6/respostas.txt`:**

- [ ] **Q21.** Rode cada benchmark três vezes. Os tempos variam muito entre execuções? Qual mecanismo foi consistentemente mais rápido, e por qual fator (aproximadamente quantas vezes)?
- [ ] **Q22.** O `bench_shm` não está fazendo exatamente a mesma coisa que o `bench_pipe` — ele só **percorre** dados que já estão compartilhados, sem nenhuma chamada `write()`/`read()`. Isso é trapaça, ou é exatamente o ponto que o exercício quer mostrar sobre memória compartilhada? Justifique.

### 6.2 — Qual mecanismo usar?

Para cada cenário abaixo, escreva em `ex6/respostas.txt` **qual** mecanismo de IPC (pipe anônimo, FIFO, memória compartilhada ou fila de mensagens) você usaria, e **uma frase** justificando:

- [ ] **Q23.** Um processo de captura de vídeo precisa entregar continuamente quadros de ~8 MB para um processo de codificação, o mais rápido possível, sem nenhum processamento especial de prioridade.
- [ ] **Q24.** Um script dispara um processo filho para rodar um comando longo e só precisa capturar a saída dele no final.
- [ ] **Q25.** Um serviço de impressão recebe trabalhos de impressão de vários programas diferentes no sistema (sem parentesco entre eles), e um trabalho marcado como "urgente" deve ser impresso antes dos que já estavam na fila.
- [ ] **Q26.** Dois programas de fabricantes diferentes, sem nenhuma relação de processo pai/filho, precisam trocar comandos curtos de texto plano, e um precisa poder iniciar mesmo que o outro ainda não tenha sido aberto.

---

## Entrega <!-- e critérios de avaliação -->

### O que entregar

Uma pasta `lab10-nome/` contendo:

**Parte A (entrega ao final do encontro):**

- [ ] `ex1/pipe_demo.c`, `ex1/pipe_loop_bug.c`, `ex1/pipe_loop.c`, `ex1/respostas.txt`
- [ ] `ex2/fifo_writer.c`, `ex2/fifo_reader.c`, `ex2/respostas.txt`
- [ ] `ex3/shm_produtor.c`, `ex3/shm_consumidor.c`, `ex3/shm_consumidor_bug.c`, `ex3/shm_criar_vazio.c`, `ex3/respostas.txt`

**Parte B (entrega até o encontro seguinte):**

- [ ] `ex4/pipe_bidirecional.c` (com a variante do cubo), `ex4/respostas.txt`
- [ ] `ex5/mq_prioridade.c`, `ex5/respostas.txt`
- [ ] `ex6/bench_pipe.c`, `ex6/bench_shm.c`, `ex6/respostas.txt`

<!--
### Rubrica (10 pontos)

| Critério | Pontos | O que é avaliado |
|---|---|---|
| Ex. 1 — Pipe e disciplina do `close()` | 2,0 | Reproduziu o travamento de propósito; entende por que o `close()` esquecido causa isso; correção testada e funcional |
| Ex. 2 — FIFO | 1,5 | FIFO criada e usada corretamente; entende o comportamento bloqueante de `open()`; diferencia pipe de FIFO |
| Ex. 3 — Memória compartilhada + semáforo | 2,5 | `shm_open`/`ftruncate`/`mmap` corretos; sincronização com `sem_open` funcional; reproduziu e explicou o caso sem sincronização |
| Ex. 4 — Pipe bidirecional | 1,0 | Dois pipes funcionando nos dois sentidos; variante do cubo implementada |
| Ex. 5 — Fila de mensagens | 1,5 | Prioridade demonstrada corretamente; entende a diferença de unidade de dados (mensagens x fluxo de bytes) |
| Ex. 6 — Escolha do mecanismo | 1,5 | Benchmark executado de fato; respostas do 6.2 coerentes com as características de cada mecanismo |

> **Dica:** Nos Exercícios 1 e 3, o travamento/erro silencioso **de propósito** é a parte pedagogicamente mais importante — o professor vai olhar se você reproduziu o problema antes de corrigi-lo, não só se o código final funciona.

---

## Anexo A — Referência rápida: pipes e FIFOs

```c
#include <unistd.h>   // pipe(), read(), write(), close()

int pipe(int fd[2]);
// fd[0] = leitura, fd[1] = escrita
// EOF (read() == 0) so' ocorre quando TODAS as pontas de escrita fecharam

#include <sys/stat.h>
int mkfifo(const char *caminho, mode_t modo);
// cria uma FIFO no sistema de arquivos -- mesma semantica do pipe,
// mas qualquer processo que souber o caminho pode abri-la
```

```bash
$ mkfifo /caminho/para/fifo     # equivalente shell de mkfifo()
$ ls -l /caminho/para/fifo      # tipo "p" na primeira coluna
```

## Anexo B — Referência rápida: memória compartilhada e filas de mensagens

```c
#include <sys/mman.h>
#include <fcntl.h>

int shm_open(const char *nome, int flags, mode_t modo);   // nome comeca com "/"
int ftruncate(int fd, off_t tamanho);                      // define o tamanho
void *mmap(NULL, tamanho, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);
int shm_unlink(const char *nome);                           // remove o objeto

#include <semaphore.h>
sem_t *sem_open(const char *nome, int flags, ...);  // semaforo NOMEADO (entre processos)
int sem_wait(sem_t *sem);                            // bloqueia ate o valor ser > 0
int sem_post(sem_t *sem);                             // incrementa, acorda quem espera
int sem_unlink(const char *nome);

#include <mqueue.h>
mqd_t mq_open(const char *nome, int flags, mode_t modo, struct mq_attr *attr);
int mq_send(mqd_t mq, const char *msg, size_t tam, unsigned int prioridade);
ssize_t mq_receive(mqd_t mq, char *buf, size_t tam, unsigned int *prioridade);
// entrega sempre a mensagem de MAIOR prioridade primeiro
```

## Anexo C — Tabela-resumo: qual mecanismo usar

| Situação | Mecanismo recomendado |
|---|---|
| Pai e filho, fluxo simples de bytes | Pipe anônimo |
| Processos sem parentesco, fluxo simples de bytes | FIFO |
| Volume grande de dados, ou acesso repetido ao mesmo dado | Memória compartilhada (+ semáforo nomeado para sincronizar) |
| Mensagens discretas que precisam de urgência/prioridade | Fila de mensagens |
| Só uma notificação, sem dado nenhum | *(fora do escopo deste laboratório --- ver Aula 12, sinais)* |

**Testado e validado em sandbox (2026-10-14):** todos os programas em C deste guia (`pipe_demo.c`, `pipe_loop_bug.c`, `pipe_loop.c`, `fifo_writer.c`, `fifo_reader.c`, `shm_produtor.c`, `shm_consumidor.c`, `shm_consumidor_bug.c`, `shm_criar_vazio.c`, `pipe_bidirecional.c`, `mq_prioridade.c`, `bench_pipe.c`, `bench_shm.c`) foram compilados com `gcc -Wall -Wextra` sem nenhum warning e executados; as saídas mostradas são reais, não simuladas. O travamento do Exercício 1 (Etapa 2) foi confirmado com `timeout 3` (código de saída 124). A condição de corrida do Exercício 3.2 foi reproduzida isolando a criação do shm (sem dados) da leitura, confirmando soma 0 em vez de 285. O benchmark do Exercício 6.1 foi executado três vezes para cada mecanismo antes de registrar os tempos no guia.

-->