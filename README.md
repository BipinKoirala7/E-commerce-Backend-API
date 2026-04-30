# E-Commerce Platform - Backend Project Overview

## Project Structure
This is a **microservices-based e-commerce platform** built with **Spring Boot 4.0.1** and **Spring Cloud 2025.1.0** using Java 25. The architecture follows the **service mesh pattern** with centralized configuration, service discovery, and API gateway.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        API Gateway (8080)                       │
│  - Handles authentication, filtering, rate limiting             │
│  - Routes requests to microservices                             │
│  - JWT token validation & refresh token management              │
│  - CORS configuration                                           │
└───────────────┬────────────────────────────────────────────────-┘
                │
    ┌───────────┼────────────┬──────────────┬──────────────┐
    │           │            │              │              │
    ▼           ▼            ▼              ▼              ▼
┌─────────┐ ┌────────┐ ┌──────────┐ ┌─────────────┐ ┌──────────────┐
│  Eureka │ │ Config │ │   User   │ │  Product    │ │   Order      │
│ Server  │ │ Server │ │ Service  │ │  Service    │ │  Service     │
│ (8761)  │ │ (8888) │ │ (8081)   │ │  (8082)     │ │  (8083)      │
└─────────┘ └────────┘ └──────────┘ └─────────────┘ └──────────────┘
                │           │            │              │
                └───────────┼────────────┴──────────────┘
                            │
                    ┌───────┴───────┬────────────────┐
                    ▼               ▼                ▼
                ┌─────────────┐ ┌──────────────┐ ┌────────────────┐
                │    Cart     │ │ Notification │ │   Databases    │
                │   Service   │ │   Service    │ │  (PostgreSQL)  │
                │   (8084)    │ │   (8085)     │ │                │
                └─────────────┘ └──────────────┘ │  - Elasticsearch
                                                  │  - RabbitMQ
                                                  └────────────────┘
```

---

## Services Overview

### 1. **API Gateway** (Port: 8080)
**Framework:** Spring Cloud Gateway MVC (Non-reactive)  
**Role:** Entry point for all client requests

**Key Features:**
- JWT authentication and authorization
- Token validation with fallback to refresh token from cookies
- Route traffic to microservices with load balancing
- CORS configuration (configurable origins, methods, headers)
- Cookie management (secure, HTTPOnly)
- Gateway secret for internal service communication
- Request header forwarding (Authorization, X-Gateway-Secret)

**Routes:**
```
/api/v1/user/**     → User Service
/api/v1/auth/**     → User Service (Authentication)
/api/v1/oauth2/**   → User Service (OAuth2)
/api/v1/product/**  → Product Service
/api/v1/cart/**     → Cart Service
/api/v1/cart-item/**  → Cart Service
/api/v1/wishlist/** → Cart Service
/api/v1/order/**    → Order Service
/api/v1/payment/**  → Order Service
```

**Configuration:** `CloudConfig/api-gateway.yaml`

---

### 2. **User Service** (Port: 8081)
**Database:** PostgreSQL (shared: `ecommerce`)  
**Role:** User authentication, registration, and OAuth2 integration

**Key Features:**
- JWT token generation (access & refresh tokens)
- OAuth2 integration (Google login)
- User registration and authentication
- Cookie management for token storage
- AMQP/RabbitMQ integration for event publishing
- Tracing with Brave/Zipkin

**Dependencies:**
- Spring Data JPA
- PostgreSQL driver
- Micrometer Tracing Brave
- JWT (JJWT 0.12.3)
- OAuth2 Client
- MapStruct (DTO mapping)

**Key Models:**
- User (with OAuth2 provider info)

**Configuration:** `CloudConfig/user-service.yaml`

---

### 3. **Product Service** (Port: 8082)
**Database:** PostgreSQL (shared: `ecommerce`)  
**Role:** Product catalog management and search

**Key Features:**
- Product CRUD operations
- Category management
- Elasticsearch integration for full-text search
- Product specifications and attributes
- Internal APIs for other services (via gateway secret)
- OpenTelemetry tracing with OTLP metrics
- Source authentication (service-to-service verification)

**Dependencies:**
- Spring Data JPA
- Spring Data Elasticsearch
- PostgreSQL driver
- OpenTelemetry
- Micrometer Prometheus

**Key Models:**
- Product (with images, categories, specifications)
- Category
- ProductSpecification

**Controllers:**
- `ProductController` - Public endpoints
- `InternalController` - Internal service endpoints

**Configuration:** `CloudConfig/product-service.yaml`

---

### 4. **Order Service** (Port: 8083)
**Database:** PostgreSQL (shared: `ecommerce`)  
**Role:** Order management and payment processing

**Key Features:**
- Order creation and management
- Payment processing with Stripe integration
- Order status tracking
- Stripe webhook handling
- JWT validation for service-to-service calls
- Snowflake ID generation for distributed IDs
- Feign client for Product Service communication
- Source authentication (inter-service)

**Dependencies:**
- Spring Data JPA
- Stripe Java SDK (v32.0.0)
- hutool-core (Chinese utility library)
- JWT (JJWT 0.12.3)
- MapStruct

**Key Models:**
- Order (with order items)
- OrderItem
- Payment (with PaymentMethod & PaymentStatus)
- OrderStatus enum

**Key Services:**
- `OrderService` - Order business logic
- `PaymentService` - Payment processing
- `StripeService` - Stripe integration
- `JwtService` - Token validation

**Stripe Configuration:**
```yaml
stripe:
  secret-key: ${STRIPE_SECRET_KEY}
  publishable-key: ${STRIPE_PUBLISHABLE_KEY}
  webhook-secret: ${STRIPE_WEBHOOK_SECRET}
```

**Configuration:** `CloudConfig/order-service.yaml`

---

### 5. **Cart Service** (Port: 8084)
**Database:** PostgreSQL (shared: `ecommerce`)  
**Role:** Shopping cart and wishlist management

**Key Features:**
- Cart item management (add, update, remove)
- Wishlist management
- Product information caching from Product Service
- JWT token validation
- OpenTelemetry tracing
- Source authentication for inter-service calls

**Dependencies:**
- Spring Data JPA
- Spring Data Elasticsearch (implicit)
- PostgreSQL driver
- OpenTelemetry
- JWT (JJWT 0.12.3)
- MapStruct

**Key Models:**
- CartItem (userId, productId, quantity)
- Wishlist (userId, productId)

**Key Services:**
- `CartItemService` - Cart operations
- `WishlistService` - Wishlist operations
- `JwtService` - Token validation

**Feign Client:**
- `ProductServiceClient` - Fetch product details for cart items

**Configuration:** `CloudConfig/cart-service.yaml`

---

### 7. **Eureka Server** (Port: 8761)
**Framework:** Spring Cloud Netflix Eureka  
**Role:** Service registry and discovery

**Key Features:**
- Service registration and deregistration
- Health check monitoring
- Client-side service discovery
- Prometheus metrics export
- OpenTelemetry integration

**Configuration:** `CloudConfig/eureka-server.yaml`

---

### 8. **Config Server** (Port: 8888)
**Framework:** Spring Cloud Config Server  
**Role:** Centralized configuration management

**Key Features:**
- Centralized YAML configuration for all services
- Dynamic configuration refresh
- Prometheus metrics export

**Configuration Files:**
```
CloudConfig/
├── application.yaml (default config for all services)
├── api-gateway.yaml
├── eureka-server.yaml
├── user-service.yaml
├── product-service.yaml
├── order-service.yaml
├── cart-service.yaml
├── notification-service.yaml
```

**Configuration:** `CloudConfig/application.yaml`

---

## Infrastructure & Dependencies

### Database
- **PostgreSQL** - Shared database for all services
- **Host:** `${DB_HOST:localhost}`
- **Port:** `${DB_PORT:5432}`
- **Database:** `ecommerce`
- **Timezone:** UTC (configured via connection-init-sql)

### Messaging
- **RabbitMQ** - Message broker (used by User Service for AMQP)

### Search & Analytics
- **Elasticsearch** - Full-text search (Product Service)
- **Prometheus** - Metrics collection
- **Zipkin** - Distributed tracing (available at `http://localhost:9411`)
- **OpenTelemetry (OTLP)** - Observability (metrics & traces at `http://localhost:4318`)

### Payment
- **Stripe** - Payment processing and webhooks

### External Services
- **Google OAuth2** - Social login integration

---

## Common Patterns & Utilities

### JWT & Authentication
- **JWT Library:** JJWT 0.12.3 (all services)
- **Token Types:**
  - Access Token: 3600 seconds (1 hour)
  - Refresh Token: 259200 seconds (3 days)
- **Storage:** Cookies (secure, HTTPOnly)
- **Claims:** `issuer` & `audience` configurable per environment

**Token Configuration:**
```yaml
app:
  jwt:
    issuer: api.ecommerce.com
    audience: ecommerce.com
    accessTokenExpiration: 3600
    refreshTokenExpiration: 259200
```

### Security
- **Service-to-Service Auth:** `X-Gateway-Secret` header
- **User-to-Service Auth:** Bearer token in `Authorization` header
- **CORS:** Configurable per service
- **Custom Filters:**
  - `JwtFilter` - JWT validation
  - `SourceAuthenticationFilter` - Service identity verification
  - `FilterExceptionHandler` - Exception handling in filters

### Data Mapping
- **MapStruct** v1.5.5.Final - DTO mapping (UserService, OrderService, CartService, ProductService)

### Lombok
- **Version:** Latest - Used for boilerplate reduction (getters, setters, constructors)

### Exception Handling
- **Custom Exceptions:**
  - `InValidTokenException` - Invalid JWT token
  - `EmptyTokenException` - Missing token
  - `TokenAuthenticationException` - Token authentication failure
  - `UnVerifiedSourceException` - Unverified service source
  - Service-specific exceptions (OrderNotFound, ProductNotFoundException, CartItemNotFound, etc.)

- **Global Exception Handler:** Each service has `GlobalExceptionHandler` for unified error responses

### ID Generation
- **Snowflake ID:** Used in Order Service for distributed ID generation

### Feign Clients
- **Product Service Clients:** Used by Order Service and Cart Service
- **User Service Clients:** Used by API Gateway
- **Configuration:** Centralized timeout settings (10s connection, 45s read)

---

## Build & Compilation

### Build Tool
- **Maven** - All services use Maven for build management
- **Parent POM:** Spring Boot 4.0.1

### Java Version
- **Java 25** - All services

### Compiler Plugins
- **Annotation Processing:**
  - Lombok
  - Spring Boot Configuration Processor
  - MapStruct Processor
  - Lombok-MapStruct Binding

---

## Configuration Management

### Environment Variables
All sensitive configurations use environment variables with defaults:

```yaml
# JWT
APP_GATEWAY_SECRET=${APP_GATEWAY_SECRET}
APP_SERVICE_SECRET=${APP_SERVICE_SECRET}

# Database
DB_HOST=localhost
DB_PORT=5432
DB_NAME=ecommerce
DB_USERNAME=postgres
DB_PASSWORD=postgres

# Stripe
STRIPE_SECRET_KEY=${STRIPE_SECRET_KEY}
STRIPE_PUBLISHABLE_KEY=${STRIPE_PUBLISHABLE_KEY}
STRIPE_WEBHOOK_SECRET=${STRIPE_WEBHOOK_SECRET}

# Google OAuth2
GOOGLE_CLIENT_ID=${GOOGLE_CLIENT_ID}
GOOGLE_CLIENT_SECRET=${GOOGLE_CLIENT_SECRET}
GOOGLE_REDIRECT_URI=${GOOGLE_REDIRECT_URI}

# Mail (Notification Service)
SPRING_MAIL_HOST=${SPRING_MAIL_HOST}
SPRING_MAIL_PORT=${SPRING_MAIL_PORT}
SPRING_MAIL_USERNAME=${SPRING_MAIL_USERNAME}
SPRING_MAIL_PASSWORD=${SPRING_MAIL_PASSWORD}

# Tracing
ZIPKIN_ENDPOINT=http://localhost:9411/v2/spans
ZIPKIN_ENABLE=true
MANAGEMENT_OPEN_TELEMETRY_TRACING_ENDPOINT=http://localhost:4318/v1/traces
```

---

## Observability & Monitoring

### Metrics
- **Prometheus** - All services expose metrics at `/actuator/prometheus`
- **Micrometer Registry** - Metrics collection framework

### Tracing
- **Brave + Zipkin** - Distributed tracing for request tracking
- **OTLP (OpenTelemetry)** - Modern observability protocol
- **Tracing Sampling:** 100% (all traces collected)

### Health Checks
- **Spring Boot Actuator** - All services expose `/actuator/health`
- **Endpoints Exposed:** `health`, `info`, `prometheus`, `metrics`

---

## CORS Configuration

### Default Settings
```yaml
cors:
  enabled: true
  allowed-origins: http://localhost:4000 (Gateway)
  allowed-origin-patterns: http://localhost:* (Services)
  allowed-methods: GET, POST, PUT, DELETE, OPTIONS
  allowed-headers: *
  exposed-headers: Authorization, Access-Control-Allow-Origin, Access-Control-Allow-Credentials
  allow-credentials: true
  max-age: 3600
```

---

## Docker & Deployment Notes

### Service Ports Summary
| Service | Port | Database | Auth Method |
|---------|------|----------|-------------|
| API Gateway | 8080 | N/A | JWT |
| User Service | 8081 | PostgreSQL | JWT |
| Product Service | 8082 | PostgreSQL | Gateway Secret + JWT |
| Order Service | 8083 | PostgreSQL | JWT |
| Cart Service | 8084 | PostgreSQL | JWT |
| Notification Service | 8085 | N/A | JWT |
| Eureka Server | 8761 | N/A | N/A |
| Config Server | 8888 | N/A | N/A |

### External Dependencies
- PostgreSQL 12+ (shared database)
- RabbitMQ 3.8+ (messaging)
- Elasticsearch 7+ (search)
- Redis (optional, for caching)
- Zipkin (tracing, optional)

---

## File Structure Reference

```
Backend/
├── CloudConfig/                    # Centralized configuration files
│   ├── application.yaml            # Base config (JWT, Feign, Logging, Eureka)
│   ├── api-gateway.yaml
│   ├── eureka-server.yaml
│   ├── user-service.yaml
│   ├── product-service.yaml
│   ├── order-service.yaml
│   ├── cart-service.yaml
│   └── notification-service.yaml
│
├── APIGateway/                     # Spring Cloud Gateway
│   ├── src/main/java/.../
│   │   ├── Config/
│   │   │   ├── GatewayRoutesConfig.java
│   │   │   ├── SecurityConfig.java
│   │   │   ├── CorsConfig.java
│   │   │   └── FeignConfig.java
│   │   ├── Filters/ (JWT, Exception Handling)
│   │   ├── Security/ (JWT, Custom Auth)
│   │   └── Client/ (UserServiceClient via Feign)
│   └── pom.xml
│
├── UserService/                    # User & Auth Service
│   ├── src/main/java/.../
│   │   ├── Service/ (UserService, AuthService, JwtService, CookieService)
│   │   ├── Security/
│   │   └── Controller/
│   └── pom.xml
│
├── ProductService/                 # Product Catalog & Search
│   ├── src/main/java/.../
│   │   ├── Service/
│   │   ├── Repository/
│   │   ├── Model/ (Product, Category, ProductSpecification)
│   │   ├── Controller/
│   │   └── Mapper/
│   └── pom.xml
│
├── OrderService/                   # Order & Payment Processing
│   ├── src/main/java/.../
│   │   ├── Service/ (OrderService, PaymentService, StripeService, JwtService)
│   │   ├── Model/ (Order, OrderItem, Payment, OrderStatus, PaymentMethod, PaymentStatus)
│   │   ├── Controller/
│   │   ├── Repository/
│   │   ├── Config/ (StripeConfig, SnowflakeIdConfig, SecurityConfig)
│   │   ├── Client/ (ProductServiceClient)
│   │   └── Filters/ (JWT, SourceAuthentication)
│   └── pom.xml
│
├── CartService/                    # Cart & Wishlist Management
│   ├── src/main/java/.../
│   │   ├── Service/ (CartItemService, WishlistService, JwtService)
│   │   ├── Model/ (CartItem, Wishlist)
│   │   ├── Controller/ (CartItemController, WishlistController)
│   │   ├── Repository/
│   │   ├── Mapper/
│   │   ├── Client/ (ProductServiceClient)
│   │   └── Filters/ (JWT, SourceAuthentication)
│   └── pom.xml
│
├── ConfigServer/                   # Spring Cloud Config Server
│   ├── src/main/java/.../
│   └── pom.xml
│
├── EurekaServer/                   # Service Discovery
│   ├── src/main/java/.../
│   └── pom.xml
│
└── README.md                       # Project documentation
```

---

## Key Technologies Stack

| Category | Technology | Version |
|----------|-----------|---------|
| Framework | Spring Boot | 4.0.1 |
| Cloud | Spring Cloud | 2025.1.0 |
| Language | Java | 25 |
| Database | PostgreSQL | 12+ |
| Build Tool | Maven | 3.6+ |
| JWT | JJWT | 0.12.3 |
| Mapping | MapStruct | 1.5.5.Final |
| ORM | Hibernate/JPA | Via Spring Data |
| Payment | Stripe | 32.0.0 |
| Search | Elasticsearch | 7+ |
| Messaging | RabbitMQ | 3.8+ |
| Tracing | Brave/Zipkin | Spring Boot integrated |
| Observability | OpenTelemetry | Spring Boot integrated |
| Metrics | Prometheus | Spring Boot integrated |
| Annotations | Lombok | Latest |
| Discovery | Netflix Eureka | Spring Cloud integrated |

---

## Startup Order (Recommended)

1. **PostgreSQL** - Database
2. **RabbitMQ** - Message broker
3. **Elasticsearch** (Optional) - Search
4. **Eureka Server** - Service registry
5. **Config Server** - Configuration provider
6. **User Service** - Authentication service
7. **Product Service** - Product catalog
8. **Order Service** - Order processing
9. **Cart Service** - Shopping cart
10. **API Gateway** - Entry point
11. **Notification Service** - Email notifications
12. **Zipkin** (Optional) - Distributed tracing

---

## Development Notes

- **Lombok:** Reduces boilerplate code with annotations
- **MapStruct:** Type-safe DTO mapping with compile-time code generation
- **Feign Clients:** Declarative HTTP clients for inter-service communication
- **OpenTelemetry:** Modern observability platform replacing traditional tracing
- **Snowflake ID:** Distributed ID generation without database dependency
- **Stripe Integration:** Handles payment processing with webhook support

---

## Summary

This is a **production-ready microservices architecture** with:
- Centralized configuration management
- Service-to-service communication via Feign
- JWT-based authentication & authorization
- OAuth2 social login support
- Payment processing with Stripe
- Full-text search with Elasticsearch
- Distributed tracing & observability
- Health checks & metrics collection
- CORS configuration
- Comprehensive error handling

The architecture is scalable, resilient, and follows microservices best practices.

