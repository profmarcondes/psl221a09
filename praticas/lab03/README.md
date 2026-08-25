# Laboratório 03: <br> Sistema de Arquivos Linux

<!--
**PSL221A09 — Programação em Sistemas Linux**

| | |
|---|---|
| **Duração estimada** | 80–90 minutos (parte prática) |
| **Formato** | Duplas ou trios, no LSC/LSI |
| **Pré-requisito** | GCC instalado (Ubuntu/Debian); acesso a um terminal com usuário comum |
| **Entregável** | Pasta `laboratorio03-nomes/` com os 3 exercícios |

---
-->

Este guia consolida a hierarquia de diretórios do Linux (**FHS**), os **tipos de arquivo**, o sistema de **permissões** (incluindo SUID, SGID e *sticky bit*) e as **APIs POSIX** de arquivo (`open`, `read`, `stat`, `opendir`/`readdir`). Siga os três exercícios em ordem. Sempre que houver uma seção de **saída esperada**, compare com o que apareceu no seu terminal antes de seguir em frente.

## Preparação do ambiente

1. Confirme que o GCC está instalado:

   ```bash
   $ gcc --version
   ```

2. Crie a pasta de trabalho do laboratório:

   ```bash
   $ mkdir -p working_dir/lab03-nome/{ex1,ex2,ex3}
   $ cd working_dir/lab03-nome
   ```

---

## Exercício 1 — Explorando a hierarquia FHS <!--(15 min)-->

> **Objetivo:** Reconhecer os principais diretórios do FHS e a diferença entre um sistema de arquivos "de verdade" e um virtual como `/proc`.

Rode os comandos abaixo e anote o que observar:

```bash
$ ls -la /
$ df -h
$ mount | grep -E "^(proc|sysfs|tmpfs) "
$ cat /proc/cpuinfo | head -10
$ cat /proc/meminfo | head -5
$ echo $$
$ cat /proc/$$/status | head -5
```

**Responda no arquivo `ex1/respostas.txt`:**

- [ ] **Q1.** Liste 5 diretórios de primeiro nível (`/algo`) e, para cada um, escreva em uma frase o que ele guarda (consulte `man 7 hier` se tiver dúvida).
- [ ] **Q2.** O comando `df -h` mostra `/proc` e `/sys` como sistemas de arquivo? Eles ocupam espaço em disco? Justifique olhando o tamanho reportado.
- [ ] **Q3.** `echo $$` imprime o PID do seu próprio shell. O que aconteceu quando você rodou `cat /proc/$$/status`? De onde vêm essas informações, já que elas não estão gravadas em nenhum HD?
- [ ] **Q4.** Rode `cat /proc/cpuinfo` duas vezes, com alguns segundos de intervalo. O conteúdo poderia mudar entre uma leitura e outra? Por quê (ou por que não)?

---

## Exercício 2 — Tipos de arquivo e permissões na prática <!--(25–30 min)-->

> **Objetivo:** Manipular permissões com `chmod`/`chown`, e observar SUID, SGID e *sticky bit* em arquivos reais do sistema.

### Parte A — Tipos de arquivo

```bash
$ cd ex2
$ touch arquivo_normal.txt
$ mkdir subpasta
$ ln -s arquivo_normal.txt link_simbolico.txt
$ ln arquivo_normal.txt hard_link.txt
$ mkfifo meu_pipe
$ ls -l
```

**Responda no arquivo `ex2/respostas.txt`:**

- [ ] **Q5.** Cole a saída do `ls -l`. Para cada linha, identifique o tipo de arquivo pelo primeiro caractere.
- [ ] **Q6.** Rode `ls -li arquivo_normal.txt hard_link.txt`. Os dois têm o mesmo número de inode? O que isso significa na prática?
- [ ] **Q7.** Apague `arquivo_normal.txt` com `rm`. O `hard_link.txt` ainda tem conteúdo? E o `link_simbolico.txt` (tente `cat link_simbolico.txt`)? Explique a diferença.

### Parte B — Permissões numéricas e simbólicas

```bash
$ echo "conteudo de teste" > dados.txt
$ chmod 644 dados.txt
$ ls -l dados.txt
$ chmod u+x,go-r dados.txt
$ ls -l dados.txt
$ chmod 600 dados.txt
$ ls -l dados.txt
```

**Continue em `ex2/respostas.txt`:**

- [ ] **Q8.** Escreva, em notação simbólica (`rwxr-xr--` etc.), o resultado de cada um dos três comandos `chmod` acima.
- [ ] **Q9.** Calcule manualmente: qual é o valor numérico (tipo `755`) equivalente a `rw-r-----`?

### Parte C — SUID, SGID e sticky bit

```bash
$ ls -l /usr/bin/passwd
$ ls -ld /tmp
$ mkdir pasta_compartilhada
$ chmod g+s pasta_compartilhada
$ ls -ld pasta_compartilhada
$ mkdir pasta_protegida
$ chmod +t pasta_protegida
$ ls -ld pasta_protegida
```

**Finalize em `ex2/respostas.txt`:**

- [ ] **Q10.** No `ls -l /usr/bin/passwd`, qual caractere aparece no lugar do `x` do dono? O que isso indica sobre com quais privilégios o programa `passwd` roda quando executado?
- [ ] **Q11.** No `ls -ld /tmp`, qual caractere aparece no lugar do `x` de "outros"? Pesquise (ou teste com um colega, cada um como um usuário diferente) o que o *sticky bit* impede que aconteça em uma pasta com permissão de escrita para todos, como `/tmp`.
- [ ] **Q12.** Depois do `chmod g+s pasta_compartilhada`, crie um arquivo dentro dela (`touch pasta_compartilhada/teste.txt`) e rode `ls -l pasta_compartilhada/teste.txt`. De qual grupo o arquivo herdou, e por quê?

<!-- >> **Dica:** Se algum desses testes não mostrar o comportamento esperado, confirme que você **não** está logado como `root` — como vimos, o `root` ignora várias dessas checagens. -->

---

## Exercício 3 — Um `ls -l` em C, do zero <!--(30–35 min)-->

> **Objetivo:** Usar `opendir`/`readdir`/`lstat` para escrever um programa que lista um diretório mostrando tipo, permissões, tamanho e inode de cada entrada — na prática, uma versão simplificada do `ls -l`.

Crie `ex3/listadir.c`. Você pode usar o esqueleto abaixo como ponto de partida (as funções `tipo_arquivo` e `permissoes_str` já estão prontas — sua tarefa principal é completar o `main`):

```c
#define _DEFAULT_SOURCE
#include <stdio.h>
#include <dirent.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <string.h>

const char *tipo_arquivo(mode_t modo) {
    if (S_ISREG(modo))  return "regular";
    if (S_ISDIR(modo))  return "diretorio";
    if (S_ISLNK(modo))  return "link simb.";
    if (S_ISCHR(modo))  return "disp. char";
    if (S_ISBLK(modo))  return "disp. bloco";
    if (S_ISFIFO(modo)) return "fifo";
    if (S_ISSOCK(modo)) return "socket";
    return "desconhecido";
}

void permissoes_str(mode_t modo, char *saida) {
    const char *bits = "rwxrwxrwx";
    for (int i = 0; i < 9; i++) {
        saida[i] = (modo & (1 << (8 - i))) ? bits[i] : '-';
    }
    saida[9] = '\0';
}

int main(int argc, char *argv[]) {
    const char *caminho = (argc > 1) ? argv[1] : ".";

    /* TODO 1: abra o diretorio "caminho" com opendir().          */
    /* Se falhar (retorno NULL), chame perror("opendir") e saia.  */

    /* TODO 2: em um loop, chame readdir() ate ele devolver NULL. */
    /* Para cada entrada->d_name:                                 */
    /*   - monte o caminho completo (caminho + "/" + d_name)      */
    /*   - chame lstat() nesse caminho completo                   */
    /*   - imprima tipo, permissoes, tamanho e inode              */

    /* TODO 3: feche o diretorio com closedir().                  */

    return 0;
}
```

**Requisitos:**

- [ ] **R1.** Usa `opendir()` e `readdir()` para percorrer o diretório indicado em `argv[1]` (ou `.` se nenhum argumento for passado).
- [ ] **R2.** Usa `lstat()` (não `stat()`) em cada entrada — assim links simbólicos aparecem como `link simb.`, não como o tipo do arquivo apontado.
- [ ] **R3.** Para cada entrada, imprime: tipo do arquivo, permissões em formato `rwxrwxrwx`, tamanho em bytes e número do inode.
- [ ] **R4.** Trata o erro de `opendir()` falhar (diretório inexistente ou sem permissão) com `perror()`, sem travar o programa.

Compile e teste:

```bash
$ cd ex3
$ gcc -g -Wall -Wextra -std=c11 -o listadir listadir.c
$ touch arquivo_normal.txt
$ mkdir subpasta
$ ln -sf arquivo_normal.txt link_simbolico.txt
$ ./listadir .
```

**Saída esperada (a ordem e os números de inode variam):**

```
regular    rwxr-xr-x    19872 bytes  inode=794656   listadir
diretorio  rwxr-xr-x     4096 bytes  inode=794651   .
diretorio  rwxr-xr-x     4096 bytes  inode=794654   subpasta
link simb. rwxrwxrwx       18 bytes  inode=794657   link_simbolico.txt
regular    rw-r--r--        0 bytes  inode=794653   arquivo_normal.txt
diretorio  rwxr-xr-x     4096 bytes  inode=794650   ..
regular    rw-r--r--     1558 bytes  inode=794652   listadir.c
```

Teste também com um caminho que não existe:

```bash
$ ./listadir /caminho/que/nao/existe
```

**Saída esperada:**

```
opendir: No such file or directory
```

**Em `ex3/respostas.txt`:**

- [ ] **Q13.** Por que o programa usa `lstat()` em vez de `stat()`? O que mudaria no resultado para `link_simbolico.txt` se você trocasse para `stat()`?
- [ ] **Q14.** Rode `./listadir .` no seu próprio diretório `ex3/`. Quais entradas aparecem que você **não** criou explicitamente? (dica: toda pasta tem pelo menos duas entradas "de brinde")
- [ ] **Q15.** O tamanho reportado para um diretório (por exemplo, `subpasta`) normalmente é `4096 bytes`, mesmo estando "vazio". O que esse número representa? (dica: não é o tamanho dos arquivos dentro dele)

<!-- Se travar na implementação, o gabarito completo está no [Anexo A](#anexo-a--gabarito-do-listadirc). -->

---

## Entrega <!-- e critérios de avaliação -->

### O que entregar

Uma pasta `lab03-nome/`, compactada, contendo:

- [ ] `ex1/respostas.txt`
- [ ] `ex2/respostas.txt`
- [ ] `ex3/listadir.c`, `ex3/respostas.txt`

<!-- 
### Rubrica (10 pontos)

| Critério | Pontos | O que é avaliado |
|---|---|---|
| Exercício 1 — FHS e `/proc` | 2,5 | Respostas corretas sobre a hierarquia e sobre a natureza virtual de `/proc` |
| Exercício 2 — Tipos e permissões | 3,5 | Testes de `chmod`/hard link/symlink e de SUID/SGID/*sticky bit* reproduzidos corretamente |
| Exercício 3 — `listadir.c` | 4,0 | Programa atende aos requisitos (R1–R4); trata erro de diretório inexistente; saída correta |

> **Dica:** O objetivo do Exercício 3 não é decorar a API POSIX, mas entender o que cada chamada faz. Um programa que "funciona" mas não usa `lstat`/`opendir`/`readdir` como pedido não demonstra o aprendizado esperado.

---

## Anexo A — Gabarito do `listadir.c`

*Consulte apenas depois de tentar o Exercício 3 por conta própria.*

```c
#define _DEFAULT_SOURCE
#include <stdio.h>
#include <dirent.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <string.h>

const char *tipo_arquivo(mode_t modo) {
    if (S_ISREG(modo))  return "regular";
    if (S_ISDIR(modo))  return "diretorio";
    if (S_ISLNK(modo))  return "link simb.";
    if (S_ISCHR(modo))  return "disp. char";
    if (S_ISBLK(modo))  return "disp. bloco";
    if (S_ISFIFO(modo)) return "fifo";
    if (S_ISSOCK(modo)) return "socket";
    return "desconhecido";
}

void permissoes_str(mode_t modo, char *saida) {
    const char *bits = "rwxrwxrwx";
    for (int i = 0; i < 9; i++) {
        saida[i] = (modo & (1 << (8 - i))) ? bits[i] : '-';
    }
    saida[9] = '\0';
}

int main(int argc, char *argv[]) {
    const char *caminho = (argc > 1) ? argv[1] : ".";

    DIR *dir = opendir(caminho);
    if (!dir) {
        perror("opendir");
        return 1;
    }

    struct dirent *entrada;
    struct stat info;
    char permstr[10];
    char caminho_completo[1024];

    while ((entrada = readdir(dir)) != NULL) {
        snprintf(caminho_completo, sizeof(caminho_completo), "%s/%s", caminho, entrada->d_name);
        if (lstat(caminho_completo, &info) == -1) {
            perror("lstat");
            continue;
        }
        permissoes_str(info.st_mode, permstr);
        printf("%-10s %s %8ld bytes  inode=%-8lu %s\n",
               tipo_arquivo(info.st_mode), permstr,
               (long) info.st_size, (unsigned long) info.st_ino,
               entrada->d_name);
    }

    closedir(dir);
    return 0;
}
```

**Testado e validado:** lista corretamente arquivos regulares, diretórios e links simbólicos, com tipo/permissões/tamanho/inode; trata `opendir()` falhando em caminho inexistente com `perror` e código de saída 1.

## Anexo B — Referência rápida de comandos

```bash
# Permissões
chmod 755 arquivo          # numerico: rwxr-xr-x
chmod u+x arquivo          # simbolico: adiciona execucao ao dono
chmod g+s pasta            # SGID em diretorio
chmod +t pasta             # sticky bit
chown usuario:grupo arquivo

# Tipos de arquivo e links
ls -l                      # primeiro caractere = tipo
ls -li                     # mostra tambem o numero de inode
ln -s alvo link             # link simbolico
ln alvo link                # hard link
mkfifo nome                 # cria um pipe nomeado

# Sistemas de arquivo
df -h                       # espaco por sistema de arquivos montado
mount                       # o que esta montado e onde
findmnt /                   # sistema de arquivos raiz
```

-->