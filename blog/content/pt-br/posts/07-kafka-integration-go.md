---
title: "Mensageria Escalável: Integração de Microsserviços Go com Apache Kafka"
date: 2026-05-20T08:45:46-08:00
draft: false
tags: ["go", "golang", "backend", "senior"]
categories: ["Treinamento Avançado"]
---

# Mensageria Escalável: Integração de Microsserviços Go com Apache Kafka

Em arquiteturas de microsserviços, a comunicação assíncrona é um pilar fundamental para alcançar escalabilidade, resiliência e desacoplamento. Apache Kafka emergiu como a plataforma de streaming de eventos de facto para esses cenários, oferecendo alta vazão, baixa latência e durabilidade. Quando combinado com a eficiência e concorrência nativa do Go, é possível construir sistemas de mensageria robustos e escaláveis. Este artigo técnico aprofundará na integração de microsserviços Go com Kafka, focando estritamente na produção e consumo assíncrono de eventos, e nas estratégias essenciais de Retry e Dead Letter Queue (DLQ) para garantir a confiabilidade do sistema.

## 1. Fundamentos da Mensageria Assíncrona com Kafka e Go

A mensageria assíncrona permite que componentes de um sistema se comuniquem sem a necessidade de estarem disponíveis simultaneamente. Um produtor envia uma mensagem e continua suas operações, sem esperar por uma resposta imediata do consumidor. Isso reduz o acoplamento temporal e espacial, melhorando a resiliência e a capacidade de resposta do sistema.

Apache Kafka atua como um *event bus* distribuído, onde eventos são publicados em *tópicos* e consumidos por um ou mais *grupos de consumidores*. Cada tópico é dividido em *partições*, que são as unidades de paralelismo e ordenação. Go, com suas goroutines e canais, é excepcionalmente adequado para lidar com a natureza concorrente da produção e consumo de eventos.

Utilizaremos a biblioteca `github.com/confluentinc/confluent-kafka-go/kafka`, que é um *wrapper* Go para a biblioteca `librdkafka` da Confluent, conhecida por sua performance e conjunto de recursos abrangente.

## 2. Produção de Eventos Assíncrona em Go

A produção assíncrona envolve enviar mensagens para o Kafka sem bloquear a thread de execução do produtor, aguardando a confirmação de entrega. Em vez disso, a confirmação (ou falha) é recebida posteriormente através de um canal de eventos.

### Configuração e Envio

Um produtor Kafka em Go é configurado com parâmetros como o endereço dos *brokers* e o nível de confirmação (`acks`). Para produção assíncrona, o método `ProduceChannel` é preferível, pois permite que o produtor envie mensagens para um canal interno e receba relatórios de entrega de forma não bloqueante.

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/confluentinc/confluent-kafka-go/kafka"
)

// Event representa a estrutura de um evento a ser publicado.
type Event struct {
	ID        string `json:"id"`
	Payload   string `json:"payload"`
	Timestamp int64  `json:"timestamp"`
}

// ProducerConfig contém as configurações para o produtor Kafka.
type ProducerConfig struct {
	Broker string
	Topic  string
}

// NewKafkaProducer cria e retorna um novo produtor Kafka.
func NewKafkaProducer(config ProducerConfig) (*kafka.Producer, error) {
	p, err := kafka.NewProducer(&kafka.ConfigMap{
		"bootstrap.servers": config.Broker,
		"acks":              "all", // Garante que todos os ISRs confirmem a escrita
		"retries":           5,     // Retentativas internas do produtor
		"message.timeout.ms": 5000, // Tempo limite para entrega da mensagem
	})
	if err != nil {
		return nil, fmt.Errorf("falha ao criar produtor: %w", err)
	}
	return p, nil
}

// ProduceEvents envia eventos para o Kafka de forma assíncrona.
func ProduceEvents(ctx context.Context, p *kafka.Producer, topic string) {
	deliveryChan := make(chan kafka.Event)

	go func() {
		for e := range p.Events() {
			switch ev := e.(type) {
			case *kafka.Message:
				if ev.TopicPartition.Error != nil {
					log.Printf("Falha na entrega da mensagem para o tópico %s [%d] em offset %v: %v\n",
						*ev.TopicPartition.Topic, ev.TopicPartition.Partition, ev.TopicPartition.Offset, ev.TopicPartition.Error)
				} else {
					log.Printf("Mensagem entregue com sucesso para o tópico %s [%d] em offset %v\n",
						*ev.TopicPartition.Topic, ev.TopicPartition.Partition, ev.TopicPartition.Offset)
				}
			case kafka.Error:
				log.Printf("Erro do Kafka: %v\n", ev)
			default:
				log.Printf("Evento ignorado: %v\n", ev)
			}
		}
	}()

	ticker := time.NewTicker(1 * time.Second)
	defer ticker.Stop()

	eventID := 0
	for {
		select {
		case <-ctx.Done():
			log.Println("Contexto cancelado, encerrando produtor de eventos.")
			return
		case <-ticker.C:
			eventID++
			event := Event{
				ID:        fmt.Sprintf("event-%d", eventID),
				Payload:   fmt.Sprintf("Dados do evento %d", eventID),
				Timestamp: time.Now().UnixMilli(),
			}
			eventBytes, err := json.Marshal(event)
			if err != nil {
				log.Printf("Erro ao serializar evento: %v\n", err)
				continue
			}

			err = p.Produce(&kafka.Message{
				TopicPartition: kafka.TopicPartition{Topic: &topic, Partition: kafka.PartitionAny},
				Value:          eventBytes,
				Headers:        []kafka.Header{{Key: "source", Value: []byte("go-producer")}},
			}, deliveryChan)

			if err != nil {
				log.Printf("Falha ao enfileirar mensagem para produção: %v\n", err)
				if err.(kafka.Error).Code() == kafka.ErrQueueFull {
					// A fila interna do produtor está cheia, tentar novamente ou lidar com backpressure
					log.Println("Fila do produtor cheia, aguardando...")
					time.Sleep(100 * time.Millisecond)
				}
			}
		}
	}
}

func main() {
	broker := os.Getenv("KAFKA_BROKER")
	if broker == "" {
		broker = "localhost:9092"
	}
	topic := os.Getenv("KAFKA_TOPIC")
	if topic == "" {
		topic = "my-events"
	}

	producerConfig := ProducerConfig{
		Broker: broker,
		Topic:  topic,
	}

	p, err := NewKafkaProducer(producerConfig)
	if err != nil {
		log.Fatalf("Erro ao inicializar produtor: %v", err)
	}
	defer p.Close()

	log.Printf("Produtor Kafka iniciado, enviando eventos para o tópico '%s' no broker '%s'\n", topic, broker)

	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	// Captura sinais de interrupção para um desligamento gracioso
	sigchan := make(chan os.Signal, 1)
	signal.Notify(sigchan, syscall.SIGINT, syscall.SIGTERM)

	go ProduceEvents(ctx, p, topic)

	<-sigchan // Aguarda o sinal de interrupção
	log.Println("Sinal de desligamento recebido, encerrando...")
	cancel() // Cancela o contexto para sinalizar o encerramento das goroutines
	p.Flush(5000) // Aguarda até 5 segundos para todas as mensagens serem entregues
	log.Println("Produtor encerrado.")
}

```

**Explicação:**
- `p.Events()`: Retorna um canal onde os relatórios de entrega (sucesso ou falha) são enviados. Uma goroutine separada deve consumir este canal para processar esses relatórios.
- `kafka.Message`: Contém o tópico, partição, valor (payload) e cabeçalhos. `PartitionAny` permite que o Kafka escolha a partição.
- `deliveryChan`: Um canal opcional para receber relatórios de entrega específicos para uma mensagem, embora `p.Events()` seja mais comum para um fluxo geral.
- `acks: "all"`: Garante a maior durabilidade, esperando que todos os *In-Sync Replicas* (ISRs) confirmem a escrita.
- `retries`: O produtor `librdkafka` possui um mecanismo de retry interno para falhas transitórias de rede ou broker.

## 3. Consumo de Eventos Assíncrona em Go

Consumir eventos assincronamente significa que o consumidor está continuamente lendo mensagens de um tópico, processando-as e, em seguida, confirmando seu *offset*. O `confluent-kafka-go` facilita isso com um *loop* de `Poll` e processamento de eventos.

### Configuração e Consumo

Um consumidor Kafka é configurado com um `group.id` (para participar de um grupo de consumidores), `bootstrap.servers` e `auto.offset.reset` (para definir o comportamento inicial do offset).

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/confluentinc/confluent-kafka-go/kafka"
)

// Event representa a estrutura de um evento a ser consumido.
type Event struct {
	ID        string `json:"id"`
	Payload   string `json:"payload"`
	Timestamp int64  `json:"timestamp"`
}

// ConsumerConfig contém as configurações para o consumidor Kafka.
type ConsumerConfig struct {
	Broker  string
	GroupID string
	Topic   string
}

// NewKafkaConsumer cria e retorna um novo consumidor Kafka.
func NewKafkaConsumer(config ConsumerConfig) (*kafka.Consumer, error) {
	c, err := kafka.NewConsumer(&kafka.ConfigMap{
		"bootstrap.servers": config.Broker,
		"group.id":          config.GroupID,
		"auto.offset.reset": "earliest", // Começa a ler do início se não houver offset salvo
		"enable.auto.commit": false,     // Desabilita o commit automático para controle manual
	})
	if err != nil {
		return nil, fmt.Errorf("falha ao criar consumidor: %w", err)
	}

	err = c.SubscribeTopics([]string{config.Topic}, nil)
	if err != nil {
		c.Close()
		return nil, fmt.Errorf("falha ao subscrever tópico %s: %w", config.Topic, err)
	}
	return c, nil
}

// ConsumeEvents consome eventos do Kafka e os processa.
func ConsumeEvents(ctx context.Context, c *kafka.Consumer) {
	for {
		select {
		case <-ctx.Done():
			log.Println("Contexto cancelado, encerrando consumidor de eventos.")
			return
		default:
			msg, err := c.Poll(100 * time.Millisecond) // Poll por 100ms
			if err != nil {
				if err.(kafka.Error).IsFatal() {
					log.Printf("Erro fatal do Kafka: %v\n", err)
					return // Encerrar em caso de erro fatal
				}
				if err.(kafka.Error).Code() == kafka.ErrTimedOut {
					// log.Println("Nenhuma mensagem em 100ms, continuando...")
					continue
				}
				log.Printf("Erro ao pollar mensagem: %v\n", err)
				continue
			}

			if msg == nil {
				continue // Nenhuma mensagem disponível
			}

			var event Event
			if err := json.Unmarshal(msg.Value, &event); err != nil {
				log.Printf("Erro ao deserializar mensagem: %v. Mensagem: %s\n", err, string(msg.Value))
				// Em um cenário real, esta mensagem pode ir para uma DLQ imediatamente.
				// Para este exemplo, apenas logamos e continuamos.
				continue
			}

			log.Printf("Mensagem recebida: Tópico %s [%d] Offset %v | Chave: %s | Valor: %+v\n",
				*msg.TopicPartition.Topic, msg.TopicPartition.Partition, msg.TopicPartition.Offset, string(msg.Key), event)

			// Simula o processamento da mensagem
			if err := processEvent(event); err != nil {
				log.Printf("Erro ao processar evento %s: %v\n", event.ID, err)
				// Aqui é onde a lógica de retry/DLQ seria acionada.
				// Por enquanto, apenas logamos e não comitamos o offset para reprocessamento.
			} else {
				// Confirma o offset manualmente após o processamento bem-sucedido
				_, err := c.CommitMessage(msg)
				if err != nil {
					log.Printf("Erro ao comitar offset para mensagem %s: %v\n", event.ID, err)
				}
			}
		}
	}
}

// processEvent simula o processamento de um evento.
// Pode retornar um erro para simular falhas.
func processEvent(event Event) error {
	// Simula uma falha de processamento para eventos com ID par
	if event.ID == "event-2" || event.ID == "event-4" { // Exemplo de falha específica
		return fmt.Errorf("falha simulada ao processar evento %s", event.ID)
	}
	// Simula um processamento demorado
	time.Sleep(50 * time.Millisecond)
	log.Printf("Evento %s processado com sucesso.\n", event.ID)
	return nil
}

func main() {
	broker := os.Getenv("KAFKA_BROKER")
	if broker == "" {
		broker = "localhost:9092"
	}
	topic := os.Getenv("KAFKA_TOPIC")
	if topic == "" {
		topic = "my-events"
	}
	groupID := os.Getenv("KAFKA_GROUP_ID")
	if groupID == "" {
		groupID = "my-go-consumer-group"
	}

	consumerConfig := ConsumerConfig{
		Broker:  broker,
		GroupID: groupID,
		Topic:   topic,
	}

	c, err := NewKafkaConsumer(consumerConfig)
	if err != nil {
		log.Fatalf("Erro ao inicializar consumidor: %v", err)
	}
	defer c.Close()

	log.Printf("Consumidor Kafka iniciado, lendo do tópico '%s' no broker '%s' como grupo '%s'\n", topic, broker, groupID)

	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	sigchan := make(chan os.Signal, 1)
	signal.Notify(sigchan, syscall.SIGINT, syscall.SIGTERM)

	go ConsumeEvents(ctx, c)

	<-sigchan
	log.Println("Sinal de desligamento recebido, encerrando...")
	cancel()
	log.Println("Consumidor encerrado.")
}

```

**Explicação:**
- `group.id`: Essencial para o balanceamento de carga e tolerância a falhas. Mensagens de uma partição são entregues a apenas um consumidor dentro do mesmo `group.id`.
- `enable.auto.commit: false`: Permite o controle manual do commit de offsets. Isso é crucial para garantir que uma mensagem só seja marcada como processada após o sucesso real, evitando perda ou reprocessamento desnecessário.
- `c.Poll(timeout)`: Bloqueia por um tempo especificado para buscar mensagens. Retorna `nil` se nenhuma mensagem estiver disponível no timeout.
- `c.CommitMessage(msg)`: Confirma o offset da mensagem processada. Se o processamento falhar, o offset não é comitado, e a mensagem será reprocessada na próxima vez que o consumidor (ou outro consumidor do mesmo grupo) buscar mensagens daquela partição.

## 4. Estratégias de Retry para Consumidores

Mesmo com o commit manual de offsets, falhas persistentes no processamento de uma mensagem podem bloquear o progresso do consumidor. Estratégias de retry são necessárias para lidar com erros transitórios e dar tempo para que os sistemas dependentes se recuperem.

### Retry Baseado em Tópicos Kafka

Uma estratégia robusta de retry envolve o uso de tópicos dedicados para retentativas. Quando uma mensagem falha no processamento, ela é republicada em um tópico de retry com um atraso. Isso libera o consumidor principal para processar outras mensagens, enquanto a mensagem falha aguarda sua vez de ser reprocessada.

**Fluxo:**
1.  **Consumidor Principal:** Lê do `original-topic`. Se o processamento falhar, publica a mensagem (com metadados de retry) para `original-topic.retry`.
2.  **Tópico de Retry (`original-topic.retry`):** Contém mensagens que falharam e precisam ser reprocessadas.
3.  **Consumidor de Retry:** Lê de `original-topic.retry`. Aplica um atraso (e.g., `time.Sleep`) e, se o número máximo de retries não foi atingido, republica a mensagem de volta para o `original-topic`. Se o máximo de retries for atingido, move a mensagem para a DLQ.

Para rastrear o número de retries, podemos incluir um campo `RetryCount` no payload da mensagem ou usar cabeçalhos Kafka.

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"os"
	"os/signal"
	"strconv"
	"syscall"
	"time"

	"github.com/confluentinc/confluent-kafka-go/kafka"
)

const (
	MaxRetries      = 3
	RetryTopicSuffix = ".retry"
	DLQTopicSuffix   = ".dlq"
)

// EventWithMetadata estende Event para incluir metadados de retry.
type EventWithMetadata struct {
	Event
	RetryCount int    `json:"retryCount"`
	Error      string `json:"error,omitempty"`
}

// ProducerConfig e NewKafkaProducer são os mesmos do exemplo de Produtor.
// ... (código do produtor NewKafkaProducer aqui) ...

// ConsumerConfig e NewKafkaConsumer são os mesmos do exemplo de Consumidor.
// ... (código do consumidor NewKafkaConsumer aqui) ...

// processEventWithRetry simula o processamento de um evento com lógica de retry.
func processEventWithRetry(event EventWithMetadata) error {
	// Simula uma falha de processamento para eventos com ID par
	if event.ID == "event-2" || event.ID == "event-4" { // Exemplo de falha específica
		return fmt.Errorf("falha simulada ao processar evento %s", event.ID)
	}
	// Simula um processamento demorado
	time.Sleep(50 * time.Millisecond)
	log.Printf("Evento %s processado com sucesso (tentativa %d).\n", event.ID, event.RetryCount)
	return nil
}

// ConsumeAndProcessEvents consome eventos do tópico principal e aplica lógica de retry/DLQ.
func ConsumeAndProcessEvents(ctx context.Context, c *kafka.Consumer, p *kafka.Producer, originalTopic string) {
	for {
		select {
		case <-ctx.Done():
			log.Println("Contexto cancelado, encerrando consumidor principal.")
			return
		default:
			msg, err := c.Poll(100 * time.Millisecond)
			if err != nil {
				if err.(kafka.Error).IsFatal() {
					log.Printf("Erro fatal do Kafka: %v\n", err)
					return
				}
				if err.(kafka.Error).Code() == kafka.ErrTimedOut {
					continue
				}
				log.Printf("Erro ao pollar mensagem: %v\n", err)
				continue
			}

			if msg == nil {
				continue
			}

			var eventWithMeta EventWithMetadata
			if err := json.Unmarshal(msg.Value, &eventWithMeta); err != nil {
				log.Printf("Erro ao deserializar mensagem para %s [%d] Offset %v: %v. Mensagem: %s\n",
					*msg.TopicPartition.Topic, msg.TopicPartition.Partition, msg.TopicPartition.Offset, err, string(msg.Value))
				// Se a deserialização falhar, a mensagem é irrecuperável, envia para DLQ.
				sendToDLQ(p, originalTopic, msg.Value, "Falha na deserialização", nil)
				_, commitErr := c.CommitMessage(msg) // Comita para não reprocessar
				if commitErr != nil {
					log.Printf("Erro ao comitar offset após falha de deserialização: %v\n", commitErr)
				}
				continue
			}

			log.Printf("Consumidor Principal: Recebida mensagem para %s [%d] Offset %v | ID: %s | Tentativa: %d\n",
				*msg.TopicPartition.Topic, msg.TopicPartition.Partition, msg.TopicPartition.Offset, eventWithMeta.ID, eventWithMeta.RetryCount)

			if err := processEventWithRetry(eventWithMeta); err != nil {
				log.Printf("Consumidor Principal: Falha ao processar evento %s (tentativa %d): %v\n", eventWithMeta.ID, eventWithMeta.RetryCount, err)

				if eventWithMeta.RetryCount >= MaxRetries {
					log.Printf("Consumidor Principal: Max retries atingido para evento %s. Enviando para DLQ.\n", eventWithMeta.ID)
					sendToDLQ(p, originalTopic, msg.Value, err.Error(), msg.Headers)
				} else {
					log.Printf("Consumidor Principal: Enviando evento %s para tópico de retry.\n", eventWithMeta.ID)
					eventWithMeta.RetryCount++
					eventWithMeta.Error = err.Error()
					sendToRetryTopic(p, originalTopic, eventWithMeta, msg.Headers)
				}
				// Não comita o offset para a mensagem falha, ela será reprocessada pelo retry handler ou movida para DLQ.
			} else {
				// Processamento bem-sucedido, comita o offset.
				_, commitErr := c.CommitMessage(msg)
				if commitErr != nil {
					log.Printf("Erro ao comitar offset para mensagem %s: %v\n", eventWithMeta.ID, commitErr)
				}
			}
		}
	}
}

// sendToRetryTopic publica a mensagem para o tópico de retry.
func sendToRetryTopic(p *kafka.Producer, originalTopic string, event EventWithMetadata, headers []kafka.Header) {
	retryTopic := originalTopic + RetryTopicSuffix
	eventBytes, err := json.Marshal(event)
	if err != nil {
		log.Printf("Erro ao serializar evento para retry: %v\n", err)
		return
	}

	// Adiciona ou atualiza o cabeçalho de retry count
	updatedHeaders := make([]kafka.Header, len(headers))
	copy(updatedHeaders, headers)
	found := false
	for i, h := range updatedHeaders {
		if h.Key == "retry-count" {
			updatedHeaders[i].Value = []byte(strconv.Itoa(event.RetryCount))
			found = true
			break
		}
	}
	if !found {
		updatedHeaders = append(updatedHeaders, kafka.Header{Key: "retry-count", Value: []byte(strconv.Itoa(event.RetryCount))})
	}


	err = p.Produce(&kafka.Message{
		TopicPartition: kafka.TopicPartition{Topic: &retryTopic, Partition: kafka.PartitionAny},
		Value:          eventBytes,
		Headers:        updatedHeaders,
	}, nil) // Não precisamos de deliveryChan aqui para simplificar

	if err != nil {
		log.Printf("Falha ao enviar mensagem para tópico de retry %s: %v\n", retryTopic, err)
	} else {
		log.Printf("Mensagem %s enviada para tópico de retry %s (tentativa %d).\n", event.ID, retryTopic, event.RetryCount)
	}
}

// sendToDL