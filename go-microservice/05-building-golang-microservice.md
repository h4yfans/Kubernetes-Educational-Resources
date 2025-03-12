# Building a Golang Microservice

This guide provides a step-by-step approach to building a production-ready microservice in Go.

## Introduction

Building microservices with Go is a common choice due to Go's simplicity, performance, and excellent standard library. This document walks through the process of creating a well-structured, maintainable microservice.

## Project Structure

A well-organized project structure is crucial for maintainability and scalability. Here's a recommended structure for a Go microservice:

```
my-service/
├── cmd/                   # Application entry points
│   └── server/            # Server binary
│       └── main.go        # Main application
├── internal/              # Private code
│   ├── config/            # Configuration
│   ├── domain/            # Domain models
│   ├── handlers/          # HTTP handlers
│   ├── middleware/        # HTTP middleware
│   ├── repository/        # Data access
│   └── service/           # Business logic
├── pkg/                   # Public code that can be imported by others
│   ├── models/            # Shared models
│   └── utils/             # Utilities
├── api/                   # API definitions (OpenAPI/Swagger)
├── deploy/                # Deployment manifests (K8s, Docker, etc.)
├── docs/                  # Documentation
├── migrations/            # Database migrations
├── scripts/               # Build and utility scripts
├── tests/                 # Tests
│   ├── integration/       # Integration tests
│   └── mock/              # Mock implementations
├── .gitignore
├── go.mod
├── go.sum
├── Dockerfile
├── Makefile
└── README.md
```

### Key Directories

- **cmd**: Contains the entry point for the application. Each subdirectory here represents a binary.
- **internal**: Contains private code that should not be imported by other projects.
- **pkg**: Contains code that is safe to use by external applications.
- **api**: Contains API definitions, such as OpenAPI/Swagger docs or Protocol Buffer definitions.

## Setting Up the Project

Let's create a new microservice project:

```bash
# Create project directory
mkdir -p users-service/cmd/server
mkdir -p users-service/internal/{config,domain,handlers,middleware,repository,service}
mkdir -p users-service/pkg/{models,utils}
mkdir -p users-service/api
mkdir -p users-service/deploy
mkdir -p users-service/docs
mkdir -p users-service/migrations
mkdir -p users-service/scripts
mkdir -p users-service/tests/{integration,mock}

# Initialize Go module
cd users-service
go mod init github.com/yourusername/users-service

# Create basic files
touch cmd/server/main.go
touch internal/config/config.go
touch internal/domain/user.go
touch README.md
touch Dockerfile
touch .gitignore
touch Makefile
```

### Setting Up Go Modules

Go modules manage dependencies in your application. Here's how to set up a basic `go.mod` file:

```go
// go.mod
module github.com/yourusername/users-service

go 1.21

require (
	github.com/gin-gonic/gin v1.9.1
	github.com/jackc/pgx/v4 v4.18.1
	github.com/spf13/viper v1.15.0
	go.uber.org/zap v1.24.0
)
```

## Configuration Management

Use a configuration package to manage your application settings:

```go
// internal/config/config.go
package config

import (
	"github.com/spf13/viper"
)

type Config struct {
	Server struct {
		Port int
		Host string
	}
	Database struct {
		Host     string
		Port     int
		User     string
		Password string
		Name     string
		SSLMode  string
	}
	JWT struct {
		Secret    string
		ExpiresIn int
	}
	LogLevel string
}

func Load() (*Config, error) {
	viper.SetConfigName("config")
	viper.SetConfigType("yaml")
	viper.AddConfigPath("./config")
	viper.AddConfigPath(".")
	viper.AutomaticEnv()

	if err := viper.ReadInConfig(); err != nil {
		return nil, err
	}

	var cfg Config
	if err := viper.Unmarshal(&cfg); err != nil {
		return nil, err
	}

	return &cfg, nil
}
```

Create a corresponding `config.yaml` file:

```yaml
# config.yaml
server:
	port: 8080
	host: "0.0.0.0"

database:
	host: "localhost"
	port: 5432
	user: "postgres"
	password: "postgres"
	name: "users"
	sslmode: "disable"

jwt:
	secret: "your-secret-key"
	expiresIn: 3600

logLevel: "info"
```

## Domain Models

Domain models represent your core business entities. Let's create a simple user model:

```go
// internal/domain/user.go
package domain

import (
    "time"
)

type User struct {
    ID        string    `json:"id"`
    Email     string    `json:"email"`
    FirstName string    `json:"first_name"`
    LastName  string    `json:"last_name"`
    Password  string    `json:"-"` // Never expose passwords in JSON
    CreatedAt time.Time `json:"created_at"`
    UpdatedAt time.Time `json:"updated_at"`
}

// For creating a new user
type CreateUserRequest struct {
    Email     string `json:"email" binding:"required,email"`
    FirstName string `json:"first_name" binding:"required"`
    LastName  string `json:"last_name" binding:"required"`
    Password  string `json:"password" binding:"required,min=8"`
}

// For updating a user
type UpdateUserRequest struct {
    FirstName *string `json:"first_name"`
    LastName  *string `json:"last_name"`
    Password  *string `json:"password" binding:"omitempty,min=8"`
}
```

## Repository Layer

The repository layer is responsible for data access. Let's implement a PostgreSQL repository for users:

```go
// internal/repository/user_repository.go
package repository

import (
    "context"
    "database/sql"
    "errors"
    "time"

    "github.com/google/uuid"
    "github.com/yourusername/users-service/internal/domain"
)

var (
    ErrUserNotFound = errors.New("user not found")
    ErrUserExists   = errors.New("user with this email already exists")
)

type UserRepository interface {
    Create(ctx context.Context, user *domain.User) error
    GetByID(ctx context.Context, id string) (*domain.User, error)
    GetByEmail(ctx context.Context, email string) (*domain.User, error)
    Update(ctx context.Context, user *domain.User) error
    Delete(ctx context.Context, id string) error
}

type PostgresUserRepository struct {
    db *sql.DB
}

func NewPostgresUserRepository(db *sql.DB) UserRepository {
    return &PostgresUserRepository{
        db: db,
    }
}

func (r *PostgresUserRepository) Create(ctx context.Context, user *domain.User) error {
    // Check if user with this email already exists
    existingUser, err := r.GetByEmail(ctx, user.Email)
    if err == nil && existingUser != nil {
        return ErrUserExists
    }

    // Generate a new UUID for the user
    user.ID = uuid.New().String()
    user.CreatedAt = time.Now()
    user.UpdatedAt = time.Now()

    query := `
        INSERT INTO users (id, email, first_name, last_name, password, created_at, updated_at)
        VALUES ($1, $2, $3, $4, $5, $6, $7)
    `

    _, err = r.db.ExecContext(
        ctx,
        query,
        user.ID,
        user.Email,
        user.FirstName,
        user.LastName,
        user.Password,
        user.CreatedAt,
        user.UpdatedAt,
    )

    return err
}

func (r *PostgresUserRepository) GetByID(ctx context.Context, id string) (*domain.User, error) {
    query := `
        SELECT id, email, first_name, last_name, password, created_at, updated_at
        FROM users
        WHERE id = $1
    `

    var user domain.User
    err := r.db.QueryRowContext(ctx, query, id).Scan(
        &user.ID,
        &user.Email,
        &user.FirstName,
        &user.LastName,
        &user.Password,
        &user.CreatedAt,
        &user.UpdatedAt,
    )

    if err == sql.ErrNoRows {
        return nil, ErrUserNotFound
    }

    if err != nil {
        return nil, err
    }

    return &user, nil
}

func (r *PostgresUserRepository) GetByEmail(ctx context.Context, email string) (*domain.User, error) {
    query := `
        SELECT id, email, first_name, last_name, password, created_at, updated_at
        FROM users
        WHERE email = $1
    `

    var user domain.User
    err := r.db.QueryRowContext(ctx, query, email).Scan(
        &user.ID,
        &user.Email,
        &user.FirstName,
        &user.LastName,
        &user.Password,
        &user.CreatedAt,
        &user.UpdatedAt,
    )

    if err == sql.ErrNoRows {
        return nil, ErrUserNotFound
    }

    if err != nil {
        return nil, err
    }

    return &user, nil
}

func (r *PostgresUserRepository) Update(ctx context.Context, user *domain.User) error {
    user.UpdatedAt = time.Now()

    query := `
        UPDATE users
        SET first_name = $1, last_name = $2, password = $3, updated_at = $4
        WHERE id = $5
    `

    result, err := r.db.ExecContext(
        ctx,
        query,
        user.FirstName,
        user.LastName,
        user.Password,
        user.UpdatedAt,
        user.ID,
    )

    if err != nil {
        return err
    }

    rowsAffected, err := result.RowsAffected()
    if err != nil {
        return err
    }

    if rowsAffected == 0 {
        return ErrUserNotFound
    }

    return nil
}

func (r *PostgresUserRepository) Delete(ctx context.Context, id string) error {
    query := `DELETE FROM users WHERE id = $1`

    result, err := r.db.ExecContext(ctx, query, id)
    if err != nil {
        return err
    }

    rowsAffected, err := result.RowsAffected()
    if err != nil {
        return err
    }

    if rowsAffected == 0 {
        return ErrUserNotFound
    }

    return nil
}
```

### Database Connection

Let's create a simple database connection setup:

```go
// internal/repository/db.go
package repository

import (
    "database/sql"
    "fmt"

    _ "github.com/lib/pq"
    "github.com/yourusername/users-service/internal/config"
)

func NewPostgresConnection(cfg *config.Config) (*sql.DB, error) {
    connStr := fmt.Sprintf(
        "host=%s port=%d user=%s password=%s dbname=%s sslmode=%s",
        cfg.Database.Host,
        cfg.Database.Port,
        cfg.Database.User,
        cfg.Database.Password,
        cfg.Database.Name,
        cfg.Database.SSLMode,
    )

    db, err := sql.Open("postgres", connStr)
    if err != nil {
        return nil, err
    }

    if err := db.Ping(); err != nil {
        return nil, err
    }

    return db, nil
}
```

## Service Layer

The service layer contains business logic and orchestrates operations. Let's implement a user service:

```go
// internal/service/user_service.go
package service

import (
    "context"
    "errors"

    "golang.org/x/crypto/bcrypt"
    "github.com/yourusername/users-service/internal/domain"
    "github.com/yourusername/users-service/internal/repository"
)

var (
    ErrInvalidCredentials = errors.New("invalid credentials")
)

type UserService interface {
    CreateUser(ctx context.Context, req domain.CreateUserRequest) (*domain.User, error)
    GetUserByID(ctx context.Context, id string) (*domain.User, error)
    UpdateUser(ctx context.Context, id string, req domain.UpdateUserRequest) (*domain.User, error)
    DeleteUser(ctx context.Context, id string) error
    Authenticate(ctx context.Context, email, password string) (*domain.User, error)
}

type userService struct {
    userRepo repository.UserRepository
}

func NewUserService(userRepo repository.UserRepository) UserService {
    return &userService{
        userRepo: userRepo,
    }
}

func (s *userService) CreateUser(ctx context.Context, req domain.CreateUserRequest) (*domain.User, error) {
    // Hash the password
    hashedPassword, err := bcrypt.GenerateFromPassword([]byte(req.Password), bcrypt.DefaultCost)
    if err != nil {
        return nil, err
    }

    user := &domain.User{
        Email:     req.Email,
        FirstName: req.FirstName,
        LastName:  req.LastName,
        Password:  string(hashedPassword),
    }

    if err := s.userRepo.Create(ctx, user); err != nil {
        return nil, err
    }

    return user, nil
}

func (s *userService) GetUserByID(ctx context.Context, id string) (*domain.User, error) {
    return s.userRepo.GetByID(ctx, id)
}

func (s *userService) UpdateUser(ctx context.Context, id string, req domain.UpdateUserRequest) (*domain.User, error) {
    user, err := s.userRepo.GetByID(ctx, id)
    if err != nil {
        return nil, err
    }

    if req.FirstName != nil {
        user.FirstName = *req.FirstName
    }

    if req.LastName != nil {
        user.LastName = *req.LastName
    }

    if req.Password != nil {
        hashedPassword, err := bcrypt.GenerateFromPassword([]byte(*req.Password), bcrypt.DefaultCost)
        if err != nil {
            return nil, err
        }
        user.Password = string(hashedPassword)
    }

    if err := s.userRepo.Update(ctx, user); err != nil {
        return nil, err
    }

    return user, nil
}

func (s *userService) DeleteUser(ctx context.Context, id string) error {
    return s.userRepo.Delete(ctx, id)
}

func (s *userService) Authenticate(ctx context.Context, email, password string) (*domain.User, error) {
    user, err := s.userRepo.GetByEmail(ctx, email)
    if err != nil {
        if errors.Is(err, repository.ErrUserNotFound) {
            return nil, ErrInvalidCredentials
        }
        return nil, err
    }

    if err := bcrypt.CompareHashAndPassword([]byte(user.Password), []byte(password)); err != nil {
        return nil, ErrInvalidCredentials
    }

    return user, nil
}
```

## HTTP Handlers

Let's implement HTTP handlers using the Gin framework:

```go
// internal/handlers/user_handler.go
package handlers

import (
    "net/http"

    "github.com/gin-gonic/gin"
    "github.com/yourusername/users-service/internal/domain"
    "github.com/yourusername/users-service/internal/service"
)

type UserHandler struct {
    userService service.UserService
}

func NewUserHandler(userService service.UserService) *UserHandler {
    return &UserHandler{
        userService: userService,
    }
}

func (h *UserHandler) CreateUser(c *gin.Context) {
    var req domain.CreateUserRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    user, err := h.userService.CreateUser(c.Request.Context(), req)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }

    c.JSON(http.StatusCreated, user)
}

func (h *UserHandler) GetUser(c *gin.Context) {
    id := c.Param("id")

    user, err := h.userService.GetUserByID(c.Request.Context(), id)
    if err != nil {
        c.JSON(http.StatusNotFound, gin.H{"error": "User not found"})
        return
    }

    c.JSON(http.StatusOK, user)
}

func (h *UserHandler) UpdateUser(c *gin.Context) {
    id := c.Param("id")

    var req domain.UpdateUserRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    user, err := h.userService.UpdateUser(c.Request.Context(), id, req)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }

    c.JSON(http.StatusOK, user)
}

func (h *UserHandler) DeleteUser(c *gin.Context) {
    id := c.Param("id")

    err := h.userService.DeleteUser(c.Request.Context(), id)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }

    c.Status(http.StatusNoContent)
}

func (h *UserHandler) Login(c *gin.Context) {
    var req struct {
        Email    string `json:"email" binding:"required,email"`
        Password string `json:"password" binding:"required"`
    }

    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    user, err := h.userService.Authenticate(c.Request.Context(), req.Email, req.Password)
    if err != nil {
        c.JSON(http.StatusUnauthorized, gin.H{"error": "Invalid credentials"})
        return
    }

    // In a real app, generate a JWT token here
    c.JSON(http.StatusOK, gin.H{
        "message": "Login successful",
        "user_id": user.ID,
    })
}
```

## Middleware

Let's implement some common middleware:

```go
// internal/middleware/logger.go
package middleware

import (
    "time"

    "github.com/gin-gonic/gin"
    "go.uber.org/zap"
)

func Logger(logger *zap.Logger) gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()
        path := c.Request.URL.Path
        query := c.Request.URL.RawQuery

        c.Next()

        end := time.Now()
        latency := end.Sub(start)

        logger.Info("Request",
            zap.Int("status", c.Writer.Status()),
            zap.String("method", c.Request.Method),
            zap.String("path", path),
            zap.String("query", query),
            zap.String("ip", c.ClientIP()),
            zap.String("user-agent", c.Request.UserAgent()),
            zap.Duration("latency", latency),
        )
    }
}

// internal/middleware/cors.go
package middleware

import (
    "github.com/gin-gonic/gin"
)

func CORS() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Writer.Header().Set("Access-Control-Allow-Origin", "*")
        c.Writer.Header().Set("Access-Control-Allow-Credentials", "true")
        c.Writer.Header().Set("Access-Control-Allow-Headers", "Content-Type, Content-Length, Accept-Encoding, X-CSRF-Token, Authorization, accept, origin, Cache-Control, X-Requested-With")
        c.Writer.Header().Set("Access-Control-Allow-Methods", "POST, OPTIONS, GET, PUT, DELETE")

        if c.Request.Method == "OPTIONS" {
            c.AbortWithStatus(204)
            return
        }

        c.Next()
    }
}

// internal/middleware/auth.go
package middleware

import (
    "net/http"
    "strings"

    "github.com/gin-gonic/gin"
    // In a real application, you'd use a JWT library
    // "github.com/golang-jwt/jwt/v4"
)

func AuthRequired() gin.HandlerFunc {
    return func(c *gin.Context) {
        authHeader := c.GetHeader("Authorization")
        if authHeader == "" {
            c.JSON(http.StatusUnauthorized, gin.H{"error": "Authorization header is required"})
            c.Abort()
            return
        }

        parts := strings.Split(authHeader, " ")
        if len(parts) != 2 || parts[0] != "Bearer" {
            c.JSON(http.StatusUnauthorized, gin.H{"error": "Authorization header format must be Bearer {token}"})
            c.Abort()
            return
        }

        // In a real application, validate the token here
        token := parts[1]
        if token == "invalid" { // Simplified check for demo
            c.JSON(http.StatusUnauthorized, gin.H{"error": "Invalid token"})
            c.Abort()
            return
        }

        // Store user ID from token in context
        c.Set("user_id", "user-123") // Simplified for demo
        c.Next()
    }
}
```

## Setting Up Routes

Let's set up the router with our handlers and middleware:

```go
// internal/server/router.go
package server

import (
    "github.com/gin-gonic/gin"
    "go.uber.org/zap"

    "github.com/yourusername/users-service/internal/handlers"
    "github.com/yourusername/users-service/internal/middleware"
)

func SetupRouter(
    logger *zap.Logger,
    userHandler *handlers.UserHandler,
) *gin.Engine {
    router := gin.New()

    // Apply global middleware
    router.Use(gin.Recovery())
    router.Use(middleware.Logger(logger))
    router.Use(middleware.CORS())

    // Health check
    router.GET("/health", func(c *gin.Context) {
        c.JSON(200, gin.H{
            "status": "UP",
        })
    })

    // API routes
    api := router.Group("/api/v1")
    {
        // Public routes
        api.POST("/users", userHandler.CreateUser)
        api.POST("/login", userHandler.Login)

        // Protected routes
        protected := api.Group("")
        protected.Use(middleware.AuthRequired())
        {
            protected.GET("/users/:id", userHandler.GetUser)
            protected.PUT("/users/:id", userHandler.UpdateUser)
            protected.DELETE("/users/:id", userHandler.DeleteUser)
        }
    }

    return router
}
```

## Main Application

Let's bring everything together in the main application:

```go
// cmd/server/main.go
package main

import (
    "context"
    "fmt"
    "log"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"

    "go.uber.org/zap"

    "github.com/yourusername/users-service/internal/config"
    "github.com/yourusername/users-service/internal/handlers"
    "github.com/yourusername/users-service/internal/repository"
    "github.com/yourusername/users-service/internal/server"
    "github.com/yourusername/users-service/internal/service"
)

func main() {
    // Load configuration
    cfg, err := config.Load()
    if err != nil {
        log.Fatalf("Failed to load config: %v", err)
    }

    // Initialize logger
    logger, err := zap.NewProduction()
    if err != nil {
        log.Fatalf("Failed to initialize logger: %v", err)
    }
    defer logger.Sync()

    // Set up database connection
    db, err := repository.NewPostgresConnection(cfg)
    if err != nil {
        logger.Fatal("Failed to connect to database", zap.Error(err))
    }
    defer db.Close()

    // Initialize repositories
    userRepo := repository.NewPostgresUserRepository(db)

    // Initialize services
    userService := service.NewUserService(userRepo)

    // Initialize handlers
    userHandler := handlers.NewUserHandler(userService)

    // Set up the router
    router := server.SetupRouter(logger, userHandler)

    // Configure the HTTP server
    srv := &http.Server{
        Addr:    fmt.Sprintf("%s:%d", cfg.Server.Host, cfg.Server.Port),
        Handler: router,
    }

    // Start the server in a goroutine
    go func() {
        logger.Info("Starting server", zap.String("address", srv.Addr))
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            logger.Fatal("Server failed", zap.Error(err))
        }
    }()

    // Wait for interrupt signal to gracefully shut down the server
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit
    logger.Info("Shutting down server...")

    // Create a deadline to wait for
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    if err := srv.Shutdown(ctx); err != nil {
        logger.Fatal("Server forced to shutdown", zap.Error(err))
    }

    logger.Info("Server exiting")
}
```

## Docker Setup

Let's create a Dockerfile for our microservice:

```dockerfile
# Dockerfile
FROM golang:1.21-alpine AS builder

WORKDIR /app

# Copy go mod and sum files
COPY go.mod go.sum ./

# Download dependencies
RUN go mod download

# Copy source code
COPY . .

# Build the application
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o users-service ./cmd/server

# Use a lightweight alpine image for the final image
FROM alpine:3.18

RUN apk --no-cache add ca-certificates

WORKDIR /root/

# Copy the binary from the builder stage
COPY --from=builder /app/users-service .
COPY --from=builder /app/config.yaml .

# Expose the application port
EXPOSE 8080

# Command to run the application
CMD ["./users-service"]
```

## Database Migrations

Let's create a migration script for our users table:

```sql
-- migrations/001_create_users_table.sql
CREATE TABLE IF NOT EXISTS users (
    id UUID PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    first_name VARCHAR(255) NOT NULL,
    last_name VARCHAR(255) NOT NULL,
    password VARCHAR(255) NOT NULL,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL
);

-- For rolling back the migration
-- DROP TABLE IF EXISTS users;
```

## Makefile

Create a Makefile to simplify common tasks:

```makefile
# Makefile
.PHONY: build run test clean docker-build docker-run

# Application name
APP_NAME=users-service

# Go variables
GOBASE=$(shell pwd)
GOBIN=$(GOBASE)/bin

# Build the application
build:
	@echo "Building $(APP_NAME)..."
	@go build -o $(GOBIN)/$(APP_NAME) ./cmd/server

# Run the application
run:
	@echo "Running $(APP_NAME)..."
	@go run ./cmd/server

# Run tests
test:
	@echo "Running tests..."
	@go test -v ./...

# Clean build artifacts
clean:
	@echo "Cleaning..."
	@rm -rf $(GOBIN)

# Build Docker image
docker-build:
	@echo "Building Docker image..."
	@docker build -t $(APP_NAME) .

# Run Docker container
docker-run:
	@echo "Running Docker container..."
	@docker run -p 8080:8080 $(APP_NAME)
```

## Interview Questions and Answers

### 1. How would you structure a Golang microservice for maintainability and scalability?

**Answer:**
I follow a layered architecture that separates concerns:

- **Entry Point Layer** (cmd): Contains the main application entry points.
- **API/Handler Layer**: Handles HTTP requests/responses and input validation.
- **Service Layer**: Implements business logic and orchestrates operations.
- **Repository Layer**: Manages data access and persistence.
- **Domain Layer**: Defines core business entities and interfaces.

This structure provides several benefits:
- Clear separation of concerns
- Easier testing through dependency injection
- Simpler onboarding for new team members
- Better maintainability as the codebase grows

For scalability, I design stateless services that can be horizontally scaled, use connection pooling for databases, implement caching where appropriate, and ensure efficient error handling.

### 2. What are some best practices for error handling in a Go microservice?

**Answer:**
In Go microservices, I follow these error handling best practices:

1. **Use meaningful custom errors**: Define domain-specific error types that provide context.
   ```go
   var ErrUserNotFound = errors.New("user not found")
   ```

2. **Wrap errors with context**: Use `fmt.Errorf` with `%w` to add context while preserving the original error.
   ```go
   if err != nil {
       return fmt.Errorf("failed to fetch user data: %w", err)
   }
   ```

3. **Error types for different status codes**: Map error types to appropriate HTTP status codes.
   ```go
   switch {
   case errors.Is(err, repository.ErrUserNotFound):
       c.JSON(http.StatusNotFound, gin.H{"error": "User not found"})
   case errors.Is(err, repository.ErrUserExists):
       c.JSON(http.StatusConflict, gin.H{"error": "User already exists"})
   default:
       c.JSON(http.StatusInternalServerError, gin.H{"error": "Internal server error"})
   }
   ```

4. **Centralized error handling**: Use middleware for consistent error responses.

5. **Don't expose sensitive information**: Sanitize error messages that are sent to clients.

6. **Log detailed errors internally**: Use structured logging for debugging.

7. **Handle panics**: Use middleware like `gin.Recovery()` to catch panics.

### 3. How would you handle database transactions in a microservice?

**Answer:**
For database transactions in a microservice, I implement the following approach:

1. **Transaction management at the repository layer**:
   ```go
   func (r *UserRepository) CreateUserWithProfile(ctx context.Context, user *User, profile *Profile) error {
       tx, err := r.db.BeginTx(ctx, nil)
       if err != nil {
           return err
       }
       defer tx.Rollback() // Rollback if not committed

       // Insert user
       if err := insertUser(ctx, tx, user); err != nil {
           return err
       }

       // Insert profile
       if err := insertProfile(ctx, tx, profile); err != nil {
           return err
       }

       return tx.Commit()
   }
   ```

2. **Context propagation**: Pass context through all layers to support transaction timeouts and cancellations.

3. **Idempotent operations**: Design operations that can be safely retried.

4. **Saga pattern for distributed transactions**: For operations spanning multiple services, implement compensating transactions.

5. **Connection pooling**: Configure proper connection pools to avoid resource exhaustion.

6. **Monitoring and observability**: Track transaction metrics and errors.

7. **Optimistic locking**: Use version fields to detect concurrent modifications.

By keeping transactions short-lived and focused on a single responsibility, the service remains scalable and resilient.