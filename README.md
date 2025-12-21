# golang-boilerplate

This is a sample Go project that demonstrates a binary search tree implementation.

## Project Structure

The project is organized as follows:

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

- **`cmd/`**: Contains the main application's entry point.
  - **`golang-boilerplate/`**: The main application directory.
    - **`main.go`**: The program's entry point. It creates a binary search tree and prints it in order.

- **`internal/`**: Contains internal packages, not intended for use by other applications.
  - **`tree/`**: A package that implements a binary search tree for strings.
    - **`tree.go`**: The binary search tree implementation.

## How to Run

To run the project, use the following command:

```bash
go run ./cmd/golang-boilerplate
```

## Releasing

To create a new release, you need to create a Git tag and push it to the remote repository. The pipeline will be triggered automatically when a tag in the format `v*` (e.g., `v1.0.0` or `v0.1.2`) is pushed.

1. **Create a Git tag**:
   ```bash
   git tag v1.0.0
   ```

2. **Push the tag to the remote repository**:
   ```bash
   git push origin v1.0.0
   ```