# InventHold Codebase Analysis Report

**Date**: August 21, 2026  
**Repository**: https://github.com/93GUS18/InventHold  
**Current State**: Project initialization phase (specification defined, source code not yet implemented)

---

## Executive Summary

InventHold is a **microservice-based inventory management system** designed to handle temporary holds on inventory items during checkout processes. The project specification is comprehensive and well-defined in `.github/copilot-instructions.md`. All source code must be implemented according to this specification using Domain-Driven Design (DDD) architecture.

---

## Project Scope & Architecture

### Core Domain
- **Purpose**: Place temporary holds on inventory items with configurable expiration (default: 15 minutes)
- **Key Operations**: Create hold, retrieve hold details, release hold, list inventory
- **Concurrency Model**: Atomic operations to prevent race conditions on stock deduction

### Technology Stack

#### Backend
- **.NET 10** (C#) - Primary runtime
- **MongoDB** - Primary data store for inventory and holds
- **Redis** - Caching layer for high-frequency reads
- **RabbitMQ** - Event broker for domain event publishing
- **ASP.NET Core** - Web API framework

#### Frontend
- **React 18+** with TypeScript (Vite)
- **API Client**: TypeScript-based HTTP client
- **UI Components**: Dashboard, Forms, Active Holds List with countdown

#### DevOps
- **Docker & Docker Compose** - Container orchestration
- **Multi-stage builds** for both backend and frontend
- **Service networking** on shared user-defined bridge network

### Expected Directory Structure
```
InventHold/
├── docker/
│   ├── docker-compose.yml         # Orchestrates 5 services
│   ├── db.env                     # MongoDB connection config
│   └── cache.env                  # Redis config
├── src/
│   ├── InventoryHold.Contracts/   # Shared DTOs, Enums, Models
│   ├── InventoryHold.Domain/      # Business logic, Services, Repositories
│   ├── InventoryHold.Infrastructure/ # MongoDB, Redis, RabbitMQ implementations
│   ├── InventoryHold.WebApi/      # ASP.NET Core API, Controllers
│   └── InventoryHold.UnitTests/   # xUnit or nUnit tests
└── web/                           # React + Vite frontend
	├── src/
	│   ├── api/                   # API client
	│   ├── components/            # React components
	│   ├── types/                 # TypeScript interfaces
	│   └── ...
	└── vite.config.ts

---

## Architectural Patterns & Constraints

### 1. Domain-Driven Design (DDD)
- **Contracts Layer** (`Contracts/`): Shared DTOs, enums, and models across all layers
- **Domain Layer** (`Domain/`): Core business logic, repository interfaces, domain services
- **Infrastructure Layer** (`Infrastructure/`): Implementations of repositories, cache, messaging
- **API Layer** (`WebApi/`): Controllers, dependency injection, startup configuration

### 2. MongoDB & Concurrency Safety
**CRITICAL INVARIANT**: Stock deduction and hold placement must use **atomic operations**
```csharp
// Pattern to use: FindOneAndUpdateAsync with filters
var result = await collection.FindOneAndUpdateAsync(
	filter: Builders<Inventory>.Filter.And(
		Builders<Inventory>.Filter.Eq(i => i.ProductId, productId),
		Builders<Inventory>.Filter.Gte(i => i.Stock, holdQuantity)
	),
	update: Builders<Inventory>.Update.Inc(i => i.Stock, -holdQuantity),
	options: new FindOneAndUpdateOptions<Inventory> { ReturnDocument = ReturnDocument.After }
);

// Fail if no document matched (stock insufficient or product not found)
if (result == null) throw new InsufficientStockException();
```

### 3. Caching Strategy (Redis)
- **Cache Keys**: Use consistent naming (e.g., `inventory:all`, `hold:{holdId}`)
- **TTL**: Explicit expiration for all cached data
- **Eviction**: Immediate cache invalidation on mutations (POST, DELETE)
- **Pattern**: Cache-aside (check cache → miss → fetch → populate cache)

### 4. Event-Driven Architecture (RabbitMQ)
**Domain Events to Publish**:
- `HoldCreated`: When hold is successfully created
- `HoldReleased`: When hold is manually released or expired
- `HoldExpired`: When hold's TTL expires
- `InventoryRestored`: When inventory is restored after hold release

**Event Payload Structure**:
```csharp
public class DomainEvent
{
	public string EventId { get; set; }
	public string EventType { get; set; }
	public DateTime Timestamp { get; set; }
	public object Payload { get; set; }
}
```

### 5. API Contract (HTTP Status Codes & Responses)
| Endpoint | Method | Behavior | Status Codes |
|----------|--------|----------|--------------|
| `/api/holds` | POST | Create hold with stock verification | 201, 400, 409, 422 |
| `/api/holds/{holdId}` | GET | Retrieve hold details (handles expired) | 200, 404 |
| `/api/holds/{holdId}` | DELETE | Release hold & restore stock | 200, 404, 409 |
| `/api/inventory` | GET | List all inventory (cached) | 200 |

**Status Code Usage**:
- `200 OK`: Successful read or mutation
- `201 Created`: Resource created (POST /api/holds)
- `400 Bad Request`: Invalid input
- `404 Not Found`: Resource not found
- `409 Conflict`: Business logic violation (e.g., insufficient stock, hold already released)
- `422 Unprocessable Entity`: Validation failed

---

## Testing Requirements

### Framework & Tools
- **Test Framework**: xUnit or nUnit
- **Mocking**: Moq or NSubstitute
- **Assertions**: FluentAssertions

### Test Coverage Requirements
**Minimum 5 functional unit tests** covering:

1. **Hold Creation**
   - ✅ Valid hold creation with sufficient stock
   - ❌ Hold creation with insufficient stock (409)
   - ❌ Hold creation for non-existent product (404)

2. **Hold Lifecycle**
   - ✅ Retrieve existing hold
   - ❌ Retrieve non-existent hold (404)
   - ✅ Retrieve expired hold (check state)

3. **Race Conditions**
   - ✅ Concurrent hold attempts on same inventory (only one succeeds)
   - ✅ Concurrent release and hold creation (no data corruption)

4. **Edge Cases**
   - ✅ Release non-existent hold (idempotent behavior)
   - ✅ Release already-released hold (409 or 200 based on idempotency choice)
   - ✅ TTL expiration on holds

### Test Isolation
- **No live infrastructure**: Mock MongoDB, Redis, and RabbitMQ dependencies
- **Use in-memory collections** for testing repository logic
- **Fake message broker** for testing event publishing

---

## Deployment & Local Development

### Docker Compose Services
1. **mongodb**: Port 27017, persistent volume `mongodb-data`
2. **redis**: Port 6379, explicit memory configuration
3. **rabbitmq**: Port 5672 (AMQP), 15672 (Management UI)
4. **backend**: Port 5000/5001, depends on all three infrastructure services
5. **frontend**: Port 80/3000, proxies `/api` to backend

### Health Checks
All infrastructure services must implement health checks for startup verification.

### Local Development Workflow
```bash
# Full stack (all services)
docker-compose up --build

# Backend only
dotnet build
dotnet run --project src/InventoryHold.WebApi/

# Frontend only
cd web && npm run dev

# Tests
dotnet test src/InventoryHold.UnitTests/

# Tear down
docker-compose down -v  # -v removes volumes
```

---

## Critical Implementation Notes

### 1. Configuration Management
- **No hardcoded credentials** - All connection strings from environment variables
- **Environment Variables Required**:
  - `MONGODB_CONNECTION_STRING`
  - `REDIS_CONNECTION_STRING`
  - `RABBITMQ_CONNECTION_STRING`
  - `HOLD_EXPIRATION_MINUTES` (default: 15)

### 2. Startup Seeding
- Automatically seed **at least 5 products** with stock levels on application startup
- Products should have varied stock levels for testing (0, 1, 10, 100, 1000)

### 3. Frontend-Backend Integration
- **API Base URL**: Configurable via environment variable or hardcoded during build
- **CORS**: Enable CORS in ASP.NET Core for frontend requests
- **Instant UI sync**: No page refresh on mutations; update local state immediately and show optimistic updates

### 4. Hold Expiration Strategy
- **TTL-based**: MongoDB TTL index on `ExpiryTime` field automatically removes expired holds
- **Background check**: Optional background service to publish `HoldExpired` events
- **Lazy evaluation**: On GET, check if hold has expired and return appropriate state

---

## Development Checklist

### Phase 1: Project Setup
- [ ] Create project structure (all .csproj files)
- [ ] Configure Docker Compose with all 5 services
- [ ] Setup appsettings.json with environment variable bindings
- [ ] Configure dependency injection (Program.cs)
- [ ] Seed database on startup

### Phase 2: Domain & Infrastructure Layer
- [ ] Define DTOs and Contracts
- [ ] Implement MongoDB repository with atomic operations
- [ ] Implement Redis cache layer
- [ ] Implement RabbitMQ event publisher
- [ ] Create domain models and business logic

### Phase 3: API Layer
- [ ] Implement HoldsController with all 4 endpoints
- [ ] Add input validation and error handling
- [ ] Implement proper HTTP status codes
- [ ] Add authorization/authentication (if required)

### Phase 4: Frontend Layer
- [ ] Setup React + Vite project
- [ ] Create API client (instances of inventoryApi.ts)
- [ ] Build UI components (Dashboard, Form, List)
- [ ] Integrate countdown timer for expiring holds
- [ ] Implement optimistic UI updates

### Phase 5: Testing & Hardening
- [ ] Write unit tests (minimum 5, covering race conditions)
- [ ] Integration tests (optional)
- [ ] Load testing for concurrency scenarios
- [ ] Documentation updates

---

## Key Files to Monitor

| File | Purpose | Priority |
|------|---------|----------|
| `docker-compose.yml` | Service orchestration | 🔴 Critical |
| `src/InventoryHold.Domain/` | Business logic | 🔴 Critical |
| `src/InventoryHold.Infrastructure/` | Data & messaging | 🔴 Critical |
| `src/InventoryHold.WebApi/Program.cs` | DI & startup | 🔴 Critical |
| `src/InventoryHold.UnitTests/` | Test coverage | 🟡 Important |
| `web/src/api/inventoryApi.ts` | Frontend API layer | 🟡 Important |

---

## Risks & Considerations

| Risk | Mitigation |
|------|-----------|
| Race conditions on concurrent stock operations | Use MongoDB atomic operations (`FindOneAndUpdateAsync`) |
| Cache invalidation complexity | Clear cache keys immediately on mutations; use short explicit TTLs |
| Event loss in RabbitMQ | Implement durable queues and acknowledge patterns |
| Frontend state synchronization | Optimistic updates + server confirmation; refresh on errors |
| MongoDB connection pooling | Configure connection pool size for expected load |
| TTL index cleanup overhead | Test TTL index performance with anticipated data volume |

---

## AI Agent Workflow Guidance

1. **Before implementing any feature**: Run analysis of specification against current codebase
2. **Code generation**: Follow DDD pattern exactly (Contracts → Domain → Infrastructure → API)
3. **Testing first**: Write test expectations before implementing feature logic
4. **Concurrency focus**: Extra scrutiny on stock operations; validate atomic operations
5. **Integration points**: Verify Docker Compose networking and environment variable propagation
6. **Frontend-backend sync**: Ensure API contracts match TypeScript types in `web/src/types/`

---

## References & Documentation Links

- [DDD in .NET](https://docs.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/)
- [MongoDB .NET Driver](https://www.mongodb.com/docs/drivers/csharp/)
- [Redis .NET Client](https://stackexchange.github.io/StackExchange.Redis/)
- [RabbitMQ .NET](https://www.rabbitmq.com/dotnet-api-guide.html)
- [ASP.NET Core Dependency Injection](https://docs.microsoft.com/en-us/dotnet/core/extensions/dependency-injection)
- [React + TypeScript Best Practices](https://react-typescript-cheatsheet.netlify.app/)

---

**Analysis Complete** - Ready for implementation phase.
