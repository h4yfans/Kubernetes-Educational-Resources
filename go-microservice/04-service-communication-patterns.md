# Service Communication Patterns for Microservices

## Introduction

In a microservice architecture, services need to communicate with each other to fulfill business operations. This document explores various communication patterns, their trade-offs, and implementation approaches in Go.

## Synchronous Communication Patterns

Synchronous communication involves a service making a request and waiting for a response before proceeding.

### 1. Request-Response

The most basic pattern where a service makes a direct HTTP/gRPC call to another service and waits for a response.

**Implementation with HTTP:**

```go
func (s *OrderService) GetUserDetails(ctx context.Context, userID string) (*UserDetails, error) {
    req, err := http.NewRequestWithContext(ctx, "GET",
        fmt.Sprintf("http://user-service/users/%s", userID), nil)
    if err != nil {
        return nil, fmt.Errorf("failed to create request: %w", err)
    }

    resp, err := s.httpClient.Do(req)
    if err != nil {
        return nil, fmt.Errorf("failed to get user details: %w", err)
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusOK {
        return nil, fmt.Errorf("user service returned status: %d", resp.StatusCode)
    }

    var userDetails UserDetails
    if err := json.NewDecoder(resp.Body).Decode(&userDetails); err != nil {
        return nil, fmt.Errorf("failed to decode response: %w", err)
    }

    return &userDetails, nil
}
```

**Implementation with gRPC:**

```go
func (s *OrderService) GetUserDetails(ctx context.Context, userID string) (*UserDetails, error) {
    resp, err := s.userClient.GetUser(ctx, &pb.GetUserRequest{Id: userID})
    if err != nil {
        return nil, fmt.Errorf("failed to get user details: %w", err)
    }

    return &UserDetails{
        ID:    resp.Id,
        Name:  resp.Name,
        Email: resp.Email,
    }, nil
}
```

**Advantages:**
- Simple to implement and understand
- Immediate response with success or failure
- Strong consistency guarantees

**Disadvantages:**
- Increased latency as the client waits for a response
- Tight coupling between services
- Reduced availability (if the called service is down, the caller fails)

### 2. API Gateway / Backend for Frontend (BFF)

An intermediary that sits between clients and services, routing requests and aggregating responses.

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

**Implementation in Go:**

```go
func (g *Gateway) GetOrderDetails(w http.ResponseWriter, r *http.Request) {
    orderID := chi.URLParam(r, "id")
    ctx := r.Context()

    // Get order data
    orderResp, err := g.orderClient.GetOrder(ctx, &pb.GetOrderRequest{Id: orderID})
    if err != nil {
        http.Error(w, "Failed to get order", http.StatusInternalServerError)
        return
    }

    // Get user data
    userResp, err := g.userClient.GetUser(ctx, &pb.GetUserRequest{Id: orderResp.UserId})
    if err != nil {
        http.Error(w, "Failed to get user", http.StatusInternalServerError)
        return
    }

    // Get product data for each item in the order
    var products []*ProductDetails
    for _, item := range orderResp.Items {
        productResp, err := g.productClient.GetProduct(ctx, &pb.GetProductRequest{Id: item.ProductId})
        if err != nil {
            http.Error(w, "Failed to get product", http.StatusInternalServerError)
            return
        }

        products = append(products, &ProductDetails{
            ID:    productResp.Id,
            Name:  productResp.Name,
            Price: productResp.Price,
        })
    }

    // Combine all data
    response := OrderDetailsResponse{
        Order: OrderDetails{
            ID:        orderResp.Id,
            Status:    orderResp.Status,
            CreatedAt: orderResp.CreatedAt.AsTime(),
        },
        User: UserDetails{
            ID:    userResp.Id,
            Name:  userResp.Name,
            Email: userResp.Email,
        },
        Products: products,
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(response)
}
```

**Advantages:**
- Reduces client complexity by providing a single entry point
- Can aggregate data from multiple services
- Enables protocol translation (e.g., REST to gRPC)
- Centralizes cross-cutting concerns like authentication

**Disadvantages:**
- Can become a bottleneck if not properly designed
- Adds another layer of complexity
- May introduce a single point of failure

### 3. Service Mesh

A dedicated infrastructure layer for handling service-to-service communication, providing features like service discovery, load balancing, encryption, and observability.

Popular service mesh implementations include:
- Istio
- Linkerd
- Consul Connect

**Example configuration with Istio:**

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: order-service
spec:
  hosts:
  - order-service
  http:
  - route:
    - destination:
        host: order-service
        subset: v1
      weight: 90
    - destination:
        host: order-service
        subset: v2
      weight: 10
  - match:
    - headers:
        end-user:
          exact: beta-tester
    route:
    - destination:
        host: order-service
        subset: v2
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: order-service
spec:
  host: order-service
  trafficPolicy:
    loadBalancer:
      simple: ROUND_ROBIN
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
    trafficPolicy:
      loadBalancer:
        simple: LEAST_CONN
```

**Advantages:**
- Provides advanced traffic management capabilities
- Handles service discovery and load balancing
- Offers built-in observability and security features
- Decouples communication concerns from application code

**Disadvantages:**
- Adds complexity to the infrastructure
- May introduce performance overhead
- Requires expertise to set up and maintain

### 4. Circuit Breaker Pattern

Prevents cascading failures by stopping calls to a failing service and allowing it time to recover.

**Implementation in Go using gobreaker:**

```go
package main

import (
    "fmt"
    "time"

    "github.com/sony/gobreaker"
)

type UserService struct {
    client     *http.Client
    baseURL    string
    cb         *gobreaker.CircuitBreaker
}

func NewUserService(baseURL string) *UserService {
    // Configure the circuit breaker
    cbSettings := gobreaker.Settings{
        Name:        "user-service",
        MaxRequests: 5,                // Max requests allowed in half-open state
        Interval:    10 * time.Second, // Interval for clearing counters
        Timeout:     30 * time.Second, // Time to wait before transitioning from open to half-open
        ReadyToTrip: func(counts gobreaker.Counts) bool {
            failureRatio := float64(counts.TotalFailures) / float64(counts.Requests)
            return counts.Requests >= 10 && failureRatio >= 0.5
        },
        OnStateChange: func(name string, from gobreaker.State, to gobreaker.State) {
            log.Printf("Circuit breaker %s state changed from %s to %s", name, from, to)
        },
    }

    return &UserService{
        client:  &http.Client{Timeout: 5 * time.Second},
        baseURL: baseURL,
        cb:      gobreaker.NewCircuitBreaker(cbSettings),
    }
}

func (s *UserService) GetUser(ctx context.Context, userID string) (*User, error) {
    // Execute the request through the circuit breaker
    result, err := s.cb.Execute(func() (interface{}, error) {
        url := fmt.Sprintf("%s/users/%s", s.baseURL, userID)
        req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
        if err != nil {
            return nil, err
        }

        resp, err := s.client.Do(req)
        if err != nil {
            return nil, err
        }
        defer resp.Body.Close()

        if resp.StatusCode != http.StatusOK {
            return nil, fmt.Errorf("user service returned status: %d", resp.StatusCode)
        }

        var user User
        if err := json.NewDecoder(resp.Body).Decode(&user); err != nil {
            return nil, err
        }

        return &user, nil
    })

    if err != nil {
        return nil, err
    }

    return result.(*User), nil
}
```

**Advantages:**
- Prevents cascading failures across services
- Allows failing services time to recover
- Fails fast when a service is known to be down
- Provides graceful degradation

**Disadvantages:**
- Adds complexity to the client code
- Requires careful configuration of thresholds
- May need fallback mechanisms for critical operations

## Asynchronous Communication Patterns

Asynchronous communication allows services to communicate without waiting for a response, enabling better scalability and resilience.

### 1. Message Queuing

Services communicate by sending messages through a queue, where they are stored until the receiving service processes them.

**Implementation with RabbitMQ in Go:**

```go
// Publisher
func (s *OrderService) CreateOrder(ctx context.Context, order *Order) error {
    // Save order to database
    if err := s.repo.SaveOrder(ctx, order); err != nil {
        return fmt.Errorf("failed to save order: %w", err)
    }

    // Publish event to message queue
    orderCreatedEvent := OrderCreatedEvent{
        OrderID:    order.ID,
        UserID:     order.UserID,
        TotalPrice: order.TotalPrice,
        Items:      order.Items,
        CreatedAt:  time.Now(),
    }

    payload, err := json.Marshal(orderCreatedEvent)
    if err != nil {
        return fmt.Errorf("failed to marshal event: %w", err)
    }

    err = s.channel.PublishWithContext(
        ctx,
        "orders",  // exchange
        "created", // routing key
        false,     // mandatory
        false,     // immediate
        amqp.Publishing{
            ContentType: "application/json",
            Body:        payload,
        },
    )
    if err != nil {
        return fmt.Errorf("failed to publish event: %w", err)
    }

    return nil
}

// Consumer
func (s *InventoryService) StartConsumer() error {
    msgs, err := s.channel.Consume(
        "inventory_queue", // queue
        "",                // consumer
        false,             // auto-ack
        false,             // exclusive
        false,             // no-local
        false,             // no-wait
        nil,               // args
    )
    if err != nil {
        return fmt.Errorf("failed to register consumer: %w", err)
    }

    go func() {
        for msg := range msgs {
            var event OrderCreatedEvent
            if err := json.Unmarshal(msg.Body, &event); err != nil {
                log.Printf("Error unmarshaling message: %v", err)
                msg.Nack(false, true) // negative acknowledgement, requeue
                continue
            }

            // Process the order
            err := s.processOrder(context.Background(), &event)
            if err != nil {
                log.Printf("Error processing order: %v", err)
                msg.Nack(false, true) // negative acknowledgement, requeue
                continue
            }

            msg.Ack(false) // acknowledge message
        }
    }()

    return nil
}

func (s *InventoryService) processOrder(ctx context.Context, event *OrderCreatedEvent) error {
    // Update inventory for each item in the order
    for _, item := range event.Items {
        if err := s.repo.DecrementStock(ctx, item.ProductID, item.Quantity); err != nil {
            return fmt.Errorf("failed to update inventory: %w", err)
        }
    }

    return nil
}
```

**Advantages:**
- Decouples services temporally
- Provides a buffer during traffic spikes
- Enables reliable delivery with message persistence
- Allows for retry mechanisms

**Disadvantages:**
- Adds complexity with message broker infrastructure
- May introduce eventual consistency challenges
- Requires handling of duplicate messages

### 2. Publish-Subscribe (Pub/Sub)

A messaging pattern where publishers categorize messages into topics, and subscribers express interest in topics to receive relevant messages.

**Implementation with NATS in Go:**

```go
// Publisher
func (s *PaymentService) ProcessPayment(ctx context.Context, payment *Payment) error {
    // Process payment logic
    result, err := s.paymentGateway.Charge(ctx, payment)
    if err != nil {
        return fmt.Errorf("payment processing failed: %w", err)
    }

    // Publish event
    paymentProcessedEvent := PaymentProcessedEvent{
        PaymentID:  payment.ID,
        OrderID:    payment.OrderID,
        Amount:     payment.Amount,
        Status:     result.Status,
        ProcessedAt: time.Now(),
    }

    payload, err := json.Marshal(paymentProcessedEvent)
    if err != nil {
        return fmt.Errorf("failed to marshal event: %w", err)
    }

    err = s.natsConn.Publish("payments.processed", payload)
    if err != nil {
        return fmt.Errorf("failed to publish event: %w", err)
    }

    return nil
}

// Subscriber (Order Service)
func (s *OrderService) subscribeToPaymentEvents() error {
    _, err := s.natsConn.Subscribe("payments.processed", func(msg *nats.Msg) {
        var event PaymentProcessedEvent
        if err := json.Unmarshal(msg.Data, &event); err != nil {
            log.Printf("Error unmarshaling payment event: %v", err)
            return
        }

        // Update order status based on payment status
        ctx := context.Background()
        if event.Status == "succeeded" {
            if err := s.repo.UpdateOrderStatus(ctx, event.OrderID, "paid"); err != nil {
                log.Printf("Error updating order status: %v", err)
                return
            }
        } else {
            if err := s.repo.UpdateOrderStatus(ctx, event.OrderID, "payment_failed"); err != nil {
                log.Printf("Error updating order status: %v", err)
                return
            }
        }
    })

    return err
}

// Subscriber (Notification Service)
func (s *NotificationService) subscribeToPaymentEvents() error {
    _, err := s.natsConn.Subscribe("payments.processed", func(msg *nats.Msg) {
        var event PaymentProcessedEvent
        if err := json.Unmarshal(msg.Data, &event); err != nil {
            log.Printf("Error unmarshaling payment event: %v", err)
            return
        }

        // Get order details
        ctx := context.Background()
        order, err := s.orderClient.GetOrder(ctx, event.OrderID)
        if err != nil {
            log.Printf("Error getting order details: %v", err)
            return
        }

        // Send notification to user
        if event.Status == "succeeded" {
            if err := s.emailSender.SendPaymentConfirmation(order.UserEmail, event.OrderID); err != nil {
                log.Printf("Error sending payment confirmation: %v", err)
                return
            }
        } else {
            if err := s.emailSender.SendPaymentFailure(order.UserEmail, event.OrderID); err != nil {
                log.Printf("Error sending payment failure notification: %v", err)
                return
            }
        }
    })

    return err
}
```

**Advantages:**
- One-to-many communication model
- Publishers and subscribers are decoupled
- Multiple services can react to the same event
- Enables event-driven architecture

**Disadvantages:**
- May lead to complex event dependencies
- Harder to track the flow of operations
- Potential for message loss if not configured for persistence

### 3. Event Sourcing

Stores all changes to application state as a sequence of events, which can be replayed to reconstruct the current state.

**Implementation in Go:**

```go
// Event definitions
type Event interface {
    EventType() string
    AggregateID() string
    Timestamp() time.Time
}

type OrderCreatedEvent struct {
    ID        string    `json:"id"`
    UserID    string    `json:"user_id"`
    Items     []Item    `json:"items"`
    CreatedAt time.Time `json:"created_at"`
}

func (e OrderCreatedEvent) EventType() string    { return "order_created" }
func (e OrderCreatedEvent) AggregateID() string  { return e.ID }
func (e OrderCreatedEvent) Timestamp() time.Time { return e.CreatedAt }

type OrderItemAddedEvent struct {
    OrderID   string    `json:"order_id"`
    ProductID string    `json:"product_id"`
    Quantity  int       `json:"quantity"`
    Price     float64   `json:"price"`
    AddedAt   time.Time `json:"added_at"`
}

func (e OrderItemAddedEvent) EventType() string    { return "order_item_added" }
func (e OrderItemAddedEvent) AggregateID() string  { return e.OrderID }
func (e OrderItemAddedEvent) Timestamp() time.Time { return e.AddedAt }

// Event store
type EventStore interface {
    SaveEvent(ctx context.Context, event Event) error
    GetEvents(ctx context.Context, aggregateID string) ([]Event, error)
}

// PostgreSQL implementation of event store
type PostgresEventStore struct {
    db *sql.DB
}

func (s *PostgresEventStore) SaveEvent(ctx context.Context, event Event) error {
    payload, err := json.Marshal(event)
    if err != nil {
        return fmt.Errorf("failed to marshal event: %w", err)
    }

    _, err = s.db.ExecContext(
        ctx,
        "INSERT INTO events (aggregate_id, event_type, payload, created_at) VALUES ($1, $2, $3, $4)",
        event.AggregateID(),
        event.EventType(),
        payload,
        event.Timestamp(),
    )
    if err != nil {
        return fmt.Errorf("failed to save event: %w", err)
    }

    return nil
}

func (s *PostgresEventStore) GetEvents(ctx context.Context, aggregateID string) ([]Event, error) {
    rows, err := s.db.QueryContext(
        ctx,
        "SELECT event_type, payload FROM events WHERE aggregate_id = $1 ORDER BY created_at ASC",
        aggregateID,
    )
    if err != nil {
        return nil, fmt.Errorf("failed to query events: %w", err)
    }
    defer rows.Close()

    var events []Event
    for rows.Next() {
        var eventType string
        var payload []byte

        if err := rows.Scan(&eventType, &payload); err != nil {
            return nil, fmt.Errorf("failed to scan row: %w", err)
        }

        var event Event
        switch eventType {
        case "order_created":
            var e OrderCreatedEvent
            if err := json.Unmarshal(payload, &e); err != nil {
                return nil, fmt.Errorf("failed to unmarshal event: %w", err)
            }
            event = e
        case "order_item_added":
            var e OrderItemAddedEvent
            if err := json.Unmarshal(payload, &e); err != nil {
                return nil, fmt.Errorf("failed to unmarshal event: %w", err)
            }
            event = e
        default:
            return nil, fmt.Errorf("unknown event type: %s", eventType)
        }

        events = append(events, event)
    }

    return events, nil
}

// Order aggregate
type Order struct {
    ID        string
    UserID    string
    Items     []Item
    CreatedAt time.Time
}

type Item struct {
    ProductID string
    Quantity  int
    Price     float64
}

// Rebuild order from events
func RebuildOrder(events []Event) (*Order, error) {
    if len(events) == 0 {
        return nil, fmt.Errorf("no events found")
    }

    var order *Order

    for _, event := range events {
        switch e := event.(type) {
        case OrderCreatedEvent:
            order = &Order{
                ID:        e.ID,
                UserID:    e.UserID,
                Items:     e.Items,
                CreatedAt: e.CreatedAt,
            }
        case OrderItemAddedEvent:
            if order == nil {
                return nil, fmt.Errorf("order not initialized")
            }

            order.Items = append(order.Items, Item{
                ProductID: e.ProductID,
                Quantity:  e.Quantity,
                Price:     e.Price,
            })
        }
    }

    return order, nil
}

// Order service
type OrderService struct {
    eventStore EventStore
}

func (s *OrderService) CreateOrder(ctx context.Context, userID string) (string, error) {
    orderID := uuid.New().String()

    event := OrderCreatedEvent{
        ID:        orderID,
        UserID:    userID,
        Items:     []Item{},
        CreatedAt: time.Now(),
    }

    if err := s.eventStore.SaveEvent(ctx, event); err != nil {
        return "", fmt.Errorf("failed to save order created event: %w", err)
    }

    return orderID, nil
}

func (s *OrderService) AddItem(ctx context.Context, orderID, productID string, quantity int, price float64) error {
    event := OrderItemAddedEvent{
        OrderID:   orderID,
        ProductID: productID,
        Quantity:  quantity,
        Price:     price,
        AddedAt:   time.Now(),
    }

    if err := s.eventStore.SaveEvent(ctx, event); err != nil {
        return fmt.Errorf("failed to save order item added event: %w", err)
    }

    return nil
}

func (s *OrderService) GetOrder(ctx context.Context, orderID string) (*Order, error) {
    events, err := s.eventStore.GetEvents(ctx, orderID)
    if err != nil {
        return nil, fmt.Errorf("failed to get events: %w", err)
    }

    order, err := RebuildOrder(events)
    if err != nil {
        return nil, fmt.Errorf("failed to rebuild order: %w", err)
    }

    return order, nil
}
```

**Advantages:**
- Complete audit trail of all changes
- Ability to reconstruct past states
- Natural fit for event-driven architectures
- Enables powerful event replay capabilities

**Disadvantages:**
- Increased complexity in application design
- Potential performance issues with large event streams
- Requires careful event schema evolution
- Learning curve for developers

### 4. Command Query Responsibility Segregation (CQRS)

Separates read and write operations into different models, allowing each to be optimized independently.

**Implementation in Go:**

```go
// Command model (write operations)
type OrderCommandRepository interface {
    CreateOrder(ctx context.Context, order *Order) error
    UpdateOrderStatus(ctx context.Context, orderID, status string) error
    AddOrderItem(ctx context.Context, orderID string, item *OrderItem) error
}

type OrderCommandService struct {
    repo OrderCommandRepository
    eventBus EventBus
}

func (s *OrderCommandService) CreateOrder(ctx context.Context, cmd CreateOrderCommand) (string, error) {
    order := &Order{
        ID:        uuid.New().String(),
        UserID:    cmd.UserID,
        Status:    "created",
        CreatedAt: time.Now(),
    }

    if err := s.repo.CreateOrder(ctx, order); err != nil {
        return "", fmt.Errorf("failed to create order: %w", err)
    }

    // Publish event
    event := OrderCreatedEvent{
        OrderID:   order.ID,
        UserID:    order.UserID,
        Status:    order.Status,
        CreatedAt: order.CreatedAt,
    }

    if err := s.eventBus.Publish(ctx, "orders.created", event); err != nil {
        log.Printf("Failed to publish order created event: %v", err)
    }

    return order.ID, nil
}

// Query model (read operations)
type OrderQueryRepository interface {
    GetOrderByID(ctx context.Context, orderID string) (*OrderReadModel, error)
    GetOrdersByUserID(ctx context.Context, userID string) ([]*OrderReadModel, error)
}

type OrderReadModel struct {
    ID            string
    UserID        string
    Status        string
    Items         []OrderItemReadModel
    TotalAmount   float64
    CreatedAt     time.Time
    LastUpdatedAt time.Time
}

type OrderItemReadModel struct {
    ProductID   string
    ProductName string
    Quantity    int
    Price       float64
    Subtotal    float64
}

type OrderQueryService struct {
    repo OrderQueryRepository
}

func (s *OrderQueryService) GetOrderByID(ctx context.Context, orderID string) (*OrderReadModel, error) {
    return s.repo.GetOrderByID(ctx, orderID)
}

func (s *OrderQueryService) GetOrdersByUserID(ctx context.Context, userID string) ([]*OrderReadModel, error) {
    return s.repo.GetOrdersByUserID(ctx, userID)
}

// Event handler to update the read model
type OrderEventHandler struct {
    db *sql.DB
}

func (h *OrderEventHandler) HandleOrderCreated(ctx context.Context, event OrderCreatedEvent) error {
    _, err := h.db.ExecContext(
        ctx,
        `INSERT INTO order_read_model (id, user_id, status, total_amount, created_at, last_updated_at)
         VALUES ($1, $2, $3, $4, $5, $6)`,
        event.OrderID,
        event.UserID,
        event.Status,
        0.0, // Initial total amount
        event.CreatedAt,
        event.CreatedAt,
    )
    if err != nil {
        return fmt.Errorf("failed to insert order read model: %w", err)
    }

    return nil
}

func (h *OrderEventHandler) HandleOrderItemAdded(ctx context.Context, event OrderItemAddedEvent) error {
    tx, err := h.db.BeginTx(ctx, nil)
    if err != nil {
        return fmt.Errorf("failed to begin transaction: %w", err)
    }
    defer tx.Rollback()

    // Insert order item
    _, err = tx.ExecContext(
        ctx,
        `INSERT INTO order_item_read_model (order_id, product_id, product_name, quantity, price, subtotal)
         VALUES ($1, $2, $3, $4, $5, $6)`,
        event.OrderID,
        event.ProductID,
        event.ProductName,
        event.Quantity,
        event.Price,
        event.Quantity * event.Price,
    )
    if err != nil {
        return fmt.Errorf("failed to insert order item read model: %w", err)
    }

    // Update order total amount
    _, err = tx.ExecContext(
        ctx,
        `UPDATE order_read_model
         SET total_amount = total_amount + $1, last_updated_at = $2
         WHERE id = $3`,
        event.Quantity * event.Price,
        event.Timestamp,
        event.OrderID,
    )
    if err != nil {
        return fmt.Errorf("failed to update order read model: %w", err)
    }

    if err := tx.Commit(); err != nil {
        return fmt.Errorf("failed to commit transaction: %w", err)
    }

    return nil
}

// HTTP handlers
func (h *OrderHandler) CreateOrder(w http.ResponseWriter, r *http.Request) {
    var cmd CreateOrderCommand
    if err := json.NewDecoder(r.Body).Decode(&cmd); err != nil {
        http.Error(w, "Invalid request body", http.StatusBadRequest)
        return
    }

    orderID, err := h.commandService.CreateOrder(r.Context(), cmd)
    if err != nil {
        http.Error(w, "Failed to create order", http.StatusInternalServerError)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(map[string]string{"order_id": orderID})
}

func (h *OrderHandler) GetOrder(w http.ResponseWriter, r *http.Request) {
    orderID := chi.URLParam(r, "id")

    order, err := h.queryService.GetOrderByID(r.Context(), orderID)
    if err != nil {
        http.Error(w, "Failed to get order", http.StatusInternalServerError)
        return
    }

    if order == nil {
        http.Error(w, "Order not found", http.StatusNotFound)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(order)
}
```

**Advantages:**
- Optimized read and write models
- Scalability for read-heavy workloads
- Separation of concerns
- Flexibility in storage technologies

**Disadvantages:**
- Increased complexity with multiple models
- Eventual consistency between read and write models
- Higher development and maintenance costs
- Steeper learning curve

## Choosing the Right Communication Pattern

Selecting the appropriate communication pattern depends on various factors:

| Pattern | When to Use | When to Avoid |
|---------|-------------|---------------|
| **Request-Response** | - Simple operations<br>- When immediate response is required<br>- For queries that need consistent data | - Long-running operations<br>- When high availability is critical<br>- When the called service might be slow |
| **API Gateway** | - When clients need data from multiple services<br>- When you need centralized authentication<br>- For protocol translation | - For simple architectures with few services<br>- When performance is critical (adds latency) |
| **Service Mesh** | - Complex microservice deployments<br>- When you need advanced traffic management<br>- When security between services is important | - Simple architectures with few services<br>- Resource-constrained environments |
| **Message Queuing** | - For decoupling services<br>- When operations can be processed asynchronously<br>- For handling traffic spikes | - When immediate response is required<br>- For simple query operations |
| **Pub/Sub** | - When multiple services need to react to events<br>- For broadcasting notifications<br>- For building event-driven architectures | - When strong consistency is required<br>- For request-response interactions |
| **Event Sourcing** | - When you need a complete audit trail<br>- For complex business domains<br>- When you need to reconstruct past states | - Simple CRUD applications<br>- When query performance is critical |
| **CQRS** | - Read-heavy workloads<br>- Complex domains with different read/write requirements<br>- When scaling reads independently is important | - Simple applications<br>- When consistency is more important than performance |

## Implementation Best Practices

### 1. Error Handling and Resilience

- **Implement proper timeouts**: Set appropriate timeouts for all service calls to prevent cascading failures.
  ```go
  ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
  defer cancel()
  response, err := client.DoSomething(ctx, request)
  ```

- **Use circuit breakers**: Prevent cascading failures by stopping calls to failing services.
  ```go
  // Using github.com/sony/gobreaker as shown earlier
  ```

- **Implement retries with backoff**: Retry failed operations with exponential backoff.
  ```go
  // Using github.com/cenkalti/backoff/v4
  operation := func() error {
      return client.DoSomething(ctx, request)
  }

  err := backoff.Retry(operation, backoff.NewExponentialBackOff())
  ```

- **Provide fallbacks**: Have fallback mechanisms when services are unavailable.
  ```go
  result, err := circuitBreaker.Execute(func() (interface{}, error) {
      return primaryService.GetData(ctx)
  })

  if err != nil {
      // Fallback to cache or default value
      return fallbackService.GetData(ctx)
  }
  ```

### 2. Service Discovery

- **Use a service registry**: Register and discover services dynamically.
  ```go
  // Using Consul client
  config := consulapi.DefaultConfig()
  client, _ := consulapi.NewClient(config)

  // Register service
  registration := &consulapi.AgentServiceRegistration{
      ID:      "order-service-1",
      Name:    "order-service",
      Port:    8080,
      Address: "192.168.1.100",
      Check: &consulapi.AgentServiceCheck{
          HTTP:     "http://192.168.1.100:8080/health",
          Interval: "10s",
      },
  }
  client.Agent().ServiceRegister(registration)

  // Discover service
  services, _, _ := client.Health().Service("user-service", "", true, nil)
  serviceURL := fmt.Sprintf("http://%s:%d", services[0].Service.Address, services[0].Service.Port)
  ```

- **Use DNS-based discovery**: Leverage Kubernetes service discovery.
  ```go
  // In Kubernetes, simply use the service name
  resp, err := http.Get("http://user-service:8080/users/123")
  ```

### 3. Monitoring and Tracing

- **Implement distributed tracing**: Use OpenTelemetry to trace requests across services.
  ```go
  // Initialize tracer
  tp := initTracer()
  defer tp.Shutdown(context.Background())

  // Create a span
  ctx, span := tp.Tracer("order-service").Start(ctx, "GetOrderDetails")
  defer span.End()

  // Add attributes to the span
  span.SetAttributes(attribute.String("order.id", orderID))

  // Create a child span for a downstream call
  ctx, childSpan := tp.Tracer("order-service").Start(ctx, "GetUserDetails")
  defer childSpan.End()

  // Make the call with the context that has the span
  userDetails, err := userClient.GetUserDetails(ctx, userID)
  if err != nil {
      childSpan.RecordError(err)
      childSpan.SetStatus(codes.Error, err.Error())
  }
  ```

- **Add metrics**: Instrument your code with Prometheus metrics.
  ```go
  // Define metrics
  requestDuration := prometheus.NewHistogramVec(
      prometheus.HistogramOpts{
          Name:    "http_request_duration_seconds",
          Help:    "Duration of HTTP requests in seconds",
          Buckets: prometheus.DefBuckets,
      },
      []string{"handler", "method", "status"},
  )
  prometheus.MustRegister(requestDuration)

  // Use middleware to record metrics
  func metricsMiddleware(next http.Handler) http.Handler {
      return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
          start := time.Now()

          ww := middleware.NewWrapResponseWriter(w, r.ProtoMajor)
          next.ServeHTTP(ww, r)

          duration := time.Since(start).Seconds()
          requestDuration.WithLabelValues(
              r.URL.Path,
              r.Method,
              strconv.Itoa(ww.Status()),
          ).Observe(duration)
      })
  }
  ```

### 4. API Versioning

- **Include version in URL path**: `/api/v1/users`
- **Use content negotiation**: `Accept: application/vnd.myapp.v1+json`
- **Maintain backward compatibility**: Add fields without removing existing ones

### 5. Security

- **Use mutual TLS (mTLS)**: Authenticate both client and server.
  ```go
  // Load client certificate and key
  cert, err := tls.LoadX509KeyPair("client.crt", "client.key")
  if err != nil {
      log.Fatalf("Error loading client cert: %v", err)
  }

  // Load CA certificate
  caCert, err := os.ReadFile("ca.crt")
  if err != nil {
      log.Fatalf("Error loading CA cert: %v", err)
  }

  caCertPool := x509.NewCertPool()
  caCertPool.AppendCertsFromPEM(caCert)

  // Create TLS config
  tlsConfig := &tls.Config{
      Certificates: []tls.Certificate{cert},
      RootCAs:      caCertPool,
  }

  // Create HTTP client with TLS config
  transport := &http.Transport{TLSClientConfig: tlsConfig}
  client := &http.Client{Transport: transport}
  ```

- **Implement authentication and authorization**: Use JWT tokens for service-to-service communication.
  ```go
  // Middleware to verify JWT
  func authMiddleware(next http.Handler) http.Handler {
      return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
          tokenString := r.Header.Get("Authorization")
          if tokenString == "" {
              http.Error(w, "Unauthorized", http.StatusUnauthorized)
              return
          }

          // Remove "Bearer " prefix
          tokenString = strings.TrimPrefix(tokenString, "Bearer ")

          // Parse and validate token
          token, err := jwt.Parse(tokenString, func(token *jwt.Token) (interface{}, error) {
              if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
                  return nil, fmt.Errorf("unexpected signing method: %v", token.Header["alg"])
              }
              return []byte(os.Getenv("JWT_SECRET")), nil
          })

          if err != nil || !token.Valid {
              http.Error(w, "Unauthorized", http.StatusUnauthorized)
              return
          }

          // Extract claims and add to context
          if claims, ok := token.Claims.(jwt.MapClaims); ok {
              ctx := context.WithValue(r.Context(), "user", claims)
              next.ServeHTTP(w, r.WithContext(ctx))
          } else {
              http.Error(w, "Unauthorized", http.StatusUnauthorized)
          }
      })
  }
  ```

## Interview Questions and Answers

### 1. What are the main differences between synchronous and asynchronous communication patterns in microservices?

**Answer:**
Synchronous communication involves a service making a request and waiting for a response before proceeding, creating a direct dependency between services. Examples include HTTP/REST and gRPC calls. This pattern is simpler to implement and provides immediate feedback but can lead to increased latency and reduced availability.

Asynchronous communication allows services to communicate without waiting for a response, typically through message brokers or event streams. Examples include message queues, pub/sub, and event sourcing. This pattern provides better decoupling, scalability, and resilience but introduces complexity with eventual consistency and more complex error handling.

### 2. When would you choose gRPC over REST for service-to-service communication?

**Answer:**
I would choose gRPC over REST in the following scenarios:

- **Performance is critical**: gRPC uses Protocol Buffers which are more efficient in serialization/deserialization compared to JSON.
- **Strong typing is needed**: gRPC provides contract-first API development with strict typing.
- **Bidirectional streaming**: When services need to exchange data in real-time in both directions.
- **Service-to-service communication**: gRPC is particularly well-suited for internal service communication.
- **Polyglot environments**: When services are written in different languages, gRPC provides consistent client libraries.

However, REST might be preferred when:
- External clients need to access the API (browser compatibility)
- Simplicity and wide tooling support are more important than performance
- The team is more familiar with REST concepts

### 3. How would you handle failures in asynchronous communication patterns?

**Answer:**
To handle failures in asynchronous communication, I would implement:

1. **Message persistence**: Ensure messages are persisted before acknowledging them.
2. **Dead letter queues (DLQ)**: Route messages that cannot be processed to a separate queue for analysis.
3. **Retry mechanisms**: Implement exponential backoff for retrying failed operations.
4. **Idempotent operations**: Design receivers to handle duplicate messages safely.
5. **Circuit breakers**: Prevent overwhelming failing services with requests.
6. **Monitoring and alerting**: Set up comprehensive monitoring to detect failures quickly.
7. **Compensating transactions**: Implement patterns to undo operations when part of a workflow fails.
8. **Message TTL (Time-To-Live)**: Set expiration for messages to prevent processing stale data.

### 4. What are the challenges of implementing CQRS and Event Sourcing in a microservice architecture?

**Answer:**
Implementing CQRS and Event Sourcing in microservices presents several challenges:

1. **Complexity**: Both patterns add significant complexity to the system design and development process.
2. **Eventual consistency**: Read models may lag behind write operations, requiring careful handling of stale data.
3. **Event schema evolution**: As the system evolves, managing changes to event schemas becomes challenging.
4. **Performance considerations**: Event stores can grow large over time, affecting query performance.
5. **Learning curve**: Teams need to understand new concepts and patterns, which can slow development initially.
6. **Testing complexity**: Testing event-sourced systems requires different approaches than traditional systems.
7. **Debugging difficulty**: Tracing issues across event streams can be more complex than in traditional systems.
8. **Transaction boundaries**: Defining consistent transaction boundaries across services becomes more challenging.

To address these challenges, I would recommend starting with a bounded context where the benefits clearly outweigh the costs, investing in developer training, implementing proper monitoring, and carefully designing event schemas with future evolution in mind.

### 5. How would you design a system that needs both real-time updates and guaranteed message delivery?

**Answer:**
To design a system requiring both real-time updates and guaranteed message delivery, I would implement a hybrid approach:

1. **Primary flow using message queues**:
   - Use a reliable message broker like RabbitMQ or Apache Kafka for guaranteed delivery
   - Implement acknowledgments, persistence, and dead letter queues
   - Process messages asynchronously with proper retry mechanisms

2. **Real-time notification layer**:
   - Add WebSockets or Server-Sent Events for real-time client updates
   - Use a pub/sub system like Redis or NATS for internal real-time communication

3. **Consistency mechanisms**:
   - Implement an event-driven architecture where the guaranteed delivery system is the source of truth
   - Use the real-time system for immediate feedback but confirm with the guaranteed system
   - Add reconciliation processes to handle discrepancies

4. **Implementation example**:
   - When an order is placed, publish to a durable queue for processing
   - Simultaneously notify relevant services via a pub/sub channel
   - The order service processes the queued message and confirms completion
   - Clients receive immediate feedback via WebSockets but UI shows "processing" until confirmed
   - If real-time notification fails, the guaranteed delivery system ensures the operation completes

This approach provides the best of both worlds: immediate feedback for users while ensuring that all operations are eventually processed correctly.