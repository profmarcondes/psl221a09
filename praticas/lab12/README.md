# Laboratório 12 <br> Rede e Servidores Concorrentes

<!-- **PSL221A09 — Programação em Sistemas Linux** · versão **b** (acompanha a Aula 13b)

| | |
|---|---|
| **Duração estimada** | 2h — Parte A (presencial, ~90 min) + Parte B (extraclasse) |
| **Formato** | Individual, em dupla ou trio |
| **Pré-requisito** | GCC, `nc` (pacote `netcat-openbsd`) e `ss` (pacote `iproute2`) — ambos já vêm no Ubuntu; **três terminais** abertos (ou abas) para os Exercícios 1, 3 e 5 |
| **Entregável** | Pasta `working_dir/lab12-nome/` com os exercícios da Parte A; Parte B entregue até o encontro seguinte |

---
-->

Este guia percorre o conteúdo da Aula 13: **sockets TCP** (servidor e cliente,
ordem de bytes, `SO_REUSEADDR`), a diferença entre **fluxo** (TCP) e
**datagrama** (UDP), **sockets do domínio Unix**, e as formas de um servidor
atender **vários clientes ao mesmo tempo** — um processo por cliente e uma
thread por cliente, e o que muda quando os clientes compartilham estado. Todos os testes usam a interface de *loopback*
(`127.0.0.1`): não é preciso rede nem uma segunda máquina.

**Como usar este guia.** A **Parte A** (Exercícios 1 a 3) é feita durante o
encontro de laboratório, com o professor disponível. A **Parte B** (Exercícios 4
a 6) é atividade extraclasse: mesmo formato e mesmo nível de detalhe, para
consolidar sozinho o que não coube no tempo de aula. Sempre que houver uma seção
de **saída esperada**, compare com o que apareceu no seu terminal antes de
seguir em frente.

> **Atenção: vários passos falham de propósito.** "Address already in use",
> "Connection refused", um cliente que fica pendurado até o `timeout` matar, um
> servidor que morre com *segmentation fault* — nos Exercícios 1, 3, 4 e 5 esses
> erros **são** o experimento. Leia a etapa
> inteira antes de concluir que algo deu errado. Números de porta efêmera e de
> PID variam de máquina para máquina; o que importa é o comportamento.

> **Porta usada.** Todos os servidores TCP e UDP deste laboratório usam a porta
> **5012**. Se outro colega estiver usando a mesma máquina (ou você esquecer um
> servidor rodando), o `bind()` vai falhar — confira com
> `ss -tlnp | grep 5012` e encerre o processo que estiver lá.

## Preparação do ambiente

```bash
$ mkdir -p working_dir/lab12-nome/{ex1,ex2,ex3,ex4,ex5,ex6}
$ cd working_dir/lab12-nome
```

Substitua `nome` pelo seu nome (ou pelos nomes da dupla/trio). Cada exercício tem
sua própria pasta (`ex1/` a `ex6/`) e seu próprio `respostas.txt`.

Todos os programas usam só cabeçalhos do sistema (sem bibliotecas extras):

```bash
$ gcc -Wall -Wextra -o programa programa.c
```

Os programas com threads do Exercício 5 precisam também da opção `-pthread`.

---

# Parte A — presencial (~90 min)

## Exercício 1 — Servidor e cliente TCP de eco (30 min)

> **Objetivo:** Escrever o par servidor/cliente TCP mais simples possível, e
> provocar dois erros clássicos: esquecer a ordem de bytes da rede e reiniciar
> um servidor na mesma porta.

### Etapa 1 — Servidor e cliente

Crie `ex1/eco_servidor.c`:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <netinet/in.h>
#include <sys/socket.h>

#define PORTA 5012

int main(void) {
    int srv = socket(AF_INET, SOCK_STREAM, 0);       /* TCP sobre IPv4 */
    if (srv < 0) { perror("socket"); return 1; }

    struct sockaddr_in end;
    memset(&end, 0, sizeof(end));
    end.sin_family = AF_INET;
    end.sin_addr.s_addr = htonl(INADDR_ANY);         /* qualquer interface */
    end.sin_port = htons(PORTA);                     /* ordem de bytes da rede */

    if (bind(srv, (struct sockaddr *)&end, sizeof(end)) < 0) { perror("bind"); return 1; }
    if (listen(srv, 8) < 0) { perror("listen"); return 1; }
    printf("[servidor] escutando na porta %d (PID %d)\n", PORTA, getpid());
    fflush(stdout);

    for (;;) {
        struct sockaddr_in cli;
        socklen_t tam = sizeof(cli);
        int con = accept(srv, (struct sockaddr *)&cli, &tam);   /* bloqueia */
        if (con < 0) { perror("accept"); continue; }

        char ip[INET_ADDRSTRLEN];
        inet_ntop(AF_INET, &cli.sin_addr, ip, sizeof(ip));
        printf("[servidor] conexao de %s:%d\n", ip, ntohs(cli.sin_port));
        fflush(stdout);

        char buf[256];
        ssize_t n;
        while ((n = read(con, buf, sizeof(buf))) > 0)
            write(con, buf, (size_t)n);                          /* eco */

        printf("[servidor] %s:%d desconectou\n", ip, ntohs(cli.sin_port));
        fflush(stdout);
        close(con);
    }
}
```

E `ex1/eco_cliente.c`:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <netinet/in.h>
#include <sys/socket.h>

#define PORTA 5012

int main(int argc, char *argv[]) {
    if (argc < 2) { fprintf(stderr, "uso: %s msg1 [msg2 ...]\n", argv[0]); return 1; }

    int s = socket(AF_INET, SOCK_STREAM, 0);
    if (s < 0) { perror("socket"); return 1; }

    struct sockaddr_in srv;
    memset(&srv, 0, sizeof(srv));
    srv.sin_family = AF_INET;
    srv.sin_port = htons(PORTA);
    inet_pton(AF_INET, "127.0.0.1", &srv.sin_addr);

    if (connect(s, (struct sockaddr *)&srv, sizeof(srv)) < 0) { perror("connect"); return 1; }

    for (int i = 1; i < argc; i++) {
        write(s, argv[i], strlen(argv[i]));
        char buf[256];
        ssize_t n = read(s, buf, sizeof(buf) - 1);
        if (n <= 0) break;
        buf[n] = '\0';
        printf("[cliente] enviei \"%s\", recebi \"%s\"\n", argv[i], buf);
    }
    close(s);
    return 0;
}
```

No **terminal 1**, compile e suba o servidor:

```bash
$ cd ex1
$ gcc -Wall -Wextra -o eco_servidor eco_servidor.c
$ gcc -Wall -Wextra -o eco_cliente eco_cliente.c
$ ./eco_servidor
```

No **terminal 2**, rode o cliente, depois o `nc` (um cliente TCP genérico) e,
por fim, veja quem está escutando na porta:

```bash
$ ./eco_cliente ola mundo PSL221A09
$ printf 'linha via nc\n' | nc -N 127.0.0.1 5012
$ ss -tlnp | grep 5012
```

**Saída real capturada** (terminal 2):

```
[cliente] enviei "ola", recebi "ola"
[cliente] enviei "mundo", recebi "mundo"
[cliente] enviei "PSL221A09", recebi "PSL221A09"
linha via nc
LISTEN 0      8            0.0.0.0:5012       0.0.0.0:*    users:(("eco_servidor",pid=4417,fd=3))
```

**Saída real capturada** (terminal 1, o servidor):

```
[servidor] escutando na porta 5012 (PID 4417)
[servidor] conexao de 127.0.0.1:42674
[servidor] 127.0.0.1:42674 desconectou
[servidor] conexao de 127.0.0.1:42688
[servidor] 127.0.0.1:42688 desconectou
```

**Em `ex1/respostas.txt`:**

- [ ] **Q1.** O servidor trabalha com **dois** sockets: `srv` e `con`. Qual
  chamada cria cada um, e para que serve cada um? Quantos `srv` e quantos
  `con` existem depois que dois clientes passaram pelo servidor?
- [ ] **Q2.** O servidor imprimiu que o cliente veio da porta `42674` (no seu
  caso, outro número). O cliente nunca chamou `bind()`: quem escolheu essa
  porta? E o que significa o `0.0.0.0:5012` na saída do `ss`?

### Etapa 2 — Esquecendo o `htons()`

Pare o servidor (Ctrl+C no terminal 1). Copie o servidor para
`ex1/eco_servidor_semhtons.c` e troque **apenas** a linha da porta:

```c
    end.sin_port = PORTA;                            /* SEM htons */
```

```bash
$ cp eco_servidor.c eco_servidor_semhtons.c
$ nano eco_servidor_semhtons.c
$ gcc -Wall -Wextra -o eco_servidor_semhtons eco_servidor_semhtons.c
$ ./eco_servidor_semhtons                # terminal 1
```

```bash
$ ss -tlnp | grep eco_servidor           # terminal 2
$ ./eco_cliente teste
```

**Saída real capturada:**

```
[servidor] escutando na porta 5012 (PID 4741)
```
```
LISTEN 0      8            0.0.0.0:37907      0.0.0.0:*    users:(("eco_servidor_se",pid=4741,fd=3))
connect: Connection refused
```

**Em `ex1/respostas.txt`:**

- [ ] **Q3.** O servidor jura que está na porta 5012, mas o `ss` mostra `37907`.
  Explique o número usando a representação hexadecimal de 5012 (dica:
  `printf '%04X\n' 5012`) e a ordem de bytes do seu processador. Por que o
  próprio `printf` do servidor não percebe o erro?

### Etapa 3 — "Address already in use"

Pare o servidor da Etapa 2. Volte ao `eco_servidor` original e faça, nesta ordem:

1. **Terminal 1:** `./eco_servidor`
2. **Terminal 2:** `nc 127.0.0.1 5012` — e deixe conectado, sem digitar nada
3. **Terminal 1:** Ctrl+C no servidor, e **imediatamente** `./eco_servidor` de novo
4. **Terminal 3:** `ss -tan | grep 5012`

**Saída real capturada** (terminal 1, depois do Ctrl+C):

```
$ ./eco_servidor
bind: Address already in use
```

**Saída real capturada** (terminal 3):

```
$ ss -tan | grep 5012
TIME-WAIT  0      0          127.0.0.1:42674     127.0.0.1:5012
TIME-WAIT  0      0          127.0.0.1:42688     127.0.0.1:5012
CLOSE-WAIT 0      0          127.0.0.1:42692     127.0.0.1:5012
FIN-WAIT-2 0      0          127.0.0.1:5012      127.0.0.1:42692
```

Encerre o `nc` do terminal 2 (Ctrl+C). Rode `ss -tan | grep 5012` algumas vezes
ao longo de um minuto e tente subir o servidor de novo de vez em quando.

**Em `ex1/respostas.txt`:**

- [ ] **Q4.** Das quatro linhas do `ss`, só **uma** impede o `bind()` do
  servidor. Qual, e como você identificou (olhe a coluna do endereço **local**)?
  Quem fechou a conexão primeiro — o servidor ou o `nc`? Quanto tempo,
  aproximadamente, até o servidor conseguir subir de novo?

### Etapa 4 — `SO_REUSEADDR`

Copie o servidor para `ex1/eco_servidor_reuse.c` e acrescente duas linhas logo
depois do teste do `socket()`:

```c
    int sim = 1;                                     /* permite reusar a porta */
    setsockopt(srv, SOL_SOCKET, SO_REUSEADDR, &sim, sizeof(sim));
```

```bash
$ cp eco_servidor.c eco_servidor_reuse.c
$ nano eco_servidor_reuse.c
$ gcc -Wall -Wextra -o eco_servidor_reuse eco_servidor_reuse.c
```

Repita a Etapa 3 **duas vezes**:

- **(a)** derrubando o `eco_servidor` **antigo** (sem a opção) e subindo o
  `eco_servidor_reuse` na hora;
- **(b)** espere a porta ficar livre (`ss -tan | grep 5012` sem nenhuma linha
  com `127.0.0.1:5012` como endereço local) e repita usando o
  `eco_servidor_reuse` **nos dois papéis** — derrubar e subir de novo.

**Resultado real** das quatro combinações, testadas com um script que repete a
Etapa 3 automaticamente (sempre partindo da porta livre):

```
antigo=eco_servidor_reuse -> novo=eco_servidor_reuse | porta: FIN-WAIT-2 | SUBIU
antigo=eco_servidor -> novo=eco_servidor | porta: FIN-WAIT-2 | FALHOU (bind: Address already in use)
antigo=eco_servidor -> novo=eco_servidor_reuse | porta: FIN-WAIT-2 | FALHOU (bind: Address already in use)
antigo=eco_servidor_reuse -> novo=eco_servidor | porta: FIN-WAIT-2 | FALHOU (bind: Address already in use)
```

Por fim, com um `eco_servidor_reuse` rodando no terminal 1, tente subir
**outro** `eco_servidor_reuse` no terminal 3.

**Em `ex1/respostas.txt`:**

- [ ] **Q5.** No caso (a), a versão nova já tinha `SO_REUSEADDR` e mesmo assim
  falhou. O que isso diz sobre **de quem** é a permissão que o kernel confere?
  Na prática, qual a consequência para quem corrige esse bug num servidor que
  já está em produção?
- [ ] **Q6.** O segundo `eco_servidor_reuse` rodando ao mesmo tempo que o
  primeiro conseguiu subir? O que isso mostra sobre o que `SO_REUSEADDR`
  **não** permite?

---

## Exercício 2 — TCP é fluxo, UDP é datagrama (25 min)

> **Objetivo:** Ver na prática que o TCP não preserva as fronteiras dos
> `write()`, e comparar com o UDP, que preserva — mas não garante mais nada.

### 2.1 — Três `write()`, quantos `read()`?

Crie `ex2/fluxo_servidor.c`:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <netinet/in.h>
#include <sys/socket.h>

#define PORTA 5012

int main(void) {
    int srv = socket(AF_INET, SOCK_STREAM, 0);
    int sim = 1;
    setsockopt(srv, SOL_SOCKET, SO_REUSEADDR, &sim, sizeof(sim));

    struct sockaddr_in end;
    memset(&end, 0, sizeof(end));
    end.sin_family = AF_INET;
    end.sin_addr.s_addr = htonl(INADDR_ANY);
    end.sin_port = htons(PORTA);
    if (bind(srv, (struct sockaddr *)&end, sizeof(end)) < 0) { perror("bind"); return 1; }
    listen(srv, 8);
    printf("[TCP] esperando um cliente na porta %d...\n", PORTA);
    fflush(stdout);

    int con = accept(srv, NULL, NULL);
    printf("[TCP] conectado; dormindo 1s antes de ler\n");
    fflush(stdout);
    sleep(1);                       /* da tempo de o cliente mandar tudo */

    char buf[256];
    ssize_t n;
    int leituras = 0;
    while ((n = read(con, buf, sizeof(buf) - 1)) > 0) {
        buf[n] = '\0';
        printf("[TCP] read() #%d devolveu %zd bytes: [%s]\n", ++leituras, n, buf);
    }
    printf("[TCP] EOF depois de %d leitura(s)\n", leituras);
    close(con);
    close(srv);
    return 0;
}
```

E `ex2/fluxo_cliente.c`:

```c
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <netinet/in.h>
#include <sys/socket.h>

#define PORTA 5012

int main(void) {
    int s = socket(AF_INET, SOCK_STREAM, 0);
    struct sockaddr_in srv;
    memset(&srv, 0, sizeof(srv));
    srv.sin_family = AF_INET;
    srv.sin_port = htons(PORTA);
    inet_pton(AF_INET, "127.0.0.1", &srv.sin_addr);
    if (connect(s, (struct sockaddr *)&srv, sizeof(srv)) < 0) { perror("connect"); return 1; }

    const char *msgs[] = {"msg1", "msg2", "msg3"};
    for (int i = 0; i < 3; i++) {
        write(s, msgs[i], strlen(msgs[i]));        /* tres write() separados */
        printf("[cliente TCP] write() de \"%s\"\n", msgs[i]);
    }
    close(s);
    return 0;
}
```

```bash
$ cd ex2
$ gcc -Wall -Wextra -o fluxo_servidor fluxo_servidor.c
$ gcc -Wall -Wextra -o fluxo_cliente fluxo_cliente.c
$ ./fluxo_servidor &
$ ./fluxo_cliente
```

**Saída real capturada** (o `read()` devolveu os 12 bytes juntos nas seis
vezes em que foi rodado; a ordem entre as linhas do cliente e as do servidor
pode variar um pouco):

```
[TCP] esperando um cliente na porta 5012...
[cliente TCP] write() de "msg1"
[TCP] conectado; dormindo 1s antes de ler
[cliente TCP] write() de "msg2"
[cliente TCP] write() de "msg3"
[TCP] read() #1 devolveu 12 bytes: [msg1msg2msg3]
[TCP] EOF depois de 1 leitura(s)
```

**Em `ex2/respostas.txt`:**

- [ ] **Q7.** O cliente fez três `write()` separados, e o servidor recebeu tudo
  num único `read()`. Onde ficaram os 12 bytes durante o segundo em que o
  servidor dormia, e por que o `read()` não "sabe" que eram três mensagens?

Agora copie o servidor para `ex2/fluxo_servidor5.c` e troque o tamanho máximo
da leitura de `sizeof(buf) - 1` para `5`:

```c
    while ((n = read(con, buf, 5)) > 0) {
```

```bash
$ gcc -Wall -Wextra -o fluxo_servidor5 fluxo_servidor5.c
$ ./fluxo_servidor5 &
$ ./fluxo_cliente
```

**Saída real capturada** (só as linhas do servidor):

```
[TCP] esperando um cliente na porta 5012...
[TCP] conectado; dormindo 1s antes de ler
[TCP] read() #1 devolveu 5 bytes: [msg1m]
[TCP] read() #2 devolveu 5 bytes: [sg2ms]
[TCP] read() #3 devolveu 2 bytes: [g3]
[TCP] EOF depois de 3 leitura(s)
```

- [ ] **Q8.** Agora as mensagens chegaram **partidas**. Se você estivesse
  projetando um protocolo de comandos de texto sobre TCP (ex.: `LIGAR`,
  `DESLIGAR`, `STATUS`), como o receptor saberia onde termina cada comando?
  Proponha **duas** formas diferentes.

### 2.2 — O mesmo teste com UDP

Crie `ex2/udp_servidor.c`:

```c
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <netinet/in.h>
#include <sys/socket.h>

#define PORTA 5012

int main(void) {
    int s = socket(AF_INET, SOCK_DGRAM, 0);          /* UDP */
    struct sockaddr_in end;
    memset(&end, 0, sizeof(end));
    end.sin_family = AF_INET;
    end.sin_addr.s_addr = htonl(INADDR_ANY);
    end.sin_port = htons(PORTA);
    if (bind(s, (struct sockaddr *)&end, sizeof(end)) < 0) { perror("bind"); return 1; }
    printf("[UDP] escutando na porta %d; dormindo 2s antes de ler\n", PORTA);
    fflush(stdout);
    sleep(2);                       /* os tres datagramas chegam durante o sono */

    for (int i = 1; i <= 3; i++) {
        char buf[256];
        struct sockaddr_in de;
        socklen_t tam = sizeof(de);
        ssize_t n = recvfrom(s, buf, sizeof(buf) - 1, 0, (struct sockaddr *)&de, &tam);
        buf[n] = '\0';
        printf("[UDP] recvfrom() #%d devolveu %zd bytes: [%s] (de porta %d)\n",
               i, n, buf, ntohs(de.sin_port));
    }
    close(s);
    return 0;
}
```

E `ex2/udp_cliente.c`:

```c
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <netinet/in.h>
#include <sys/socket.h>

#define PORTA 5012

int main(void) {
    int s = socket(AF_INET, SOCK_DGRAM, 0);
    struct sockaddr_in srv;
    memset(&srv, 0, sizeof(srv));
    srv.sin_family = AF_INET;
    srv.sin_port = htons(PORTA);
    inet_pton(AF_INET, "127.0.0.1", &srv.sin_addr);

    const char *msgs[] = {"msg1", "msg2", "msg3"};
    for (int i = 0; i < 3; i++) {                     /* sem connect(): cada */
        ssize_t n = sendto(s, msgs[i], strlen(msgs[i]), 0,  /* envio leva o destino */
                           (struct sockaddr *)&srv, sizeof(srv));
        printf("[cliente UDP] sendto() de \"%s\" retornou %zd\n", msgs[i], n);
    }
    close(s);
    return 0;
}
```

```bash
$ gcc -Wall -Wextra -o udp_servidor udp_servidor.c
$ gcc -Wall -Wextra -o udp_cliente udp_cliente.c
$ ./udp_servidor &
$ ./udp_cliente
```

**Saída real capturada:**

```
[UDP] escutando na porta 5012; dormindo 2s antes de ler
[cliente UDP] sendto() de "msg1" retornou 4
[cliente UDP] sendto() de "msg2" retornou 4
[cliente UDP] sendto() de "msg3" retornou 4
[UDP] recvfrom() #1 devolveu 4 bytes: [msg1] (de porta 60923)
[UDP] recvfrom() #2 devolveu 4 bytes: [msg2] (de porta 60923)
[UDP] recvfrom() #3 devolveu 4 bytes: [msg3] (de porta 60923)
```

Agora rode **só** o cliente, sem nenhum servidor:

```bash
$ ./udp_cliente; echo "exit=$?"
```

**Saída real capturada:**

```
[cliente UDP] sendto() de "msg1" retornou 4
[cliente UDP] sendto() de "msg2" retornou 4
[cliente UDP] sendto() de "msg3" retornou 4
exit=0
```

**Em `ex2/respostas.txt`:**

- [ ] **Q9.** Mesmo com o servidor dormindo 2s (como no TCP), cada
  `recvfrom()` devolveu exatamente uma mensagem. O que o UDP preserva que o
  TCP não preserva? Repare também que o servidor UDP conseguiu usar a porta
  5012 mesmo com o servidor TCP do 2.1 tendo usado a mesma porta — por quê?
- [ ] **Q10.** Sem servidor nenhum, o `sendto()` retornou `4` (sucesso). No
  Exercício 1, Etapa 2, o cliente TCP para uma porta sem ninguém recebeu
  `Connection refused`. O que "sucesso" significa no `sendto()` do UDP — e o
  que ele **não** significa?

Por último, copie o servidor UDP para `ex2/udp_servidor3.c` e troque o tamanho
máximo de leitura do `recvfrom()` de `sizeof(buf) - 1` para `3`:

```c
        ssize_t n = recvfrom(s, buf, 3, 0, (struct sockaddr *)&de, &tam);
```

```bash
$ gcc -Wall -Wextra -o udp_servidor3 udp_servidor3.c
$ ./udp_servidor3 &
$ ./udp_cliente
```

**Saída real capturada** (só as linhas do servidor):

```
[UDP] escutando na porta 5012; dormindo 2s antes de ler
[UDP] recvfrom() #1 devolveu 3 bytes: [msg] (de porta 46635)
[UDP] recvfrom() #2 devolveu 3 bytes: [msg] (de porta 46635)
[UDP] recvfrom() #3 devolveu 3 bytes: [msg] (de porta 46635)
```

- [ ] **Q11.** Compare com o `fluxo_servidor5` do TCP: lá, o resto de cada
  mensagem veio na leitura seguinte. Para onde foram os caracteres `1`, `2` e
  `3` no UDP? Que cuidado isso exige ao escolher o tamanho do buffer de
  recepção num programa UDP?

---

## Exercício 3 — Vários clientes ao mesmo tempo (35 min)

> **Objetivo:** Ver o servidor iterativo travar com dois clientes, corrigir
> com um processo por cliente, e reproduzir o bug do `close()` esquecido.

### 3.1 — O servidor iterativo

Use o `eco_servidor_reuse` do Exercício 1:

1. **Terminal 1:** `cd ex3 && ../ex1/eco_servidor_reuse`
2. **Terminal 2:** `nc 127.0.0.1 5012` — cliente **A**, conectado e parado
3. **Terminal 3:** cliente **B**, em segundo plano, e logo em seguida o `ss`:

```bash
$ (echo "oi do B" | timeout 3 nc -N 127.0.0.1 5012; echo "B saiu com codigo $?") &
$ ss -tln | grep -E 'State|5012'
```

**Saída real capturada** (terminal 3):

```
State  Recv-Q Send-Q Local Address:Port  Peer Address:PortProcess
LISTEN 1      8            0.0.0.0:5012       0.0.0.0:*
B saiu com codigo 124
```

Agora encerre o cliente A (Ctrl+C no terminal 2) e olhe o terminal 1.

**Saída real capturada** (terminal 1):

```
[servidor] escutando na porta 5012 (PID 4134)
[servidor] conexao de 127.0.0.1:57704
[servidor] 127.0.0.1:57704 desconectou
[servidor] conexao de 127.0.0.1:57708
[servidor] 127.0.0.1:57708 desconectou
```

**Em `ex3/respostas.txt`:**

- [ ] **Q12.** O cliente B não recebeu o eco e saiu pelo `timeout` (código
  `124`) — mas também **não** levou `Connection refused`. Onde a conexão de B
  ficou esperando, e o que o `Recv-Q 1` da linha `LISTEN` está contando? Por
  que B só aparece no log do servidor depois que A sai?

### 3.2 — Um processo por cliente

Pare o servidor. Crie `ex3/eco_fork.c`:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <signal.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <netinet/in.h>
#include <sys/socket.h>
#include <sys/wait.h>

#define PORTA 5012

static void colhe_filhos(int signo) {          /* padrao do Laboratorio 11 */
    (void)signo;
    while (waitpid(-1, NULL, WNOHANG) > 0)
        ;
}

int main(void) {
    struct sigaction sa;
    memset(&sa, 0, sizeof(sa));
    sa.sa_handler = colhe_filhos;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = SA_RESTART;                  /* accept() volta a esperar */
    sigaction(SIGCHLD, &sa, NULL);

    int srv = socket(AF_INET, SOCK_STREAM, 0);
    int sim = 1;
    setsockopt(srv, SOL_SOCKET, SO_REUSEADDR, &sim, sizeof(sim));

    struct sockaddr_in end;
    memset(&end, 0, sizeof(end));
    end.sin_family = AF_INET;
    end.sin_addr.s_addr = htonl(INADDR_ANY);
    end.sin_port = htons(PORTA);
    if (bind(srv, (struct sockaddr *)&end, sizeof(end)) < 0) { perror("bind"); return 1; }
    listen(srv, 8);
    printf("[pai %d] escutando na porta %d\n", getpid(), PORTA);
    fflush(stdout);

    for (;;) {
        struct sockaddr_in cli;
        socklen_t tam = sizeof(cli);
        int con = accept(srv, (struct sockaddr *)&cli, &tam);
        if (con < 0) { perror("accept"); continue; }

        pid_t pid = fork();
        if (pid == 0) {                        /* FILHO: atende este cliente */
            close(srv);                        /* nao vai aceitar ninguem */
            printf("[filho %d] atendendo porta %d\n", getpid(), ntohs(cli.sin_port));
            fflush(stdout);
            char buf[256];
            ssize_t n;
            while ((n = read(con, buf, sizeof(buf))) > 0)
                write(con, buf, (size_t)n);
            printf("[filho %d] cliente saiu\n", getpid());
            fflush(stdout);
            close(con);
            _exit(0);
        }
        close(con);                            /* PAI: o socket e do filho agora */
    }
}
```

```bash
$ gcc -Wall -Wextra -o eco_fork eco_fork.c
$ ./eco_fork                              # terminal 1
```

Repita os passos 2 e 3 da etapa anterior (A parado no terminal 2; B e `ss` no
terminal 3).

**Saída real capturada** (terminal 3):

```
oi do B
B saiu com codigo 0
State  Recv-Q Send-Q Local Address:Port  Peer Address:PortProcess
LISTEN 0      8            0.0.0.0:5012       0.0.0.0:*
```

Desta vez B terminou antes mesmo de o `ss` rodar.

**Saída real capturada** (terminal 1, depois de encerrar A):

```
[pai 4156] escutando na porta 5012
[filho 4162] atendendo porta 42686
[filho 4171] atendendo porta 42702
[filho 4171] cliente saiu
[filho 4162] cliente saiu
```

**Em `ex3/respostas.txt`:**

- [ ] **Q13.** O código tem dois `close()` "estranhos": o filho fecha `srv` e o
  pai fecha `con`. Explique cada um. Por que fechar `con` no pai **não**
  derruba a conexão que o filho está atendendo?
- [ ] **Q14.** Para que serve o `sigaction(SIGCHLD, ...)` com
  `colhe_filhos()`? O que você veria em `ps -o pid,stat,comm --ppid <PID do pai>`
  depois de alguns clientes, se essa linha fosse removida?

### 3.3 — O `close(con)` esquecido

Copie o servidor para `ex3/eco_fork_bug.c` e comente **apenas** o `close(con)`
do pai (a última linha do laço `for`):

```c
        /* close(con);   <-- esquecido de proposito */
```

```bash
$ gcc -Wall -Wextra -o eco_fork_bug eco_fork_bug.c
$ ./eco_fork_bug                          # terminal 1
```

No terminal 3, um cliente:

```bash
$ echo "oi do B" | timeout 3 nc -N 127.0.0.1 5012; echo "codigo $?"
```

**Saída real capturada:**

```
oi do B
codigo 124
```

Rode mais dois clientes do mesmo jeito e então liste os descritores do pai
(o PID aparece na primeira linha do terminal 1). Depois repita tudo com o
`eco_fork` correto, para comparar:

```bash
$ ls -l /proc/<PID do pai>/fd
```

**Saída real capturada** (só o fim de cada linha):

```
eco_fork_bug, depois de 3 clientes:      eco_fork, depois de 3 clientes:
0 -> /dev/pts/0                          0 -> /dev/pts/0
1 -> /dev/pts/0                          1 -> /dev/pts/0
2 -> /dev/pts/0                          2 -> /dev/pts/0
3 -> socket:[7755]                       3 -> socket:[7798]
4 -> socket:[7756]
5 -> socket:[7761]
6 -> socket:[7767]
```

Com o `eco_fork` correto, os três clientes saíram com `codigo 0`.

**Em `ex3/respostas.txt`:**

- [ ] **Q15.** O cliente recebeu o eco, mas ficou esperando até o `timeout`.
  O filho já tinha terminado. Por que o `nc` não viu o fim da conexão? Que
  regra do Laboratório 10 (pipes) é exatamente a mesma aqui?
- [ ] **Q16.** Cada cliente deixa um descritor aberto no pai. O limite padrão
  é 1024 por processo (`ulimit -n`). O que vai acontecer com esse servidor
  depois de cerca de mil clientes — e quem vai sentir o problema: os clientes
  antigos ou os novos?

---

# Parte B — atividade extraclasse

Os três exercícios abaixo não cabem no encontro de laboratório, mas cobrem
conteúdo cobrado na Avaliação 2. Faça-os antes do encontro seguinte e entregue
junto com a Parte A.

## Exercício 4 — Sockets do domínio Unix (20 min)

> **Objetivo:** Usar um socket local, identificado por um caminho no sistema de
> arquivos, e descobrir quem é o processo do outro lado com `SO_PEERCRED`.

Crie `ex4/uds_servidor.c`:

```c
#define _GNU_SOURCE                             /* struct ucred / SO_PEERCRED */
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <sys/un.h>

#define CAMINHO "/tmp/psl_lab12.sock"

int main(void) {
    int srv = socket(AF_UNIX, SOCK_STREAM, 0);     /* dominio local, fluxo */
    if (srv < 0) { perror("socket"); return 1; }

    struct sockaddr_un end;
    memset(&end, 0, sizeof(end));
    end.sun_family = AF_UNIX;
    strncpy(end.sun_path, CAMINHO, sizeof(end.sun_path) - 1);

    if (bind(srv, (struct sockaddr *)&end, sizeof(end)) < 0) { perror("bind"); return 1; }
    listen(srv, 8);
    printf("[servidor] escutando em %s (PID %d)\n", CAMINHO, getpid());
    fflush(stdout);

    for (;;) {
        int con = accept(srv, NULL, NULL);
        if (con < 0) { perror("accept"); continue; }

        struct ucred quem;                         /* quem esta do outro lado? */
        socklen_t tam = sizeof(quem);
        getsockopt(con, SOL_SOCKET, SO_PEERCRED, &quem, &tam);

        char buf[128];
        ssize_t n = read(con, buf, sizeof(buf) - 1);
        if (n > 0) {
            buf[n] = '\0';
            printf("[servidor] cliente PID=%d UID=%d disse: \"%s\"\n",
                   quem.pid, quem.uid, buf);
            fflush(stdout);
            char resp[64];
            int len = snprintf(resp, sizeof(resp), "ola, processo %d", quem.pid);
            write(con, resp, (size_t)len);
        }
        close(con);
    }
}
```

E `ex4/uds_cliente.c`:

```c
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <sys/un.h>

#define CAMINHO "/tmp/psl_lab12.sock"

int main(void) {
    int s = socket(AF_UNIX, SOCK_STREAM, 0);
    struct sockaddr_un srv;
    memset(&srv, 0, sizeof(srv));
    srv.sun_family = AF_UNIX;
    strncpy(srv.sun_path, CAMINHO, sizeof(srv.sun_path) - 1);
    if (connect(s, (struct sockaddr *)&srv, sizeof(srv)) < 0) { perror("connect"); return 1; }

    char msg[64];
    int len = snprintf(msg, sizeof(msg), "oi, sou o PID %d", getpid());
    write(s, msg, (size_t)len);

    char buf[128];
    ssize_t n = read(s, buf, sizeof(buf) - 1);
    if (n > 0) { buf[n] = '\0'; printf("[cliente %d] resposta: \"%s\"\n", getpid(), buf); }
    close(s);
    return 0;
}
```

```bash
$ cd ex4
$ gcc -Wall -Wextra -o uds_servidor uds_servidor.c
$ gcc -Wall -Wextra -o uds_cliente uds_cliente.c
$ rm -f /tmp/psl_lab12.sock
$ ./uds_servidor &
$ ./uds_cliente; ./uds_cliente
$ ls -l /tmp/psl_lab12.sock
$ cat /tmp/psl_lab12.sock
```

**Saída real capturada:**

```
[servidor] escutando em /tmp/psl_lab12.sock (PID 4256)
[servidor] cliente PID=4258 UID=0 disse: "oi, sou o PID 4258"
[cliente 4258] resposta: "ola, processo 4258"
[servidor] cliente PID=4259 UID=0 disse: "oi, sou o PID 4259"
[cliente 4259] resposta: "ola, processo 4259"
srwxr-xr-x 1 root root 0 Oct 26 19:45 /tmp/psl_lab12.sock
cat: /tmp/psl_lab12.sock: No such device or address
```

> No ambiente de validação os programas rodaram como `root` (UID 0). Na sua
> máquina o UID será o do seu usuário — confira com `id -u`.

**Em `ex4/respostas.txt`:**

- [ ] **Q17.** O que indica o `s` no início da linha do `ls -l`? O arquivo tem
  tamanho 0 e o `cat` falhou: onde ficam, então, os dados trocados entre
  cliente e servidor?

Agora derrube o servidor e tente subir de novo, e rode um cliente:

```bash
$ kill %1
$ ./uds_servidor; echo "exit=$?"
$ ./uds_cliente; echo "exit=$?"
```

**Saída real capturada:**

```
bind: Address already in use
exit=1
connect: Connection refused
exit=1
```

- [ ] **Q18.** O servidor morreu, mas o `bind()` diz que o endereço está em
  uso — e o cliente leva `Connection refused`, apesar de o arquivo existir.
  Explique as duas mensagens. Compare com o `TIME-WAIT` do Exercício 1: a
  causa é a mesma?

Copie o servidor para `ex4/uds_servidor2.c` e acrescente uma linha
imediatamente antes do `bind()`:

```c
    unlink(CAMINHO);                               /* remove o arquivo de uma execucao anterior */
```

```bash
$ gcc -Wall -Wextra -o uds_servidor2 uds_servidor2.c
$ ./uds_servidor2 &
$ ./uds_cliente
```

**Saída real capturada:**

```
[servidor] escutando em /tmp/psl_lab12.sock (PID 4265)
[servidor] cliente PID=4267 UID=0 disse: "oi, sou o PID 4267"
[cliente 4267] resposta: "ola, processo 4267"
```

- [ ] **Q19.** Com o `uds_servidor2` ainda rodando, suba um **segundo**
  `uds_servidor2` em outro terminal e rode um cliente. Quem respondeu? O
  primeiro servidor ainda está vivo (`ps`)? Que risco o `unlink()` cego
  introduz, e como um servidor real evita isso? *(Dica: pense num PID file,
  Laboratório 08, ou em tentar um `connect()` antes do `unlink()`.)*
- [ ] **Q20.** Com `SO_PEERCRED`, o servidor sabe o PID e o UID de quem
  conectou, sem o cliente informar nada. Por que um servidor TCP não tem como
  obter essa garantia sobre um cliente? Dê um exemplo de uso: o que um daemon
  local poderia decidir com base no UID?

---

## Exercício 5 — Uma thread por cliente e estado compartilhado (30 min)

> **Objetivo:** Atender vários clientes com uma thread por cliente, ver o que
> as threads compartilham e os processos do Exercício 3 não compartilham — e o
> preço disso: um erro numa thread derruba todas.

### 5.1 — O servidor com threads

Crie `ex5/eco_thread.c`:

```c
#define _GNU_SOURCE                             /* gettid() */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <signal.h>
#include <unistd.h>
#include <pthread.h>
#include <arpa/inet.h>
#include <netinet/in.h>
#include <sys/socket.h>

#define PORTA 5012

static long mensagens = 0;                      /* estado COMPARTILHADO por todas as threads */
static pthread_mutex_t trava = PTHREAD_MUTEX_INITIALIZER;

static void *atende(void *arg) {
    int con = *(int *)arg;                      /* copia o descritor...           */
    free(arg);                                  /* ...e libera a memoria do main  */
    printf("[thread %d] atendendo o fd %d\n", gettid(), con);
    fflush(stdout);

    char buf[256];
    ssize_t n;
    while ((n = read(con, buf, sizeof(buf))) > 0) {
        pthread_mutex_lock(&trava);
        long minha = ++mensagens;               /* secao critica: le, soma, grava */
        pthread_mutex_unlock(&trava);

        write(con, buf, (size_t)n);             /* eco */
        printf("[thread %d] eco de %zd bytes (mensagem #%ld do servidor)\n",
               gettid(), n, minha);
        fflush(stdout);
    }
    printf("[thread %d] cliente saiu\n", gettid());
    fflush(stdout);
    close(con);                                 /* nenhuma outra copia deste fd */
    return NULL;
}

int main(void) {
    signal(SIGPIPE, SIG_IGN);                   /* cliente que some nao derruba o servidor */

    int srv = socket(AF_INET, SOCK_STREAM, 0);
    int sim = 1;
    setsockopt(srv, SOL_SOCKET, SO_REUSEADDR, &sim, sizeof(sim));

    struct sockaddr_in end;
    memset(&end, 0, sizeof(end));
    end.sin_family = AF_INET;
    end.sin_addr.s_addr = htonl(INADDR_ANY);
    end.sin_port = htons(PORTA);
    if (bind(srv, (struct sockaddr *)&end, sizeof(end)) < 0) { perror("bind"); return 1; }
    listen(srv, 8);
    printf("[main] PID %d, porta %d, uma thread por cliente\n", getpid(), PORTA);
    fflush(stdout);

    for (;;) {
        int con = accept(srv, NULL, NULL);
        if (con < 0) { perror("accept"); continue; }

        int *arg = malloc(sizeof(int));         /* cada thread recebe a SUA copia */
        *arg = con;
        pthread_t t;
        if (pthread_create(&t, NULL, atende, arg) != 0) {
            fprintf(stderr, "pthread_create falhou\n");
            close(con);
            free(arg);
            continue;
        }
        pthread_detach(t);                      /* ninguem vai dar join: libera ao terminar */
    }
}
```

Programas com threads são compilados com `-pthread`:

```bash
$ cd ex5
$ gcc -Wall -Wextra -pthread -o eco_thread eco_thread.c
$ ./eco_thread                                # terminal 1
```

No **terminal 2**, um cliente A que conecta e fica parado: `nc 127.0.0.1 5012`.
No **terminal 3**, um cliente B que manda duas linhas com 2s de intervalo, um
segundo depois um cliente C de uma linha só, e a lista de threads do servidor
(`-L` mostra uma linha por thread):

```bash
$ ( (echo "B linha 1"; sleep 2; echo "B linha 2") | nc -N 127.0.0.1 5012 ) &
$ sleep 1; echo "C sozinho" | nc -N 127.0.0.1 5012
$ ps -L -o pid,lwp,nlwp,comm -p <PID do servidor>
```

Depois de alguns segundos, encerre o cliente A (Ctrl+C no terminal 2).

**Saída real capturada** (terminal 3):

```
B linha 1
C sozinho
  PID   LWP NLWP COMMAND
 1151  1151    3 eco_thread
 1151  1154    3 eco_thread
 1151  1161    3 eco_thread
B linha 2
```

**Saída real capturada** (terminal 1):

```
[main] PID 1151, porta 5012, uma thread por cliente
[thread 1154] atendendo o fd 4
[thread 1161] atendendo o fd 5
[thread 1161] eco de 10 bytes (mensagem #1 do servidor)
[thread 1164] atendendo o fd 6
[thread 1164] eco de 10 bytes (mensagem #2 do servidor)
[thread 1164] cliente saiu
[thread 1161] eco de 10 bytes (mensagem #3 do servidor)
[thread 1161] cliente saiu
[thread 1154] cliente saiu
```

**Em `ex5/respostas.txt`:**

- [ ] **Q21.** Quantas threads o `ps -L` mostrou, e a quem pertence cada uma?
  (Compare a coluna `LWP` com os números `[thread ...]` do log.) Por que a
  thread de C não aparece? No `eco_fork` do Exercício 3, cada cliente ganhava
  um processo filho: quantos PIDs diferentes aparecem aqui? E por que o
  cliente A, parado, não atrapalhou B nem C — a mesma situação que travou o
  servidor iterativo?
- [ ] **Q22.** Três detalhes do `main()`:
  - a) Por que o descritor vai num `malloc()`, em vez de simplesmente
    `pthread_create(&t, NULL, atende, &con)`? *(Dica: o que acontece com
    `con` na próxima volta do laço se dois clientes chegarem quase juntos?)*
  - b) Para que serve o `pthread_detach()`? O que aconteceria com o servidor,
    depois de milhares de clientes, sem ele?
  - c) No `eco_fork`, o pai **precisava** fechar `con` (Q13 e Q15). Aqui o
    `main()` não fecha — quem fecha é a thread. Por que a regra mudou?

### 5.2 — Estado compartilhado: threads × processos

Repare no log do 5.1: as mensagens foram numeradas **#1, #2, #3** somando
todos os clientes — o contador `mensagens` é uma variável global, e todas as
threads enxergam a mesma.

Agora faça o mesmo contador no servidor com `fork()`. Copie o `eco_fork` do
Exercício 3 para `ex5/eco_fork_contador.c` e faça duas mudanças. Logo abaixo
do `#define PORTA`, acrescente:

```c
static long mensagens = 0;                     /* contador global, como no eco_thread */
```

E troque o laço do filho (o `while` com `read`/`write`) por:

```c
            while ((n = read(con, buf, sizeof(buf))) > 0) {
                long minha = ++mensagens;      /* a mesma conta do eco_thread */
                write(con, buf, (size_t)n);
                printf("[filho %d] eco de %zd bytes (mensagem #%ld do servidor)\n",
                       getpid(), n, minha);
                fflush(stdout);
            }
```

```bash
$ cp ../ex3/eco_fork.c eco_fork_contador.c
$ nano eco_fork_contador.c
$ gcc -Wall -Wextra -o eco_fork_contador eco_fork_contador.c
$ ./eco_fork_contador                         # terminal 1
```

No terminal 3, os mesmos clientes B e C do 5.1 (o cliente A não é necessário):

```bash
$ ( (echo "B linha 1"; sleep 2; echo "B linha 2") | nc -N 127.0.0.1 5012 ) &
$ sleep 1; echo "C sozinho" | nc -N 127.0.0.1 5012
```

**Saída real capturada** (terminal 1):

```
[pai 1389] escutando na porta 5012
[filho 1396] atendendo porta 48148
[filho 1396] eco de 10 bytes (mensagem #1 do servidor)
[filho 1399] atendendo porta 48152
[filho 1399] eco de 10 bytes (mensagem #1 do servidor)
[filho 1399] cliente saiu
[filho 1396] eco de 10 bytes (mensagem #2 do servidor)
[filho 1396] cliente saiu
```

**Em `ex5/respostas.txt`:**

- [ ] **Q23.** No `eco_thread`, a contagem foi #1, #2, #3; no
  `eco_fork_contador`, B contou #1 e #2 e C **também** contou #1. Explique
  usando o que o `fork()` faz com a memória do processo (Aula 9). Por que o
  `eco_thread` precisa do mutex em volta de `++mensagens` e o
  `eco_fork_contador` não precisa? O que seria preciso para o servidor com
  `fork()` ter um contador único?

### 5.3 — Uma thread que falha derruba todas

Copie o servidor com threads para `ex5/eco_thread_crash.c` e acrescente, logo
no início do laço `while` da função `atende()`, um erro de programação que só
acontece quando o cliente manda `bug`:

```c
        if (strncmp(buf, "bug", 3) == 0) {      /* simula um erro de programacao */
            int *p = NULL;
            *p = 42;                            /* segmentation fault */
        }
```

Faça o mesmo no servidor com processos: copie `eco_fork_contador.c` para
`ex5/eco_fork_crash.c` e acrescente as mesmas linhas no início do laço
`while` do filho.

```bash
$ cp eco_thread.c eco_thread_crash.c
$ nano eco_thread_crash.c
$ gcc -Wall -Wextra -pthread -o eco_thread_crash eco_thread_crash.c
$ cp eco_fork_contador.c eco_fork_crash.c
$ nano eco_fork_crash.c
$ gcc -Wall -Wextra -o eco_fork_crash eco_fork_crash.c
```

Suba o `eco_thread_crash` no **terminal 1**, pedindo ao shell o código de
saída:

```bash
$ ./eco_thread_crash; echo "servidor saiu com codigo $?"
```

No **terminal 3**, um cliente A que conecta, espera 3s e só então manda uma
linha e, um segundo depois de A conectar, um cliente B que manda `bug`:

```bash
$ ( (sleep 3; echo "A ainda esta ai?") | nc -N 127.0.0.1 5012; echo "A saiu com codigo $?" ) &
$ sleep 1; echo "bug" | nc -N 127.0.0.1 5012; echo "B saiu com codigo $?"
```

**Saída real capturada** (terminal 1):

```
$ ./eco_thread_crash; echo "servidor saiu com codigo $?"
[main] PID 1638, porta 5012, uma thread por cliente
[thread 1645] atendendo o fd 4
[thread 1648] atendendo o fd 5
Segmentation fault
servidor saiu com codigo 139
```

**Saída real capturada** (terminal 3):

```
B saiu com codigo 0
A saiu com codigo 0
```

Repita com o `eco_fork_crash` no terminal 1
(`./eco_fork_crash; echo "servidor saiu com codigo $?"`) e os mesmos dois
comandos no terminal 3. Depois, encerre o servidor com Ctrl+C.

**Saída real capturada** (terminal 1, antes do Ctrl+C):

```
$ ./eco_fork_crash; echo "servidor saiu com codigo $?"
[pai 1658] escutando na porta 5012
[filho 1665] atendendo porta 55322
[filho 1668] atendendo porta 55334
[filho 1665] eco de 17 bytes (mensagem #1 do servidor)
[filho 1665] cliente saiu
```

**Saída real capturada** (terminal 3):

```
B saiu com codigo 0
A ainda esta ai?
A saiu com codigo 0
```

> Dependendo da configuração da sua máquina, a mensagem pode aparecer como
> `Segmentation fault (core dumped)`. O código de saída é o mesmo.

- [ ] **Q24.** Compare as duas versões: o que aconteceu com o **servidor** e
  com o **cliente A** em cada uma? Explique o código `139` (lembre do `141` do
  `SIGPIPE` na Aula 13). Por que um *segmentation fault* numa thread encerra
  o processo inteiro, enquanto o de um filho não afeta o pai nem os outros
  filhos? Por que, no `eco_fork_crash`, nenhuma mensagem de erro apareceu no
  terminal 1?

---

## Exercício 6 — Latência, vazão e a escolha certa (25 min)

> **Objetivo:** Medir a diferença real entre TCP e socket Unix na mesma
> máquina, e decidir que tipo de socket e que modelo de servidor usar em
> cenários diferentes.

Crie `ex6/bench_latencia.c` (100.000 idas e voltas de 1 byte):

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <netinet/in.h>
#include <sys/socket.h>
#include <sys/un.h>
#include <sys/wait.h>

#define PORTA   5012
#define CAMINHO "/tmp/psl_bench_lab12.sock"
#define N       100000                        /* idas e voltas de 1 byte */

/* Cria o socket de escuta (TCP ou Unix) e devolve o endereco para o connect() */
static int escutar(int tcp, struct sockaddr_storage *end, socklen_t *tam) {
    memset(end, 0, sizeof(*end));
    int s;
    if (tcp) {
        struct sockaddr_in *in = (struct sockaddr_in *)end;
        s = socket(AF_INET, SOCK_STREAM, 0);
        int sim = 1;
        setsockopt(s, SOL_SOCKET, SO_REUSEADDR, &sim, sizeof(sim));
        in->sin_family = AF_INET;
        in->sin_port = htons(PORTA);
        inet_pton(AF_INET, "127.0.0.1", &in->sin_addr);
        *tam = sizeof(*in);
    } else {
        struct sockaddr_un *un = (struct sockaddr_un *)end;
        s = socket(AF_UNIX, SOCK_STREAM, 0);
        unlink(CAMINHO);
        un->sun_family = AF_UNIX;
        strncpy(un->sun_path, CAMINHO, sizeof(un->sun_path) - 1);
        *tam = sizeof(*un);
    }
    if (bind(s, (struct sockaddr *)end, *tam) < 0) { perror("bind"); exit(1); }
    listen(s, 1);
    return s;
}

int main(int argc, char *argv[]) {
    if (argc != 2 || (strcmp(argv[1], "tcp") && strcmp(argv[1], "unix"))) {
        fprintf(stderr, "uso: %s tcp|unix\n", argv[0]);
        return 1;
    }
    int tcp = strcmp(argv[1], "tcp") == 0;
    struct sockaddr_storage end;
    socklen_t tam;
    int srv = escutar(tcp, &end, &tam);

    if (fork() == 0) {                          /* FILHO: devolve cada byte */
        close(srv);
        int s = socket(tcp ? AF_INET : AF_UNIX, SOCK_STREAM, 0);
        if (connect(s, (struct sockaddr *)&end, tam) < 0) { perror("connect"); _exit(1); }
        char c;
        while (read(s, &c, 1) == 1)
            if (write(s, &c, 1) != 1) break;
        _exit(0);
    }

    int con = accept(srv, NULL, NULL);          /* PAI: mede N idas e voltas */
    struct timespec t0, t1;
    clock_gettime(CLOCK_MONOTONIC, &t0);
    char c = 'x';
    for (int i = 0; i < N; i++) {
        if (write(con, &c, 1) != 1 || read(con, &c, 1) != 1) { perror("ping-pong"); break; }
    }
    clock_gettime(CLOCK_MONOTONIC, &t1);
    close(con);
    wait(NULL);

    double us = ((t1.tv_sec - t0.tv_sec) * 1e9 + (t1.tv_nsec - t0.tv_nsec)) / 1e3;
    printf("[%s] %d idas e voltas em %.1f ms -> %.2f us por ida e volta\n",
           tcp ? "TCP " : "UNIX", N, us / 1e3, us / N);
    if (!tcp) unlink(CAMINHO);
    return 0;
}
```

E `ex6/bench_vazao.c` (256 MB num sentido só, em blocos de 64 KB):

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <netinet/in.h>
#include <sys/socket.h>
#include <sys/un.h>
#include <sys/wait.h>

#define PORTA   5012
#define CAMINHO "/tmp/psl_bench_lab12.sock"
#define TOTAL   (256L * 1024 * 1024)
#define BLOCO   65536

/* Cria o socket de escuta (TCP ou Unix) e devolve o endereco para o connect() */
static int escutar(int tcp, struct sockaddr_storage *end, socklen_t *tam) {
    memset(end, 0, sizeof(*end));
    int s;
    if (tcp) {
        struct sockaddr_in *in = (struct sockaddr_in *)end;
        s = socket(AF_INET, SOCK_STREAM, 0);
        int sim = 1;
        setsockopt(s, SOL_SOCKET, SO_REUSEADDR, &sim, sizeof(sim));
        in->sin_family = AF_INET;
        in->sin_port = htons(PORTA);
        inet_pton(AF_INET, "127.0.0.1", &in->sin_addr);
        *tam = sizeof(*in);
    } else {
        struct sockaddr_un *un = (struct sockaddr_un *)end;
        s = socket(AF_UNIX, SOCK_STREAM, 0);
        unlink(CAMINHO);
        un->sun_family = AF_UNIX;
        strncpy(un->sun_path, CAMINHO, sizeof(un->sun_path) - 1);
        *tam = sizeof(*un);
    }
    if (bind(s, (struct sockaddr *)end, *tam) < 0) { perror("bind"); exit(1); }
    listen(s, 1);
    return s;
}

int main(int argc, char *argv[]) {
    if (argc != 2 || (strcmp(argv[1], "tcp") && strcmp(argv[1], "unix"))) {
        fprintf(stderr, "uso: %s tcp|unix\n", argv[0]);
        return 1;
    }
    int tcp = strcmp(argv[1], "tcp") == 0;
    struct sockaddr_storage end;
    socklen_t tam;
    int srv = escutar(tcp, &end, &tam);

    if (fork() == 0) {                          /* FILHO: envia TOTAL bytes */
        close(srv);
        int s = socket(tcp ? AF_INET : AF_UNIX, SOCK_STREAM, 0);
        if (connect(s, (struct sockaddr *)&end, tam) < 0) { perror("connect"); _exit(1); }
        static char buf[BLOCO];
        for (long env = 0; env < TOTAL; env += BLOCO)
            if (write(s, buf, BLOCO) != BLOCO) break;
        _exit(0);
    }

    int con = accept(srv, NULL, NULL);          /* PAI: recebe tudo e mede */
    struct timespec t0, t1;
    clock_gettime(CLOCK_MONOTONIC, &t0);
    static char buf[BLOCO];
    long lido = 0; ssize_t r;
    while ((r = read(con, buf, BLOCO)) > 0) lido += r;
    clock_gettime(CLOCK_MONOTONIC, &t1);
    close(con);
    wait(NULL);

    double us = ((t1.tv_sec - t0.tv_sec) * 1e9 + (t1.tv_nsec - t0.tv_nsec)) / 1e3;
    printf("[%s] %ld MB em %.1f ms -> %.0f MB/s\n", tcp ? "TCP " : "UNIX",
           lido >> 20, us / 1e3, (lido / 1048576.0) / (us / 1e6));
    if (!tcp) unlink(CAMINHO);
    return 0;
}
```

```bash
$ cd ex6
$ gcc -O2 -Wall -Wextra -o bench_latencia bench_latencia.c
$ gcc -O2 -Wall -Wextra -o bench_vazao bench_vazao.c
$ for i in 1 2 3; do ./bench_latencia tcp; ./bench_latencia unix; done
$ for i in 1 2 3; do ./bench_vazao tcp; ./bench_vazao unix; done
```

**Saída real capturada:**

```
[TCP ] 100000 idas e voltas em 3005.4 ms -> 30.05 us por ida e volta
[UNIX] 100000 idas e voltas em 2457.5 ms -> 24.57 us por ida e volta
[TCP ] 100000 idas e voltas em 3159.5 ms -> 31.60 us por ida e volta
[UNIX] 100000 idas e voltas em 2010.0 ms -> 20.10 us por ida e volta
[TCP ] 100000 idas e voltas em 2799.0 ms -> 27.99 us por ida e volta
[UNIX] 100000 idas e voltas em 2070.4 ms -> 20.70 us por ida e volta
```
```
[TCP ] 256 MB em 120.4 ms -> 2127 MB/s
[UNIX] 256 MB em 62.8 ms -> 4075 MB/s
[TCP ] 256 MB em 110.9 ms -> 2308 MB/s
[UNIX] 256 MB em 85.9 ms -> 2980 MB/s
[TCP ] 256 MB em 121.0 ms -> 2116 MB/s
[UNIX] 256 MB em 47.9 ms -> 5347 MB/s
```

**Em `ex6/respostas.txt`:**

- [ ] **Q25.** Monte uma tabela com a média das três execuções de cada caso.
  Quantas vezes o socket Unix foi mais rápido na **latência**, e quantas na
  **vazão**? Por que a vantagem é tão diferente nas duas medidas? *(Dica: no
  ping-pong de 1 byte, o que cada ida e volta obriga o kernel a fazer com os
  dois processos?)*
- [ ] **Q26.** Para cada cenário, indique o **tipo de socket** (TCP, UDP ou
  Unix) e o **modelo de servidor** (iterativo, processo por cliente ou thread
  por cliente), com uma frase de justificativa:
  - a) Um sensor envia 10 leituras de temperatura por segundo a um coletor na
    rede; perder uma leitura de vez em quando não importa.
  - b) Um daemon local recebe comandos de uma ferramenta de linha de comando
    na mesma máquina e só pode aceitar comandos vindos do usuário `root`.
  - c) Um jogo de perguntas e respostas pela rede mantém um placar único para
    cerca de 30 jogadores conectados ao mesmo tempo; cada resposta atualiza o
    placar, e todos precisam ver o placar mais recente.
  - d) Um servidor executa, para cada cliente, um interpretador de scripts
    enviados pelo próprio cliente, que pode travar ou causar *segmentation
    fault* — e a falha de um cliente não pode derrubar os outros.

---

## Entrega <!-- e critérios de avaliação -->

### O que entregar

Uma pasta `lab12-nome/` contendo:

**Parte A (entrega ao final do encontro):**

- [ ] `ex1/eco_servidor.c`, `ex1/eco_cliente.c`, `ex1/eco_servidor_semhtons.c`, `ex1/eco_servidor_reuse.c`, `ex1/respostas.txt`
- [ ] `ex2/fluxo_servidor.c`, `ex2/fluxo_servidor5.c`, `ex2/fluxo_cliente.c`, `ex2/udp_servidor.c`, `ex2/udp_servidor3.c`, `ex2/udp_cliente.c`, `ex2/respostas.txt`
- [ ] `ex3/eco_fork.c`, `ex3/eco_fork_bug.c`, `ex3/respostas.txt`

**Parte B (entrega até o encontro seguinte):**

- [ ] `ex4/uds_servidor.c`, `ex4/uds_servidor2.c`, `ex4/uds_cliente.c`, `ex4/respostas.txt`
- [ ] `ex5/eco_thread.c`, `ex5/eco_fork_contador.c`, `ex5/eco_thread_crash.c`, `ex5/eco_fork_crash.c`, `ex5/respostas.txt`
- [ ] `ex6/bench_latencia.c`, `ex6/bench_vazao.c`, `ex6/respostas.txt`

<!-- 
### Rubrica (10 pontos)

| Critério | Pontos | O que é avaliado |
|---|---|---|
| Ex. 1 — TCP, `htons` e `SO_REUSEADDR` | 2,0 | Servidor e cliente funcionando; explica a porta 37907; reproduziu o "Address already in use" e as combinações com `SO_REUSEADDR` |
| Ex. 2 — Fluxo × datagrama | 1,5 | Mensagens grudadas e partidas no TCP; fronteiras e truncamento no UDP; propõe enquadramento |
| Ex. 3 — Vários clientes | 2,5 | Explica a fila do `accept()`; `eco_fork` correto (dois `close()`, `SIGCHLD`); reproduziu e explicou o descritor vazado |
| Ex. 4 — Sockets Unix | 1,0 | Arquivo de socket, `unlink()` e seu risco, `SO_PEERCRED` |
| Ex. 5 — Thread por cliente | 1,5 | Servidor com threads funcionando; explica o `malloc()` e o `pthread_detach()`; contador compartilhado × `fork()`; reproduziu o *crash* e explica por que uma thread derruba todas |
| Ex. 6 — Medição e escolha | 1,5 | Benchmarks executados de fato; explica a diferença latência × vazão; Q26 coerente |

> **Dica:** Nos Exercícios 1, 3, 4 e 5, o erro **de propósito** é a parte mais
> importante — o professor vai olhar se você reproduziu o problema antes de
> corrigi-lo, não só se o código final funciona.

---

## Anexo A — Referência rápida: sockets

```c
#include <sys/socket.h>
#include <netinet/in.h>     /* struct sockaddr_in, INADDR_ANY */
#include <arpa/inet.h>      /* htons, ntohs, inet_pton, inet_ntop */
#include <sys/un.h>         /* struct sockaddr_un */

int socket(int dominio, int tipo, 0);        /* AF_INET/AF_UNIX, SOCK_STREAM/SOCK_DGRAM */
int setsockopt(int s, SOL_SOCKET, SO_REUSEADDR, &sim, sizeof(sim));   /* antes do bind */
int bind(int s, const struct sockaddr *end, socklen_t tam);
int listen(int s, int fila);
int accept(int s, struct sockaddr *cli, socklen_t *tam);   /* devolve um socket NOVO */
int connect(int s, const struct sockaddr *srv, socklen_t tam);
ssize_t sendto(int s, const void *buf, size_t n, 0, const struct sockaddr *dest, socklen_t tam);
ssize_t recvfrom(int s, void *buf, size_t n, 0, struct sockaddr *de, socklen_t *tam);

uint16_t htons(uint16_t porta);              /* host -> rede; ntohs faz o contrario */
inet_pton(AF_INET, "127.0.0.1", &end.sin_addr);
inet_ntop(AF_INET, &cli.sin_addr, texto, INET_ADDRSTRLEN);
```

```bash
$ ss -tlnp                  # quem escuta em TCP (t=TCP, l=listen, n=numerico, p=processo)
$ ss -tan                   # todas as conexoes TCP, com o estado
$ ss -ulnp                  # quem escuta em UDP
$ nc 127.0.0.1 5012         # cliente TCP generico (Ctrl+D envia EOF)
$ nc -u 127.0.0.1 5012      # cliente UDP
$ nc -l 5012                # servidor TCP generico
```

## Anexo B — Referência rápida: threads

```c
#define _GNU_SOURCE                      /* gettid() */
#include <pthread.h>                     /* compile com -pthread */

void *atende(void *arg);                 /* a funcao que a thread executa */
pthread_t t;
pthread_create(&t, NULL, atende, arg);   /* retorna 0 ou um codigo de erro (nao usa errno) */
pthread_detach(t);                       /* libera tudo sozinha ao terminar...        */
pthread_join(t, NULL);                   /* ...ou alguem espera por ela (nunca os dois) */

static pthread_mutex_t trava = PTHREAD_MUTEX_INITIALIZER;
pthread_mutex_lock(&trava);
/* secao critica: leitura e escrita do dado compartilhado */
pthread_mutex_unlock(&trava);

pid_t gettid(void);                      /* numero da thread (o LWP do ps -L) */
```

```bash
$ ps -L -o pid,lwp,nlwp,comm -p <PID>    # uma linha por thread
$ ls /proc/<PID>/task                    # um diretorio por thread
```

Regras de ouro: tudo o que é **global** ou foi alocado com `malloc()` é visto
por todas as threads — proteja com mutex (Q23); e um erro fatal numa thread
encerra o **processo inteiro** (Q24).

## Anexo C — Tabela-resumo: que socket e que servidor usar

| Situação | Escolha recomendada |
|---|---|
| Comunicação confiável pela rede | TCP |
| Mensagens curtas e frequentes, perda tolerável | UDP |
| Processos na mesma máquina, ou é preciso saber quem é o cliente | Socket Unix (+ `SO_PEERCRED`) |
| Poucos clientes, requisições curtas | Servidor iterativo |
| Isolamento entre clientes (um não pode derrubar o outro) | Processo por cliente (`fork`) |
| Clientes que compartilham estado (placar, contadores, cache) | Thread por cliente (+ mutex) |

**Testado e validado em sandbox.** Os Exercícios 1 a 4 e 6 são idênticos aos
do Laboratório 12 (código comparado byte a byte com o guia original). Todos os
programas foram compilados com `gcc -Wall -Wextra` (e `-pthread` nos de
threads) sem nenhum warning e executados: os dos Exercícios 1 a 4 e 6 na
validação de 2026-10-26, e os quatro do Exercício 5 (`eco_thread.c`,
`eco_fork_contador.c`, `eco_thread_crash.c`, `eco_fork_crash.c`) em 2026-11-16.
O código mostrado no guia foi injetado a partir dos próprios arquivos testados
e conferido byte a byte com eles. As saídas são reais e foram capturadas com
os comandos exatamente como estão no guia, num pseudo-terminal (como o aluno
verá). As quatro combinações de `SO_REUSEADDR` do Exercício 1 foram testadas
uma a uma, sempre partindo da porta livre; a porta ficou bloqueada por 62 s
depois de encerrado o cliente.
-->