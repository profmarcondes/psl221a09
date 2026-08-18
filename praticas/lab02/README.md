# Laboratório 02<br>Ambiente e Ferramentas de Programação (Parte 2)

<!--
**PSL221A09 — Programação em Sistemas Linux**

| | |
|---|---|
| **Duração estimada** | 80–90 minutos (parte prática) |
| **Formato** | Individual, no LSC/LSI |
| **Pré-requisito** | Git, Valgrind, strace, CMake e GCC instalados (Ubuntu/Debian) |
| **Entregável** | Pasta `lab02-nome/` com os 4 exercícios |

---
--->

Este guia consolida quatro ferramentas que complementam o GCC, o Make e o GDB: **Git** para controle de versão, **Valgrind** para encontrar vazamentos de memória, **strace** para rastrear chamadas de sistema, e **CMake** para descrever builds de forma portável. Siga os quatro exercícios em ordem. Sempre que houver uma seção de **saída esperada**, compare com o que apareceu no seu terminal antes de seguir em frente.

## Preparação do ambiente

1. Confirme que as ferramentas estão instaladas:

   ```bash
   $ git --version
   $ valgrind --version
   $ strace -V
   $ cmake --version
   $ gcc --version
   ```

   Se algum comando não for encontrado:

   ```bash
   $ sudo apt update
   $ sudo apt install git valgrind strace cmake build-essential
   ```

2. Configure sua identidade no Git (só precisa fazer uma vez por máquina):

   ```bash
   $ git config --global user.name "Seu Nome"
   $ git config --global user.email "seu.email@exemplo.com"
   ```

3. Crie a pasta de trabalho do laboratório:

   ```bash
   $ mkdir -p ~/psl/lab02/{ex1,ex2,ex3,ex4}
   $ cd ~/psl/lab02
   ```#include "estatistica.h"

double media(const int *valores, int tamanho) {
    double soma = 0.0;
    for (int i = 0; i < tamanho; i++) {
        soma += valores[i];
    }
    return soma / tamanho;
}

int maior(const int *valores, int tamanho) {
    int max = valores[0];
    for (int i = 1; i < tamanho; i++) {
        if (valores[i] > max) max = valores[i];
    }
    return max;
}

int menor(const int *valores, int tamanho) {
    int min = valores[0];
    for (int i = 1; i < tamanho; i++) {
        if (valores[i] < min) min = valores[i];
    }
    return min;
}


---

## Exercício 1 — Git: fluxo básico com branches (20 min)

> **Objetivo:** Praticar o ciclo `init` → `add` → `commit` → `branch` → `merge`, incluindo a resolução manual de um conflito.

### Parte A — Primeiro repositório

Entre em `ex1/` e crie um repositório com um arquivo de configuração simples:

```bash
$ cd ex1
$ git init -b main
$ cat > config.txt << 'EOF'
modo=texto
tamanho_buffer=64
EOF
$ git add config.txt
$ git commit -m "Adiciona configuracao inicial"
```

**Responda no arquivo `ex1/respostas.txt`:**

- [ ] **Q1.** O que `git status` mostrava **antes** do `git add`? E depois?
- [ ] **Q2.** O que aparece em `git log --oneline` depois do commit?

### Parte B — Trabalhando em uma branch

Crie uma branch para uma mudança específica, sem afetar `main` diretamente:

```bash
$ git switch -c feature-buffer
$ sed -i 's/tamanho_buffer=64/tamanho_buffer=4096/' config.txt
$ git commit -am "Aumenta buffer para 4096"
```

Agora volte para `main` e faça **outra** mudança na **mesma linha**:

```bash
$ git switch main
$ sed -i 's/tamanho_buffer=64/tamanho_buffer=128/' config.txt
$ git commit -am "Ajusta buffer para 128"
```

Visualize as duas linhas de desenvolvimento divergindo:

```bash
$ git log --oneline --graph --all
```

**Continue em `ex1/respostas.txt`:**

- [ ] **Q3.** Quantos commits aparecem no total? Quantas branches existem agora?

### Parte C — Merge com conflito

Tente juntar a branch `feature-buffer` em `main`:

```bash
$ git merge feature-buffer
```

**Saída esperada:**

```
Auto-merging config.txt
CONFLICT (content): Merge conflict in config.txt
Automatic merge failed; fix conflicts and then commit the result.
```

Abra `config.txt` — ele agora contém marcações de conflito:

```
modo=texto
<<<<<<< HEAD
tamanho_buffer=128
=======
tamanho_buffer=4096
>>>>>>> feature-buffer
```

Resolva o conflito manualmente: edite o arquivo, escolha `tamanho_buffer=4096` (mantendo a mudança da branch), remova as três linhas de marcação (`<<<<<<<`, `=======`, `>>>>>>>`), e finalize o merge:

```bash
$ git add config.txt
$ git commit -m "Resolve conflito: usa buffer de 4096"
$ git log --oneline --graph --all
```

**Finalize em `ex1/respostas.txt`:**

- [ ] **Q4.** Cole a saída do `git log --oneline --graph --all` final. Quantos commits o merge criou?
- [ ] **Q5.** Se você tivesse escolhido manter `tamanho_buffer=128` em vez de `4096` na resolução, o conteúdo final do arquivo mudaria? Por quê?

> **Dica:** Se quiser recomeçar o exercício do zero, apague a pasta `ex1` e volte ao início — não tem problema errar um merge enquanto está aprendendo.

---

## Exercício 2 — Valgrind: encontrando e corrigindo um memory leak (20–25 min)

> **Objetivo:** Usar `valgrind --leak-check=full` para localizar exatamente a linha de um `malloc()` sem `free()` correspondente, e confirmar a correção.

Crie `ex2/leak.c`:

```c
#include <stdio.h>
#include <stdlib.h>

int *processa_lote(int tamanho) {
    int *buffer = malloc(tamanho * sizeof(int));
    for (int i = 0; i < tamanho; i++) {
        buffer[i] = i * i;
    }
    return buffer;
}

int main(void) {
    int *resultado = processa_lote(100);
    printf("Primeiro elemento: %d\n", resultado[0]);
    printf("Ultimo elemento: %d\n", resultado[99]);
    /* falta liberar "resultado" aqui */
    return 0;
}
```

Compile **com símbolos de depuração** (o Valgrind precisa deles para apontar a linha certa) e rode normalmente primeiro:

```bash
$ gcc -g -Wall -Wextra -std=c11 -o leak leak.c
$ ./leak
```

**Saída esperada:**

```
Primeiro elemento: 0
Ultimo elemento: 9801
```

O programa parece funcionar perfeitamente — não trava, não imprime erro. É exatamente por isso que memory leaks são perigosos: eles não avisam. Agora rode sob o Valgrind:

```bash
$ valgrind --leak-check=full ./leak
```

**Saída esperada (resumida):**

```
==12291== HEAP SUMMARY:
==12291==     in use at exit: 400 bytes in 1 blocks
==12291==   total heap usage: 2 allocs, 1 frees, 4,496 bytes allocated
==12291==
==12291== 400 bytes in 1 blocks are definitely lost in loss record 1 of 1
==12291==    at 0x4846828: malloc (vg_replace_malloc.c:...)
==12291==    by 0x109188: processa_lote (leak.c:5)
==12291==    by 0x1091D9: main (leak.c:13)
==12291==
==12291== LEAK SUMMARY:
==12291==    definitely lost: 400 bytes in 1 blocks
==12291==      indirectly lost: 0 bytes in 0 blocks
```

**Responda no arquivo `ex2/respostas.txt`:**

- [ ] **Q6.** Quantos bytes foram perdidos (`definitely lost`)? Bate com `100 * sizeof(int)`?
- [ ] **Q7.** Qual linha exata do `leak.c` o Valgrind aponta como origem do `malloc()` que vazou?
- [ ] **Q8.** `total heap usage` mostra `2 allocs, 1 frees`. Por que há 2 alocações se o código só tem um `malloc()` explícito? (dica: pense no que a própria `libc`/`printf` pode alocar internamente)

### Corrija e confirme

Edite `ex2/leak.c`, adicionando `free(resultado);` antes do `return 0;`. Recompile e rode o Valgrind de novo:

```bash
$ gcc -g -Wall -Wextra -std=c11 -o leak leak.c
$ valgrind --leak-check=full ./leak
```

**Saída esperada:**

```
==12655== HEAP SUMMARY:
==12655==     in use at exit: 0 bytes in 0 blocks
==12655==   total heap usage: 2 allocs, 2 frees, 4,496 bytes allocated
==12655==
==12655== All heap blocks were freed -- no leaks are possible
```

Se a sua saída terminou assim, o exercício está concluído.

---

## Exercício 3 — strace: por que meu programa está lento? (20–25 min)

> **Objetivo:** Usar `strace -c` para medir o número de chamadas de sistema de um programa e demonstrar, na prática, como o tamanho de um buffer de leitura afeta o desempenho.

Crie `ex3/copia.c` — um programa que copia um arquivo usando `read()`/`write()` diretamente (sem passar pela camada com buffer da `stdio`):

```c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>

#define TAM_BUFFER 64

int main(int argc, char *argv[]) {
    if (argc != 3) {
        fprintf(stderr, "Uso: %s origem destino\n", argv[0]);
        return 1;
    }

    int origem = open(argv[1], O_RDONLY);
    int destino = open(argv[2], O_WRONLY | O_CREAT | O_TRUNC, 0644);
    if (origem < 0 || destino < 0) {
        perror("Erro ao abrir arquivo");
        return 1;
    }

    char buffer[TAM_BUFFER];
    ssize_t bytes_lidos;
    while ((bytes_lidos = read(origem, buffer, TAM_BUFFER)) > 0) {
        write(destino, buffer, (size_t) bytes_lidos);
    }

    close(origem);
    close(destino);
    printf("Copia concluida.\n");
    return 0;
}
```

Compile e crie um arquivo de teste de ~20 KB:

```bash
$ gcc -g -Wall -Wextra -std=c11 -o copia copia.c
$ head -c 20000 /dev/urandom > origem.dat
```

Rastreie a execução contando as chamadas de sistema:

```bash
$ strace -c ./copia origem.dat destino.dat
```

**Saída esperada (resumida, com `TAM_BUFFER 64`):**

```
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 32.14    0.000909           2       314           write
 31.36    0.000887           2       315           read
 ...
------ ----------- ----------- --------- --------- ----------------
100.00    0.002828           4       665         1 total
```

Confirme que a cópia foi feita corretamente:

```bash
$ diff origem.dat destino.dat && echo "Arquivos identicos"
```

**Responda no arquivo `ex3/respostas.txt`:**

- [ ] **Q9.** Quantas chamadas de `read()` apareceram? Isso bate com aproximadamente `tamanho_do_arquivo / TAM_BUFFER`?
- [ ] **Q10.** Rode `strace -e trace=open,openat,read,write,close ./copia origem.dat destino.dat` (sem o `-c`) e cole as 5 primeiras linhas da saída.

### Agora aumente o buffer

Altere `#define TAM_BUFFER 64` para `#define TAM_BUFFER 4096`, recompile e rode o `strace -c` de novo:

```bash
$ gcc -g -Wall -Wextra -std=c11 -o copia copia.c
$ strace -c ./copia origem.dat destino.dat
```

**Continue em `ex3/respostas.txt`:**

- [ ] **Q11.** Quantas chamadas de `read()` e `write()` apareceram agora, com o buffer de 4096 bytes? Quantas vezes o número de chamadas caiu, comparado ao buffer de 64 bytes?
- [ ] **Q12.** Em que situação real (fora da sala de aula) um número excessivo de chamadas de `read()`/`write()` pode ser um problema sério?

---

## Exercício 4 — CMake: build de um projeto com múltiplos arquivos (20–25 min)

> **Objetivo:** Escrever um `CMakeLists.txt` do zero para um projeto C com múltiplos arquivos-fonte, usando o fluxo de build fora da árvore de código (*out-of-source*).

Copie os três arquivos abaixo para `ex4/`.

**`ex4/estatistica.h`**

```c
#ifndef ESTATISTICA_H
#define ESTATISTICA_H

double media(const int *valores, int tamanho);
int maior(const int *valores, int tamanho);
int menor(const int *valores, int tamanho);

#endif
```

**`ex4/estatistica.c`**

```c
#include "estatistica.h"

double media(const int *valores, int tamanho) {
    double soma = 0.0;
    for (int i = 0; i < tamanho; i++) {
        soma += valores[i];
    }
    return soma / tamanho;
}

int maior(const int *valores, int tamanho) {
    int max = valores[0];
    for (int i = 1; i < tamanho; i++) {
        if (valores[i] > max) max = valores[i];
    }
    return max;
}

int menor(const int *valores, int tamanho) {
    int min = valores[0];
    for (int i = 1; i < tamanho; i++) {
        if (valores[i] < min) min = valores[i];
    }
    return min;
}
```

**`ex4/main.c`**

```c
#include <stdio.h>
#include "estatistica.h"

int main(void) {
    int notas[] = {7, 8, 5, 9, 6, 10, 4};
    int tamanho = sizeof(notas) / sizeof(notas[0]);

    printf("Notas da turma: ");
    for (int i = 0; i < tamanho; i++) {
        printf("%d ", notas[i]);
    }
    printf("\n");

    printf("Media:  %.2f\n", media(notas, tamanho));
    printf("Maior:  %d\n", maior(notas, tamanho));
    printf("Menor:  %d\n", menor(notas, tamanho));

    return 0;
}
```

### Sua tarefa

Escreva `ex4/CMakeLists.txt` **do zero** que atenda a estes requisitos:

- [ ] **R1.** `cmake_minimum_required(VERSION 3.16)` e `project(turma C)`.
- [ ] **R2.** Define o padrão da linguagem C11 (`CMAKE_C_STANDARD`).
- [ ] **R3.** Um alvo executável `turma`, gerado a partir de `main.c` e `estatistica.c`.
- [ ] **R4.** As flags `-Wall -Wextra -g` aplicadas ao alvo `turma` (pesquise `target_compile_options`).

Teste seu `CMakeLists.txt` com um build fora da árvore de código:

```bash
$ cd ex4
$ mkdir build && cd build
$ cmake ..
$ make
$ ./turma
```

**Saída esperada:**

```
Notas da turma: 7 8 5 9 6 10 4
Media:  7.00
Maior:  10
Menor:  4
```

Agora teste a recompilação incremental, como fizemos com o Make:

```bash
$ touch ../main.c
$ make        # deve recompilar so main.c, nao estatistica.c
```

**Em `ex4/respostas.txt`:**

- [ ] **Q13.** Qual arquivo o `make` recompilou depois do `touch ../main.c`? O CMake preservou o comportamento incremental do Make?
- [ ] **Q14.** O que aconteceria se você rodasse `cmake ..` e `make` **dentro** da pasta `ex4/` (sem criar `build/`)? Por que a prática recomendada é sempre usar uma pasta separada?
- [ ] **Q15.** Como você apagaria completamente os artefatos de build e recomeçaria do zero, sem risco de apagar código-fonte?

<!-- 
Se travar, o gabarito completo está no [Anexo A](#anexo-a--gabarito-do-cmakeliststxt).

---

## Entrega e critérios de avaliação

### O que entregar

Uma pasta `laboratorio02-nomes-da-dupla/` contendo:

- [ ] `ex1/config.txt` (versão final), `ex1/respostas.txt`
- [ ] `ex2/leak.c` (já corrigido, com `free()`), `ex2/respostas.txt`
- [ ] `ex3/copia.c` (versão final, com `TAM_BUFFER 4096`), `ex3/respostas.txt`
- [ ] `ex4/main.c`, `ex4/estatistica.c`, `ex4/estatistica.h`, `ex4/CMakeLists.txt`, `ex4/respostas.txt`

### Rubrica (10 pontos)

| Critério | Pontos | O que é avaliado |
|---|---|---|
| Exercício 1 — Git | 2,5 | Histórico de commits e branches correto; conflito resolvido corretamente (não apagando código de nenhum dos dois lados sem justificar) |
| Exercício 2 — Valgrind | 2,5 | Leak localizado com a ferramenta (linha correta identificada); correção confirmada por uma segunda execução do Valgrind sem erros |
| Exercício 3 — strace | 2,5 | Contagem de syscalls correta nos dois cenários de buffer; conclusão sobre o efeito do tamanho do buffer bem justificada |
| Exercício 4 — CMake | 2,5 | `CMakeLists.txt` atende aos requisitos (R1–R4); build fora da árvore funciona; recompilação incremental testada |

> **Dica:** Assim como no laboratório anterior, o que conta é **mostrar o uso das ferramentas** — um `CMakeLists.txt` copiado sem entender, ou um conflito de merge resolvido apagando um dos dois lados sem motivo, não demonstra o aprendizado esperado.

---

## Anexo A — Gabarito do `CMakeLists.txt`

*Consulte apenas depois de tentar o Exercício 4 por conta própria.*

```cmake
cmake_minimum_required(VERSION 3.16)
project(turma C)

set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)

add_executable(turma main.c estatistica.c)
target_compile_options(turma PRIVATE -Wall -Wextra -g)
```

**Testado e validado:** `cmake ..`, `make`, `./turma` (saída correta), `touch ../main.c && make` (recompila só `main.c.o`).

## Anexo B — `leak.c` corrigido

```c
#include <stdio.h>
#include <stdlib.h>

int *processa_lote(int tamanho) {
    int *buffer = malloc(tamanho * sizeof(int));
    for (int i = 0; i < tamanho; i++) {
        buffer[i] = i * i;
    }
    return buffer;
}

int main(void) {
    int *resultado = processa_lote(100);
    printf("Primeiro elemento: %d\n", resultado[0]);
    printf("Ultimo elemento: %d\n", resultado[99]);
    free(resultado);
    return 0;
}
```

## Anexo C — Referência rápida de comandos

```bash
# Git
git init -b main              # cria repositorio com branch "main"
git status                    # o que mudou?
git add <arquivo>             # marca para o proximo commit
git commit -m "mensagem"      # registra uma fotografia do projeto
git switch -c <branch>        # cria e muda para uma branch
git merge <branch>            # junta uma branch na atual
git log --oneline --graph --all

# Valgrind
valgrind --leak-check=full ./programa

# strace
strace ./programa                          # rastreia tudo
strace -c ./programa                       # resumo por syscall
strace -e trace=open,read,write ./programa # filtra so I/O

# CMake
mkdir build && cd build
cmake ..
make
```
-->