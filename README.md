
Cart API
========

## Overview

This repository contains a Go programming exercise for interview candidates.  
You'll be developing an API for an online shopping cart in the Go programming language.


## Requirements

This is a REST API for basic CRUD operations for an online shopping cart. Data
should be persisted in a storage layer which can use Postgres.

You should use the default `net/http` package for REST implementation; `sqlx` or `sqlc` for interacting with Postgres;
all the queries should be written manually (no ORM); your repo should be private.

#### Additional requirements

- Cover your code with unit tests (you could use `testify`).
- Create a `Dockerfile` and `Docker compose` for your application.
- For parsing your environment files use `viper`.
- To apply migrations for Postgres use `goose` in your application.

### Domain Types

The Cart API consists of two simple types: `Cart` and `CartItem`. The `Cart`  
holds zero or more `CartItem` objects.

`CartItem` objects should be created in DB exactly (not from application).

- The maximum number of distinct products in one cart is **5**.

### Create Cart

A new cart should be created and an ID generated. The new empty cart should be returned.

```sh
POST http://localhost:3000/carts -d '{}'
```

```json
{
  "id": 1,
  "items": []
}
```

### Add to Cart

Cart can contain only 5 products

A new item should be added to an existing cart. 
The new item should be returned.

Should fail if:
  - The cart does not exist.
  - The product name is blank.
  - The price is non-positive.
  - The cart already contains 5 products.

```sh
POST http://localhost:3000/carts/1/items -d '{
  "product": "Shoes",
  "price": 2500.50
}'
```

```json
{
  "id": 1,
  "cart_id": 1,
  "product": "Shoes",
  "price": 2500.50
}
```

### Remove from Cart

An existing item should be removed from a cart. Should fail if the cart does not
exist or if the item does not exist.

```sh
DELETE http://localhost:3000/carts/1/items/1
```

```json
{}
```


### View Cart

An existing cart should be able to be viewed with its items. Should fail if the
cart does not exist.

```sh
GET http://localhost:3000/carts/1
```

```json
{
  "id": 1,
  "items": [
    {
      "id": 1,
      "cart_id": 1,
      "product": "Shoes",
      "price": 2500.50
    },
    {
      "id": 2,
      "cart_id": 1,
      "product": "Socks",
      "price": 1200.00
    }
  ]
}
```

### Calculate Cart Price and Discounts

Add an endpoint to calculate the total price of the cart and apply discounts.
The total price is the sum of prices of all items in the cart.

Discount rules:
  - If the total price > 5000 → apply a 10% discount.
  - If the total number of items > 3 → apply a 5% discount.
  - If both conditions are met, apply the larger discount (10%).

Example request:

```sh
GET http://localhost:3000/carts/1/price
```

Example response:

```json
{
  "cart_id": 1,
  "total_price": 6200.00,
  "discount_percent": 10,
  "final_price": 5580.00
}
```
## Project Structure

```
cart_api/
│
├── cmd/                          # Application entry point
│   └── main.go                   # Main initialization and server startup
│
├── config/                       # Configuration files
│   └── config.yaml               # Application settings (DB, HTTP server, timeouts)
│
├── internal/                     # Internal application packages
│   │
│   ├── config/                   # Configuration management layer
│   │   └── config.go             # Config loading with viper, structs with mapstructure tags
│   │
│   ├── db/                       # Database layer
│   │   ├── db.go                 # DB connection initialization with sqlx
│   │   └── migrations/           # SQL migrations directory
│   │       └── 00001_init.sql    # Initial database schema
│   │
│   ├── entity/                   # Domain entities and DTOs
│   │   ├── cart.go               # Cart domain entity
│   │   ├── cart_item.go          # CartItem domain entity
│   │   └── add_cart_item.go      # DTO for adding items to cart
│   │
│   ├── errorsx/                  # Custom application errors
│   │   └── repository_errors.go  # Sentinel errors (ErrCartNotFound, etc.)
│   │
│   ├── handlers/                 # HTTP handlers layer (Controllers)
│   │   │                         # Each business function = separate file
│   │   ├── server.go             # Server setup and routing
│   │   ├── create_cart.go        # POST   /api/v1/carts
│   │   ├── view_cart.go          # GET    /api/v1/carts/{id}
│   │   ├── add_to_cart.go        # POST   /api/v1/carts/{id}/items
│   │   ├── remove_from_cart.go   # DELETE /api/v1/carts/{id}/items/{item_id}
│   │   └── helpers.go            # HTTPErrorResponse and utilities
│   │
│   ├── repository/               # Data access layer
│   │   │                         # Each business function = separate file
│   │   ├── repository.go         # Repository struct and constructor
│   │   ├── create_cart.go        # AddCart() - insert cart
│   │   ├── view_cart.go          # GetCart() - select cart with items
│   │   ├── add_to_cart.go        # AddCartItem() - insert cart item
│   │   ├── remove_from_cart.go   # RemoveCartItem() - delete cart item
│   │   └── repository_test.go    # All repository tests
│   │
│   └── service/                  # Business logic layer
│       │                         # Each business function = separate file
│       ├── service.go            # Service struct and constructor
│       ├── repositorier.go       # Repository interface abstraction
│       ├── servicer.go           # Service interface for testing
│       ├── create_cart.go        # CreateCart() - business logic
│       ├── view_cart.go          # ViewCart() - business logic
│       ├── add_to_cart.go        # AddCartItemToCart() - business logic
│       ├── remove_from_cart.go   # RemoveCartItemFromCart() - business logic
│       ├── service_test.go       # All service tests
│       └── mock_repositorier_test.go  # Mock repository for testing
│
├── docker-compose.yaml           # Docker Compose configuration
├── Dockerfile                    # Application container definition
├── go.mod                        # Go module dependencies
└── go.sum                        # Dependencies checksums

```

## Layer Descriptions

### **cmd/** - Application Entry Point
**Purpose**: Application initialization and startup  
**Responsibilities**:
- Load configuration from `config/config.yaml`
- Initialize database connection
- Create repository, service, and handlers (Dependency Injection)
- Start HTTP server
- Handle critical initialization errors with `log.Fatalf()`
- Gracefull shutdown

**Key Rules**:
- Simple and readable `main()` function
- Explicit dependency passing through constructors
- Context with timeout for initialization operations
- No business logic in this layer

---

### **config/** - Configuration Files
**Purpose**: Application configuration storage  
**Contains**:
- `config.yaml` - YAML file with all application settings
- Environment-specific configurations
- Database connection parameters
- HTTP server settings
- Timeout values

---

### **internal/config/** - Configuration Management
**Purpose**: Configuration loading and parsing  
**Responsibilities**:
- Load configuration from YAML files using `viper`
- Map configuration to Go structs with `mapstructure` tags
- Provide typed configuration structures
- Export `LoadConfig(path string)` function

**Key Rules**:
- All struct fields must be exported (PascalCase)
- All fields must have `mapstructure` tags
- Use typed values (`time.Duration` for timeouts)
- Group related settings into separate structs

---

### **internal/db/** - Database Layer
**Purpose**: Database connection and migrations management  
**Responsibilities**:
- Initialize PostgreSQL connection using `sqlx`
- Manage database migrations with `goose`
- Use embedded migrations via `embed.FS`
- Export `NewPostgres(ctx, config)` function

**Key Rules**:
- Use context for connection initialization
- Execute migrations before returning connection
- Handle all connection and migration errors
- Use `sqlx.DB` (not `sql.DB`)

**Subdirectories**:
- `migrations/` - SQL migration files managed by goose

---

### **internal/entity/** - Domain Entities and DTOs
**Purpose**: Define domain models and data transfer objects  
**Responsibilities**:
- Domain entities (Cart, CartItem) - core business objects
- Request DTOs (AddItemRequest) - incoming data structures
- Response DTOs (ErrorResponse) - outgoing data structures
- JSON and database mapping

**Key Rules**:
- All structs and fields must be exported
- Required tags: `json` for API serialization, `db` for database mapping
- Use `db:"-"` for fields not stored in database
- Use `int64` for IDs and numeric values
- snake_case in JSON and DB tags, PascalCase in struct names

---

### **internal/errorsx/** - Custom Errors
**Purpose**: Application-specific typed errors  
**Responsibilities**:
- Define sentinel errors using `errors.New()`
- Provide typed errors for business logic
- Enable error type checking with `errors.Is()`
- Group errors by logical domains

**Key Rules**:
- Prefix all error variables with `Err`
- Export all error variables
- Lowercase messages without punctuation
- Use descriptive, specific error names
- English language for consistency

---

### **internal/handlers/** - HTTP Handlers Layer
**Purpose**: HTTP request handling and API endpoints  
**Responsibilities**:
- Handle HTTP requests and responses
- **Input validation at handler level** (mandatory!)
- Extract and validate URL parameters
- Decode request bodies
- Create context with timeout
- Call service layer methods
- Handle service errors and map to HTTP status codes
- Serialize and send responses

**Key Rules**:
- **Each business function in a separate file** (create_cart.go, view_cart.go, etc.)
- Validation MUST be at handler level, NOT in service
- Context with timeout before calling service
- Use typed error checking with `errors.Is()`
- Distinguish error types (404 vs 500)
- No business logic in handlers

---

### **internal/repository/** - Data Access Layer
**Purpose**: Database operations and data persistence  
**Responsibilities**:
- **Each business function in a separate file** (create_cart.go, view_cart.go, etc.)
- Execute SQL queries using `sqlx`
- Map database rows to domain entities
- Handle database-specific errors
- Return typed errors from `errorsx`
- Provide CRUD operations

**Key Rules**:
- Context as first parameter in all methods
- Use Context-aware methods (QueryContext, ExecContext)
- Use RETURNING clause to get inserted IDs
- Handle PostgreSQL-specific errors (foreign key violations)
- Check `RowsAffected()` for DELETE/UPDATE
- Use LEFT JOIN for optional relations
- Use COALESCE for NULL handling
- Return predefined errors from errorsx

---

### **internal/service/** - Business Logic Layer
**Purpose**: Application business logic and orchestration  
**Responsibilities**:
- **Each business function in a separate file** (create_cart.go, view_cart.go, etc.)
- Implement business rules and workflows
- Coordinate repository calls
- Validate business constraints
- Log all errors with descriptive messages
- Return domain entities

**Key Rules**:
- Use `Repositorier` interface for repository dependency
- Define `Servicer` interface for testing
- Context as first parameter in all methods
- Log errors before returning them
- Use dependency injection through constructor
- No technical details (SQL, HTTP) in this layer
- Business-oriented method names

---

## Clean Architecture Flow

```
HTTP Request
    ↓
[handlers] - HTTP layer, validation, context creation
    ↓
[service]  - Business logic, error logging
    ↓
[repository] - Database queries, data mapping
    ↓
PostgreSQL Database
```

**Dependency Direction**: handlers → service → repository → database

**Context Flow**: Created in handlers with timeout → passed through service → used in repository DB calls

**Error Flow**: Repository (DB errors) → Service (logs + returns) → Handlers (maps to HTTP status)

---

## Key Architectural Principles

1. **Clean Architecture**: Clear separation of concerns across layers
2. **Dependency Inversion**: Higher layers depend on interfaces, not implementations
3. **Explicit Dependencies**: All dependencies passed through constructors
4. **Context Propagation**: Context passed through all layers for cancellation and timeouts
5. **Typed Errors**: Use sentinel errors for type-safe error handling
6. **Validation at Boundaries**: Input validation at handler level
7. **Single Responsibility**: Each layer has a specific, well-defined purpose
8. **Testability**: Interfaces enable easy mocking and unit testing

---

## Testing Strategy

- **Repository Tests**: Test database operations with sql.Mock
- **Service Tests**: Test business logic with mocked repository (unit tests)
- **Handler Tests**: Test HTTP handling with mocked service (unit tests)
- Use `mockery` for generating interface mocks
- Tests co-located with implementation files

---

## Technology Stack

- **Language**: Go
- **HTTP Router**: Native `net/http` with ServeMux
- **Database**: PostgreSQL
- **DB Library**: sqlx (SQL extensions for Go)
- **Migrations**: goose with embedded SQL files
- **Configuration**: viper with YAML
- **Testing**: Native `testing` package + mockery
- **Containerization**: Docker + Docker Compose

