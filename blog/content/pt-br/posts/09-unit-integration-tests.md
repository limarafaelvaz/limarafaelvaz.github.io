---
title: "Testes em Nível de Produção: Mocks, HTTPTest e Benchmarks"
date: 2026-05-20T08:47:31-08:00
draft: false
tags: ["go", "golang", "backend", "senior"]
categories: ["Treinamento Avançado"]
---

# Testes em Nível de Produção em Go: Mocks, HTTPTest, Benchmarks e Validação de Concorrência

Em sistemas de produção, a robustez e a confiabilidade são primordiais. Em Go, uma linguagem projetada para concorrência e performance, a estratégia de testes precisa ser igualmente sofisticada. Este artigo aprofunda-se em técnicas e ferramentas essenciais para garantir a qualidade de software em nível de produção, cobrindo desde os fundamentos do pacote `testing` até a validação de concorrência e análise de performance.

## Fundamentos do Pacote `testing` em Go

O pacote `testing` é a base de todo o ecossistema de testes em Go. Ele fornece as primitivas necessárias para escrever testes unitários, de integração e benchmarks. A execução é gerenciada pelo comando `go test`.

Um teste em Go é uma função que começa com `Test` e recebe um argumento do tipo `*testing.T`.

```go
// mypackage/calculator.go
package mypackage

// Add retorna a soma de dois inteiros.
func Add(a, b int) int {
	return a + b
}

// Subtract retorna a diferença entre dois inteiros.
func Subtract(a, b int) int {
	return a - b
}
```

```go
// mypackage/calculator_test.go
package mypackage_test

import (
	"testing"

	"meuprojeto/mypackage" // Ajuste o caminho do módulo conforme necessário
)

func TestAdd(t *testing.T) {
	result := mypackage.Add(2, 3)
	expected := 5
	if result != expected {
		t.Errorf("Add(2, 3) = %d; esperado %d", result, expected)
	}
}

func TestSubtract(t *testing.T) {
	tests := []struct {
		name     string
		a, b     int
		expected int
	}{
		{"positive result", 5, 2, 3},
		{"negative result", 2, 5, -3},
		{"zero result", 5, 5, 0},
	}

	for _, tt := range tests {
		// t.Run permite agrupar subtestes, melhorando a organização e o reporte.
		t.Run(tt.name, func(t *testing.T) {
			result := mypackage.Subtract(tt.a, tt.b)
			if result != tt.expected {
				t.Errorf("Subtract(%d, %d) = %d; esperado %d", tt.a, tt.b, result, tt.expected)
			}
		})
	}
}

func TestParallelExample(t *testing.T) {
	// t.Parallel() marca o teste para ser executado em paralelo com outros testes paralelos.
	// Isso é útil para testes de integração ou testes que demoram.
	t.Parallel()
	// Simula um trabalho demorado
	// time.Sleep(100 * time.Millisecond)
	t.Log("Teste paralelo executado.")
}
```

**Execução:** `go test ./mypackage`

As funções `t.Error`, `t.Errorf`, `t.Fatal`, `t.Fatalf` são usadas para reportar falhas. `t.Fatal` e `t.Fatalf` interrompem a execução do teste imediatamente, enquanto `t.Error` e `t.Errorf` permitem que o teste continue. `t.Log` e `t.Logf` são para mensagens informativas.

## Aprimorando Asserções com `testify`

Embora o pacote `testing` seja funcional, as asserções manuais (`if result != expected`) podem se tornar repetitivas e menos legíveis em testes complexos. O pacote `testify` (github.com/stretchr/testify) oferece um conjunto rico de funções de asserção que melhoram a clareza e a concisão dos testes.

`testify` oferece dois subpacotes principais para asserções:
*   `assert`: Reporta falhas mas permite que o teste continue.
*   `require`: Reporta falhas e interrompe o teste imediatamente (similar a `t.Fatal`).

```go
// mypackage/calculator_test.go (com testify)
package mypackage_test

import (
	"testing"

	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/require"

	"meuprojeto/mypackage" // Ajuste o caminho do módulo conforme necessário
)

func TestAddWithTestify(t *testing.T) {
	result := mypackage.Add(2, 3)
	expected := 5
	assert.Equal(t, expected, result, "Add(2, 3) deveria ser 5") // Mais legível
}

func TestSubtractWithTestify(t *testing.T) {
	tests := []struct {
		name     string
		a, b     int
		expected int
	}{
		{"positive result", 5, 2, 3},
		{"negative result", 2, 5, -3},
		{"zero result", 5, 5, 0},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			result := mypackage.Subtract(tt.a, tt.b)
			// require.Equal interrompe o subteste se a asserção falhar.
			require.Equal(t, tt.expected, result, "Subtract(%d, %d) deveria ser %d", tt.a, tt.b, tt.expected)
		})
	}
}

func TestComplexAssertions(t *testing.T) {
	var ptr *int
	assert.Nil(t, ptr, "Ponteiro deveria ser nil")

	err := mypackage.SimulateError() // Suponha uma função que retorna um erro
	assert.Error(t, err, "Deveria ter retornado um erro")
	assert.Contains(t, err.Error(), "simulado", "Mensagem de erro deveria conter 'simulado'")

	slice := []int{1, 2, 3}
	assert.Len(t, slice, 3, "Slice deveria ter 3 elementos")
	assert.Contains(t, slice, 2, "Slice deveria conter o elemento 2")
}

// mypackage/calculator.go (adicionar para o exemplo acima)
func SimulateError() error {
	return assert.AnError // testify.AnError é um erro genérico útil para testes
}
```

**Instalação:** `go get github.com/stretchr/testify`

`testify` oferece uma vasta gama de asserções para diferentes tipos de dados e cenários, tornando os testes mais expressivos e fáceis de manter.

## Testando Componentes HTTP com `net/http/httptest`

Para aplicações web e APIs, testar handlers HTTP é crucial. O pacote `net/http/httptest` fornece utilitários para simular requisições e respostas HTTP sem a necessidade de levantar um servidor real, tornando os testes rápidos e isolados.

```go
// myapp/handler.go
package myapp

import (
	"encoding/json"
	"fmt"
	"net/http"
)

// User representa um usuário.
type User struct {
	ID   string `json:"id"`
	Name string `json:"name"`
}

// GetUserHandler é um handler HTTP que retorna um usuário por ID.
func GetUserHandler(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodGet {
		http.Error(w, "Método não permitido", http.StatusMethodNotAllowed)
		return
	}

	userID := r.URL.Query().Get("id")
	if userID == "" {
		http.Error(w, "ID do usuário é obrigatório", http.StatusBadRequest)
		return
	}

	// Simula a busca de um usuário no banco de dados
	user := User{
		ID:   userID,
		Name: fmt.Sprintf("User %s", userID),
	}

	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusOK)
	json.NewEncoder(w).Encode(user)
}
```

```go
// myapp/handler_test.go
package myapp_test

import (
	"encoding/json"
	"io"
	"net/http"
	"net/http/httptest"
	"testing"

	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/require"

	"meuprojeto/myapp" // Ajuste o caminho do módulo conforme necessário
)

func TestGetUserHandler(t *testing.T) {
	tests := []struct {
		name           string
		method         string
		userID         string
		expectedStatus int
		expectedBody   string // Para erros ou JSON esperado
		expectedUser   *myapp.User
	}{
		{
			name:           "Sucesso ao obter usuário",
			method:         http.MethodGet,
			userID:         "123",
			expectedStatus: http.StatusOK,
			expectedUser:   &myapp.User{ID: "123", Name: "User 123"},
		},
		{
			name:           "ID do usuário ausente",
			method:         http.MethodGet,
			userID:         "",
			expectedStatus: http.StatusBadRequest,
			expectedBody:   "ID do usuário é obrigatório\n",
		},
		{
			name:           "Método não permitido",
			method:         http.MethodPost, // Usando POST em vez de GET
			userID:         "456",
			expectedStatus: http.StatusMethodNotAllowed,
			expectedBody:   "Método não permitido\n",
		},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			// 1. Criar uma requisição HTTP
			req := httptest.NewRequest(tt.method, "/users?id="+tt.userID, nil)

			// 2. Criar um ResponseRecorder para capturar a resposta
			rr := httptest.NewRecorder()

			// 3. Chamar o handler diretamente
			myapp.GetUserHandler(rr, req)

			// 4. Assertions
			assert.Equal(t, tt.expectedStatus, rr.Code, "Status code deveria ser o esperado")

			bodyBytes, err := io.ReadAll(rr.Body)
			require.NoError(t, err)
			bodyString := string(bodyBytes)

			if tt.expectedUser != nil {
				var actualUser myapp.User
				err := json.Unmarshal(bodyBytes, &actualUser)
				require.NoError(t, err, "Erro ao decodificar JSON da resposta")
				assert.Equal(t, tt.expectedUser, &actualUser, "Corpo da resposta deveria corresponder ao usuário esperado")
			} else {
				assert.Equal(t, tt.expectedBody, bodyString, "Corpo da resposta deveria corresponder ao erro esperado")
			}
		})
	}
}

// Exemplo de teste de integração com httptest.NewServer
func TestGetUserHandlerIntegration(t *testing.T) {
	// httptest.NewServer inicia um servidor HTTP real em uma porta aleatória.
	// Útil para testar clientes HTTP ou interações mais complexas.
	ts := httptest.NewServer(http.HandlerFunc(myapp.GetUserHandler))
	defer ts.Close() // Garante que o servidor seja fechado após o teste

	client := ts.Client() // Cliente HTTP configurado para o servidor de teste

	resp, err := client.Get(ts.URL + "/users?id=789")
	require.NoError(t, err)
	defer resp.Body.Close()

	assert.Equal(t, http.StatusOK, resp.StatusCode)

	var user myapp.User
	err = json.NewDecoder(resp.Body).Decode(&user)
	require.NoError(t, err)
	assert.Equal(t, "789", user.ID)
	assert.Equal(t, "User 789", user.Name)
}
```

`httptest.NewRecorder` e `httptest.NewRequest` são ideais para testes unitários de handlers, enquanto `httptest.NewServer` é mais adequado para testes de integração onde um cliente HTTP real precisa interagir com o servidor.

## Mocks e Stubs para Isolamento e Controle

Em sistemas complexos, componentes frequentemente dependem de outros (bancos de dados, serviços externos, etc.). Para testar uma unidade de código isoladamente, controlando suas dependências, usamos mocks e stubs. Em Go, a flexibilidade das interfaces torna a criação de mocks muito natural.

**Mocks Manuais:**

```go
// myapp/service.go
package myapp

import (
	"errors"
	"fmt"
)

// UserRepository define a interface para operações de persistência de usuários.
type UserRepository interface {
	FindByID(id string) (*User, error)
	Save(user *User) error
}

// UserService é um serviço de aplicação que gerencia usuários.
type UserService struct {
	repo UserRepository
}

// NewUserService cria uma nova instância de UserService.
func NewUserService(repo UserRepository) *UserService {
	return &UserService{repo: repo}
}

// GetUser busca um usuário pelo ID.
func (s *UserService) GetUser(id string) (*User, error) {
	if id == "" {
		return nil, errors.New("ID do usuário não pode ser vazio")
	}
	user, err := s.repo.FindByID(id)
	if err != nil {
		return nil, fmt.Errorf("falha ao buscar usuário: %w", err)
	}
	if user == nil {
		return nil, errors.New("usuário não encontrado")
	}
	return user, nil
}

// CreateUser cria um novo usuário.
func (s *UserService) CreateUser(user *User) error {
	if user == nil || user.ID == "" || user.Name == "" {
		return errors.New("dados do usuário inválidos")
	}
	return s.repo.Save(user)
}
```

```go
// myapp/service_test.go
package myapp_test

import (
	"errors"
	"testing"

	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/require"

	"meuprojeto/myapp" // Ajuste o caminho do módulo conforme necessário
)

// MockUserRepository é uma implementação mock da interface UserRepository.
type MockUserRepository struct {
	FindByIDFunc func(id string) (*myapp.User, error)
	SaveFunc     func(user *myapp.User) error
}

// FindByID implementa o método FindByID da interface UserRepository.
func (m *MockUserRepository) FindByID(id string) (*myapp.User, error) {
	if m.FindByIDFunc != nil {
		return m.FindByIDFunc(id)
	}
	return nil, nil // Comportamento padrão se não for mockado
}

// Save implementa o método Save da interface UserRepository.
func (m *MockUserRepository) Save(user *myapp.User) error {
	if m.SaveFunc != nil {
		return m.SaveFunc(user)
	}
	return nil // Comportamento padrão se não for mockado
}

func TestUserService_GetUser(t *testing.T) {
	t.Run("Usuário encontrado com sucesso", func(t *testing.T) {
		expectedUser := &myapp.User{ID: "1", Name: "Alice"}
		mockRepo := &MockUserRepository{
			FindByIDFunc: func(id string) (*myapp.User, error) {
				return expectedUser, nil
			},
		}
		service := myapp.NewUserService(mockRepo)

		user, err := service.GetUser("1")
		require.NoError(t, err)
		assert.Equal(t, expectedUser, user)
	})

	t.Run("Usuário não encontrado", func(t *testing.T) {
		mockRepo := &MockUserRepository{
			FindByIDFunc: func(id string) (*myapp.User, error) {
				return nil, nil // Retorna nil para simular não encontrado
			},
		}
		service := myapp.NewUserService(mockRepo)

		user, err := service.GetUser("2")
		assert.Error(t, err)
		assert.Nil(t, user)
		assert.Contains(t, err.Error(), "usuário não encontrado")
	})

	t.Run("Erro no repositório", func(t *testing.T) {
		mockRepo := &MockUserRepository{
			FindByIDFunc: func(id string) (*myapp.User, error) {
				return nil, errors.New("erro de banco de dados")
			},
		}
		service := myapp.NewUserService(mockRepo)

		user, err := service.GetUser("3")
		assert.Error(t, err)
		assert.Nil(t, user)
		assert.Contains(t, err.Error(), "falha ao buscar usuário: erro de banco de dados")
	})

	t.Run("ID vazio", func(t *testing.T) {
		mockRepo := &MockUserRepository{} // O mock não será chamado
		service := myapp.NewUserService(mockRepo)

		user, err := service.GetUser("")
		assert.Error(t, err)
		assert.Nil(t, user)
		assert.Contains(t, err.Error(), "ID do usuário não pode ser vazio")
	})
}
```

**Mocks com `testify/mock`:**

Para cenários mais complexos, onde você precisa verificar se métodos específicos foram chamados, com quais argumentos e quantas vezes, `github.com/stretchr/testify/mock` pode ser muito útil.

```go
// myapp/service_test.go (com testify/mock)
package myapp_test

import (
	"errors"
	"testing"

	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/mock"
	"github.com/stretchr/testify/require"

	"meuprojeto/myapp" // Ajuste o caminho do módulo conforme necessário
)

// MockUserRepositoryTestify é uma implementação mock da interface UserRepository usando testify/mock.
type MockUserRepositoryTestify struct {
	mock.Mock
}

// FindByID implementa o método FindByID da interface UserRepository.
func (m *MockUserRepositoryTestify) FindByID(id string) (*myapp.User, error) {
	args := m.Called(id)
	var user *myapp.User
	if args.Get(0) != nil {
		user = args.Get(0).(*myapp.User)
	}
	return user, args.Error(1)
}

// Save implementa o método Save da interface UserRepository.
func (m *MockUserRepositoryTestify) Save(user *myapp.User) error {
	args := m.Called(user)
	return args.Error(0)
}

func TestUserService_CreateUserWithTestifyMock(t *testing.T) {
	t.Run("Criação de usuário com sucesso", func(t *testing.T) {
		mockRepo := new(MockUserRepositoryTestify)
		newUser := &myapp.User{ID: "new-id", Name: "Bob"}

		// Configura o mock para esperar uma chamada a Save com o newUser
		// e retornar nil (sem erro).
		mockRepo.On("Save", newUser).Return(nil).Once()

		service := myapp.NewUserService(mockRepo)
		err := service.CreateUser(newUser)

		require.NoError(t, err)
		// Verifica se todas as expectativas do mock foram atendidas.
		mockRepo.AssertExpectations(t)
		// Verifica se o método Save foi chamado exatamente uma vez com newUser.
		mockRepo.AssertCalled(t, "Save", newUser)
	})

	t.Run("Falha na criação de usuário devido a erro no repositório", func(t *testing.T) {
		mockRepo := new(MockUserRepositoryTestify)
		newUser := &myapp.User{ID: "fail-id", Name: "Charlie"}
		repoError := errors.New("erro de persistência")

		mockRepo.On("Save", newUser).Return(repoError).Once()

		service := myapp.NewUserService(mockRepo)
		err := service.CreateUser(newUser)

		assert.Error(t, err)
		assert.Equal(t, repoError, err)
		mockRepo.AssertExpectations(t)
		mockRepo.AssertCalled(t, "Save", newUser)
	})

	t.Run("Dados de usuário inválidos", func(t *testing.T) {
		mockRepo := new(MockUserRepositoryTestify)
		invalidUser := &myapp.User{ID: "", Name: "Invalid"} // ID vazio

		// Não configuramos o mock para Save, pois esperamos que o serviço
		// retorne um erro antes de chamar o repositório.
		service := myapp.NewUserService(mockRepo)
		err := service.CreateUser(invalidUser)

		assert.Error(t, err)
		assert.Contains(t, err.Error(), "dados do usuário inválidos")
		// Assegura que o método Save nunca foi chamado.
		mockRepo.AssertNotCalled(t, "Save")
	})
}
```

**Instalação:** `go get github.com/stretchr/testify/mock`

Mocks são ferramentas poderosas para garantir que a lógica de negócios seja testada de forma isolada e que as interações com dependências sejam as esperadas.

## Validação de Concorrência com o Detector de Corrida (`-race`)

Go foi projetado com concorrência em mente, mas isso não elimina a possibilidade de *race conditions* (condições de corrida), onde o acesso não sincronizado a memória compartilhada pode levar a comportamentos imprevisíveis. O Go runtime inclui um detector de corrida integrado que pode ser ativado com a flag `-race`.

**Como funciona:** O detector de corrida instrumenta o código durante a compilação para monitorar acessos a memória compartilhada. Se dois goroutines acessam a mesma variável sem sincronização adequada, e pelo menos um dos acessos é uma escrita, o detector reporta uma condição de corrida.

**Importância:** Race conditions são notoriamente difíceis de depurar, pois podem ser intermitentes e dependem do agendamento das goroutines. O detector de corrida é uma ferramenta indispensável para identificar esses problemas proativamente.

**Exemplo de Race Condition:**

```go
// mypackage/counter.go
package mypackage

import "sync"

// BadCounter demonstra uma race condition.
type BadCounter struct {
	value int
}

func (c *BadCounter) Increment() {
	c.value++ // Acesso não sincronizado
}

func (c *BadCounter) Value() int {
	return c.value
}

// SafeCounter demonstra como evitar uma race condition com um Mutex.
type SafeCounter struct {
	value int
	mu    sync.Mutex
}

func (c *SafeCounter) Increment() {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.value++
}

func (c *SafeCounter) Value() int {
	c.mu.Lock()
	defer c.mu.Unlock()
	return c.value
}
```

```go
// mypackage/counter_test.go
package mypackage_test

import (
	"sync"
	"testing"

	"meuprojeto/mypackage" // Ajuste o caminho do módulo conforme necessário
)

func TestBadCounterRaceCondition(t *testing.T) {
	counter := mypackage.BadCounter{}
	numGoroutines := 100
	incrementsPerGoroutine := 1000

	var wg sync.WaitGroup
	wg.Add(numGoroutines)

	for i := 0; i < numGoroutines; i++ {
		go func() {
			defer wg.Done()
			for j := 0; j < incrementsPerGoroutine; j++ {
				counter.Increment() // Acesso concorrente sem sincronização
			}
		}()
	}
	wg.Wait()

	// O valor final pode não ser o esperado devido à race condition
	expected := numGoroutines * incrementsPerGoroutine
	if counter.Value() != expected {
		t.Logf("Race condition detectada (ou valor inesperado): esperado %d, obtido %d", expected, counter.Value())
		// t.Error("Race condition provável, valor final inconsistente")
		// Não usamos t.Error aqui porque o detector de corrida fará o trabalho.
	}
}

func TestSafeCounterNoRaceCondition(t *testing.T) {
	counter := mypackage.SafeCounter{}
	numGoroutines := 100
	incrementsPerGoroutine := 1000

	var wg sync.WaitGroup
	wg.Add(numGoroutines)

	for i := 0; i < numGoroutines; i++ {
		go func() {
			defer wg.Done()
			for j := 0; j < incrementsPerGoroutine; j++ {
				counter.Increment() // Acesso sincronizado com Mutex
			}
		}()
	}
	wg.Wait()

	expected := numGoroutines * incrementsPerGoroutine
	if counter.Value() != expected {
		t.Errorf("Erro: SafeCounter deveria ser %d, mas obteve %d", expected, counter.Value())
	}
}
```

**Execução com Detector de Corrida:**

Para o `TestBadCounterRaceCondition`:
`go test -race ./mypackage`

Você verá uma saída similar a:
```
==================
WARNING: DATA RACE
Write at 0x00c00010a008 by goroutine 8:
  meuprojeto/mypackage.(*BadCounter).Increment()
      /path/to/meuprojeto/mypackage/counter.go:10 +0x38
  meuprojeto/mypackage_test.TestBadCounterRaceCondition.func1()
      /path/to/meuprojeto/mypackage/counter_test.go:21 +0x4e
... (muitas linhas de stack trace)
==================
--- FAIL: TestBadCounterRaceCondition (0.00s)
    counter_test.go:30: Race condition detectada (ou valor inesperado): esperado 100000, obtido 99999
FAIL
exit status 1
FAIL    meuprojeto/mypackage    0.008s
```

Para o `TestSafeCounterNoRaceCondition`:
`go test -race ./mypackage`

A saída não mostrará nenhum aviso de race condition, e o teste passará.

Sempre execute seus testes com `go test -race ./...` em seu pipeline de CI/CD para garantir que novas race conditions não sejam introduzidas.

## Análise de Performance com Benchmarks

A performance é um aspecto crítico em sistemas de produção. Go oferece um framework de benchmarking integrado no pacote `testing` que permite medir o desempenho do código de forma precisa.

**Estrutura de um Benchmark:**
Uma função de benchmark começa com `Benchmark` e recebe um argumento `*testing.B`. Dentro da função, o loop `b.N` é executado para medir o tempo médio de uma operação.

```go
// mypackage/stringops.go
package mypackage

import "strings"

// ConcatStringsBuilder concatena strings usando strings.Builder (eficiente).
func ConcatStringsBuilder(parts []string) string {
	var sb strings.Builder
	for _, part := range parts {
		sb.WriteString(part)
	}
	return sb.String()
}

// ConcatStringsPlus concatena strings usando o operador '+' (menos eficiente para muitos itens).
func ConcatStringsPlus(parts []string) string {
	var result string
	for _, part := range parts {
		result += part
	}
	return result
}

// SumSlice calcula a soma dos elementos de um slice.
func SumSlice(s []int) int {
	sum := 0
	for _, v := range s {
		sum += v
	}
	return sum
}
```

```go
// mypackage/stringops_test.go
package mypackage_test

import (
	"strconv"
	"testing"

	"meuprojeto/mypackage" // Ajuste o caminho do módulo conforme necessário
)

// Prepara um slice de strings para benchmarks de concatenação.
func generateStrings(n int) []string {
	parts := make([]string, n)
	for i := 0; i < n; i++ {
		parts[i] = strconv.Itoa(i)
	}
	return parts