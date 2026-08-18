
# Laboratório 01:<br>Ambiente e Ferramentas de Programação (Parte 1)

<!--
**PSL221A09 — Programação em Sistemas Linux**

| | |
|---|---|
| **Duração estimada** | 70–80 minutos (parte prática do encontro) |
| **Formato** | Duplas ou trios, no LSC/LSI |
| **Pré-requisito** | GCC, Make e GDB instalados (Ubuntu/Debian) |
| **Entregável** | Pasta `lab01-nome/` com os 3 exercícios |
---
-->

Este guia consolida os conceitos vistos hoje: compilação com **GCC**, automação de build com **Make** e depuração com **GDB**. Siga os três exercícios em ordem — cada um assume que o anterior foi concluído. Sempre que houver uma seção de **saída esperada**, compare com o que apareceu no seu terminal antes de seguir em frente.

## Preparação do ambiente

1. Abra um terminal e confirme que as ferramentas estão instaladas:

   ```bash
   $ gcc --version
   $ make --version
   $ gdb --version
   ```

   Se algum comando não for encontrado: `sudo apt install build-essential gdb`.

2. Crie a pasta de trabalho do encontro:

   Execute este comando na raiz do repositório.

   ```bash
   $ mkdir -p working_dir/lab01-nome/{ex1,ex2,ex3}
   $ cd working_dir/lab01-nome
   ```
   
---

## Exercício 1 — GCC: flags de compilação (20 min)

> **Objetivo:** Entender o que cada flag do GCC faz na prática: warnings que pegam bugs, e otimizações que mudam o desempenho sem mudar o resultado.

### Parte A — Warnings

Crie o arquivo `ex1/warnings.c` com o conteúdo abaixo (o bug é proposital):

```c
#include <stdio.h>

int calcula_soma(int *v, int tamanho) {
    int soma = 0;
    int i;
    for (i = 0; i < tamanho; i++) {
        soma = soma + v[i];
    }
    return soma;
}

int compara_limite(int valor, unsigned int limite) {
    if (valor < limite) {
        return 1;
    }
    return 0;
}

int main() {
    int notas[5] = {8, 7, 9, 6, 10};
    int total = calcula_soma(notas, 5);
    float media = total / 5;
    int fator_bonus = 2;

    printf("Total: %d\n", total);
    printf("Media: %.2f\n", media);
    printf("Esta acima do limite? %d\n", compara_limite(total, 30));

    return 0;
}
```

Compile a mesma fonte de quatro formas diferentes e observe a diferença:

```bash
$ gcc warnings.c -o v1
$ gcc -Wall warnings.c -o v2
$ gcc -Wall -Wextra warnings.c -o v3
$ gcc -Wall -Wextra -Werror warnings.c -o v4
```

**Responda no arquivo `ex1/respostas.txt`:**

- [ ] **Q1.** Quantos warnings o `-Wall` sozinho mostrou? Qual variável ele reclamou?
- [ ] **Q2.** O que `-Wextra` encontrou a mais que o `-Wall` não tinha encontrado?
- [ ] **Q3.** O comando com `-Werror` gerou um executável (`v4`)? Por quê?
<!-- - [ ] **Q4.** O programa tem um bug de **divisão inteira** na linha da variável `media` (o resultado devia ser `8.00`, não um número inteiro "arredondado"). Nenhuma das flags de warning aponta esse problema. Por que você acha que o compilador não consegue detectar esse tipo de bug?
-->

### Parte B — Otimização

Crie `ex1/otimizacao.c`:

```c
#include <stdio.h>

#define N 450

static double a[N][N], b[N][N], c[N][N];

void inicializa(void) {
    for (int i = 0; i < N; i++) {
        for (int j = 0; j < N; j++) {
            a[i][j] = (double)((i + j) % 7);
            b[i][j] = (double)((i * j) % 5);
            c[i][j] = 0.0;
        }
    }
}

void multiplica(void) {
    for (int i = 0; i < N; i++) {
        for (int j = 0; j < N; j++) {
            double soma = 0.0;
            for (int k = 0; k < N; k++) {
                soma += a[i][k] * b[k][j];
            }
            c[i][j] = soma;
        }
    }
}

int main(void) {
    inicializa();
    multiplica();

    double checksum = 0.0;
    for (int i = 0; i < N; i++) {
        checksum += c[i][i];
    }

    printf("Multiplicacao de matrizes %dx%d concluida. Checksum: %.2f\n", N, N, checksum);
    return 0;
}
```

Compile com dois níveis de otimização diferentes e meça o tempo de execução:

```bash
$ gcc -O0 otimizacao.c -o otim_O0
$ gcc -O2 otimizacao.c -o otim_O2
$ time ./otim_O0
$ time ./otim_O2
```

**Saída esperada:**

```
Multiplicacao de matrizes 450x450 concluida. Checksum: 971967.00

real  0m0.378s   # aproximado, com -O0
real  0m0.071s   # aproximado, com -O2 (cerca de 5x mais rápido)
```

**Continue em `ex1/respostas.txt`:**

- [ ] **Q5.** Qual foi o tempo real (`real`) com `-O0` e com `-O2` na sua máquina?
- [ ] **Q6.** O **checksum** impresso foi igual nas duas versões? O que isso prova sobre o que a otimização muda (e o que ela *não* muda)?
- [ ] **Q7.** Em qual situação você compilaria com `-O0` em vez de `-O2`?

> **Dica:** Se os tempos ficarem parecidos entre `-O0` e `-O2` na sua máquina, aumente o valor de `N` (por exemplo, para 600) e recompile os dois.

---

## Exercício 2 — Make: automatizando o build (25 min)

> **Objetivo:** Escrever, do zero, um Makefile funcional para um projeto com múltiplos arquivos `.c`, usando variáveis, regras de padrão e alvos `.PHONY`.

Copie os três arquivos abaixo para `ex2/`.

**`ex2/estatistica.h`**

```c
#ifndef ESTATISTICA_H
#define ESTATISTICA_H

double media(const int *valores, int tamanho);
int maior(const int *valores, int tamanho);
int menor(const int *valores, int tamanho);

#endif
```

**`ex2/estatistica.c`**

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

**`ex2/main.c`**

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

Escreva `ex2/Makefile` **do zero** que atenda a estes requisitos:

- [ ] **R1.** Variáveis `CC` (compilador) e `CFLAGS` contendo pelo menos `-Wall -Wextra -g -std=c11`.
- [ ] **R2.** Um alvo `all` que gera o executável `turma`.
- [ ] **R3.** Uma **regra de padrão** (*pattern rule*) `%.o: %.c` que compila qualquer `.c` em `.o` — sem repetir a regra para `main.c` e `estatistica.c` separadamente.
- [ ] **R4.** Um alvo `run` que compila (se preciso) e executa o programa.
- [ ] **R5.** Um alvo `clean` que remove o executável e os arquivos `.o`.
- [ ] **R6.** Os alvos que não geram arquivo com o próprio nome (`run`, `clean`, `all`) devem estar marcados como `.PHONY`.

> **Dica — Cuidado com o TAB!** A linha de comando dentro de uma regra do Make precisa começar com um caractere de *tabulação*, não com espaços. Se seu editor converte TAB em espaços automaticamente, desative essa opção só para este arquivo (ou consulte o professor).

Teste seu Makefile:

```bash
$ cd ex2
$ make          # deve compilar main.c e estatistica.c, e linkar
$ make run      # deve executar e mostrar media/maior/menor
$ make          # rodar de novo: nao deve recompilar nada
$ touch main.c
$ make          # deve recompilar so main.c (nao estatistica.c!)
$ make clean    # deve remover turma, main.o, estatistica.o
```

**Saída esperada:**

```
Notas da turma: 7 8 5 9 6 10 4
Media:  7.00
Maior:  10
Menor:  4
```

**Em `ex2/respostas.txt`:**

- [ ] **Q8.** Depois de `touch main.c`, quais arquivos o `make` recompilou? Por que `estatistica.c` não foi recompilado?
- [ ] **Q9.** O que aconteceria se você removesse a linha `.PHONY: ... clean` e existisse, por acidente, um arquivo chamado `clean` na pasta? (pode testar: `touch clean && make clean`)

---

## Exercício 3 — GDB: caçando um bug (25–30 min)

> **Objetivo:** Usar `break`, `next`/`step`, `watch`, `backtrace` e `print` para localizar a causa exata de um resultado errado — sem adicionar um único `printf`.

Crie `ex3/notas.c`:

```c
#include <stdio.h>

double calcula_media(int *notas, int n) {
    int soma = 0;
    int i;
    for (i = 0; i <= n; i++) {
        soma = soma + notas[i];
    }
    return (double) soma / n;
}

void processa_turma(int *notas, int n) {
    double resultado = calcula_media(notas, n);
    printf("Media da turma: %.2f\n", resultado);
}

int main(void) {
    int notas[5] = {8, 7, 9, 6, 10};
    int quantidade = 5;

    printf("Processando turma com %d notas...\n", quantidade);
    processa_turma(notas, quantidade);

    return 0;
}
```

Compile **com símbolos de depuração** e rode:

```bash
$ gcc -Wall -Wextra -g -std=c11 -o notas notas.c
$ ./notas
```

**Saída esperada:**

```
Processando turma com 5 notas...
Media da turma: 6561.00
```

A média de `{8, 7, 9, 6, 10}` deveria ser **8.00**. Algo está muito errado. Vamos investigar com o GDB, passo a passo.

### Roteiro de depuração

**Passo 1.** Inicie o GDB e coloque um breakpoint na função onde a conta é feita:

```
$ gdb ./notas
(gdb) break calcula_media
(gdb) run
```

O GDB deve parar na primeira linha de `calcula_media`, mostrando `notas=..., n=5`.

**Passo 2.** Avance duas linhas com `next` até chegar no laço `for`:

```
(gdb) next
(gdb) next
```

**Passo 3.** Crie um *watchpoint* na variável `i`, para o GDB parar sozinho toda vez que ela mudar de valor:

```
(gdb) watch i
(gdb) continue
```

Repita `continue` várias vezes e observe os valores de `i` indo de `0` até `5`. **Não pare ainda** — dê `continue` mais uma vez, deixando `i` chegar a `6`.

> **Dica:** Se `i` chegou a `5` e o laço **ainda não terminou**, algo está errado: o vetor `notas` só tem 5 posições válidas (`notas[0]` até `notas[4]`). Um acesso com `i == 5` já está fora dos limites do vetor.

**Passo 4.** Veja a pilha de chamadas para entender quem chamou quem até chegar aqui:

```
(gdb) backtrace
```

Saída esperada (endereços variam):

```
#0  calcula_media (notas=..., n=5) at notas.c:8
#1  processa_turma (notas=..., n=5) at notas.c:15
#2  main () at notas.c:24
```

**Passo 5.** Inspecione os valores que explicam o bug:

```
(gdb) print soma
(gdb) print notas[i]
(gdb) print n
```

Você deve ver que `notas[i]` (com `i` igual a `n`, ou seja `notas[5]`) contém um valor de "lixo de memória" — um número grande e sem sentido, porque essa posição nunca foi inicializada pelo programa.

**Passo 6.** Saia do GDB:

```
(gdb) quit
```

**Em `ex3/respostas.txt`:**

- [ ] **Q10.** Qual foi o valor de `notas[i]` quando `i` chegou a `5`? (esse é o "lixo de memória" que explica o resultado `6561.00`)
- [ ] **Q11.** Cole a saída do seu `backtrace`. Quantos níveis (frames) a pilha de chamadas tem?
- [ ] **Q12.** Qual linha exata do código tem o bug, e qual é a correção?

### Corrija e confirme

Edite `ex3/notas.c`, trocando a condição do laço de `i <= n` para `i < n`. Recompile e rode de novo:

```bash
$ gcc -Wall -Wextra -g -std=c11 -o notas notas.c
$ ./notas
```

**Saída esperada:**

```
Processando turma com 5 notas...
Media da turma: 8.00
```

Se o resultado bateu com `8.00`, o exercício está concluído.

---

## Entrega <!-- e critérios de avaliação -->

Uma pasta `lab01-nome/` contendo:

- [ ] `ex1/warnings.c`, `ex1/otimizacao.c`, `ex1/respostas.txt`
- [ ] `ex2/main.c`, `ex2/estatistica.c`, `ex2/estatistica.h`, `ex2/Makefile`, `ex2/respostas.txt`
- [ ] `ex3/notas.c` (já corrigido), `ex3/respostas.txt`

<!--
### Avaliação (10 pontos)

| Critério | Pontos | O que é avaliado |
|---|---|---|
| Exercício 1 — Flags do GCC | 3,0 | Respostas corretas sobre warnings e otimização; tempos medidos e reportados |
| Exercício 2 — Makefile | 3,5 | Makefile atende aos 6 requisitos (R1–R6); recompilação incremental funciona |
| Exercício 3 — GDB | 3,5 | Bug localizado com as ferramentas corretas (não só "adivinhado"); backtrace e correção corretos |

> **Dica:** O objetivo não é só chegar na resposta certa, mas **mostrar o uso das ferramentas**. Um Makefile copiado do gabarito sem entender, ou um bug "achado" só de olhar o código sem usar o GDB, não demonstra o aprendizado esperado para este encontro.

<!-- 
---

## Anexo A — Gabarito do Makefile

*Consulte apenas depois de tentar o Exercício 2 por conta própria.*

```makefile
CC = gcc
CFLAGS = -Wall -Wextra -g -std=c11
TARGET = turma
OBJS = main.o estatistica.o

.PHONY: all clean run

all: $(TARGET)

$(TARGET): $(OBJS)
	$(CC) $(CFLAGS) -o $@ $^

%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@

run: $(TARGET)
	./$(TARGET)

clean:
	rm -f $(TARGET) $(OBJS)
```

**Testado e validado:** `make`, `make run`, `make` (idempotente), `touch main.c && make` (recompila só `main.o`), e `make clean` — todos produzem o comportamento esperado.

## Anexo B — `notas.c` corrigido

```c
#include <stdio.h>

double calcula_media(int *notas, int n) {
    int soma = 0;
    int i;
    for (i = 0; i < n; i++) {          /* corrigido: i < n */
        soma = soma + notas[i];
    }
    return (double) soma / n;
}

void processa_turma(int *notas, int n) {
    double resultado = calcula_media(notas, n);
    printf("Media da turma: %.2f\n", resultado);
}

int main(void) {
    int notas[5] = {8, 7, 9, 6, 10};
    int quantidade = 5;

    printf("Processando turma com %d notas...\n", quantidade);
    processa_turma(notas, quantidade);

    return 0;
}
```
-->