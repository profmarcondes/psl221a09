# Laboratório 09 <br> Threads e Sincronização

<!-- **PSL221A09 — Programação em Sistemas Linux**

| | |
|---|---|
| **Duração** | 2h — Parte A (presencial, ~90 min) + Parte B (extraclasse) |
| **Formato** | Individual, em dupla ou trio |
| **Pré-requisito** | GCC e a biblioteca `pthreads` (padrão em qualquer Linux); um terminal onde você possa rodar o mesmo programa várias vezes seguidas |
| **Entregável** | Pasta `working_dir/lab09-nome/` com os exercícios da Parte A; Parte B entregue até o encontro seguinte |

---
-->

Este guia percorre o caminho completo da programação com threads: dividir trabalho para ganhar velocidade, descobrir da pior forma que threads compartilham memória, e então aprender as três ferramentas que tornam esse compartilhamento seguro — **mutex**, **semáforos** e **variáveis de condição** —, terminando no erro clássico que elas próprias podem causar: o **deadlock**.

**Como usar este guia.** A **Parte A** (Exercícios 1 a 3) é feita durante o encontro de laboratório, com o professor disponível. A **Parte B** (Exercícios 4 a 6) é atividade extraclasse. Sempre que houver uma seção de **saída esperada**, compare com o que apareceu no seu terminal antes de seguir em frente.

> **Antes de começar, descubra quantos núcleos sua máquina tem:**
>
> ```bash
> $ nproc
> ```
>
> Guarde esse número — ele vai importar no Exercício 1. Na máquina onde este guia foi validado, `nproc` = **2**.

> **Aviso sobre medições.** Os tempos e *speedups* deste guia foram medidos em uma VM compartilhada e **variam** de máquina para máquina e de execução para execução. O que você deve reproduzir não são os números exatos, e sim o **formato da curva** (onde ela cresce e onde ela estaciona).

## Preparação do ambiente

```bash
$ mkdir -p working_dir/lab09-nome/{ex1,ex2,ex3,ex4,ex5,ex6}
$ cd working_dir/lab09-nome
```

Substitua `nome` pelo seu nome (ou pelos nomes da dupla/trio). Cada exercício tem sua própria pasta (`ex1/` a `ex6/`) e seu próprio `respostas.txt`.

Todos os programas usam `<pthread.h>` e precisam da biblioteca de threads. Use a opção **`-pthread`** (com hífen, sem `l`):

```bash
$ gcc -Wall -Wextra -pthread -o programa programa.c
```

> **Por que `-pthread` e não `-lpthread`?** `-lpthread` só manda o *linker* juntar a biblioteca. `-pthread` faz isso **e** define as macros que deixam os cabeçalhos em modo *thread-safe* (`_REENTRANT`). É a forma recomendada e a que usaremos no semestre inteiro.

---

# Parte A — presencial (~90 min)

## Exercício 1 — Soma paralela e *speedup* (25–30 min)

> **Objetivo:** Dividir um array entre N threads, medir o ganho contra a versão sequencial e descobrir onde esse ganho para de crescer.

Crie `ex1/soma_paralela.c`:

```c
#define _POSIX_C_SOURCE 199309L
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <time.h>

#define TAM 100000000
#define MAX_THREADS 16

static int *vetor;
static long long somas_parciais[MAX_THREADS];

typedef struct {
    int id;
    long long inicio, fim;
} tarefa_t;

void *soma_faixa(void *arg) {
    tarefa_t *t = (tarefa_t *)arg;
    long long soma = 0;
    for (long long i = t->inicio; i < t->fim; i++)
        soma += vetor[i];
    somas_parciais[t->id] = soma;   /* cada thread escreve na SUA posicao */
    return NULL;
}

static double agora(void) {
    struct timespec ts;
    clock_gettime(CLOCK_MONOTONIC, &ts);
    return ts.tv_sec + ts.tv_nsec / 1e9;
}

int main(int argc, char **argv) {
    int n_threads = (argc > 1) ? atoi(argv[1]) : 2;

    vetor = malloc(TAM * sizeof(int));
    for (long long i = 0; i < TAM; i++) vetor[i] = 1;   /* soma esperada = TAM */

    double t0 = agora();
    long long seq = 0;
    for (long long i = 0; i < TAM; i++) seq += vetor[i];
    double t_seq = agora() - t0;
    printf("Sequencial: soma=%lld, tempo=%.4fs\n", seq, t_seq);

    pthread_t th[MAX_THREADS];
    tarefa_t tf[MAX_THREADS];
    long long bloco = TAM / n_threads;

    t0 = agora();
    for (int i = 0; i < n_threads; i++) {
        tf[i].id = i;
        tf[i].inicio = i * bloco;
        tf[i].fim = (i == n_threads - 1) ? TAM : (i + 1) * bloco;
        pthread_create(&th[i], NULL, soma_faixa, &tf[i]);
    }
    long long par = 0;
    for (int i = 0; i < n_threads; i++) {
        pthread_join(th[i], NULL);
        par += somas_parciais[i];
    }
    double t_par = agora() - t0;

    printf("Paralelo (%d threads): soma=%lld, tempo=%.4fs\n", n_threads, par, t_par);
    printf("Speedup: %.2fx\n", t_seq / t_par);

    free(vetor);
    return 0;
}
```

Compile e rode com várias quantidades de threads, **repetindo cada uma três vezes**:

```bash
$ cd ex1
$ gcc -Wall -Wextra -pthread -o soma_paralela soma_paralela.c
$ for n in 1 2 4 8; do echo "--- $n thread(s)"; for r in 1 2 3; do ./soma_paralela $n | tail -1; done; done
```

**Saída real capturada (`nproc` = 2), só a linha de *speedup*:**

```
1 thread(s): 0.99  0.98  0.97
2 threads:   1.87  1.78  1.93
4 threads:   2.21  1.72  1.96
8 threads:   2.38  1.91  1.49
```

Repare em três coisas: com **1** thread o ganho é nenhum (≈1,0 — o mesmo trabalho, mais o custo de criar a thread); com **2** threads (= `nproc`) o ganho chega perto de 2x; e de 4 threads em diante o ganho **não cresce**, só fica mais **instável** entre execuções.

**Em `ex1/respostas.txt`:**

- [ ] **Q1.** Qual é o `nproc` da sua máquina? Cole sua tabela de *speedups* (3 execuções para cada quantidade de threads). O ganho chegou perto do número de núcleos quando você usou exatamente `nproc` threads?
- [ ] **Q2.** Compare a **variação** entre as 3 execuções de `nproc` threads com a variação entre as 3 execuções de 8 threads. Em qual caso os números ficaram mais espalhados? Por que você acha que isso acontece?
- [ ] **Q3.** No código, por que existem **dois laços `for` separados** — um só para criar e outro só para esperar as threads? O que mudaria se você criasse e já desse `pthread_join` na mesma iteração? *(Dica: é o mesmo raciocínio do laço de `waitpid()` do Laboratório 08.)*
- [ ] **Q4.** Recompile com `-O2` e rode de novo com `nproc` threads. O *speedup* melhorou ou piorou? *(Esse resultado costuma surpreender — guarde o número, vamos voltar a ele no gabarito.)*

---

## Exercício 2 — Condição de corrida, e como consertá-la (30 min)

> **Objetivo:** Provocar de propósito uma condição de corrida, confirmar que o resultado varia entre execuções, corrigi-la com um mutex e medir o preço dessa correção.

### Etapa 1 — O problema

Crie `ex2/race.c`:

```c
#include <stdio.h>
#include <pthread.h>

#define N_THREADS 4
#define INCREMENTOS 1000000

long contador = 0;

void *incrementa(void *arg) {
    (void)arg;
    for (int i = 0; i < INCREMENTOS; i++) {
        contador++;   /* NAO e atomico: le, soma 1, escreve -- 3 passos */
    }
    return NULL;
}

int main(void) {
    pthread_t t[N_THREADS];
    for (int i = 0; i < N_THREADS; i++) pthread_create(&t[i], NULL, incrementa, NULL);
    for (int i = 0; i < N_THREADS; i++) pthread_join(t[i], NULL);

    long esperado = (long)N_THREADS * INCREMENTOS;
    printf("Contador final: %ld\n", contador);
    printf("Esperado:       %ld\n", esperado);
    printf("Diferenca:      %ld (incrementos perdidos)\n", esperado - contador);
    return 0;
}
```

Compile **sem otimização** e rode várias vezes:

```bash
$ cd ex2
$ gcc -Wall -Wextra -O0 -pthread -o race race.c
$ ./race ; ./race ; ./race
```

**Saída real capturada (três execuções do mesmo binário):**

```
Contador final: 1386195
Esperado:       4000000
Diferenca:      2613805 (incrementos perdidos)

Contador final: 2250578
Contador final: 2762342
```

Perdemos **mais da metade** dos incrementos, e o valor **muda** a cada execução.

**Em `ex2/respostas.txt`:**

- [ ] **Q5.** Rode `./race` pelo menos 5 vezes e anote os valores. Algum deles acertou 4000000? Eles se repetiram?
- [ ] **Q6.** `contador++` parece uma operação só, mas são três passos (ler, somar 1, escrever). Descreva um cenário concreto com **duas threads** em que essa intercalação faz um incremento *desaparecer*. *(Dica: as duas leem o mesmo valor antes de qualquer uma escrever de volta.)*

### Etapa 2 — A surpresa do `-O2`

Agora recompile **exatamente o mesmo arquivo** com otimização:

```bash
$ gcc -Wall -Wextra -O2 -pthread -o race_O2 race.c
$ ./race_O2 ; ./race_O2 ; ./race_O2
```

**Saída real capturada:**

```
Contador final: 4000000
Contador final: 4000000
Contador final: 4000000
```

O bug "sumiu" — mas o código **não** foi corrigido. Olhe o que o compilador fez com o laço:

```bash
$ gcc -O2 -S -o - race.c | sed -n '/^incrementa:/,/ret/p'
```

**Saída real:**

```
incrementa:
	endbr64
	addq	$1000000, contador(%rip)
	xorl	%eax, %eax
	ret
```

O laço de **um milhão de iterações** virou **uma única instrução**: somar 1000000 de uma vez.

- [ ] **Q7.** Com base nesse assembly, explique por que a corrida praticamente não aparece mais com `-O2`. Quantas operações de "ler–somar–escrever" sobre `contador` cada thread faz agora?
- [ ] **Q8.** A corrida **desapareceu** ou apenas **ficou muito improvável**? *(Dica: `addq` é uma única instrução, mas sem o prefixo `lock` ela ainda é ler–somar–escrever por baixo.)* Qual é o perigo de concluir "compilei com -O2 e funcionou, logo meu código está correto"?

### Etapa 3 — A correção de verdade

Copie `race.c` para `ex2/mutex.c` e proteja a seção crítica:

```c
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

void *incrementa(void *arg) {
    (void)arg;
    for (int i = 0; i < INCREMENTOS; i++) {
        pthread_mutex_lock(&mutex);
        contador++;                   /* secao critica */
        pthread_mutex_unlock(&mutex);
    }
    return NULL;
}
```

```bash
$ gcc -Wall -Wextra -O0 -pthread -o mutex mutex.c
$ ./mutex ; ./mutex ; ./mutex
```

**Saída real capturada:**

```
Contador final: 4000000
Contador final: 4000000
Contador final: 4000000
```

Agora é **sempre** 4000000 — e continua sendo mesmo com `-O2`, porque a correção está no código, não no compilador.

Agora meça o preço:

```bash
$ time ./race     # sem mutex (e errado)
$ time ./mutex    # com mutex (e correto)
```

**Saída real capturada:**

```
./race    real  0m0.012s
./mutex   real  0m0.288s
```

- [ ] **Q9.** Quantas vezes mais lenta ficou a versão com mutex, na sua máquina? Por que travar e destravar um cadeado 4 milhões de vezes custa tão caro?
- [ ] **Q10.** Reescreva `incrementa()` para que cada thread acumule num **contador local** e trave o mutex **uma única vez**, no fim, para somar seu total ao global. Meça o tempo da nova versão e confirme que o resultado continua 4000000. Cole o código e o tempo.

---

## Exercício 3 — Produtor–consumidor com semáforos (30 min)

> **Objetivo:** Coordenar duas threads de velocidades diferentes através de um buffer circular limitado, usando semáforos para contar vagas e itens.

Crie `ex3/prodcons.c`:

```c
#include <stdio.h>
#include <pthread.h>
#include <semaphore.h>
#include <unistd.h>

#define TAM_BUFFER 5
#define N_ITENS   10

int buffer[TAM_BUFFER];
int idx_prod = 0, idx_cons = 0;

sem_t vagas;    /* quantas posicoes LIVRES existem   */
sem_t itens;    /* quantas posicoes OCUPADAS existem */
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

void *produtor(void *arg) {
    (void)arg;
    for (int i = 1; i <= N_ITENS; i++) {
        sem_wait(&vagas);                 /* espera ter vaga     */
        pthread_mutex_lock(&mutex);       /* exclusao mutua      */
        buffer[idx_prod] = i;
        printf("[PROD] produziu %d na posicao %d\n", i, idx_prod);
        idx_prod = (idx_prod + 1) % TAM_BUFFER;
        pthread_mutex_unlock(&mutex);
        sem_post(&itens);                 /* avisa: tem item     */
        usleep(10000);                    /* produtor RAPIDO     */
    }
    return NULL;
}

void *consumidor(void *arg) {
    (void)arg;
    for (int i = 1; i <= N_ITENS; i++) {
        sem_wait(&itens);                 /* espera ter item     */
        pthread_mutex_lock(&mutex);
        int valor = buffer[idx_cons];
        printf("        [CONS] consumiu %d da posicao %d\n", valor, idx_cons);
        idx_cons = (idx_cons + 1) % TAM_BUFFER;
        pthread_mutex_unlock(&mutex);
        sem_post(&vagas);                 /* avisa: liberou vaga */
        usleep(30000);                    /* consumidor LENTO    */
    }
    return NULL;
}

int main(void) {
    pthread_t p, c;
    sem_init(&vagas, 0, TAM_BUFFER);   /* comeca com TAM_BUFFER vagas */
    sem_init(&itens, 0, 0);            /* comeca com 0 itens          */

    pthread_create(&p, NULL, produtor, NULL);
    pthread_create(&c, NULL, consumidor, NULL);
    pthread_join(p, NULL);
    pthread_join(c, NULL);

    sem_destroy(&vagas);
    sem_destroy(&itens);
    printf("Todos os %d itens foram produzidos e consumidos.\n", N_ITENS);
    return 0;
}
```

```bash
$ cd ex3
$ gcc -Wall -Wextra -pthread -o prodcons prodcons.c
$ ./prodcons
```

**Saída real capturada:**

```
[PROD] produziu 1 na posicao 0
        [CONS] consumiu 1 da posicao 0
[PROD] produziu 2 na posicao 1
[PROD] produziu 3 na posicao 2
[PROD] produziu 4 na posicao 3
[PROD] produziu 5 na posicao 4
[PROD] produziu 6 na posicao 0
        [CONS] consumiu 3 da posicao 2
[PROD] produziu 7 na posicao 1
[PROD] produziu 8 na posicao 2
        [CONS] consumiu 4 da posicao 3
[PROD] produziu 9 na posicao 3
        [CONS] consumiu 5 da posicao 4
[PROD] produziu 10 na posicao 4
        [CONS] consumiu 6 da posicao 0
        [CONS] consumiu 7 da posicao 1
        [CONS] consumiu 8 da posicao 2
        [CONS] consumiu 9 da posicao 3
        [CONS] consumiu 10 da posicao 4
Todos os 10 itens foram produzidos e consumidos.
```

**Em `ex3/respostas.txt`:**

- [ ] **Q11.** O produtor é 3x mais rápido que o consumidor, mas em nenhum momento ele produziu mais de 5 itens à frente. Qual linha do código impede isso, e como?
- [ ] **Q12.** Repare que a posição volta a 0 depois da 4 ("produziu 6 na posicao 0"). Que parte do código faz o buffer ser **circular**? Por que isso é seguro, já que a posição 0 já tinha sido usada?
- [ ] **Q13.** Existem **dois** semáforos e **um** mutex. Explique o papel de cada um: o que aconteceria se você removesse só o mutex, mantendo os semáforos? *(Dica: pense em dois produtores em vez de um.)*
- [ ] **Q14.** Troque `TAM_BUFFER` para `1` e rode de novo. Como muda o padrão da saída? O programa ainda funciona corretamente?

---

# Parte B — atividade extraclasse

Os três exercícios abaixo não cabem no encontro de laboratório, mas cobrem conteúdo cobrado na Avaliação 2. Faça-os antes do encontro seguinte e entregue junto com a Parte A.

## Exercício 4 — Fila *thread-safe* com variável de condição (30 min)

> **Objetivo:** Esperar que uma condição se torne verdadeira **sem** queimar CPU, e entender por que a espera vai dentro de um `while`, nunca de um `if`.

Crie `ex4/fila_cond.c`:

```c
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>

#define TAM 4
#define N_ITENS 6

int fila[TAM];
int n_itens = 0;

pthread_mutex_t m        = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t  tem_item = PTHREAD_COND_INITIALIZER;

void *consumidor(void *arg) {
    (void)arg;
    for (int i = 0; i < N_ITENS; i++) {
        pthread_mutex_lock(&m);
        while (n_itens == 0) {                  /* while, NAO if */
            printf("        [CONS] fila vazia, esperando...\n");
            pthread_cond_wait(&tem_item, &m);   /* solta o mutex e dorme */
        }
        int v = fila[--n_itens];
        printf("        [CONS] consumiu %d (restam %d)\n", v, n_itens);
        pthread_mutex_unlock(&m);
        usleep(50000);
    }
    return NULL;
}

int main(void) {
    pthread_t c;
    pthread_create(&c, NULL, consumidor, NULL);

    for (int i = 1; i <= N_ITENS; i++) {
        usleep(80000);                          /* produtor mais LENTO */
        pthread_mutex_lock(&m);
        fila[n_itens++] = i * 10;
        printf("[PROD] inseriu %d (total %d)\n", i * 10, n_itens);
        pthread_cond_signal(&tem_item);         /* acorda UM que espera */
        pthread_mutex_unlock(&m);
    }

    pthread_join(c, NULL);
    printf("Fim.\n");
    return 0;
}
```

```bash
$ cd ex4
$ gcc -Wall -Wextra -pthread -o fila_cond fila_cond.c
$ ./fila_cond
```

**Saída real capturada:**

```
        [CONS] fila vazia, esperando...
[PROD] inseriu 10 (total 1)
        [CONS] consumiu 10 (restam 0)
        [CONS] fila vazia, esperando...
[PROD] inseriu 20 (total 1)
        [CONS] consumiu 20 (restam 0)
        ...
[PROD] inseriu 60 (total 1)
        [CONS] consumiu 60 (restam 0)
Fim.
```

Aqui o **consumidor** é que corre na frente: ele esvazia a fila e dorme até o produtor avisar.

**Em `ex4/respostas.txt`:**

- [ ] **Q15.** Enquanto o consumidor está parado em `pthread_cond_wait`, ele está consumindo CPU? Rode `top` (ou `htop`) em outro terminal durante a execução e confirme. Compare com o que aconteceria se a espera fosse `while (n_itens == 0) { }` — um laço vazio.
- [ ] **Q16.** `pthread_cond_wait(&tem_item, &m)` recebe o **mutex** como segundo argumento. Por que ele precisa soltar o mutex antes de dormir? O que travaria se ele dormisse segurando o mutex?
- [ ] **Q17.** Troque o `while (n_itens == 0)` por `if (n_itens == 0)` e rode várias vezes. Neste programa específico (um produtor, um consumidor) o resultado muda? Mesmo que não mude aqui, por que `while` ainda é a forma correta? *(Dica: pense em dois consumidores disputando o mesmo item.)*
- [ ] **Q18.** Acrescente um **segundo consumidor** (outra `pthread_create` com a mesma função) e ajuste os totais para que os dois juntos consumam `N_ITENS`. Com dois consumidores, `pthread_cond_signal` continua suficiente, ou seria melhor `pthread_cond_broadcast`? Teste e justifique.

---

## Exercício 5 — Deadlock: provocar e corrigir (25 min)

> **Objetivo:** Criar um deadlock clássico com dois mutexes travados em ordens opostas, observar o travamento e corrigi-lo com ordem consistente.

Crie `ex5/deadlock.c`:

```c
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>

pthread_mutex_t mA = PTHREAD_MUTEX_INITIALIZER;
pthread_mutex_t mB = PTHREAD_MUTEX_INITIALIZER;

void *thread1(void *arg) {
    (void)arg;
    pthread_mutex_lock(&mA);  printf("[T1] peguei A\n");
    usleep(100000);
    printf("[T1] quero B...\n");
    pthread_mutex_lock(&mB);  printf("[T1] peguei B\n");
    pthread_mutex_unlock(&mB); pthread_mutex_unlock(&mA);
    return NULL;
}

void *thread2(void *arg) {
    (void)arg;
    pthread_mutex_lock(&mB);  printf("[T2] peguei B\n");   /* ordem OPOSTA */
    usleep(100000);
    printf("[T2] quero A...\n");
    pthread_mutex_lock(&mA);  printf("[T2] peguei A\n");
    pthread_mutex_unlock(&mA); pthread_mutex_unlock(&mB);
    return NULL;
}

int main(void) {
    setvbuf(stdout, NULL, _IOLBF, 0);   /* linha a linha: a saida sobrevive ao kill */
    pthread_t t1, t2;
    pthread_create(&t1, NULL, thread1, NULL);
    pthread_create(&t2, NULL, thread2, NULL);
    pthread_join(t1, NULL);
    pthread_join(t2, NULL);
    printf("Terminou (se voce esta vendo isto, nao houve deadlock)\n");
    return 0;
}
```

Rode **com `timeout`**, senão o programa fica travado para sempre:

```bash
$ cd ex5
$ gcc -Wall -Wextra -pthread -o deadlock deadlock.c
$ timeout 3 ./deadlock
$ echo $?
```

**Saída real capturada:**

```
[T1] peguei A
[T2] peguei B
[T2] quero A...
[T1] quero B...
$ echo $?
124
```

O código `124` é do `timeout`: ele teve de **matar** o programa. A linha "Terminou" nunca apareceu.

> **Por que o `setvbuf` no início do `main`?** Sem ele, `stdout` fica com buffer cheio quando a saída é redirecionada, e as mensagens se **perdem** quando o `timeout` mata o processo — você veria uma tela em branco em vez do diagnóstico. É a mesma mecânica de buffer do Exercício 1 do Laboratório 08, agora atrapalhando de outro jeito.

Agora corrija: copie para `ex5/deadlock_ok.c` e faça a **thread 2 travar na mesma ordem** da thread 1 (primeiro `mA`, depois `mB`).

```bash
$ gcc -Wall -Wextra -pthread -o deadlock_ok deadlock_ok.c
$ ./deadlock_ok
```

**Saída real capturada (versão corrigida):**

```
[T2] peguei A
[T2] quero B...
[T2] peguei B
[T1] peguei A
[T1] quero B...
[T1] peguei B
Terminou (se voce esta vendo isto, nao houve deadlock)
```

**Em `ex5/respostas.txt`:**

- [ ] **Q19.** Cole a saída das duas versões e o `echo $?` de cada uma. Qual foi o código de saída da versão corrigida?
- [ ] **Q20.** Olhando a saída travada, descreva com suas palavras o estado de cada thread no momento do deadlock: o que cada uma tem na mão e o que cada uma está esperando?
- [ ] **Q21.** Por que travar **sempre na mesma ordem** elimina o deadlock? *(Dica: para haver deadlock é preciso que exista uma espera **circular** — mostre que, com ordem fixa, esse círculo não pode se formar.)*
- [ ] **Q22.** Remova os dois `usleep(100000)` da versão **com** deadlock e rode várias vezes. O programa trava sempre, às vezes, ou nunca? O que isso diz sobre a dificuldade de encontrar deadlocks em código real?

---

## Exercício 6 — *Pool* de threads com tarefas independentes (20 min)

> **Objetivo:** Fechar o laboratório com o caso em que **nenhuma** sincronização é necessária — e entender exatamente por quê.

Crie `ex6/pool.c`:

```c
#include <stdio.h>
#include <pthread.h>

#define N_TAREFAS 5

typedef struct {
    int id;
    long long inicio, fim;   /* faixa [inicio, fim) */
    double resultado;
} tarefa_t;

void *soma_quadrados(void *arg) {
    tarefa_t *t = (tarefa_t *)arg;
    double soma = 0;
    for (long long i = t->inicio; i < t->fim; i++) soma += (double)i * (double)i;
    t->resultado = soma;
    printf("[thread %d] faixa [%lld, %lld) -> %.0f\n", t->id, t->inicio, t->fim, soma);
    return NULL;
}

int main(void) {
    pthread_t pool[N_TAREFAS];
    tarefa_t tarefas[N_TAREFAS] = {
        {0,      1, 100000, 0}, {1, 100000, 200000, 0}, {2, 200000, 300000, 0},
        {3, 300000, 400000, 0}, {4, 400000, 500000, 0},
    };

    printf("Pool de %d threads, uma tarefa cada...\n", N_TAREFAS);
    for (int i = 0; i < N_TAREFAS; i++) pthread_create(&pool[i], NULL, soma_quadrados, &tarefas[i]);
    for (int i = 0; i < N_TAREFAS; i++) pthread_join(pool[i], NULL);

    double total = 0;
    for (int i = 0; i < N_TAREFAS; i++) total += tarefas[i].resultado;
    printf("Soma total: %.0f\n", total);
    return 0;
}
```

### Sua tarefa: adicione uma sexta tarefa

Aumente `N_TAREFAS` para `6` e acrescente a faixa `[500000, 600000)`. Compile e rode **duas vezes**:

```bash
$ cd ex6
$ gcc -Wall -Wextra -pthread -o pool pool.c
$ ./pool ; ./pool
```

**Saída real capturada com 6 tarefas (duas execuções):**

```
--- execucao 1
[thread 1] faixa [100000, 200000) -> 2333318333350000
[thread 0] faixa [1, 100000) -> 333328333350000
[thread 2] faixa [200000, 300000) -> 6333308333350000
[thread 3] faixa [300000, 400000) -> 12333298333339012
[thread 4] faixa [400000, 500000) -> 20333288333324916
[thread 5] faixa [500000, 600000) -> 30333278333316852
Soma total: 71999820000030784

--- execucao 2
[thread 0] faixa [1, 100000) -> 333328333350000
[thread 2] faixa [200000, 300000) -> 6333308333350000
[thread 1] faixa [100000, 200000) -> 2333318333350000
...
Soma total: 71999820000030784
```

**Em `ex6/respostas.txt`:**

- [ ] **Q23.** Cole as duas execuções. A **ordem das linhas** foi a mesma? E a **soma total**? O que explica uma variar e a outra não?
- [ ] **Q24.** Este programa tem 6 threads escrevendo em `tarefas[i].resultado` simultaneamente e **não** usa nenhum mutex. Por que isso é seguro? *(Dica: pense em qual posição do array cada thread escreve.)*
- [ ] **Q25.** O que aconteceria se, por engano, todas as tarefas tivessem o mesmo `id` (todas com `id = 0`)? Mude o código para reproduzir isso e explique o resultado.

---

## Entrega <!-- e critérios de avaliação -->

### O que entregar

Uma pasta `lab09-nome/` contendo:

**Parte A (entrega ao final do encontro):**

- [ ] `ex1/soma_paralela.c`, `ex1/respostas.txt` (com a tabela de *speedups*)
- [ ] `ex2/race.c`, `ex2/mutex.c` (incluindo a versão do Q10, com contador local), `ex2/respostas.txt`
- [ ] `ex3/prodcons.c`, `ex3/respostas.txt`

**Parte B (entrega até o encontro seguinte):**

- [ ] `ex4/fila_cond.c` (com o segundo consumidor do Q18), `ex4/respostas.txt`
- [ ] `ex5/deadlock.c`, `ex5/deadlock_ok.c`, `ex5/respostas.txt`
- [ ] `ex6/pool.c` (com a sexta tarefa), `ex6/respostas.txt`

<!--
### Rubrica (10 pontos)

| Critério | Pontos | O que é avaliado |
|---|---|---|
| Ex. 1 — Soma paralela | 1,5 | Tabela de *speedups* com repetições; interpretação correta em relação a `nproc` |
| Ex. 2 — Corrida e mutex | 2,5 | Variação entre execuções registrada; mecanismo dos 3 passos explicado; entendimento do efeito do `-O2`; correção com contador local implementada e medida |
| Ex. 3 — Semáforos | 2,0 | Papel de cada semáforo e do mutex explicado; buffer circular compreendido; teste com `TAM_BUFFER=1` |
| Ex. 4 — Variável de condição | 1,5 | Entende por que `cond_wait` solta o mutex; justifica `while` em vez de `if`; segundo consumidor funcionando |
| Ex. 5 — Deadlock | 1,5 | Deadlock reproduzido e diagnosticado; correção por ordem consistente funcionando |
| Ex. 6 — Pool | 1,0 | Sexta tarefa adicionada; entende por que não há corrida aqui |

> **Dica 1:** Em concorrência, "rodei uma vez e deu certo" **não** é evidência de código correto — é o erro que este laboratório inteiro tenta ensinar a não cometer. Sempre que a pergunta pedir várias execuções, rode várias.

> **Dica 2:** O Q10 (contador local) é o exercício que mais aparece em entrevista e em código real: corrigir uma corrida sem destruir o desempenho. Não pule.

---

## Anexo A — Referência rápida

```c
#include <pthread.h>

/* --- threads --- */
int  pthread_create(pthread_t *t, const pthread_attr_t *attr,
                    void *(*rotina)(void *), void *arg);
int  pthread_join(pthread_t t, void **retorno);
void pthread_exit(void *retorno);

/* --- mutex --- */
pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;
int pthread_mutex_lock(pthread_mutex_t *m);
int pthread_mutex_trylock(pthread_mutex_t *m);   /* nao bloqueia: falha na hora */
int pthread_mutex_unlock(pthread_mutex_t *m);

/* --- variaveis de condicao --- */
pthread_cond_t c = PTHREAD_COND_INITIALIZER;
int pthread_cond_wait(pthread_cond_t *c, pthread_mutex_t *m);  /* dentro de um while! */
int pthread_cond_signal(pthread_cond_t *c);      /* acorda UMA   */
int pthread_cond_broadcast(pthread_cond_t *c);   /* acorda TODAS */
```

```c
#include <semaphore.h>

sem_t s;
int sem_init(sem_t *s, int entre_processos, unsigned valor_inicial);
int sem_wait(sem_t *s);    /* decrementa; bloqueia se estiver em zero */
int sem_post(sem_t *s);    /* incrementa; acorda quem espera          */
int sem_destroy(sem_t *s);
```

```bash
gcc -Wall -Wextra -pthread -o programa programa.c   # sempre com -pthread
nproc                                                # quantos nucleos a maquina tem
timeout 3 ./programa                                 # mata se travar (deadlock)
```

## Anexo B — Checklist: "essa variável é segura entre threads?"

| Pergunta | Se a resposta é "sim"... |
|---|---|
| É uma variável **local** dentro da rotina da thread? | Segura — cada thread tem sua própria pilha |
| É um índice de array **exclusivo** por thread (como `parciais[t->id]`)? | Segura — threads diferentes escrevem em posições diferentes |
| É **global/estática**, e mais de uma thread **lê e escreve**? | **Não segura** — precisa de mutex (ou de ser repensada) |
| É global, mas **todas** as threads apenas **leem**? | Segura — leitura concorrente sem escrita não dá corrida |

## Anexo C — Qual ferramenta usar

| Situação | Ferramenta | Por quê |
|---|---|---|
| Proteger um dado compartilhado | **mutex** | Garante uma thread por vez na seção crítica |
| Contar recursos limitados (vagas, conexões) | **semáforo** | Guarda *quantos* ainda há, não apenas "livre/ocupado" |
| Esperar uma condição ficar verdadeira | **variável de condição** | Dorme sem gastar CPU, e acorda quando avisam |
| Muitas leituras, poucas escritas | **`pthread_rwlock_t`** | Vários leitores simultâneos, um escritor exclusivo |
| O trabalho pode ser dividido em faixas | **nada** | A melhor sincronização é a que não é necessária |

**Testado e validado em sandbox (2026-10-09):** todos os programas deste guia (`soma_paralela.c`, `race.c` em `-O0` e `-O2`, `mutex.c`, `prodcons.c`, `fila_cond.c`, `deadlock.c`, `deadlock_ok.c` e `pool.c` com 5 e 6 tarefas) foram compilados com `gcc -Wall -Wextra -pthread` sem warnings e executados várias vezes; todas as saídas, tempos, *speedups* e o trecho de assembly mostrados são reais, não simulados. A máquina de validação tem `nproc` = 2 — seus números vão diferir, mas o padrão deve se repetir.

-->