---
title: "Concorrência em Go: O Modelo de Threads e o Scheduler (M:P:N)"
date: 2026-05-19T17:20:51-05:00
draft: false
tags: ["go", "golang", "backend", "senior"]
categories: ["Treinamento Avançado"]
---

# Concorrência em Go: O Modelo de Threads e o Scheduler (M:P:N)

A concorrência é uma pedra angular da linguagem Go, projetada desde o início para facilitar a escrita de programas que executam múltiplas tarefas simultaneamente. Diferente de outras linguagens que expõem diretamente threads do sistema operacional, Go introduz um modelo de concorrência mais leve e eficiente, abstraindo a complexidade do gerenciamento de threads através de goroutines e um scheduler sofisticado. Este artigo explora em profundidade o funcionamento interno desse modelo, conhecido como M:P:N, focando nos componentes G (Goroutine), M (Machine/OS Thread), P (Processor/Logical Processor), nas runqueues e no mecanismo de work stealing.

## G: Goroutines (As Threads Leves de Go)

No coração do modelo de concorrência de Go estão as **goroutines**, que podem ser pensadas como "threads leves" gerenciadas pelo runtime de Go, em vez do sistema operacional. Uma goroutine é uma função que executa concorrentemente com outras goroutines no mesmo espaço de endereço.

**Características Principais:**

*   **Leveza:** Uma goroutine começa com uma pilha de apenas alguns kilobytes (tipicamente 2KB), que pode crescer e encolher dinamicamente conforme necessário. Isso contrasta com as threads do sistema operacional, que geralmente alocam pilhas de megabytes, tornando a criação de milhares de threads de OS inviável.
*   **Criação Simples:** São criadas prefixando uma chamada de função com a palavra-chave `go`.
*   **Multiplexação:** Milhares de goroutines podem ser multiplexadas em um número muito menor de threads do sistema operacional (Ms).
*   **Cooperação:** Embora o scheduler de Go seja preemptivo (a partir do Go 1.14), as goroutines também podem ceder voluntariamente o controle usando `runtime.Gosched()`.
*   **Comunicação:** Go promove a comunicação entre goroutines através de canais (channels), seguindo o princípio "não se comunique compartilhando memória; compartilhe memória comunicando-se".

Cada goroutine (G) possui seu próprio contexto de execução, incluindo:
*   **Stack:** A pilha de execução da goroutine.
*   **Program Counter (PC):** O ponto de execução atual.
*   **State:** O estado da goroutine (e.g., `_Grunning`, `_Grunnable`, `_Gwaiting`).

Quando uma goroutine é criada, ela é colocada em uma fila de "prontas para executar" (runqueue) e aguarda ser agendada para execução em um M.

## M: Machines (As Threads do Sistema Operacional)

Um **M** representa uma thread do sistema operacional (OS thread). Estas são as threads reais que o kernel do sistema operacional gerencia e nas quais o código Go é executado.

**Papel e Interação:**

*   **Execução de Código:** Um M é responsável por executar o código Go. Ele interage diretamente com o hardware e o kernel do sistema operacional.
*   **Bloqueio:** Se uma goroutine em execução em um M realiza uma chamada de sistema bloqueante (e.g., I/O de rede, I/O de disco, `time.Sleep` longo), o M correspondente será bloqueado pelo sistema operacional.
*   **Criação e Destruição:** O runtime de Go pode criar novos Ms conforme necessário (e.g., quando um M existente é bloqueado por uma syscall e há goroutines prontas para executar) e pode destruí-los se ficarem ociosos por muito tempo.
*   **CGO:** Chamadas para código C (CGO) também podem bloquear Ms, e o scheduler de Go lida com isso de forma semelhante às syscalls.

É importante notar que o número de Ms pode variar dinamicamente. O runtime tenta manter um número ideal de Ms para aproveitar os núcleos de CPU disponíveis, mas pode criar mais Ms para lidar com operações bloqueantes.

## P: Processors (Os Processadores Lógicos)

Um **P** (Processor, ou Logical Processor) é uma abstração crucial no scheduler de Go. Ele representa um contexto de execução para goroutines e atua como um "processador virtual" para o runtime de Go.

**Função e Importância:**

*   **Contexto de Execução:** Um P é necessário para que um M possa executar código Go. Ele fornece os recursos necessários para a execução, incluindo uma runqueue local de goroutines.
*   **Runqueue Local:** Cada P mantém sua própria fila de goroutines prontas para executar. Isso minimiza a contenção e a necessidade de bloqueios globais, melhorando a escalabilidade.
*   **`GOMAXPROCS`:** O número de Ps é determinado pela variável de ambiente `GOMAXPROCS` ou pela função `runtime.GOMAXPROCS()`. Por padrão, `GOMAXPROCS` é definido como o número de núcleos lógicos de CPU disponíveis no sistema. Isso significa que, por padrão, Go tentará usar todos os núcleos da CPU de forma eficiente.
*   **Acoplamento M-P:** Um M deve estar associado a um P para executar goroutines. Quando um M é bloqueado por uma syscall, ele "desacopla" seu P, permitindo que outro M (existente ou recém-criado) pegue esse P e continue executando outras goroutines.

O P é a chave para a eficiência do scheduler de Go, pois ele gerencia a distribuição de trabalho e minimiza a necessidade de bloqueios globais, que são caros.

## O Modelo M:P:N: O Coração do Scheduler Go

O modelo M:P:N descreve como N goroutines (G) são multiplexadas em M threads do sistema operacional (M) através de P processadores lógicos (P).

**Visão Geral:**

1.  **G -> P:** Goroutines são agendadas para execução em Ps.
2.  **P -> M:** Cada P está associado a um M. O M executa as goroutines que estão na runqueue local do P.
3.  **N:M:** O scheduler de Go gerencia a relação N:M, onde N é o número de goroutines e M é o número de threads do sistema operacional. O número de Ps (controlado por `GOMAXPROCS`) determina o grau de paralelismo real.

O scheduler de Go opera em um loop contínuo:
*   Um M, associado a um P, pega uma goroutine de sua runqueue local.
*   Executa a goroutine até que ela bloqueie (e.g., I/O, canal), ceda o controle (`runtime.Gosched()`), ou seja preemptada (após um certo tempo, tipicamente 10ms).
*   Se a goroutine bloqueia em uma syscall, o M é bloqueado, mas o P é liberado para ser usado por outro M.
*   Se a goroutine cede ou é preemptada, ela é colocada de volta em uma runqueue (local ou global).
*   O M então procura a próxima goroutine para executar.

Este modelo permite que Go mantenha um alto grau de concorrência (muitas goroutines) com um paralelismo eficiente (aproveitando os núcleos da CPU) e resiliência a operações bloqueantes.

```go
package main

import (
	"fmt"
	"runtime"
	"sync"
	"time"
)

// worker simula uma goroutine realizando algum trabalho.
func worker(id int, wg *sync.WaitGroup) {
	defer wg.Done()
	fmt.Printf("Worker %d: Iniciando (GOMAXPROCS=%d, NumCPU=%d)\n", id, runtime.GOMAXPROCS(-1), runtime.NumCPU())
	// Simula trabalho que pode ser interrompido ou ceder
	time.Sleep(50 * time.Millisecond)
	fmt.Printf("Worker %d: Finalizado\n", id)
}

func main() {
	fmt.Println("--- Exemplo 1: Goroutines Básicas e GOMAXPROCS ---")

	// 1. Configuração com GOMAXPROCS=1
	// Isso força todas as goroutines a serem executadas sequencialmente em um único P (e M).
	runtime.GOMAXPROCS(1)
	fmt.Printf("GOMAXPROCS definido para %d. (Padrão é %d)\n", runtime.GOMAXPROCS(-1), runtime.NumCPU())

	var wg1 sync.WaitGroup
	fmt.Println("Lançando 5 workers com GOMAXPROCS=1:")
	for i := 0; i < 5; i++ {
		wg1.Add(1)
		go worker(i, &wg1)
	}
	wg1.Wait()
	fmt.Println("Todas as goroutines do Exemplo 1 (GOMAXPROCS=1) terminaram.")
	fmt.Println("--------------------------------------------------")

	// 2. Configuração com GOMAXPROCS = número de CPUs
	// Isso permite que as goroutines sejam executadas em paralelo em múltiplos Ps (e Ms).
	numCPU := runtime.NumCPU()
	runtime.GOMAXPROCS(numCPU)
	fmt.Printf("GOMAXPROCS redefinido para %d (número de CPUs).\n", runtime.GOMAXPROCS(-1))

	var wg2 sync.WaitGroup
	fmt.Println("Lançando 5 workers com GOMAXPROCS=NumCPU:")
	for i := 0; i < 5; i++ {
		wg2.Add(1)
		go worker(i+10, &wg2) // IDs diferentes para clareza
	}
	wg2.Wait()
	fmt.Println("Todas as goroutines do Exemplo 1 (GOMAXPROCS=NumCPU) terminaram.")
	fmt.Println("--------------------------------------------------")
}
```
No Exemplo 1, ao definir `GOMAXPROCS(1)`, observamos que as goroutines são executadas sequencialmente, uma após a outra, pois há apenas um P disponível para agendá-las em um M. Quando `GOMAXPROCS` é redefinido para o número de CPUs, as goroutines podem ser executadas em paralelo, aproveitando múltiplos Ps e Ms, resultando em uma conclusão mais rápida do conjunto de tarefas.

## Runqueues: Gerenciando Goroutines

O scheduler de Go utiliza duas principais estruturas de fila para gerenciar goroutines prontas para execução: as runqueues locais e a runqueue global.

### Runqueue Local (P's Runqueue)

*   **Localização:** Cada P possui sua própria runqueue local. Esta é a principal fonte de goroutines para um M associado a esse P.
*   **Eficiência:** O uso de runqueues locais minimiza a contenção entre Ms que buscam trabalho. Cada M tenta pegar goroutines de sua runqueue local primeiro, o que é uma operação sem bloqueio ou com bloqueio mínimo.
*   **Tamanho:** A runqueue local tem um tamanho fixo (tipicamente 256 goroutines).
*   **Adição/Remoção:** Quando uma goroutine é criada, ou quando uma goroutine que estava bloqueada se torna runnable novamente, ela é geralmente colocada na runqueue local do P atual. O M associado ao P pega goroutines do início da fila.

### Runqueue Global

*   **Localização:** Existe uma única runqueue global, acessível por todos os Ms.
*   **Uso:** A runqueue global é usada em cenários específicos:
    *   Quando uma runqueue local está cheia e uma nova goroutine precisa ser agendada.
    *   Quando goroutines são "desbloqueadas" de uma syscall e não há um P disponível para elas imediatamente.
    *   Como uma fonte de trabalho para Ms que não encontram goroutines em suas runqueues locais ou para Ps que estão ociosos.
*   **Contenção:** Acesso à runqueue global requer bloqueios (locks), o que a torna mais lenta do que as runqueues locais. O scheduler tenta minimizar o uso da runqueue global para manter a alta performance.

### Interação entre Runqueues

O scheduler prioriza as runqueues locais. Um M, ao procurar trabalho, primeiro verifica a runqueue local de seu P. Se estiver vazia, ele tenta buscar goroutines da runqueue global. Se a runqueue global também estiver vazia, o M entra no mecanismo de work stealing.

```go
package main

import (
	"fmt"
	"runtime"
	"sync"
	"time"
)

// blockingWorker simula uma goroutine que realiza uma syscall bloqueante.
func blockingWorker(id int, wg *sync.WaitGroup) {
	defer wg.Done()
	fmt.Printf("Blocking Worker %d: Iniciando syscall bloqueante. (M associado a P será bloqueado)\n", id)
	// time.Sleep é uma syscall que bloqueia a thread do OS (M).
	// O P associado a este M será liberado para outro M.
	time.Sleep(2 * time.Second)
	fmt.Printf("Blocking Worker %d: Syscall finalizada.\n", id)
}

// cpuBoundWorker simula uma goroutine que realiza trabalho intensivo de CPU.
func cpuBoundWorker(id int, wg *sync.WaitGroup) {
	defer wg.Done()
	fmt.Printf("CPU-bound Worker %d: Iniciando trabalho intensivo de CPU.\n", id)
	sum := 0
	for i := 0; i < 1_000_000_000; i++ { // Loop pesado
		sum += i
	}
	fmt.Printf("CPU-bound Worker %d: Trabalho intensivo finalizado. Sum: %d\n", id, sum)
}

func main() {
	fmt.Println("--- Exemplo 2: Syscall Bloqueante e Desacoplamento M-P ---")

	// Definimos GOMAXPROCS para o número de CPUs para permitir paralelismo.
	numCPU := runtime.NumCPU()
	runtime.GOMAXPROCS(numCPU)
	fmt.Printf("GOMAXPROCS definido para %d.\n", runtime.GOMAXPROCS(-1))

	var wg sync.WaitGroup
	wg.Add(2) // Uma goroutine bloqueante e uma CPU-bound

	go blockingWorker(1, &wg)
	go cpuBoundWorker(2, &wg)

	fmt.Println("Esperando que as goroutines do Exemplo 2 terminem...")
	// A goroutine CPU-bound deve ser capaz de progredir mesmo enquanto a blockingWorker está bloqueada,
	// pois o P da blockingWorker será liberado e pego por outro M (ou um novo M será criado).
	wg.Wait()
	fmt.Println("Todas as goroutines do Exemplo 2 terminaram.")
	fmt.Println("--------------------------------------------------")
}
```
Neste exemplo, `blockingWorker` simula uma operação de I/O que bloqueia a thread do sistema operacional (M) na qual está sendo executada. O scheduler de Go é inteligente o suficiente para detectar isso: ele "desacopla" o P que estava associado a esse M bloqueado. Se houver outras goroutines prontas para executar (como `cpuBoundWorker`), outro M (existente ou recém-criado) pode pegar esse P liberado e continuar executando as goroutines, garantindo que o programa não fique parado esperando por uma única operação bloqueante.

## Work Stealing: Balanceamento Dinâmico de Carga

O **work stealing** é um mecanismo crucial no scheduler de Go para garantir o balanceamento de carga e a utilização eficiente dos núcleos da CPU, especialmente em cenários onde a carga de trabalho é distribuída de forma desigual entre os Ps.

**O Problema:**

Imagine um cenário onde alguns Ps estão sobrecarregados com muitas goroutines em suas runqueues locais, enquanto outros Ps estão ociosos, sem goroutines para executar. Sem um mecanismo de balanceamento, os Ps ociosos ficariam parados, e o desempenho geral do programa seria limitado pelos Ps sobrecarregados.

**A Solução: Work Stealing:**

Quando um P (e seu M associado) fica sem goroutines em sua runqueue local e a runqueue global também está vazia, ele não fica simplesmente ocioso. Em vez disso, ele tenta "roubar" goroutines de outros Ps.

**Como Funciona:**

1.  **Verificação Local:** O M associado a um P primeiro verifica a runqueue local do seu P.
2.  **Verificação Global:** Se a runqueue local estiver vazia, ele verifica a runqueue global.
3.  **Work Stealing:** Se ambas estiverem vazias, o M escolhe aleatoriamente outro P e tenta roubar aproximadamente metade das goroutines de sua runqueue local.
    *   **Por que metade?** Roubar metade ajuda a garantir que o P "vítima" ainda tenha trabalho para fazer e que o roubo seja eficiente, movendo um bloco razoável de trabalho.
    *   **Por que do final da fila?** Roubar do final da fila (ou da "cauda") minimiza a contenção com o P "vítima", que geralmente pega goroutines do início da fila (ou da "cabeça").

**Benefícios:**

*   **Balanceamento de Carga:** Distribui dinamicamente as goroutines entre os Ps, garantindo que nenhum núcleo de CPU fique ocioso enquanto há trabalho a ser feito.
*   **Utilização Eficiente da CPU:** Maximiza o uso dos recursos de hardware disponíveis.
*   **Resiliência:** Ajuda a mitigar os efeitos de distribuições de trabalho desiguais ou de goroutines que levam mais tempo do que o esperado.

O work stealing é um algoritmo descentralizado e sem bloqueio (ou com bloqueio mínimo), o que o torna muito eficiente e escalável para um grande número de Ps.

```go
package main

import (
	"fmt"
	"runtime"
	"sync"
	"time"
)

// heavyWorker simula uma goroutine que realiza uma quantidade significativa de trabalho.
func heavyWorker(id int, wg *sync.WaitGroup) {
	defer wg.Done()
	fmt.Printf("Heavy Worker %d: Iniciando trabalho pesado.\n", id)
	sum := 0
	for i := 0; i < 5_000_000_000; i++ { // Loop muito pesado
		sum += i
	}
	fmt.Printf("Heavy Worker %d: Trabalho pesado finalizado. Sum: %d\n", id, sum)
}

// lightWorker simula uma goroutine que realiza uma quantidade menor de trabalho.
