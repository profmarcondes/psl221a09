# Laboratório 04: <br> Chamadas de Sistema: Fundamentos

<!-- **PSL221A09 — Programação em Sistemas Linux**

| | |
|---|---|
| **Duração estimada** | 80–90 minutos (parte prática) |
| **Formato** | Duplas ou trios, no LSC/LSI |
| **Pré-requisito** | GCC e strace instalados (Ubuntu/Debian) |
| **Entregável** | Pasta `laboratorio04-nomes/` com os 3 exercícios |

---
-->

Este guia consolida o mecanismo de **chamada de sistema** (*syscall*), a diferença entre uma syscall e uma função de biblioteca, e o tratamento de erros com `errno`. Siga os três exercícios em ordem. Sempre que houver uma seção de **saída esperada**, compare com o que apareceu no seu terminal antes de seguir em frente.

## Preparação do ambiente

1. Confirme que as ferramentas estão instaladas:

   ```bash
   $ gcc --version
   $ strace -V
   ```

   Se `strace` não for encontrado: `sudo apt install strace`.

2. Crie a pasta de trabalho do laboratório:

   ```bash
   $ mkdir -p working_dir/lab04-nome/{ex1,ex2,ex3}
   $ cd working_dir/lab04-nome/
   ```

---

## Exercício 1 — Chamando syscalls diretamente <!--(15–20 min)-->

> **Objetivo:** Usar a função genérica `syscall()` para invocar chamadas de sistema sem passar pelos wrappers de conveniência da libc, e comparar o resultado com os wrappers de sempre.

Crie `ex1/syscall_direta.c`:

```c
#define _GNU_SOURCE
#include <stdio.h>
#include <unistd.h>
#include <sys/syscall.h>

int main(void) {
    const char msg[] = "Ola via syscall() direta!\n";

    /* syscall() generica: numero da syscall + argumentos */
    long n = syscall(SYS_write, STDOUT_FILENO, msg, sizeof(msg) - 1);
    printf("syscall(SYS_write) escreveu %ld bytes\n", n);

    long pid = syscall(SYS_getpid);
    long uid = syscall(SYS_getuid);

    printf("PID via syscall(SYS_getpid): %ld\n", pid);
    printf("UID via syscall(SYS_getuid): %ld\n", uid);

    printf("PID via getpid() da libc:    %ld\n", (long) getpid());
    printf("UID via getuid() da libc:    %ld\n", (long) getuid());

    return 0;
}
```

Compile e rode:

```bash
$ gcc -g -Wall -Wextra -std=c11 -o syscall_direta syscall_direta.c
$ ./syscall_direta
```

**Saída esperada (PID e UID variam):**

```
Ola via syscall() direta!
syscall(SYS_write) escreveu 26 bytes
PID via syscall(SYS_getpid): 20433
UID via syscall(SYS_getuid): 0
PID via getpid() da libc:    20433
UID via getuid() da libc:    0
```

Agora rastreie a execução com `strace` e observe as chamadas de fato feitas ao kernel:

```bash
$ strace ./syscall_direta 2>&1 | grep -E "write|getpid|getuid"
```

**Em `ex1/respostas.txt`:**

- [ ] **Q1.** O PID/UID impresso via `syscall(SYS_getpid)` bateu com o impresso via `getpid()` da libc? Isso já era esperado — explique por quê.
- [ ] **Q2.** Na saída do `strace`, quantas vezes a syscall `write` aparece no total? (dica: cada `printf` também acaba virando `write`, então não é só a chamada explícita do seu código)
- [ ] **Q3.** Pesquise (`man 2 syscalls` ou online) o número da syscall `write` na arquitetura x86-64. Qual é?

---

## Exercício 2 — `printf` vs. `write`: medindo a diferença <!--(25–30 min)-->

> **Objetivo:** Comparar, com `strace -c`, o número de chamadas de sistema geradas por `printf` (com buffer da libc) contra `write()` chamado manualmente uma vez por linha.

Crie os dois programas abaixo em `ex2/`.

**`ex2/muitos_printf.c`**

```c
#include <stdio.h>

int main(void) {
    for (int i = 0; i < 200000; i++) {
        printf("linha %d\n", i);
    }
    return 0;
}
```

**`ex2/muitos_write.c`**

```c
#include <unistd.h>
#include <stdio.h>

int main(void) {
    for (int i = 0; i < 200000; i++) {
        char buf[32];
        int n = snprintf(buf, sizeof(buf), "linha %d\n", i);
        if (write(STDOUT_FILENO, buf, (size_t) n) == -1) {
            return 1;
        }
    }
    return 0;
}
```

Compile os dois **com otimização** (para não medir o custo do modo debug):

```bash
$ gcc -O2 -Wall -Wextra -std=c11 -o muitos_printf muitos_printf.c
$ gcc -O2 -Wall -Wextra -std=c11 -o muitos_write muitos_write.c
```

Meça o tempo de execução de cada um (saída redirecionada para `/dev/null`, para não gastar tempo desenhando 200 mil linhas no terminal):

```bash
$ time ./muitos_printf > /dev/null
$ time ./muitos_write > /dev/null
```

Agora conte as chamadas de sistema de cada versão:

```bash
$ strace -c ./muitos_printf > /dev/null
$ strace -c ./muitos_write > /dev/null
```

**Saída esperada (resumida, os números exatos variam por máquina):**

```
# muitos_printf
100.00    0.006230           9       642         2 total     (~608 chamadas de write)

# muitos_write
100.00    3.319794          16    200029         1 total     (200000 chamadas de write)
```

**Responda no arquivo `ex2/respostas.txt`:**

- [ ] **Q4.** Quantas chamadas de `write` o `muitos_printf` gerou? Isso é muito menor que 200\,000 — qual mecanismo da libc explica essa diferença?
- [ ] **Q5.** Quantas chamadas de `write` o `muitos_write` gerou? Bate com o número de iterações do laço?
- [ ] **Q6.** Compare o tempo do `time` **sem** `strace` (execução normal) para as duas versões. A diferença é tão dramática quanto a diferença no tempo mostrado *dentro* do `strace -c`? O que isso sugere sobre o próprio `strace` ter um custo de observação?
- [ ] **Q7.** Se você tivesse que escrever uma função que grava milhares de linhas em um arquivo, o que esse experimento sugere sobre usar `write()` direto, linha a linha, vs.\ usar `fprintf`/um buffer próprio?

---

## Exercício 3 — Provocando e tratando erros de syscall <!--(20–25 min)-->

> **Objetivo:** Usar `errno`, `perror()` e `strerror()` para diagnosticar por que uma syscall falhou, sem "adivinhar" pela mensagem genérica.

Crie `ex3/erro_arquivo.c`:

```c
#include <stdio.h>
#include <fcntl.h>
#include <errno.h>
#include <string.h>
#include <unistd.h>

void tenta_abrir(const char *caminho, int flags) {
    int fd = open(caminho, flags);
    if (fd == -1) {
        printf("open(\"%s\") falhou: errno=%d (%s)\n",
               caminho, errno, strerror(errno));
    } else {
        printf("open(\"%s\") funcionou, fd=%d\n", caminho, fd);
        close(fd);
    }
}

int main(void) {
    tenta_abrir("/caminho/que/nao/existe.txt", O_RDONLY);
    tenta_abrir("/tmp", O_WRONLY);          /* diretorio: nao pode abrir p/ escrita */
    tenta_abrir("/etc/passwd", O_RDONLY);   /* deve funcionar */
    return 0;
}
```

Compile e rode:

```bash
$ gcc -g -Wall -Wextra -std=c11 -o erro_arquivo erro_arquivo.c
$ ./erro_arquivo
```

**Saída esperada:**

```
open("/caminho/que/nao/existe.txt") falhou: errno=2 (No such file or directory)
open("/tmp") falhou: errno=21 (Is a directory)
open("/etc/passwd") funcionou, fd=3
```

**Responda no arquivo `ex3/respostas.txt`:**

- [ ] **Q8.** Qual constante de `<errno.h>` corresponde ao valor `2`? E ao valor `21`? (dica: `man errno`, ou `#include <errno.h>` e procure `ENOENT`/`EISDIR` no cabeçalho)

### Agora com `perror`

Adicione, logo depois do primeiro `tenta_abrir` no `main`, uma chamada extra usando `perror` em vez de `strerror`:

```c
int fd_extra = open("/outro/caminho/inexistente.txt", O_RDONLY);
if (fd_extra == -1) {
    perror("Falha ao abrir arquivo extra");
}
```

Recompile e rode de novo.

**Continue em `ex3/respostas.txt`:**

- [ ] **Q9.** Qual foi a mensagem impressa pelo `perror`? Compare o formato dela com o que você monta manualmente com `strerror(errno)` — qual das duas formas você preferiria usar em um programa real, e por quê?
- [ ] **Q10.** Modifique `tenta_abrir` para tentar criar um arquivo com `O_CREAT | O_EXCL` em um caminho que **já existe** (por exemplo, tente criar de novo o `/tmp` — ou melhor, crie um arquivo `ex3/existente.txt` antes e tente recriá-lo com essas flags). Qual `errno` apareceu? Isso bate com a tabela vista em aula (`EEXIST`)?

<!-- >> **Dica:** Se você rodar os testes como `root`, algumas combinações de permissão podem se comportar diferente do esperado — o `root` ignora a maior parte das checagens de permissão do sistema de arquivos. Os erros `ENOENT` (arquivo não existe) e `EISDIR` (é um diretório) usados aqui acontecem **independentemente** de privilégio, então são seguros para testar em qualquer conta.-->

---

## Entrega <!-- e critérios de avaliação -->

### O que entregar

Uma pasta `lab04-nome/`, compactada, contendo:

- [ ] `ex1/syscall_direta.c`, `ex1/respostas.txt`
- [ ] `ex2/muitos_printf.c`, `ex2/muitos_write.c`, `ex2/respostas.txt`
- [ ] `ex3/erro_arquivo.c` (com a versão final, incluindo o `perror` e o teste de `EEXIST`), `ex3/respostas.txt`

<!-- 
### Rubrica (10 pontos)

| Critério | Pontos | O que é avaliado |
|---|---|---|
| Exercício 1 — `syscall()` direta | 3,0 | Programa compila e roda; respostas mostram entendimento de syscall vs.\ wrapper |
| Exercício 2 — `printf` vs.\ `write` | 4,0 | Medições feitas corretamente com `strace -c` e `time`; conclusão sobre buffering bem justificada |
| Exercício 3 — `errno`/`perror` | 3,0 | Erros diagnosticados corretamente (`ENOENT`, `EISDIR`, `EEXIST`); uso correto de `strerror`/`perror` |

> **Dica:** O que conta aqui não é só rodar os programas, mas **explicar** por que os números saem do jeito que saem — em especial no Exercício 2, a resposta "porque sim" não demonstra o aprendizado esperado.

---

## Anexo A — Referência rápida

```c
#include <unistd.h>
#include <sys/syscall.h>
#include <errno.h>
#include <string.h>

/* syscall generica */
long syscall(long numero, ...);

/* apos uma syscall (ou wrapper de libc) falhar, retornando -1: */
if (retorno == -1) {
    fprintf(stderr, "errno = %d (%s)\n", errno, strerror(errno));
    perror("mensagem de contexto");   /* imprime "mensagem: descricao do erro" no stderr */
}
```

```bash
# strace: contar syscalls
strace -c ./programa

# strace: rastrear tudo em tempo real
strace ./programa

# medir tempo de execucao (sem strace, que distorce a medida)
time ./programa
```

## Anexo B — `erro_arquivo.c` completo (com `perror` e teste de `EEXIST`)

```c
#include <stdio.h>
#include <fcntl.h>
#include <errno.h>
#include <string.h>
#include <unistd.h>

void tenta_abrir(const char *caminho, int flags) {
    int fd = open(caminho, flags);
    if (fd == -1) {
        printf("open(\"%s\") falhou: errno=%d (%s)\n",
               caminho, errno, strerror(errno));
    } else {
        printf("open(\"%s\") funcionou, fd=%d\n", caminho, fd);
        close(fd);
    }
}

int main(void) {
    tenta_abrir("/caminho/que/nao/existe.txt", O_RDONLY);

    int fd_extra = open("/outro/caminho/inexistente.txt", O_RDONLY);
    if (fd_extra == -1) {
        perror("Falha ao abrir arquivo extra");
    }

    tenta_abrir("/tmp", O_WRONLY);
    tenta_abrir("/etc/passwd", O_RDONLY);

    /* Testa O_CREAT | O_EXCL em um arquivo que ja existe */
    int fd = open("existente.txt", O_CREAT | O_WRONLY, 0644);
    if (fd != -1) close(fd);   /* garante que o arquivo existe */

    fd = open("existente.txt", O_CREAT | O_EXCL | O_WRONLY, 0644);
    if (fd == -1) {
        printf("O_CREAT|O_EXCL em arquivo existente: errno=%d (%s)\n",
               errno, strerror(errno));
    } else {
        close(fd);
    }

    return 0;
}
```

**Testado e validado:** os erros `ENOENT` (caminho inexistente), `EISDIR` (abrir diretório para escrita) e `EEXIST` (`O_CREAT|O_EXCL` em arquivo existente) foram reproduzidos e conferem com a tabela de `errno` vista em aula.

-->