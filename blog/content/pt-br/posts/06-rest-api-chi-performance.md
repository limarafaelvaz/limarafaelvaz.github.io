---
title: "APIs REST de Alta Performance: Roteamento Otimizado com Chi"
date: 2026-05-20T07:38:24-07:00
draft: false
tags: ["go", "golang", "backend", "senior"]
categories: ["Treinamento Avançado"]
---

# APIs REST de Alta Performance: Roteamento Otimizado com Chi

No desenvolvimento de APIs REST de alta performance em Go, a escolha de um roteador eficiente e flexível é crucial. O pacote `chi` (github.com/go-chi/chi) se destaca como uma excelente opção, oferecendo uma API concisa, performance robusta e uma arquitetura modular que facilita a construção de serviços web escaláveis e de fácil manutenção. Este artigo técnico aprofunda-se em como utilizar `chi` para otimizar o roteamento, focando em design limpo de rotas, implementação de middlewares customizados e tratamento idiomático de pânicos, elementos essenciais para qualquer API de nível de produção.

## Design Limpo de Rotas com Chi

Um design de rotas limpo e intuitivo é fundamental para a manutenibilidade e escalabilidade de uma API. `chi` oferece recursos poderosos que promovem essa clareza, permitindo organizar as rotas de forma lógica e hierárquica.

### A Estrutura do Roteador Chi

O coração do `chi` é o `chi.Mux`, que implementa a interface `chi.Router`. Ele atua como um multiplexador HTTP, direcionando as requisições para os handlers corretos com base no método HTTP e no caminho da URL.

```go
package main

import (
	"fmt"
	"net/http"
	"strconv"
	"encoding/json"
	"log"

	"github.com/go-chi/chi/v5"
	"github.com/go-chi/chi/v5/middleware"
)

// User representa um modelo de usuário
type User struct {
	ID   int    `json:"id"`
	Name string `json:"name"`
}

// Product representa um modelo de produto
type Product struct {
	ID    int    `json:"id"`
	Name  string `json:"name"`
	Price float64 `json:"price"`
}

// Handler para listar usuários
func listUsers(w http.ResponseWriter, r *http.Request) {
	users := []User{
		{ID: 1, Name: "Alice"},
		{ID: 2, Name: "Bob"},
	}
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusOK)
	json.NewEncoder(w).Encode(map[string][]User{"users": users})
}

// Handler para obter um usuário por ID
func getUser(w http.ResponseWriter, r *http.Request) {
	userIDStr := chi.URLParam(r, "userID")
	userID, err := strconv.Atoi(userIDStr)
	if err != nil {
		http.Error(w, "ID de usuário inválido", http.StatusBadRequest)
		return
	}
	user := User{ID: userID, Name: fmt.Sprintf("User %d", userID)}
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusOK)
	json.NewEncoder(w).Encode(user)
}

// Handler para criar um usuário
func createUser(w http.ResponseWriter, r *http.Request) {
	var newUser User
	if err := json.NewDecoder(r.Body).Decode(&newUser); err != nil {
		http.Error(w, "Corpo da requisição inválido", http.StatusBadRequest)
		return
	}
	// Simula a atribuição de um ID
	newUser.ID = 3
	log.Printf("Usuário criado: %+v", newUser)
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusCreated)
	json.NewEncoder(w).Encode(map[string]string{"message": "Usuário criado com sucesso", "id": strconv.Itoa(newUser.ID)})
}

// Handler para listar produtos
func listProducts(w http.ResponseWriter, r *http.Request) {
	products := []Product{
		{ID: 101, Name: "Laptop", Price: 1200.00},
		{ID: 102, Name: "Mouse", Price: 25.00},
	}
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusOK)
	json.NewEncoder(w).Encode(map[string][]Product{"products": products})
}

// Handler para obter um produto por ID
func getProduct(w http.ResponseWriter, r *http.Request) {
	productIDStr := chi.URLParam(r, "productID")
	productID, err := strconv.Atoi(productIDStr)
	if err != nil {
		http.Error(w, "ID de produto inválido", http.StatusBadRequest)
		return
	}
	product := Product{ID: productID, Name: fmt.Sprintf("Product %d", productID), Price: float64(productID) * 10.0}
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusOK)
	json.NewEncoder(w).Encode(product)
}

func main() {
	r := chi.NewRouter()

	// Middlewares globais (serão explicados em detalhes na próxima seção)
	r.Use(middleware.RequestID)
	r.Use(middleware.Logger)
	r.Use(middleware.Recoverer)
	r.Use(middleware.URLFormat)

	// Agrupamento de rotas com `r.Route`
	// Isso cria um sub-roteador que herda os middlewares do roteador pai,
	// mas permite adicionar middlewares específicos ou prefixos de caminho.
	r.Route("/api/v1", func(r chi.Router) {
		// Rotas para a API de usuários
		r.Route("/users", func(r chi.Router) {
			r.Get("/", listUsers)        // GET /api/v1/users
			r.Post("/", createUser)      // POST /api/v1/users
			r.Get("/{userID}", getUser)  // GET /api/v1/users/{userID}
			// Exemplo de rota com wildcard para sub-recursos
			r.Get("/{userID}/orders/*", func(w http.ResponseWriter, r *http.Request) {
				userID := chi.URLParam(r, "userID")
				orderPath := chi.URLParam(r, "*") // Captura o restante do caminho
				w.Write([]byte(fmt.Sprintf("Listando pedidos para o usuário %s, caminho: %s", userID, orderPath)))
			})
		})

		// Rotas para a API de produtos
		r.Route("/products", func(r chi.Router) {
			r.Get("/", listProducts)          // GET /api/v1/products
			r.Get("/{productID}", getProduct) // GET /api/v1/products/{productID}
		})
	})

	// Rota para servir arquivos estáticos
	// chi.FileServer é um exemplo de como usar wildcards para servir arquivos
	// r.Handle("/static/*", http.StripPrefix("/static/", http.FileServer(http.Dir(".")))))

	fmt.Println("Servidor iniciado na porta :3000")
	log.Fatal(http.ListenAndServe(":3000", r))
}
```

### Principais Recursos para Design de Rotas:

1.  **`r.Route(pathPrefix, fn(r chi.Router))`**: Este é o recurso mais poderoso para organizar rotas. Ele permite criar sub-roteadores, onde todas as rotas definidas dentro da função `fn` terão o `pathPrefix` como base. Isso é excelente para versionamento de API (`/api/v1`, `/api/v2`) ou para agrupar recursos relacionados (`/users`, `/products`).
2.  **Parâmetros de URL**: `chi` suporta parâmetros de URL de forma simples (`/{paramName}`). Para acessá-los dentro de um handler, utiliza-se `chi.URLParam(r, "paramName