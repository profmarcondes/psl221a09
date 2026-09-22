# Laboratório 06 <br> Shell Scripting

<!-- **PSL221A09 — Programação em Sistemas Linux**

| | |
|---|---|
| **Duração estimada** | Parte A: 90–100 min (presencial) · Parte B: extraclasse |
| **Formato** | Duplas ou trios, no LSC/LSI (Parte A) — mesma dupla dá sequência na Parte B |
| **Pré-requisito** | Bash, `tar`, `grep`, `sed`, `awk` (padrão em qualquer Ubuntu/Debian) |
| **Entregável** | Pasta `laboratorio06-nomes-da-dupla/` com os 6 exercícios (`ex1` a `ex6`) |

---
-->

Este guia consolida shell scripting: variáveis, expansões, redirecionamento, estruturas de controle, funções, tratamento de opções e sinais, e os três filtros de texto mais usados no dia a dia (`grep`, `sed`, `awk`). A **Parte A** (três exercícios) é conduzida de forma presencial; a **Parte B** (três exercícios) é atividade extraclasse. A numeração das perguntas (`Qn`) é **contínua** ao longo dos seis exercícios — siga-os em ordem. Sempre que houver uma seção de **saída esperada**, compare com o que apareceu no seu terminal antes de seguir em frente.

## Preparação do ambiente

```bash
$ bash --version
$ awk --version
$ mkdir -p working_dir/lab06-nome/{ex1,ex2,ex3,ex4,ex5,ex6}
$ cd working_dir/lab06-nome
```

---

# Parte A — Presencial

## Exercício 1 — Script de backup com compressão e *timestamp* (25–30 min)

> **Objetivo:** Escrever um script que compacta uma pasta em um arquivo `.tar.gz` cujo nome inclui data e hora, tratando erros de uso (argumentos faltando, pasta inexistente).

Crie `ex1/backup.sh`:

```bash
#!/bin/bash
set -euo pipefail

ORIGEM="${1:?Uso: $0 <pasta_origem> <pasta_destino>}"
DESTINO="${2:?Uso: $0 <pasta_origem> <pasta_destino>}"

if [ ! -d "$ORIGEM" ]; then
    echo "Erro: pasta de origem '$ORIGEM' nao existe." >&2
    exit 1
fi

mkdir -p "$DESTINO"

TIMESTAMP=$(date +%Y%m%d_%H%M%S)
NOME_BASE=$(basename "$ORIGEM")
ARQUIVO_BACKUP="${DESTINO}/${NOME_BASE}_${TIMESTAMP}.tar.gz"

tar -czf "$ARQUIVO_BACKUP" -C "$(dirname "$ORIGEM")" "$NOME_BASE"

TAMANHO=$(du -h "$ARQUIVO_BACKUP" | cut -f1)
echo "Backup criado: $ARQUIVO_BACKUP ($TAMANHO)"
```

**O que cada parte faz (não precisa decorar, mas precisa entender):**

- `set -euo pipefail`: o script **para** no primeiro erro, no uso de variável não definida, ou se qualquer comando de um pipeline falhar — evita continuar silenciosamente depois de um erro
- `"${1:?mensagem}"`: se `$1` não foi passado, imprime `mensagem` e sai — é assim que o script cobra os argumentos
- `$(date +%Y%m%d_%H%M%S)`: gera o *timestamp* no formato `AAAAMMDD_HHMMSS`

Teste:

```bash
$ cd ex1
$ chmod +x backup.sh
$ mkdir -p dados_importantes
$ echo "relatorio 1" > dados_importantes/relatorio1.txt
$ echo "relatorio 2" > dados_importantes/relatorio2.txt
$ ./backup.sh dados_importantes backups
$ ls -la backups/
$ tar -tzf backups/*.tar.gz
```

**Saída esperada (o timestamp varia):**

```
Backup criado: backups/dados_importantes_20260815_235319.tar.gz (4.0K)
```

E o `tar -tzf` deve listar `dados_importantes/`, `dados_importantes/relatorio1.txt` e `dados_importantes/relatorio2.txt`.

Agora teste os dois casos de erro:

```bash
$ ./backup.sh                         # sem argumentos
$ ./backup.sh pasta_que_nao_existe backups   # origem inexistente
```

**Responda no arquivo `ex1/respostas.txt`:**

- [ ] **Q1.** O que aconteceu ao rodar `./backup.sh` sem argumentos? Qual foi o código de saída (`echo $?` logo depois)?
- [ ] **Q2.** O que aconteceu ao passar uma pasta de origem inexistente? O script criou algum arquivo de backup mesmo assim?
- [ ] **Q3.** Rode o backup **duas vezes seguidas**, sem esperar um segundo entre elas. Os dois arquivos `.tar.gz` gerados têm nomes diferentes? Por quê (ou por que não)?
- [ ] **Q4.** Modifique o script para aceitar um **terceiro argumento opcional**: um padrão de exclusão (ex.: `*.log`) a passar para `tar --exclude`. Documente o comando que você usou para testar.

---

## Exercício 2 — Script de deploy com opções de linha de comando (25–30 min)

> **Objetivo:** Escrever um script que aceita `--env`, `--version` e `--dry-run` em qualquer ordem, com validação de argumentos.

> **Nota:** `getopts` só processa opções de **uma letra**. Para opções longas como `--env`, o padrão é um laço manual com `case` — é isso que vocês vão implementar aqui.

Crie `ex2/deploy.sh`:

```bash
#!/bin/bash
set -euo pipefail

VERSION="1.0.0"
AMBIENTE=""
DRY_RUN=0

uso() {
    echo "Uso: $0 --env <ambiente> [--dry-run] [--version]"
    exit 1
}

while [[ $# -gt 0 ]]; do
    case "$1" in
        --env)
            AMBIENTE="$2"
            shift 2
            ;;
        --dry-run)
            DRY_RUN=1
            shift
            ;;
        --version)
            echo "deploy.sh versao $VERSION"
            exit 0
            ;;
        -h|--help)
            uso
            ;;
        *)
            echo "Opcao desconhecida: $1" >&2
            uso
            ;;
    esac
done

if [ -z "$AMBIENTE" ]; then
    echo "Erro: --env e obrigatorio (dev, staging ou producao)" >&2
    uso
fi

case "$AMBIENTE" in
    dev|staging|producao) ;;
    *)
        echo "Erro: ambiente invalido '$AMBIENTE'" >&2
        exit 1
        ;;
esac

if [ "$DRY_RUN" -eq 1 ]; then
    echo "[DRY-RUN] Simularia deploy para: $AMBIENTE"
else
    echo "Fazendo deploy para: $AMBIENTE"
fi
```

Teste todos os casos:

```bash
$ cd ex2
$ chmod +x deploy.sh
$ ./deploy.sh --version
$ ./deploy.sh --env staging --dry-run
$ ./deploy.sh --env producao
$ ./deploy.sh --env invalido
$ ./deploy.sh
```

**Saída esperada, na mesma ordem dos comandos acima:**

```
deploy.sh versao 1.0.0
[DRY-RUN] Simularia deploy para: staging
Fazendo deploy para: producao
Erro: ambiente invalido 'invalido'
Erro: --env e obrigatorio (dev, staging ou producao)
Uso: ./deploy.sh --env <ambiente> [--dry-run] [--version]
```

**Responda no arquivo `ex2/respostas.txt`:**

- [ ] **Q5.** No laço `while [[ $# -gt 0 ]]`, o que `shift 2` faz de diferente de um `shift` sozinho? Por que `--env` precisa de `shift 2` e `--dry-run` só de `shift`?
- [ ] **Q6.** Rode `./deploy.sh --env staging --env producao` (repetindo `--env`). Qual ambiente "venceu"? Isso é o comportamento que você esperava?
- [ ] **Q7.** Adicione uma nova opção `--force` que pula a validação da lista `dev|staging|producao`. Cole o trecho de código adicionado e mostre um teste com um ambiente "inválido" mas com `--force`.

---

## Exercício 3 — Pipeline com grep, sed e awk sobre CSV (25–30 min)

> **Objetivo:** Combinar os três filtros de texto para extrair, transformar e resumir dados de um arquivo CSV, sem escrever um programa em C ou Python para isso.

Crie `ex3/alunos.csv`:

```
nome,nota1,nota2,nota3
Ana Silva,8.5,7.0,9.0
Bruno Costa,4.0,5.5,6.0
Carla Dias,9.0,9.5,8.0
Diego Souza,3.0,4.5,5.0
Elisa Melo,7.5,8.0,7.0
```

Rode e observe cada comando **separadamente** primeiro:

```bash
$ cd ex3

# grep: linhas de alunos com nota1 abaixo de 5
$ grep -E "^[A-Za-z ]+,[0-4]\." alunos.csv

# sed: trocar virgula por ponto-e-virgula, exceto no cabecalho
$ sed '1!s/,/;/g' alunos.csv

# awk: calcular a media de cada aluno e classificar
$ awk -F',' 'NR==1 {next} {media=($2+$3+$4)/3; status=(media>=6)?"APROVADO":"REPROVADO"; printf "%-15s media=%.2f  %s\n", $1, media, status}' alunos.csv
```

**Saída esperada do `awk` acima:**

```
Ana Silva       media=8.17  APROVADO
Bruno Costa     media=5.17  REPROVADO
Carla Dias      media=8.83  APROVADO
Diego Souza     media=4.17  REPROVADO
Elisa Melo      media=7.50  APROVADO
```

Agora combine em um **pipeline único** que lista só os reprovados, ordenados pela menor média:

```bash
$ awk -F',' 'NR==1 {next} {media=($2+$3+$4)/3; if (media<6) printf "%.2f %s\n", media, $1}' alunos.csv | sort -n
```

**Saída esperada:**

```
4.17 Diego Souza
5.17 Bruno Costa
```

**Responda no arquivo `ex3/respostas.txt`:**

- [ ] **Q8.** No comando `grep -E "^[A-Za-z ]+,[0-4]\."`, explique o que cada parte da expressão regular faz: `^`, `[A-Za-z ]+`, `,`, `[0-4]`, `\.`.
- [ ] **Q9.** No `sed '1!s/,/;/g'`, o que o `1!` faz? O que aconteceria se você removesse essa parte e rodasse só `sed 's/,/;/g'`?
- [ ] **Q10.** Escreva **seu próprio** comando `awk` (uma linha) que imprime só o nome dos alunos **aprovados** (média ≥ 6), em ordem alfabética. Combine `awk` com `sort` como no exemplo do pipeline.
- [ ] **Q11.** Entre `grep`, `sed` e `awk`, qual você usaria para: (a) só **encontrar** linhas que combinam com um padrão; (b) **substituir** um padrão de texto por outro; (c) fazer **cálculos** sobre colunas numéricas? Justifique cada escolha em uma frase.

---

# Parte B — Extraclasse

## Exercício 4 — Analisador de log (25–30 min)

> **Objetivo:** Escrever um script que lê um arquivo de log, conta ocorrências de `ERROR`/`WARNING`, e destaca os erros com um alerta se passarem de um limite.

Crie `ex4/servidor.log` (dados de teste):

```
2026-08-15 10:00:01 INFO Servidor iniciado na porta 8080
2026-08-15 10:00:05 INFO Cliente 192.168.1.10 conectado
2026-08-15 10:01:12 ERROR Falha ao conectar ao banco de dados: timeout
2026-08-15 10:01:15 WARNING Reutilizando conexao expirada
2026-08-15 10:02:00 INFO Cliente 192.168.1.11 conectado
2026-08-15 10:02:30 ERROR Falha de autenticacao para usuario 'admin'
2026-08-15 10:03:00 INFO Requisicao processada em 120ms
2026-08-15 10:04:45 ERROR Disco quase cheio: 95% de uso
2026-08-15 10:05:00 WARNING Latencia acima do normal: 800ms
2026-08-15 10:06:00 INFO Cliente 192.168.1.10 desconectado
```

Crie `ex4/analisa_log.sh`:

```bash
#!/bin/bash
set -euo pipefail

ARQUIVO="${1:?Uso: $0 <arquivo_de_log>}"

if [ ! -f "$ARQUIVO" ]; then
    echo "Erro: arquivo '$ARQUIVO' nao encontrado." >&2
    exit 1
fi

TOTAL=$(wc -l < "$ARQUIVO")
TOTAL_ERROS=$(grep -c "ERROR" "$ARQUIVO" || true)
TOTAL_WARNINGS=$(grep -c "WARNING" "$ARQUIVO" || true)

echo "===== Relatorio de log: $ARQUIVO ====="
echo "Total de linhas:    $TOTAL"
echo "Total de ERROS:     $TOTAL_ERROS"
echo "Total de WARNINGS:  $TOTAL_WARNINGS"
echo ""
echo "--- Detalhe dos erros ---"

grep "ERROR" "$ARQUIVO" | while read -r linha; do
    HORA=$(echo "$linha" | awk '{print $2}')
    MSG=$(echo "$linha" | cut -d' ' -f4-)
    echo "[$HORA] $MSG"
done

if [ "$TOTAL_ERROS" -gt 5 ]; then
    echo ""
    echo "ALERTA: mais de 5 erros encontrados!"
fi
```

Teste:

```bash
$ cd ex4
$ chmod +x analisa_log.sh
$ ./analisa_log.sh servidor.log
```

**Saída esperada:**

```
===== Relatorio de log: servidor.log =====
Total de linhas:    10
Total de ERROS:     3
Total de WARNINGS:  2

--- Detalhe dos erros ---
[10:01:12] Falha ao conectar ao banco de dados: timeout
[10:02:30] Falha de autenticacao para usuario 'admin'
[10:04:45] Disco quase cheio: 95% de uso
```

**Responda no arquivo `ex4/respostas.txt`:**

- [ ] **Q12.** Por que o script usa `grep -c "ERROR" "$ARQUIVO" || true` em vez de só `grep -c "ERROR" "$ARQUIVO"`? (dica: pense no que `grep` retorna como código de saída quando **não encontra nada**, e no efeito do `set -e` no início do script)
- [ ] **Q13.** Adicione ao `servidor.log` linhas suficientes com `ERROR` para passar de 5 no total, e rode o script de novo. O alerta apareceu?
- [ ] **Q14.** Modifique o script para também reportar a **porcentagem** de linhas que são erro (`ERROR`) em relação ao total de linhas, usando `awk` ou `$(( ))`. Cole o trecho de código que você adicionou.

---

## Exercício 5 — Monitor de recursos com `trap` (25–30 min)

> **Objetivo:** Escrever um script que monitora uso de disco/memória em um laço, e usa `trap` para garantir uma saída limpa quando interrompido.

Crie `ex5/monitora_recursos.sh`:

```bash
#!/bin/bash
set -euo pipefail

LIMITE_DISCO=80
LIMITE_MEM=80
ARQUIVO_LOG=$(mktemp)

finalizar() {
    echo ""
    echo "Encerrando monitoramento. Log salvo em: $ARQUIVO_LOG"
    exit 0
}

trap finalizar SIGINT SIGTERM

echo "Monitorando recursos a cada 2s (Ctrl+C para parar)..."
echo "Log temporario: $ARQUIVO_LOG"
\
```

Teste rodando e interrompendo com `Ctrl+C`:

```bash
$ cd ex5
$ chmod +x monitora_recursos.sh
$ ./monitora_recursos.sh
# deixe rodar por ~10 segundos, depois pressione Ctrl+C
```

**Saída esperada (os valores variam):**

```
Monitorando recursos a cada 2s (Ctrl+C para parar)...
Log temporario: /tmp/tmp.XXXXXXXXXX
00:12:01 disco=29% mem=8%
00:12:03 disco=29% mem=8%
^C
Encerrando monitoramento. Log salvo em: /tmp/tmp.XXXXXXXXXX
```

**Responda no arquivo `ex5/respostas.txt`:**

- [ ] **Q15.** O que aconteceria se você **removesse** a linha `trap finalizar SIGINT SIGTERM` e desse `Ctrl+C`? O arquivo de log ainda existiria depois? A mensagem "Encerrando monitoramento" apareceria?
- [ ] **Q16.** Depois de interromper o script, confirme que o arquivo de log (`cat` no caminho impresso) tem as linhas registradas. Isso mostra que o `tee -a` estava funcionando mesmo com o script em laço infinito?
- [ ] **Q17.** Modifique o script para **também** capturar o sinal `SIGTERM` de forma diferente de `SIGINT` (por exemplo, imprimindo qual dos dois foi recebido). Dica: registre dois `trap`s separados, um por sinal, cada um chamando uma função diferente.

---

## Exercício 6 — Desafio: parser de CSV em `awk` (25–30 min)

> **Objetivo:** Escrever um script `awk` que processa um CSV de vendas e produz um relatório agregado por vendedor, sem usar nenhuma outra linguagem.

Crie `ex6/vendas.csv`:

```
data,vendedor,produto,valor
2026-08-01,Ana,Notebook,3500.00
2026-08-01,Bruno,Mouse,45.00
2026-08-02,Ana,Teclado,120.00
2026-08-02,Carla,Monitor,890.00
2026-08-03,Bruno,Notebook,3200.00
2026-08-03,Ana,Mouse,50.00
```

Crie `ex6/resumo_vendas.awk`:

```awk
BEGIN {
    FS = ","
}
NR == 1 { next }
{
    total[$2] += $4
    contagem[$2]++
    total_geral += $4
}
END {
    printf "%-10s %10s %8s\n", "Vendedor", "Total", "Vendas"
    for (vendedor in total) {
        printf "%-10s %10.2f %8d\n", vendedor, total[vendedor], contagem[vendedor]
    }
    printf "\n%-10s %10.2f\n", "GERAL", total_geral
}
```

Rode:

```bash
$ cd ex6
$ awk -f resumo_vendas.awk vendas.csv
```

**Saída esperada (a ordem dos vendedores pode variar — `awk` não garante ordem em arrays associativos):**

```
Vendedor        Total   Vendas
Carla          890.00        1
Ana           3670.00        3
Bruno         3245.00        2

GERAL         7805.00
```

**Responda no arquivo `ex6/respostas.txt`:**

- [ ] **Q18.** O que `NR == 1 { next }` faz? O que aconteceria com o `total_geral` se essa linha fosse removida?
- [ ] **Q19.** `total[$2] += $4` usa `$2` (vendedor) como **chave** de um array associativo. Isso funcionaria se `$2` fosse `$1` (a data) em vez do vendedor? O que mudaria no significado do relatório?
- [ ] **Q20.** Modifique `resumo_vendas.awk` para também imprimir, para cada vendedor, o **ticket médio** (total dividido pela contagem de vendas). Cole o trecho adicionado e o resultado.
- [ ] **Q21.** Modifique o script para imprimir os vendedores **ordenados por total, do maior para o menor** (dica: pesquise `asort`/`asorti`, disponíveis no `gawk`, ou redirecione a saída para `sort -k2 -rn`).

---

## Entrega  <!-- e critérios de avaliação -->

### O que entregar

Uma pasta `lab06-nome/` contendo:

**Parte A (presencial):**
- [ ] `ex1/backup.sh` (versão final, com o argumento de exclusão), `ex1/respostas.txt`
- [ ] `ex2/deploy.sh` (com `--force`), `ex2/respostas.txt`
- [ ] `ex3/alunos.csv`, `ex3/respostas.txt` (incluindo o comando `awk` do Q10)

**Parte B (extraclasse):**
- [ ] `ex4/servidor.log` (com as linhas extras de erro), `ex4/analisa_log.sh` (versão final, com percentual), `ex4/respostas.txt`
- [ ] `ex5/monitora_recursos.sh` (com tratamento diferenciado de sinais), `ex5/respostas.txt`
- [ ] `ex6/vendas.csv`, `ex6/resumo_vendas.awk` (com ticket médio e ordenação), `ex6/respostas.txt`

<!-- ### Rubrica (10 pontos)

| Critério | Pontos | O que é avaliado |
|---|---|---|
| Ex.1 — Backup (Parte A) | 2,0 | Script trata os dois casos de erro; *timestamp* funciona; extensão do `--exclude` funcional |
| Ex.2 — Deploy (Parte A) | 2,0 | Parsing de opções longas correto; validação de ambiente; opção `--force` funcional |
| Ex.3 — grep/sed/awk (Parte A) | 1,5 | Comandos reproduzidos corretamente; comando `awk` do Q10 funcional e correto |
| Ex.4 — Log (Parte B) | 1,5 | Contagem e detalhe de erros corretos; alerta de limite funcional; percentual adicionado corretamente |
| Ex.5 — `trap` (Parte B) | 1,5 | Encerramento limpo via `trap`; sinais tratados de forma diferenciada |
| Ex.6 — `awk` (Parte B) | 1,5 | Ticket médio e ordenação implementados corretamente |

> **Dica:** Scripts que funcionam só "no caminho feliz" (sem tratar erro de argumento faltando ou arquivo inexistente) não atendem à rubrica — isso é intencional: tratar erro é parte do que se está avaliando, não um extra. No Exercício 5, o professor pode testar o script mandando `Ctrl+C` no meio da execução — se o script não limpar/encerrar direito, isso conta contra a nota, mesmo que o "caminho feliz" funcione.

---

## Anexo A — Referência rápida: bash básico, redirecionamento e filtros de texto

```bash
# Variaveis e expansoes
var="valor"              # sem espaco ao redor do =
echo "$var" "${var}"      # uso com e sem chaves
echo "$(comando)"         # substituicao de comando
echo "$((2 + 3))"          # aritmetica inteira

# Redirecionamento
comando > arquivo          # sobrescreve stdout
comando >> arquivo         # acrescenta stdout
comando 2> erros.txt       # so stderr
comando > tudo.txt 2>&1    # stdout e stderr juntos
comando1 | comando2        # pipe: stdout de um -> stdin do outro

# Controle de fluxo
if [ -f "$1" ]; then ...; elif [ -d "$1" ]; then ...; else ...; fi
case "$1" in padrao1) ...;; padrao2|padrao3) ...;; *) ...;; esac
for item in lista; do ...; done
while [ condicao ]; do ...; done

# Filtros de texto
grep "padrao" arquivo             # filtra linhas
grep -c "padrao" arquivo          # so a contagem
grep -E "padrao_estendido" arquivo  # regex estendida (ERE): +, ?, |, (), {n,m} sem escapar
sed 's/de/para/g' arquivo         # substitui (nao edita o original sem -i)
sed '1!s/de/para/g' arquivo       # "1!" = em todas as linhas, exceto a 1 (endereco negado)
awk -F',' '{print $1}' arquivo    # imprime uma coluna
awk -F',' 'END{print NR}' arquivo # conta linhas
```

**Regex estendida (ERE, usada com `grep -E`) — tokens comuns:**

| Token | Significado |
|---|---|
| `^` / `$` | início / fim da linha |
| `.` | qualquer caractere (use `\.` para o ponto literal) |
| `[A-Za-z ]` | uma classe de caracteres — letras ou espaço, neste exemplo |
| `+` / `*` | uma-ou-mais / zero-ou-mais repetições do átomo anterior |
| `?` | zero-ou-uma ocorrência (torna o átomo anterior opcional) |
| `\|` | alternativa (ex.: `gcc\|clang`) |
| `(...)` | agrupamento — para aplicar `+`/`*`/`\|` a mais de um caractere |
| `{n,m}` | entre `n` e `m` repetições do átomo anterior |

**Subshells — o que muda ao rodar algo entre `( )` ou num pipe:**

```bash
(cd /tmp && ls)    # roda num subshell: o cd NÃO afeta o diretório do shell pai
echo "$PWD"        # continua no diretório original

contador=0
echo "a b c" | while read -r palavra; do contador=$((contador+1)); done
echo "$contador"   # imprime 0, não 3! cada lado de um pipe roda em um subshell,
                    # e variáveis alteradas dentro de um subshell não "voltam" pro pai
```

Esse é o motivo pelo qual, no Exercício 4, `grep "ERROR" "$ARQUIVO" | while read -r linha; do ...; done` só **imprime** dentro do laço — não tenta acumular um contador nessa mesma estrutura, porque o `while` roda num subshell (o `TOTAL_ERROS` do script é calculado **antes**, com `$(...)`, que roda num subshell separado mas captura a saída via *command substitution*, que sim é retornada ao shell pai).

---

## Anexo B — Referência rápida: opções longas, `getopts` e `trap`

```bash
# Parsing de opcoes longas (padrao manual)
while [[ $# -gt 0 ]]; do
    case "$1" in
        --opcao) VALOR="$2"; shift 2 ;;
        --flag)  LIGADO=1; shift ;;
        *)       echo "desconhecido: $1"; exit 1 ;;
    esac
done

# getopts (so opcoes de 1 letra)
while getopts "vo:h" opt; do
    case "$opt" in
        v) ... ;;
        o) VALOR="$OPTARG" ;;
    esac
done
shift $((OPTIND - 1))

# trap
trap 'minha_funcao' SIGINT SIGTERM   # sinais especificos
trap 'limpeza' EXIT                   # sempre, ao sair (de qualquer forma)
```

```awk
# awk: estrutura basica
BEGIN { FS = "," }              # roda uma vez, antes de ler qualquer linha
NR == 1 { next }                 # pula o cabecalho
{ acumulador[$2] += $4 }         # roda para CADA linha
END { print acumulador["Ana"] }  # roda uma vez, no final
```

---

## Anexo C — `backup.sh` com exclusão (referência)

```bash
#!/bin/bash
set -euo pipefail

ORIGEM="${1:?Uso: $0 <pasta_origem> <pasta_destino> [padrao_exclusao]}"
DESTINO="${2:?Uso: $0 <pasta_origem> <pasta_destino> [padrao_exclusao]}"
EXCLUSAO="${3:-}"

if [ ! -d "$ORIGEM" ]; then
    echo "Erro: pasta de origem '$ORIGEM' nao existe." >&2
    exit 1
fi

mkdir -p "$DESTINO"

TIMESTAMP=$(date +%Y%m%d_%H%M%S)
NOME_BASE=$(basename "$ORIGEM")
ARQUIVO_BACKUP="${DESTINO}/${NOME_BASE}_${TIMESTAMP}.tar.gz"

if [ -n "$EXCLUSAO" ]; then
    tar -czf "$ARQUIVO_BACKUP" --exclude="$EXCLUSAO" -C "$(dirname "$ORIGEM")" "$NOME_BASE"
else
    tar -czf "$ARQUIVO_BACKUP" -C "$(dirname "$ORIGEM")" "$NOME_BASE"
fi

TAMANHO=$(du -h "$ARQUIVO_BACKUP" | cut -f1)
echo "Backup criado: $ARQUIVO_BACKUP ($TAMANHO)"
```

**Testado e validado:** compacta corretamente, gera nomes únicos por *timestamp*, trata argumentos faltando e pasta de origem inexistente com mensagens de erro e código de saída `1`.

## Anexo D — `resumo_vendas.awk` com ticket médio e ordenação (referência)

```awk
BEGIN {
    FS = ","
}
NR == 1 { next }
{
    total[$2] += $4
    contagem[$2]++
    total_geral += $4
}
END {
    printf "%-10s %10s %8s %12s\n", "Vendedor", "Total", "Vendas", "Ticket medio"
    for (vendedor in total) {
        media = total[vendedor] / contagem[vendedor]
        linha[vendedor] = sprintf("%-10s %10.2f %8d %12.2f",
                                   vendedor, total[vendedor], contagem[vendedor], media)
        chave[vendedor] = total[vendedor]
    }
    n = asorti(chave, ordenado, "@val_num_desc")
    for (i = 1; i <= n; i++) {
        print linha[ordenado[i]]
    }
    printf "\n%-10s %10.2f\n", "GERAL", total_geral
}
```

**Testado e validado:** produz o mesmo total (`7805.00`) do script original, com ticket médio calculado corretamente e vendedores ordenados do maior para o menor total (requer `gawk`; em `awk` sem `asorti`, use `| sort -k2 -rn` na saída em vez da ordenação interna).

-->