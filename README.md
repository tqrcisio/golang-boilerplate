# golang-boilerplate

Este é um projeto de exemplo em Go que demonstra uma implementação de árvore de busca binária.

## Estrutura do Projeto

O projeto está organizado da seguinte forma:

```
.
├── cmd/
│   └── golang-boilerplate/
│       └── main.go
├── go.mod
├── internal/
│   └── tree/
│       └── tree.go
├── LICENSE
└── README.md
```

- **`cmd/`**: Contém o ponto de entrada da aplicação principal.
  - **`golang-boilerplate/`**: O diretório da aplicação principal.
    - **`main.go`**: O ponto de entrada do programa. Ele cria uma árvore de busca binária e a imprime em ordem.

- **`internal/`**: Contém pacotes internos, não destinados a serem utilizados por outras aplicações.
  - **`tree/`**: Um pacote que implementa uma árvore de busca binária para strings.
    - **`tree.go`**: A implementação da árvore de busca binária.

## Como Executar

Para executar o projeto, utilize o seguinte comando:

```bash
go run ./cmd/golang-boilerplate
```