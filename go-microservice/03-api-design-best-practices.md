# API Design Best Practices for Microservices

## Introduction

Well-designed APIs are crucial for microservice architectures. They form the contracts between services and define how services communicate with each other and with external clients. This document covers best practices for designing APIs in a microservice ecosystem, focusing on both REST and gRPC approaches.

## API Design Principles

### 1. API-First Development

Design your APIs before implementing them. This approach ensures that your APIs are well-thought-out and meet the needs of consumers.

**Best Practice:** Use tools like OpenAPI (Swagger) or Protocol Buffers to define your APIs declaratively.

```yaml
# OpenAPI example for a user service
openapi: 3.0.0
info:
  title: User Service API
  version: 1.0.0
  description: API for managing users
paths:
  /users:
    get:
      summary: List users
      responses:
        '200':
          description: A list of users
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/User'
    post:
      summary: Create a user
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateUserRequest'
      responses:
        '201':
          description: User created
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
components:
  schemas:
    User:
      type: object
      properties:
        id:
          type: string
        name:
          type: string
        email:
          type: string
    CreateUserRequest:
      type: object
      required:
        - name
        - email
      properties:
        name:
          type: string
        email:
          type: string
```

### 2. Consistency

Maintain consistency in your API design across all services. This includes naming conventions, error handling, versioning, and response formats.

**Interview Tip:** Be prepared to discuss how you would ensure API consistency across multiple teams working on different microservices.

### 3. Backward Compatibility

Design APIs to be backward compatible whenever possible. This allows clients to continue functioning even when the API evolves.

**Best Practice:** Add new fields instead of changing existing ones, and make new fields optional.

### 4. Security by Design

Incorporate security considerations from the beginning of the API design process.

**Key Security Aspects:**
- Authentication and authorization
- Input validation
- Rate limiting
- Transport security (TLS)
- Sensitive data handling

## RESTful API Design

### 1. Resource-Oriented Design

Design your APIs around resources (nouns) rather than actions (verbs).

```
# Good
GET /users
GET /users/123
POST /users
PUT /users/123
DELETE /users/123

# Avoid
GET /getUsers
POST /createUser
PUT /updateUser/123
DELETE /deleteUser/123
```

### 2. HTTP Methods

Use HTTP methods appropriately:

- **GET**: Retrieve a resource (read-only)
- **POST**: Create a new resource
- **PUT**: Update a resource (full update)
- **PATCH**: Partially update a resource
- **DELETE**: Remove a resource

**Interview Question:** "When would you use PUT versus PATCH in a RESTful API?"

### 3. Status Codes

Use appropriate HTTP status codes to indicate the result of operations:

- **2xx**: Success (200 OK, 201 Created, 204 No Content)
- **3xx**: Redirection (301 Moved Permanently, 304 Not Modified)
- **4xx**: Client errors (400 Bad Request, 401 Unauthorized, 404 Not Found)
- **5xx**: Server errors (500 Internal Server Error, 503 Service Unavailable)

```go
// Example of proper status code usage in Go
func handleGetUser(w http.ResponseWriter, r *http.Request) {
    id := chi.URLParam(r, "id")

    user, err := userService.GetUser(r.Context(), id)
    if err != nil {
        if errors.Is(err, ErrUserNotFound) {
            http.Error(w, "User not found", http.StatusNotFound)
            return
        }
        http.Error(w, "Internal server error", http.StatusInternalServerError)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusOK)
    json.NewEncoder(w).Encode(user)
}

func handleCreateUser(w http.ResponseWriter, r *http.Request) {
    var req CreateUserRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "Invalid request body", http.StatusBadRequest)
        return
    }

    user, err := userService.CreateUser(r.Context(), req)
    if err != nil {
        if errors.Is(err, ErrInvalidInput) {
            http.Error(w, err.Error(), http.StatusBadRequest)
            return
        }
        if errors.Is(err, ErrEmailAlreadyExists) {
            http.Error(w, "Email already in use", http.StatusConflict)
            return
        }
        http.Error(w, "Internal server error", http.StatusInternalServerError)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(user)
}
```

### 4. Query Parameters

Use query parameters for filtering, sorting, and pagination:

```
GET /users?role=admin&status=active
GET /products?category=electronics&sort=price&order=asc
GET /orders?page=2&limit=10
```

**Implementation Example:**

```go
func handleListUsers(w http.ResponseWriter, r *http.Request) {
    // Extract query parameters
    role := r.URL.Query().Get("role")
    status := r.URL.Query().Get("status")

    // Parse pagination parameters
    page, _ := strconv.Atoi(r.URL.Query().Get("page"))
    if page <= 0 {
        page = 1
    }

    limit, _ := strconv.Atoi(r.URL.Query().Get("limit"))
    if limit <= 0 || limit > 100 {
        limit = 20 // Default limit
    }

    // Build filter
    filter := UserFilter{
        Role:   role,
        Status: status,
        Page:   page,
        Limit:  limit,
    }

    // Get users with filter
    users, total, err := userService.ListUsers(r.Context(), filter)
    if err != nil {
        http.Error(w, "Internal server error", http.StatusInternalServerError)
        return
    }

    // Return response with pagination metadata
    response := ListUsersResponse{
        Users: users,
        Pagination: Pagination{
            Total:  total,
            Page:   page,
            Limit:  limit,
            Pages:  (total + limit - 1) / limit,
        },
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(response)
}
```

### 5. Versioning

Implement API versioning to allow evolution without breaking existing clients. Common approaches include:

1. **URL Path Versioning**:
   ```
   /api/v1/users
   /api/v2/users
   ```

2. **Header Versioning**:
   ```
   Accept: application/vnd.mycompany.api+json;version=1
   ```

3. **Query Parameter Versioning**:
   ```
   /api/users?version=1
   ```

**Best Practice:** URL path versioning is the most explicit and easiest to understand, though header versioning is more RESTfully pure.

```go
// URL path versioning in Go
func setupRoutes(r *chi.Mux) {
    // v1 API
    r.Route("/api/v1", func(r chi.Router) {
        r.Get("/users", v1.ListUsers)
        r.Post("/users", v1.CreateUser)
        r.Get("/users/{id}", v1.GetUser)
    })

    // v2 API
    r.Route("/api/v2", func(r chi.Router) {
        r.Get("/users", v2.ListUsers)
        r.Post("/users", v2.CreateUser)
        r.Get("/users/{id}", v2.GetUser)
    })
}
```

### 6. HATEOAS (Hypermedia as the Engine of Application State)

Include links in responses to guide clients on available actions:

```json
{
  "id": "123",
  "name": "John Doe",
  "email": "john@example.com",
  "_links": {
    "self": {
      "href": "/users/123"
    },
    "orders": {
      "href": "/users/123/orders"
    },
    "update": {
      "href": "/users/123",
      "method": "PUT"
    },
    "delete": {
      "href": "/users/123",
      "method": "DELETE"
    }
  }
}
```

**Note:** While HATEOAS is a principle of REST, it's often omitted in microservice architectures due to the added complexity and the fact that service-to-service communication is often more RPC-like.

### 7. Error Handling

Provide consistent, informative error responses:

```json
{
  "error": {
    "code": "INVALID_INPUT",
    "message": "Invalid input data",
    "details": [
      {
        "field": "email",
        "message": "Must be a valid email address"
      },
      {
        "field": "password",
        "message": "Must be at least 8 characters long"
      }
    ]
  }
}
```

**Implementation Example:**

```go
type ErrorResponse struct {
    Error struct {
        Code    string       `json:"code"`
        Message string       `json:"message"`
        Details []FieldError `json:"details,omitempty"`
    } `json:"error"`
}

type FieldError struct {
    Field   string `json:"field"`
    Message string `json:"message"`
}

func writeErrorResponse(w http.ResponseWriter, statusCode int, code, message string, details []FieldError) {
    resp := ErrorResponse{}
    resp.Error.Code = code
    resp.Error.Message = message
    resp.Error.Details = details

    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(statusCode)
    json.NewEncoder(w).Encode(resp)
}

// Usage
func handleCreateUser(w http.ResponseWriter, r *http.Request) {
    var req CreateUserRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        writeErrorResponse(w, http.StatusBadRequest, "INVALID_JSON", "Invalid JSON payload", nil)
        return
    }

    // Validate input
    var fieldErrors []FieldError
    if req.Name == "" {
        fieldErrors = append(fieldErrors, FieldError{Field: "name", Message: "Name is required"})
    }
    if req.Email == "" {
        fieldErrors = append(fieldErrors, FieldError{Field: "email", Message: "Email is required"})
    } else if !isValidEmail(req.Email) {
        fieldErrors = append(fieldErrors, FieldError{Field: "email", Message: "Invalid email format"})
    }

    if len(fieldErrors) > 0 {
        writeErrorResponse(w, http.StatusBadRequest, "VALIDATION_ERROR", "Validation failed", fieldErrors)
        return
    }

    // Process the request...
}
```

## gRPC API Design

### 1. Service Definition

Define your services and messages clearly in Protocol Buffers:

```protobuf
syntax = "proto3";
package user;

service UserService {
    rpc GetUser(GetUserRequest) returns (User) {}
    rpc CreateUser(CreateUserRequest) returns (User) {}
    rpc UpdateUser(UpdateUserRequest) returns (User) {}
    rpc DeleteUser(DeleteUserRequest) returns (DeleteUserResponse) {}
    rpc ListUsers(ListUsersRequest) returns (ListUsersResponse) {}
}

message GetUserRequest {
    string id = 1;
}

message CreateUserRequest {
    string name = 1;
    string email = 2;
}

message UpdateUserRequest {
    string id = 1;
    string name = 2;
    string email = 3;
}

message DeleteUserRequest {
    string id = 1;
}

message DeleteUserResponse {
    bool success = 1;
}

message ListUsersRequest {
    string role = 1;
    string status = 2;
    int32 page = 3;
    int32 limit = 4;
}

message ListUsersResponse {
    repeated User users = 1;
    Pagination pagination = 2;
}

message User {
    string id = 1;
    string name = 2;
    string email = 3;
    string role = 4;
    string status = 5;
    google.protobuf.Timestamp created_at = 6;
    google.protobuf.Timestamp updated_at = 7;
}

message Pagination {
    int32 total = 1;
    int32 page = 2;
    int32 limit = 3;
    int32 pages = 4;
}
```

### 2. Error Handling

Use gRPC status codes and error details:

```go
import (
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
    "google.golang.org/genproto/googleapis/rpc/errdetails"
)

func (s *userServer) CreateUser(ctx context.Context, req *pb.CreateUserRequest) (*pb.User, error) {
    // Validate input
    if req.Name == "" || req.Email == "" {
        st := status.New(codes.InvalidArgument, "Invalid input")

        var violations []*errdetails.BadRequest_FieldViolation
        if req.Name == "" {
            violations = append(violations, &errdetails.BadRequest_FieldViolation{
                Field: "name",
                Description: "Name is required",
            })
        }
        if req.Email == "" {
            violations = append(violations, &errdetails.BadRequest_FieldViolation{
                Field: "email",
                Description: "Email is required",
            })
        }

        br := &errdetails.BadRequest{FieldViolations: violations}
        st, err := st.WithDetails(br)
        if err != nil {
            return nil, status.Error(codes.Internal, "Failed to build error details")
        }

        return nil, st.Err()
    }

    // Process the request...
}
```

### 3. Streaming

Leverage gRPC streaming for real-time updates or large data transfers:

```protobuf
service OrderService {
    // Unary RPC
    rpc GetOrder(GetOrderRequest) returns (Order) {}

    // Server streaming RPC
    rpc WatchOrders(WatchOrdersRequest) returns (stream Order) {}

    // Client streaming RPC
    rpc CreateBulkOrders(stream CreateOrderRequest) returns (BulkOrderResponse) {}

    // Bidirectional streaming RPC
    rpc ProcessOrders(stream OrderOperation) returns (stream OrderResult) {}
}
```

**Implementation Example:**

```go
// Server streaming example
func (s *orderServer) WatchOrders(req *pb.WatchOrdersRequest, stream pb.OrderService_WatchOrdersServer) error {
    // Set up a subscription to order updates
    orderCh := s.orderService.SubscribeToOrders(req.UserId)
    defer s.orderService.UnsubscribeFromOrders(req.UserId)

    for {
        select {
        case order := <-orderCh:
            // Convert domain order to protobuf order
            pbOrder := convertToPbOrder(order)

            // Send the order to the client
            if err := stream.Send(pbOrder); err != nil {
                return status.Errorf(codes.Internal, "Failed to send order: %v", err)
            }

        case <-stream.Context().Done():
            // Client disconnected or context canceled
            return nil
        }
    }
}
```

### 4. Versioning

Version your gRPC APIs using package names:

```protobuf
// v1 API
syntax = "proto3";
package myservice.v1;

// v2 API
syntax = "proto3";
package myservice.v2;
```

**Best Practice:** Keep backward compatibility within a major version. When breaking changes are necessary, create a new major version.

### 5. Metadata

Use gRPC metadata for cross-cutting concerns like authentication and tracing:

```go
// Client-side: sending metadata
func callGetUser(ctx context.Context, client pb.UserServiceClient, userID string) (*pb.User, error) {
    // Create metadata with auth token
    md := metadata.New(map[string]string{
        "authorization": "Bearer " + token,
        "request-id":    uuid.New().String(),
    })

    // Create context with metadata
    ctx = metadata.NewOutgoingContext(ctx, md)

    // Make the call
    return client.GetUser(ctx, &pb.GetUserRequest{Id: userID})
}

// Server-side: receiving metadata
func (s *userServer) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.User, error) {
    // Extract metadata
    md, ok := metadata.FromIncomingContext(ctx)
    if !ok {
        return nil, status.Error(codes.Unauthenticated, "No metadata found")
    }

    // Get auth token
    authHeader := md.Get("authorization")
    if len(authHeader) == 0 {
        return nil, status.Error(codes.Unauthenticated, "No authorization token provided")
    }

    // Validate token
    // ...

    // Process the request
    // ...
}
```

## Choosing Between REST and gRPC

### When to Use REST

- Public APIs consumed by various clients
- Browser-based applications (direct API calls)
- When HTTP caching is important
- When you need maximum compatibility
- When human readability of the protocol is important

### When to Use gRPC

- Internal service-to-service communication
- Performance-critical systems
- Polyglot environments (multiple programming languages)
- When you need streaming capabilities
- When strong typing and contract enforcement are important

**Interview Question:** "How would you decide between REST and gRPC for microservice communication?"

## API Gateway Pattern

An API Gateway serves as a single entry point for all clients and handles cross-cutting concerns:

```
┌─────────────┐     ┌─────────────┐
│   Mobile    │     │    Web      │
│   Client    │     │   Client    │
└──────┬──────┘     └──────┬──────┘
        │                  │
        ▼                  ▼
┌─────────────────────────────────┐
│                                 │
│          API Gateway            │
│                                 │
└───┬─────────────┬───────────┬───┘
    │             │           │
    ▼             ▼           ▼
┌─────────┐   ┌─────────┐ ┌─────────┐
│  User    │   │ Product │ │  Order  │
│ Service  │   │ Service │ │ Service │
└─────────┘   └─────────┘ └─────────┘
```

**Key Responsibilities:**
- Request routing
- API composition
- Protocol translation (e.g., REST to gRPC)
- Authentication and authorization
- Rate limiting
- Caching
- Monitoring and logging
- SSL termination

**Implementation Options:**
- Kong
- Traefik
- AWS API Gateway
- Azure API Management
- Custom implementation with frameworks like Go's Chi or Gin

## Documentation

### OpenAPI (Swagger)

For REST APIs, use OpenAPI to document your endpoints:

```go
// Using go-swagger annotations
// @title User Service API
// @version 1.0
// @description API for managing users
// @host api.example.com
// @BasePath /api/v1

// @Summary Get a user by ID
// @Description Get a user by their unique identifier
// @Tags users
// @Accept json
// @Produce json
// @Param id path string true "User ID"
// @Success 200 {object} User
// @Failure 404 {object} ErrorResponse
// @Failure 500 {object} ErrorResponse
// @Router /users/{id} [get]
func (h *UserHandler) GetUser(w http.ResponseWriter, r *http.Request) {
    // Implementation...
}
```

### Protocol Buffers

For gRPC, the `.proto` files serve as documentation, but you can enhance them with comments:

```protobuf
// UserService provides operations for managing users
service UserService {
    // GetUser retrieves a user by their ID
    // Returns NOT_FOUND if the user doesn't exist
    rpc GetUser(GetUserRequest) returns (User) {}

    // CreateUser creates a new user
    // Returns INVALID_ARGUMENT if the input is invalid
    // Returns ALREADY_EXISTS if the email is already in use
    rpc CreateUser(CreateUserRequest) returns (User) {}
}
```

## Testing APIs

### 1. Unit Testing

Test individual handlers or service implementations:

```go
func TestUserHandler_GetUser(t *testing.T) {
    // Create a mock service
    mockService := &MockUserService{
        users: map[string]*User{
            "123": {ID: "123", Name: "Test User", Email: "test@example.com"},
        },
    }

    // Create the handler with the mock service
    handler := NewUserHandler(mockService)

    // Create a test request
    req := httptest.NewRequest("GET", "/users/123", nil)
    rec := httptest.NewRecorder()

    // Create a router and register the handler
    router := chi.NewRouter()
    router.Get("/users/{id}", handler.GetUser)

    // Serve the request
    router.ServeHTTP(rec, req)

    // Check the response
    assert.Equal(t, http.StatusOK, rec.Code)

    var user User
    err := json.NewDecoder(rec.Body).Decode(&user)
    assert.NoError(t, err)
    assert.Equal(t, "123", user.ID)
    assert.Equal(t, "Test User", user.Name)
}
```

### 2. Integration Testing

Test the API with real dependencies:

```go
func TestUserAPI_Integration(t *testing.T) {
    // Skip if not running integration tests
    if testing.Short() {
        t.Skip("Skipping integration test")
    }

    // Set up test server with real dependencies
    server := setupTestServer()
    defer server.Close()

    // Create a test client
    client := &http.Client{}

    // Test creating a user
    createReq := CreateUserRequest{
        Name:  "Test User",
        Email: "test@example.com",
    }
    createBody, _ := json.Marshal(createReq)

    resp, err := client.Post(server.URL+"/api/v1/users", "application/json", bytes.NewBuffer(createBody))
    assert.NoError(t, err)
    assert.Equal(t, http.StatusCreated, resp.StatusCode)

    var createdUser User
    json.NewDecoder(resp.Body).Decode(&createdUser)
    resp.Body.Close()

    // Test getting the created user
    resp, err = client.Get(server.URL + "/api/v1/users/" + createdUser.ID)
    assert.NoError(t, err)
    assert.Equal(t, http.StatusOK, resp.StatusCode)

    var fetchedUser User
    json.NewDecoder(resp.Body).Decode(&fetchedUser)
    resp.Body.Close()

    assert.Equal(t, createdUser.ID, fetchedUser.ID)
    assert.Equal(t, "Test User", fetchedUser.Name)
}
```

### 3. Contract Testing

Ensure your API adheres to its contract:

```go
func TestUserAPI_Contract(t *testing.T) {
    // Set up test server
    server := setupTestServer()
    defer server.Close()

    // Run contract tests using a tool like Pact
    pact := dsl.Pact{
        Consumer: "UserWebApp",
        Provider: "UserService",
    }

    // Define the expected interaction
    pact.AddInteraction().
        Given("A user exists").
        UponReceiving("A request for a user").
        WithRequest(dsl.Request{
            Method: "GET",
            Path:   dsl.String("/api/v1/users/123"),
            Headers: dsl.MapMatcher{
                "Accept": dsl.String("application/json"),
            },
        }).
        WillRespondWith(dsl.Response{
            Status: 200,
            Headers: dsl.MapMatcher{
                "Content-Type": dsl.String("application/json"),
            },
            Body: dsl.Match(User{}),
        })

    // Verify the interaction
    err := pact.Verify(func() error {
        // Make the request to the test server
        client := &http.Client{}
        req, _ := http.NewRequest("GET", server.URL+"/api/v1/users/123", nil)
        req.Header.Set("Accept", "application/json")

        resp, err := client.Do(req)
        if err != nil {
            return err
        }
        defer resp.Body.Close()

        return nil
    })

    assert.NoError(t, err)
}
```

## Common API Design Pitfalls

### 1. Chatty APIs

**Problem**: Too many small API calls required to accomplish a task.

**Solution**: Design coarse-grained APIs that provide all necessary data in a single call.

### 2. Tight Coupling

**Problem**: APIs that expose internal implementation details.

**Solution**: Design APIs around business capabilities, not internal structures.

### 3. Inconsistent Error Handling

**Problem**: Different error formats across endpoints or services.

**Solution**: Standardize error responses across all APIs.

### 4. Poor Versioning Strategy

**Problem**: Breaking changes without proper versioning.

**Solution**: Implement a clear versioning strategy and maintain backward compatibility.

### 5. Insufficient Validation

**Problem**: Accepting invalid input that causes downstream issues.

**Solution**: Implement thorough input validation at the API boundary.

## Interview Questions and Answers

### Q: How would you handle API versioning in a microservice architecture?

**A:** I prefer URL path versioning (e.g., `/api/v1/users`) for its explicitness and ease of use. When introducing a new version:

1. Create a new API version when making breaking changes
2. Support both old and new versions simultaneously during a transition period
3. Communicate deprecation timelines clearly to API consumers
4. Consider using feature flags to gradually roll out new versions
5. Monitor usage of each version to determine when it's safe to retire old versions

For implementation, I'd use separate route handlers for each version but share common business logic where possible.

### Q: How do you ensure API security in a microservice environment?

**A:** I implement multiple layers of security:

1. **Transport Security**: Always use HTTPS/TLS for all API communications
2. **Authentication**: Implement token-based auth (JWT or OAuth 2.0)
3. **Authorization**: Use role-based or attribute-based access control
4. **API Gateway**: Centralize security concerns like rate limiting and IP filtering
5. **Input Validation**: Thoroughly validate all inputs to prevent injection attacks
6. **Sensitive Data Handling**: Encrypt sensitive data and use appropriate HTTP headers
7. **Audit Logging**: Log all security-relevant events for monitoring and compliance

### Q: How would you design an API that needs to handle both synchronous and asynchronous operations?

**A:** I would implement a hybrid approach:

1. For synchronous operations that complete quickly, use standard request-response patterns
2. For long-running operations, implement an asynchronous pattern:
   - Accept the initial request and return a 202 Accepted status
   - Create a job/task resource with a unique ID
   - Provide a way to check the status of the job (e.g., GET /jobs/{id})
   - Optionally, implement webhooks or event streaming for proactive updates

```
# Initial request
POST /api/v1/reports
{
  "type": "monthly-sales",
  "parameters": { ... }
}

# Response
HTTP/1.1 202 Accepted
{
  "job_id": "abc123",
  "status": "processing",
  "status_url": "/api/v1/jobs/abc123"
}

# Status check
GET /api/v1/jobs/abc123

# Status response
HTTP/1.1 200 OK
{
  "job_id": "abc123",
  "status": "completed",
  "result_url": "/api/v1/reports/xyz789"
}
```

## Conclusion

Well-designed APIs are essential for successful microservice architectures. By following these best practices for both REST and gRPC APIs, you can create robust, maintainable, and developer-friendly interfaces between your services.

In the next section, we'll explore service communication patterns, including synchronous and asynchronous communication mechanisms for microservices.