---
title: "Arquitetura com Context: Cancelamentos, Timeouts e Metadados"
date: 2026-05-20T07:34:24-07:00
draft: false
tags: ["go", "golang", "backend", "senior"]
categories: ["Treinamento Avançado"]
---

# Arquitetura com Context: Cancelamentos, Timeouts e Metadados

A gestão de requisições em sistemas distribuídos, especialmente em arquiteturas de microsserviços, apresenta desafios complexos. Requisições podem falhar, demorar excessivamente ou consumir recursos desnecessariamente se não forem gerenciadas de forma eficaz. O pacote `context` do Go é uma ferramenta fundamental para enfrentar esses desafios, fornecendo um mecanismo robusto para propagar prazos, sinais de cancelamento e valores específicos da requisição através de limites de API e goroutines.

Este artigo aprofunda-se no uso do `context.Context` em cenários de microsserviços, focando estritamente em três pilares: a passagem de contexto entre serviços, a prevenção de *goroutine leaks* e a implementação de cancelamento em cascata.

## Passagem de Contexto em Microsserviços

Em uma arquitetura de microsserviços, uma única requisição do usuário pode atravessar múltiplos serviços. É crucial que informações como prazos, sinais de cancelamento e metadados de rastreamento (como um ID de requisição) sejam propagadas de forma consistente por toda essa cadeia de chamadas. Isso permite que cada serviço na cadeia respeite o prazo original, cancele operações desnecessárias e forneça logs e rastreamentos coerentes.

A propagação de contexto entre serviços geralmente envolve a serialização de partes do contexto em cabeçalhos HTTP (ou outros protocolos de transporte) no serviço chamador e a desserialização desses cabeçalhos de volta para um objeto `context.Context` no serviço chamado. Padrões como W3C Trace Context (usado por OpenTelemetry) são amplamente adotados para essa finalidade, mas para metadados específicos, cabeçalhos customizados podem ser empregados.

Considere um cenário onde um `requestID` é gerado no serviço de entrada e precisa ser propagado para serviços downstream para fins de rastreamento e depuração.

### Exemplo Prático: Propagação de Request ID via HTTP Headers

Vamos simular um cliente chamando um servidor, propagando um `requestID`.

#### Serviço Servidor (`server.go`)

```go
package main

import (
	"context"
	"fmt"
	"log"
	"net/http"
	"time"
)

// requestIDKey é uma chave privada para context.WithValue para evitar colisões.
type requestIDKey string

const RequestIDKey requestIDKey = "requestID"

func handleServiceA(w http.ResponseWriter, r *http.Request) {
	// Cria um novo contexto a partir do contexto da requisição HTTP.
	// O contexto da requisição já pode conter informações como cancelamento do cliente.
	ctx := r.Context()

	// Extrai o Request ID do cabeçalho HTTP.
	requestID := r.Header.Get("X-Request-ID")
	if requestID == "" {
		requestID = "UNKNOWN" // Fallback se o cabeçalho não estiver presente
	}

	// Adiciona o Request ID ao contexto para uso interno no serviço.
	// Isso permite que funções downstream acessem o requestID sem que ele precise
	// ser um argumento explícito em cada chamada de função.
	ctx = context.WithValue(ctx, RequestIDKey, requestID)

	log.Printf("[%s] Recebida requisição em /serviceA", requestID)

	// Chama uma função downstream, passando o contexto enriquecido.
	processRequest(ctx)

	w.WriteHeader(http.StatusOK)
	fmt.Fprintf(w, "Requisição processada com Request ID: %s", requestID)
}

func processRequest(ctx context.Context) {
	requestID, ok := ctx.Value(RequestIDKey).(string)
	if !ok {
		requestID = "UNKNOWN_IN_PROCESS"
	}

	log.Printf("[%s] Processando requisição internamente...", requestID)

	// Simula um trabalho que pode ser cancelado ou ter um timeout
	select {
	case <-time.After(500 * time.Millisecond):
		log.Printf("[%s] Trabalho interno concluído.", requestID)
	case <-ctx.Done():
		log.Printf("[%s] Trabalho interno cancelado: %v", requestID, ctx.Err())
	}
}

func main() {
	http.HandleFunc("/serviceA", handleServiceA)
	port := ":8080"
	log.Printf("Servidor escutando na porta %s", port)
	log.Fatal(http.ListenAndServe(port, nil))
}
```

#### Serviço Cliente (`client.go`)

```go
package main

import (
	"context"
	"fmt"
	"log"
	"net/http"
	"time"

	"github.com/google/uuid" // Para gerar um Request ID único
)

// requestIDKey é uma chave privada para context.WithValue para evitar colisões.
type requestIDKey string

const RequestIDKey requestIDKey = "requestID"

func callServiceA(ctx context.Context, url string) error {
	// Gera um Request ID único para esta requisição.
	requestID := uuid.New().String()
	// Adiciona o Request ID ao contexto para uso interno no cliente.
	// Embora não seja estritamente necessário para este exemplo simples,
	// em um cliente mais complexo, outras funções poderiam precisar do ID.
	ctx = context.WithValue(ctx, RequestIDKey, requestID)

	log.Printf("[%s] Iniciando chamada para %s", requestID, url)

	// Cria uma nova requisição HTTP, associando-a ao contexto.
	// Isso garante que o timeout ou cancelamento do contexto seja respeitado
	// pela requisição HTTP.
	req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
	if err != nil {
		return fmt.Errorf("falha ao criar requisição: %w", err)
	}

	// Adiciona o Request ID ao cabeçalho HTTP para propagação ao serviço downstream.
	req.Header.Set("X-Request-ID", requestID)

	client := &http.Client{}
	resp, err := client.Do(req)
	if err != nil {
		return fmt.Errorf("falha ao fazer requisição: %w", err)
	}
	defer resp.Body.Close()

	log.Printf("[%s] Resposta do serviço: %s", requestID, resp.Status)
	return nil
}

func main() {
	// Contexto com um timeout global para a requisição do cliente.
	// Se a requisição demorar mais de 3 segundos, ela será cancelada.
	ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
	defer cancel() // Garante que o cancelamento seja chamado para liberar recursos.

	serviceURL := "http://localhost:8080/serviceA"
	err := callServiceA(ctx, serviceURL)
	if err != nil {
		log.Printf("Erro na chamada do serviço: %v", err)
	}
}
```

Para executar:
1.  Compile e execute o `server.go` em um terminal: `go run server.go`
2.  Compile e execute o `client.go` em outro terminal: `go run client.go`

Você verá o `requestID` gerado pelo cliente sendo propagado e logado tanto no cliente quanto no servidor, demonstrando a passagem de contexto.

## Goroutine Leaks e o Papel do Contexto

Um *goroutine leak* ocorre quando uma goroutine é iniciada, mas nunca termina, permanecendo na memória e consumindo recursos indefinidamente. Isso pode levar a problemas de desempenho, esgotamento de memória e falhas no aplicativo ao longo do tempo. Uma causa comum de leaks é uma goroutine bloqueada esperando por um evento (como uma leitura de canal) que nunca acontece ou que não é mais relevante.

O `context.Context` é a principal ferramenta em Go para prevenir *goroutine leaks* em operações assíncronas. Ao passar um contexto para uma goroutine, ela pode monitorar o canal `ctx.Done()` e sair graciosamente quando o contexto é cancelado.

### Exemplo Prático: Prevenindo Goroutine Leaks

#### Exemplo com Leak (sem Contexto)

```go
package main

import (
