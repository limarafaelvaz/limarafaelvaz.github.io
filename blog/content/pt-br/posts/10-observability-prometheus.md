---
title: "Observabilidade Industrial: Métricas com Prometheus e Tracing"
date: 2026-05-20T08:48:26-08:00
draft: false
tags: ["go", "golang", "backend", "senior"]
categories: ["Treinamento Avançado"]
---

# Observabilidade Industrial: Métricas com Prometheus e Tracing

A complexidade crescente dos ambientes industriais modernos, impulsionada pela Indústria 4.0, IoT industrial (IIoT) e a convergência de TI/OT, exige abordagens sofisticadas para monitoramento e diagnóstico. A observabilidade, que vai além do monitoramento tradicional, permite que engenheiros e operadores compreendam o "porquê" de um sistema se comportar de uma certa maneira, não apenas o "o quê". Em contextos industriais, isso se traduz em insights sobre a saúde de máquinas, eficiência de processos, latência de redes de controle e a causa raiz de falhas em sistemas distribuídos.

Este artigo técnico explora a aplicação de métricas com Prometheus e rastreamento distribuído (tracing) para alcançar uma observabilidade profunda em ambientes industriais, focando na coleta de telemetria, contadores, histogramas personalizados e rastreamento distribuído, com exemplos práticos em Go.

## Coleta de Telemetria em Ambientes Industriais

A coleta de telemetria é o alicerce da observabilidade. Em ambientes industriais, isso apresenta desafios únicos devido à diversidade de protocolos (Modbus, OPC UA, EtherNet/IP, Profinet), sistemas legados, restrições de rede e a criticidade das operações. A telemetria pode incluir:

*   **Dados de Sensores:** Temperatura, pressão, vibração, nível, corrente, tensão.
*   **Dados de Atuadores:** Posição, estado (ligado/desligado), velocidade.
*   **Estados de Máquinas/PLCs:** Modo de operação, alarmes, contagens de produção, tempos de ciclo.
*   **Dados de Rede:** Latência, perda de pacotes em redes de controle.
*   **Logs de Eventos:** Mensagens de erro, avisos, eventos de segurança.

Para integrar esses dados em um sistema de observabilidade moderno, é comum empregar gateways ou agentes que traduzem protocolos industriais para formatos amigáveis à nuvem ou a sistemas de monitoramento, como o modelo de dados de séries temporais do Prometheus ou o formato OpenTelemetry para traces.

Ferramentas como o [Prometheus Node Exporter](https://github.com/prometheus/node_exporter) podem coletar métricas de sistemas operacionais em servidores industriais. Para dados específicos de PLCs ou dispositivos, agentes personalizados ou exportadores específicos (ex: [Modbus Exporter](https://github.com/RichiH/modbus_exporter), [OPC UA Exporter](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/opcuareceiver) via OpenTelemetry Collector) são frequentemente necessários. A flexibilidade de linguagens como Go permite a criação de agentes leves e eficientes para coletar e expor essas métricas.

## Métricas com Prometheus

Prometheus é um sistema de monitoramento e alerta de código aberto que coleta métricas de seus alvos configurados em intervalos definidos, avalia regras de expressão, exibe os resultados e pode acionar alertas. Seu modelo de dados de séries temporais é ideal para dados industriais, onde o valor de um sensor ou o estado de uma máquina muda ao longo do tempo.

### Fundamentos do Prometheus para Observabilidade Industrial

O Prometheus opera com um modelo de *pull*, onde ele "raspa" (scrape) um endpoint HTTP `/metrics` exposto pelos serviços ou agentes. As métricas são armazenadas como séries temporais, identificadas por um nome de métrica e um conjunto de pares chave-valor (rótulos).

Os tipos de métricas principais do Prometheus são:

*   **Counter:** Um valor numérico que só pode aumentar ou ser resetado para zero. Ideal para contagens cumulativas.
*   **Gauge:** Um valor numérico que pode subir e descer livremente. Ideal para valores instantâneos como temperatura, pressão, uso de CPU.
*   **Histogram:** Amostra observações (geralmente durações de requisições ou tamanhos de resposta) e as conta em *buckets* configuráveis. Fornece somas e contagens de todas as observações.
*   **Summary:** Semelhante ao histograma, mas calcula quantis no lado do cliente.

Neste artigo, focaremos em **Counters** e **Histograms personalizados**, que são particularmente úteis para cenários industriais.

### Contadores (Counters)

Contadores são a forma mais simples de métrica e representam um valor cumulativo que só pode aumentar. São perfeitos para registrar eventos que ocorrem ao longo do tempo.

**Cenários de Uso Industrial:**

*   **Contagem de Produção:** Número total de peças produzidas, ciclos de máquina concluídos.
*   **Contagem de Erros:** Número de falhas de sensores, erros de comunicação, alarmes acionados.
*   **Contagem de Operações:** Número de vezes que uma válvula foi aberta, um motor foi ligado.

**Exemplo Prático em Go:**

O código Go a seguir demonstra como criar e expor contadores usando a biblioteca `prometheus/client_golang`. Ele simula um processo industrial que conclui ciclos de produção e ocasionalmente reporta erros de sensores.

```go
package main

import (
	"fmt"
	"log"
	"math/rand"
	"net/http"
	"time"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
	// productionCyclesTotal é um contador para o número total de ciclos de produção concluídos.
	// Útil para monitorar a vazão da linha de produção.
	productionCyclesTotal = prometheus.NewCounter(
		prometheus.CounterOpts{
			Name: "industrial_production_cycles_total",
			Help: "Número total de ciclos de produção concluídos com sucesso.",
		},
	)

	// sensorErrorsTotal é um contador vetorial para o número de erros de sensor,
	// categorizados por ID do sensor e tipo de erro. Permite granularidade na análise de falhas.
	sensorErrorsTotal = prometheus.NewCounterVec(
		prometheus.CounterOpts{
			Name: "industrial_sensor_errors_total",
			Help: "Número total de erros reportados por sensores específicos.",
		},
		[]string{"sensor_id", "error_type"}, // Rótulos para identificar o sensor e o tipo de erro
	)
)

func init() {
	// Registra os contadores no registro padrão do Prometheus.
	// Isso os torna disponíveis para serem raspados pelo Prometheus.
	prometheus.MustRegister(productionCyclesTotal)
	prometheus.MustRegister(sensorErrorsTotal)
	rand.Seed(time.Now().UnixNano()) // Inicializa o gerador de números aleatórios
}

// simulateProductionCycle simula um único ciclo de produção e incrementa os contadores relevantes.
func simulateProductionCycle() {
	// Simula o tempo que um ciclo de produção levaria.
	time.Sleep(time.Duration(500+rand.Intn(1000)) * time.Millisecond) // Trabalho simulado entre 0.5s e 1.5s
	productionCyclesTotal.Inc()                                       // Incrementa o contador de ciclos concluídos.

	// Simula um erro de sensor ocasional (10% de chance).
	if rand.Intn(10) == 0 {
		sensorID := fmt.Sprintf("sensor_%d", rand.Intn(3)+1) // Sensores 1, 2 ou 3
		errorType := "timeout"
		if rand.Intn(2) == 0 {
			errorType = "calibration_failure" // Alterna entre tipos de erro
		}
		// Incrementa o contador de erros para o sensor e tipo de erro específicos.
		sensorErrorsTotal.WithLabelValues(sensorID, errorType).Inc()
		log.Printf("Erro simulado no %s: %s", sensorID, errorType)
	}
	log.Println("Ciclo de produção concluído.")
}

func main() {
	// Expõe as métricas Prometheus via HTTP no caminho /metrics.
	http.Handle("/metrics", promhttp.Handler())

	// Inicia uma goroutine para simular ciclos de produção continuamente.
	go func() {
		for {
			simulateProductionCycle()
			time.Sleep(1 * time.Second) // Inicia um novo ciclo a cada segundo
		}
	}()

	log.Println("Servidor de métricas Prometheus iniciado em :8080/metrics")
	// Inicia o servidor HTTP.
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

Para executar este exemplo:
1.  Salve o código como `main.go`.
2.  Execute `go mod init industrial-metrics` (se ainda não tiver um módulo).
3.  Execute `go get github.com/prometheus/client_golang/prometheus github.com/prometheus/client_golang/prometheus/promhttp`.
4.  Execute `go run main.go`.
5.  Acesse `http://localhost:8080/metrics` em seu navegador para ver as métricas.

Você verá métricas como:
```
# HELP industrial_production_cycles_total Número total de ciclos de produção concluídos com sucesso.
# TYPE industrial_production_cycles_total counter
industrial_production_cycles_total 123
# HELP industrial_sensor_errors_total Número total de erros reportados por sensores específicos.
# TYPE industrial_sensor_errors_total counter
industrial_sensor_errors_total{error_type="calibration_failure",sensor_id="sensor_1"} 2
industrial_sensor_errors_total{error_type="timeout",sensor_id="sensor_3"} 1
```

### Histogramas Personalizados (Custom Histograms)

Histogramas são métricas complexas que amostram observações (como durações de requisições ou tamanhos de dados) e as contam em *buckets* configuráveis. Eles fornecem uma visão da distribuição dos valores observados, além da soma total e da contagem de observações. Isso é crucial para entender a latência ou a variabilidade de um processo.

**A Importância dos Buckets:**
A definição dos *buckets* é fundamental. Eles representam intervalos de valores. Por exemplo, se você está medindo o tempo de ciclo de uma máquina, buckets como `[0.1, 0.2, 0.5, 1.0, 2.0]` segundos permitem ver quantas operações foram concluídas em menos de 0.1s, entre 0.1s e 0.2s, etc. Uma escolha inadequada de buckets pode ocultar informações importantes ou gerar dados irrelevantes.

**Cenários de Uso Industrial:**

*   **Tempos de Ciclo de Máquinas:** Distribuição do tempo que leva para uma máquina completar uma tarefa. Ajuda a identificar gargalos ou variabilidade indesejada.
*   **Latência de Comunicação:** Distribuição do tempo de resposta de um PLC, sensor ou atuador. Essencial para sistemas de controle em tempo real.
*   **Distribuição de Valores de Sensores:** Em vez de apenas a média, um histograma pode mostrar se a temperatura de um forno está consistentemente dentro de uma faixa ideal ou se há picos/vales frequentes.
*   **Duração de Etapas de Processo:** Medir a duração de fases específicas em um processo de fabricação.

**Exemplo Prático em Go:**

Este exemplo em Go cria e expõe histogramas para medir a duração de uma etapa crítica do processo industrial e a distribuição da temperatura de sensores.

```go
package main

import (
	"fmt"
	"log"
	"math/rand"
	"net/http"
	"time"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
	// criticalProcessDuration é um histograma para medir o tempo de processamento de uma etapa crítica.
	// Os buckets são definidos em segundos para capturar a distribuição da latência.
	criticalProcessDuration = prometheus.NewHistogram(
		prometheus.HistogramOpts{
			Name:    "industrial_critical_