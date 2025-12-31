# BYO DPP Developer Onboarding Handbook

## Table of Contents

### Quick Start
1. [Introduction](#introduction)
2. [Getting Started](#getting-started)
3. [Quick Reference Guide](#quick-reference-guide)
4. [Developer Onboarding Guide](#developer-onboarding-guide)
   - [Day 1: Getting Started](#day-1-getting-started)
   - [30-60-90 Day Learning Path](#30-60-90-day-learning-path)
   - [Your First Feature Walkthrough](#your-first-feature-walkthrough)
   - [Common Pitfalls & How to Avoid Them](#common-pitfalls--how-to-avoid-them)

### Architecture & Design
5. [Architecture Overview](#architecture-overview)
6. [Architecture Understanding A2Z: Complete Guide with Actual Code References](#architecture-understanding-a2z-complete-guide-with-actual-code-references)
   - [Application Bootstrap and Initialization Flow](#1-application-bootstrap-and-initialization-flow)
   - [Dynamic Resource Registration Architecture](#2-dynamic-resource-registration-architecture)
   - [Provider Pattern Implementation](#3-provider-pattern-implementation-with-actual-code)
   - [Request Flow with Actual Code References](#4-request-flow-with-actual-code-references)
   - [Validation Schema System](#5-validation-schema-system-with-actual-examples)
   - [Logic Map System](#6-logic-map-system-custom-business-logic-registration)
   - [Error Handling Flow](#22-error-handling-flow-with-code-references)
   - [Repository Discovery Pattern](#23-repository-discovery-pattern-explained)
   - [Complete Dependency Tree](#40-understanding-the-full-dependency-tree)
   - [Debugging Architecture](#35-debugging-architecture-with-code-references)
   - [Development Workflow](#34-complete-feature-implementation-checklist)
7. [Domain Driven Design Alignment](#domain-driven-design-alignment)
8. [Module Lifecycle Management](#module-lifecycle-management)

### Implementation Patterns
9. [Event Orchestration and Pipeline Patterns](#event-orchestration-and-pipeline-patterns)
10. [Error Handling and Retry Architecture](#error-handling-and-retry-architecture)
11. [Security Best Practices](#security-best-practices)
12. [API → Service → Domain → Infra Implementation Walk-through](#api--service--domain--infra-implementation-walk-through)

### Operations
13. [Deployment Architecture and Runtime Concerns](#deployment-architecture-and-runtime-concerns)
14. [Troubleshooting Handbook](#troubleshooting-handbook)

### Reference
15. [Code Samples](#code-samples)
16. [Diagrams](#diagrams)
17. [Tables and References](#tables-and-references)
18. [Glossary](#glossary)

### Multi-Tenancy Architecture
19. [Multi-Tenancy Implementation: Complete Architecture & Planning Guide](#-multi-tenancy-implementation-complete-architecture--planning-guide)
    - [Overview & Current Implementation](#overview--current-implementation)
    - [Multi-Tenancy Patterns Analysis](#multi-tenancy-patterns-analysis)
    - [BYO DPP Implementation Deep Dive](#byo-dpp-implementation-deep-dive)
    - [Data Isolation Strategy](#data-isolation-strategy)
    - [Security & Authorization Architecture](#security--authorization-architecture)
    - [Tenant Lifecycle Management](#tenant-lifecycle-management)
    - [Scaling Strategy](#scaling-strategy)
    - [Migration & Evolution Roadmap](#migration--evolution-roadmap)
    - [Performance Optimization](#performance-optimization)
    - [Monitoring & Observability](#monitoring--observability)
    - [Implementation Checklist](#implementation-checklist)

## Deep Repository Analysis

The **BYO DPP** codebase follows a **Clean Architecture** / **Domain‑Driven Design** (DDD) approach, separating concerns into three concentric layers:

```
┌─────────────────────┐
│   Application Layer │   – HTTP adapters, routing, controllers, middleware
├─────────────────────┤
│        Domain       │   – Business logic, entities, value objects, domain services
├─────────────────────┤
│   Infrastructure    │   – Database (Drizzle ORM), external services, adapters
└─────────────────────┘
```

### Core Directories

| Directory | Primary Responsibility | Key Patterns |
|-----------|------------------------|--------------|
| `src/bootstrap.ts` | Application start‑up, DI container wiring | **Provider**, **Factory**, **Singleton** |
| `src/application/` | HTTP entry points, request/response handling | **Controller**, **Middleware (Chain of Responsibility)** |
| `src/core/` | Domain models, validation schemas, functional utilities | **Entity**, **Value Object**, **Aggregate**, **Domain Service**, **Specification** |
| `src/infra/` | Persistence, external integrations, cross‑cutting concerns | **Repository**, **Adapter**, **Gateway**, **Adapter**, **Decorator** |
| `scripts/` | Code generation, DB seeding, build helpers | **CLI**, **Builder** |

### Frequently Used Patterns

| Pattern | Where It Appears | Why It Matters |
|---------|------------------|----------------|
| **Dependency Injection (DI)** | `src/bootstrap.ts`, providers in `src/application/providers/` | Decouples concrete implementations, enables easy testing and swapping of components (e.g., switching from PostgreSQL to MongoDB). |
| **Repository** | `src/infra/adapters/repository/drizzle-orm.repository.ts` | Abstracts data access, isolates ORM specifics from domain logic. |
| **Service Layer** | `src/application/adapters/service/base-resource.service.ts` | Coordinates use‑cases, injects business logic (logic map) and repositories. |
| **Adapter** | `src/infra/adapters/event-stream/sse-event-stream.adapter.ts`, `src/infra/adapters/mailer/mailer.adapter.ts` | Allows the application to communicate with external systems (SSE, email, blockchain) without leaking implementation details. |
| **Mediator / Event Orchestrator** | `src/infra/adapters/event-stream/event-orchestrator.ts` | Central hub for publishing domain events, enabling loose coupling between producers and consumers. |
| **Saga (Compensating Transactions)** | `src/application/logics/v1/dpps/pipeline/dpp-saga.logic.ts` (conceptual) | Guarantees eventual consistency across distributed resources. |
| **Circuit Breaker & Retry** | `src/infra/services/retry.service.ts` | Improves resilience when calling flaky external services. |
| **Unit of Work** | Transaction handling in `DrizzleORMRepository` | Guarantees atomicity for multi‑resource operations. |
| **Aspect‑Oriented Logging** | Decorators in `src/infra/logging/logger.ts` | Keeps business code clean while providing structured logs. |

### Execution Flow (Typical Request)

1. **HTTP Request** hits the **ExpressAdapter** (`src/infra/adapters/express/express.adapter.ts`).  
2. Global **middleware** (security, rate‑limiting, auth) processes the request.  
3. **Router** (`src/application/adapters/router/api.router.ts`) resolves the route and instantiates the appropriate **Controller** (`BaseController`).  
4. The **Controller** delegates to the **Service** (`BaseService`).  
5. **Service** looks up a **Logic** class from the **logic map** (e.g., `DppsCreateLogic`). If none, it falls back to the **Repository**.  
6. **Logic** executes domain rules, may publish **Domain Events** via the **Event Orchestrator**.  
7. **Repository** persists data using **Drizzle ORM**.  
8. **ResponseBuilder** formats a consistent API response.  

Understanding this flow helps developers locate where to add new features, modify validation, or replace a data store without touching unrelated layers.

### Quick Navigation for New Contributors

| Concern | Entry Point | Typical Files |
|---------|-------------|---------------|
| Adding a new **resource** (e.g., `Invoices`) | `src/bootstrap.ts` → `buildRoutes()` | `src/core/schemas/resource/invoices/*`, `src/infra/schemas/drizzle-pg-orm/byo-dpp/invoices.schema.ts`, `src/application/adapters/controller/base-resource.controller.ts` |
| Extending **business logic** | Logic map (`logicMap` in `src/bootstrap.ts`) | `src/core/logics/v1/<resource>/crud/<resource>-create.logic.ts` |
| Changing **validation** | Joi/Zod schemas | `src/core/schemas/resource/<resource>/post/create-<resource>.schema.ts` |
| Updating **database schema** | Drizzle migrations | `migrations/drizzle/*.sql` |
| Custom **event handling** | Event Orchestrator registration | `src/infra/adapters/event-stream/event-orchestrator.ts` |
| Adding **caching** | Cache manager | `src/infra/cache/cache-manager.ts` |
| Modifying **security** | Security middleware | `src/infra/middlewares/security.middleware.ts` |

---

*The above analysis is now part of the handbook to give developers a high‑level map of the codebase, making onboarding faster and more intuitive.*


13. [Troubleshooting Handbook](#troubleshooting-handbook)

## Introduction

Welcome to the BYO DPP (Build Your Own Data Processing Pipeline) Backend Developer Onboarding Handbook. This comprehensive guide is designed to help new developers understand the architecture, patterns, and implementation details of our production-grade backend system.

### What is BYO DPP?

BYO DPP is a flexible, modular backend framework for building data processing pipelines. It follows Domain-Driven Design (DDD) principles and provides a clean architecture for handling complex business logic across multiple domains.

**Framework Purpose**: BYO DPP solves the problem of building scalable, maintainable backend systems for data processing applications. Traditional monolithic architectures become unwieldy as business logic grows, leading to tight coupling, difficult testing, and maintenance challenges. BYO DPP provides a structured approach to separate concerns while maintaining flexibility.

**Key Problems Solved**:
- **Scalability**: Modular architecture allows independent scaling of components
- **Maintainability**: Clear separation of concerns makes code easier to understand and modify
- **Testability**: Dependency injection and interface-based design enable comprehensive testing
- **Flexibility**: Dynamic resource registration allows runtime configuration changes
- **Consistency**: Standardized patterns ensure predictable behavior across the application

### Who Should Read This Handbook?

- **New Backend Developers**: Start with [Getting Started](#getting-started) and [Quick Reference Guide](#quick-reference-guide)
- **Architects**: Focus on [Architecture Overview](#architecture-overview) and [Domain Driven Design](#domain-driven-design-alignment)
- **DevOps Engineers**: See [Deployment Architecture](#deployment-architecture-and-runtime-concerns) and [Troubleshooting](#troubleshooting-handbook)
- **Full-Stack Developers**: Review [API Implementation Walk-through](#api--service--domain--infra-implementation-walk-through)

## Quick Reference Guide

### Essential Commands

```bash
# Development
npm run dev                    # Start development server with hot reload
npm run build                  # Build for production
npm start                      # Run production server
npm test                       # Run tests
npm run lint                   # Check code quality

# Database
npm run db:push                # Push schema changes
npm run db:migrate             # Run migrations
npm run db:seed                # Seed database

# Docker
docker-compose up -d           # Start all services
docker-compose down            # Stop all services
docker-compose logs -f app     # View application logs
```

### Architecture at a Glance

```mermaid
flowchart TD
    Client[Client Request]
    
    subgraph Application["APPLICATION LAYER"]
        Express[Express Adapter]
        Middleware[Middleware Pipeline]
        Controllers[Controllers]
        Routers[Dynamic Routers]
    end
    
    subgraph Domain["DOMAIN LAYER"]
        Logic[Business Logic]
        Services[Services]
        Entities[Entities & Value Objects]
        Events[Domain Events]
    end
    
    subgraph Infrastructure["INFRASTRUCTURE LAYER"]
        Repositories[Repositories]
        ORM[Drizzle ORM]
        External[External Services]
        SSE[Event Streaming]
    end
    
    subgraph Persistence["DATABASE & EXTERNAL SERVICES"]
        DB[(PostgreSQL)]
        Email[Email Service]
        Storage[File Storage]
        Blockchain[Blockchain]
    end
    
    Client --> Express
    Express --> Middleware
    Middleware --> Controllers
    Controllers --> Routers
    Routers --> Logic
    Logic --> Services
    Services --> Entities
    Services --> Events
    Logic --> Repositories
    Repositories --> ORM
    ORM --> DB
    External --> Email
    External --> Storage
    External --> Blockchain
    SSE --> Client
    
    style Application fill:#e1f5ff
    style Domain fill:#fff3e0
    style Infrastructure fill:#f3e5f5
    style Persistence fill:#e8f5e9
```

### Key Design Patterns

| Pattern | Purpose | Where Used |
|---------|---------|------------|
| **Dependency Injection** | Loose coupling | Throughout, especially [`bootstrap.ts`](src/bootstrap.ts:1) |
| **Repository** | Data access abstraction | [`drizzle-orm.repository.ts`](src/infra/adapters/repository/drizzle-orm.repository.ts:1) |
| **Service Layer** | Business orchestration | [`base-resource.service.ts`](src/application/adapters/service/base-resource.service.ts:1) |
| **Adapter** | External integration | [`express.adapter.ts`](src/infra/adapters/express/express.adapter.ts:1) |
| **Observer** | Event-driven updates | [`sse-event-stream.adapter.ts`](src/infra/adapters/event-stream/sse-event-stream.adapter.ts:1) |
| **Factory** | Object creation | [`error.factory.ts`](src/infra/adapters/errors/error.factory.ts:1) |

### Common Tasks Quick Links

| Task | Where to Start |
|------|----------------|
| Add new resource | [`bootstrap.ts:buildRoutes()`](src/bootstrap.ts:638) |
| Create business logic | [`src/core/logics/v1/`](src/core/logics/v1/) |
| Add validation | [`src/core/schemas/resource/`](src/core/schemas/resource/) |
| Update database schema | [`migrations/drizzle/`](migrations/drizzle/) |
| Handle events | [`event-orchestrator.ts`](src/infra/adapters/event-stream/event-orchestrator.ts:1) |
| Configure security | [`security.middleware.ts`](src/infra/middlewares/security.middleware.ts:1) |

### Environment Variables Cheat Sheet

| Variable | Purpose | Required |
|----------|---------|----------|
| `POSTGRES_DB_URI` | Database connection | ✅ Yes |
| `KEYCLOAK_HOST_URI` | Auth server URL | ✅ Yes |
| `PORT` | Server port | ⚪ No (default: 3000) |
| `NODE_ENV` | Environment | ⚪ No (default: dev) |
| `REDIS_URL` | Cache/session store | ⚪ Optional |
| `LOG_LEVEL` | Logging verbosity | ⚪ No (default: info) |

### Request Flow Summary

```mermaid
sequenceDiagram
    participant Client
    participant Express as Express Adapter
    participant Middleware as Middleware Pipeline
    participant Router
    participant Controller
    participant Service
    participant Logic as Logic Class
    participant Repository
    participant DB as PostgreSQL
    
    Client->>Express: 1. HTTP Request
    Express->>Middleware: 2. Forward to Pipeline
    
    rect rgb(240, 248, 255)
        Note over Middleware: Authentication & Validation
        Middleware->>Middleware: Verify JWT Token
        Middleware->>Middleware: Validate Schema
    end
    
    Middleware->>Router: 3. Route Resolution
    Router->>Controller: 4. HTTP Handling
    Controller->>Service: 5. Delegate to Service
    
    alt Custom Logic Exists
        Service->>Logic: 6a. Execute Business Rules
        Logic->>Repository: 7. Data Persistence
    else No Custom Logic
        Service->>Repository: 6b. Direct Repository Call
    end
    
    Repository->>DB: SQL Query
    DB-->>Repository: Data Result
    Repository-->>Service: Repository Result
    Service-->>Controller: Service Response
    Controller-->>Express: 8. Formatted Output
    Express-->>Client: HTTP Response
```

## Developer Onboarding Guide

### Day 1: Getting Started

Welcome! This guide will help you get productive on your first day.

#### Morning (2-3 hours)

**1. Environment Setup** (60 min)
```bash
# Clone the repository
git clone https://github.com/your-org/byo-dpp-backend.git
cd byo-dpp-backend

# Install Node.js v18+ (check with node --version)
# Install Docker Desktop

# Set up environment
cp .env.dev.example .env
# Open .env and update values (ask your team lead for credentials)

# Install dependencies
npm ci

# Start services
docker-compose up -d
```

**2. Verify Setup** (30 min)
```bash
# Check services are running
docker ps
# Should see: byo-dpp-db, byo-dpp-redis

# Start dev server
npm run dev

# In another terminal, test health endpoint
curl http://localhost:3000/health
# Expected: {"status":"healthy","uptime":...}
```

**3. Explore the Codebase** (60 min)
Read these files in order:
1. [`README.md`](README.md:1) - Project overview
2. [`src/bootstrap.ts`](src/bootstrap.ts:1) - Application entry point
3. [`src/application/adapters/controller/base-resource.controller.ts`](src/application/adapters/controller/base-resource.controller.ts:1) - How controllers work
4. [`src/core/logics/v1/`](src/core/logics/v1/) - Browse logic examples

#### Afternoon (3-4 hours)

**4. Run Your First API Request** (30 min)

Using Postman or curl:
```bash
# Get all products
curl http://localhost:3000/api/v1/products

# Create a product (requires auth token from Keycloak)
curl -X POST http://localhost:3000/api/v1/products \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name":"Test Product","categoryId":1}'
```

**5. Make a Small Change** (90 min)
Try this simple exercise:
- Find the products list endpoint in [`base-resource.controller.ts`](src/application/adapters/controller/base-resource.controller.ts:1)
- Add a console.log to see when it's called
- Restart the server and test the endpoint
- Observe your log in the console

**6. Review Architecture** (60 min)
- Read [Architecture Overview](#architecture-overview)
- Study the [Request Flow Summary](#request-flow-summary)
- Understand how a request flows through layers

**7. Meet the Team** (30 min)
- Schedule 30-minute intro calls with team members
- Ask about coding standards and review process
- Get access to project management tools

### 30-60-90 Day Learning Path

#### Days 1-30: Foundation

**Week 1: Setup & Understanding**
- ✅ Complete Day 1 setup
- ✅ Read this handbook through "Architecture Overview"
- ✅ Understand the layered architecture
- ✅ Review existing PRs to understand code style
- 🎯 **Milestone**: Successfully run and test the application locally

**Week 2: Simple Features**
- ✅ Implement a simple CRUD endpoint (see [Your First Feature](#your-first-feature-walkthrough))
- ✅ Write tests for your feature
- ✅ Submit your first PR
- 🎯 **Milestone**: First PR merged

**Week 3: Business Logic**
- ✅ Study logic classes in [`src/core/logics/v1/`](src/core/logics/v1/)
- ✅ Implement custom business logic
- ✅ Understand validation schemas
- 🎯 **Milestone**: Create a logic class with business rules

**Week 4: Integration**
- ✅ Work on a feature requiring multiple resources
- ✅ Implement error handling
- ✅ Add proper logging
- 🎯 **Milestone**: Complete feature with cross-resource logic

#### Days 31-60: Intermediate

**Week 5-6: Advanced Patterns**
- ✅ Implement a saga pattern for distributed transactions
- ✅ Create a new domain event and handler
- ✅ Work with SSE for real-time updates
- 🎯 **Milestone**: Implement event-driven feature

**Week 7-8: Performance & Security**
- ✅ Add caching to a frequently accessed endpoint
- ✅ Implement rate limiting for a resource
- ✅ Review and apply security best practices
- 🎯 **Milestone**: Optimize a slow endpoint

#### Days 61-90: Advanced

**Week 9-10: System Design**
- ✅ Design a new bounded context
- ✅ Create comprehensive documentation
- ✅ Lead a feature from design to deployment
- 🎯 **Milestone**: Own a feature end-to-end

**Week 11-12: Leadership**
- ✅ Review other developers' PRs
- ✅ Mentor a new developer
- ✅ Contribute to architecture decisions
- 🎯 **Milestone**: Become a code reviewer

### Your First Feature Walkthrough

Let's build a simple feature: **Add a "Featured" flag to products**.

#### Step 1: Understand the Requirements

**User Story**: As a product manager, I want to mark products as "featured" so they appear prominently on the homepage.

**Acceptance Criteria**:
- Products can be marked as featured (boolean field)
- Only admins can change the featured status
- Featured products can be filtered in the list endpoint

#### Step 2: Database Schema Change

**Create a migration**:
```bash
# Generate migration file
npm run db:generate
```

Add to the migration file:
```sql
-- Add featured column to products table
ALTER TABLE products ADD COLUMN is_featured BOOLEAN DEFAULT FALSE;

-- Add index for filtering
CREATE INDEX idx_products_is_featured ON products(is_featured);
```

Apply migration:
```bash
npm run db:migrate
```

#### Step 3: Update the Schema Definition

Edit [`src/infra/schemas/drizzle-pg-orm/byo-dpp/products.schema.ts`](src/infra/schemas/drizzle-pg-orm/byo-dpp/products.schema.ts:1):

```typescript
export const products = pgTable('products', {
    id: serial('id').primaryKey(),
    name: varchar('name', { length: 255 }).notNull(),
    // ... existing fields
    isFeatured: boolean('is_featured').default(false).notNull(), // ADD THIS
    createdAt: timestamp('created_at').defaultNow().notNull(),
    updatedAt: timestamp('updated_at').defaultNow().notNull(),
});
```

#### Step 4: Update Validation Schema

Edit [`src/core/schemas/resource/products/post/create-products.schema.ts`](src/core/schemas/resource/products/post/create-products.schema.ts:1):

```typescript
export const createProductsSchema = Joi.object({
    name: Joi.string().required().min(1).max(255),
    description: Joi.string().allow('').max(1000),
    categoryId: Joi.number().integer().positive().required(),
    isEnabled: Joi.boolean().default(true),
    isFeatured: Joi.boolean().default(false), // ADD THIS
    metadata: Joi.object().default({})
});
```

#### Step 5: Create Business Logic (Optional)

Create [`src/core/logics/v1/products/action/products-feature-post.logic.ts`](src/core/logics/v1/products/action/products-feature-post.logic.ts:1):

```typescript
import { IRequest } from 'src/core/interfaces';
import { ILogic } from 'src/core/interfaces/logic.interface';
import { ICoreRepository } from 'src/core/interfaces/repository.interface';
import { CoreProvider } from 'src/core/providers/core.provider';

export class ProductsFeatureLogic implements ILogic {
    request: IRequest;
    repository: ICoreRepository;
    coreProvider: CoreProvider;

    constructor(request: IRequest, repository: ICoreRepository) {
        this.request = request;
        this.repository = repository;
        this.coreProvider = CoreProvider.getInstance();
    }

    async execute() {
        const { productId } = this.request.params;
        const { isFeatured } = this.request.body;

        // Business rule: Only admins can feature products
        if (!this.request.user?.roles.includes('admin')) {
            throw this.coreProvider.getErrorFactory().create(
                'ForbiddenError',
                'Only admins can feature products'
            );
        }

        // Business rule: Limit featured products to 10
        if (isFeatured) {
            const featuredCount = await this.repository.list({
                filter: { isFeatured: { eq: true } }
            });

            if (featuredCount.count >= 10) {
                throw this.coreProvider.getErrorFactory().create(
                    'BadRequestError',
                    'Maximum of 10 featured products reached'
                );
            }
        }

        // Update product
        return await this.repository.update(
            { filter: { id: { eq: productId } } },
            { isFeatured }
        );
    }
}
```

#### Step 6: Register the Logic

Edit [`src/application/mapper/logic-map.ts`](src/application/mapper/logic-map.ts:1):

```typescript
import { ProductsFeatureLogic } from 'src/core/logics/v1/products/action/products-feature-post.logic';

export const logicMap = {
    'v1': {
        'products': {
            'action': {
                'feature:post': ProductsFeatureLogic, // ADD THIS
            },
            'crud': {
                // existing CRUD operations
            }
        }
    }
};
```

#### Step 7: Test Your Feature

**Create a test file** [`src/core/logics/v1/products/__tests__/products-feature.test.ts`](src/core/logics/v1/products/__tests__/products-feature.test.ts:1):

```typescript
import { ProductsFeatureLogic } from '../action/products-feature-post.logic';

describe('ProductsFeatureLogic', () => {
    it('should allow admin to feature product', async () => {
        // Arrange
        const mockRequest = {
            params: { productId: '1' },
            body: { isFeatured: true },
            user: { roles: ['admin'] }
        };
        const mockRepository = {
            list: jest.fn().mockResolvedValue({ count: 5 }),
            update: jest.fn().mockResolvedValue({ success: true })
        };

        // Act
        const logic = new ProductsFeatureLogic(mockRequest, mockRepository);
        const result = await logic.execute();

        // Assert
        expect(result.success).toBe(true);
        expect(mockRepository.update).toHaveBeenCalledWith(
            { filter: { id: { eq: '1' } } },
            { isFeatured: true }
        );
    });

    it('should reject non-admin users', async () => {
        // Arrange
        const mockRequest = {
            params: { productId: '1' },
            body: { isFeatured: true },
            user: { roles: ['user'] }
        };

        // Act & Assert
        const logic = new ProductsFeatureLogic(mockRequest, mockRepository);
        await expect(logic.execute()).rejects.toThrow('Only admins can feature products');
    });
});
```

Run tests:
```bash
npm test
```

#### Step 8: Manual Testing

```bash
# Start the server
npm run dev

# Feature a product (as admin)
curl -X POST http://localhost:3000/api/v1/products/1/feature \
  -H "Authorization: Bearer ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"isFeatured": true}'

# List featured products
curl "http://localhost:3000/api/v1/products?filter[isFeatured][eq]=true"
```

#### Step 9: Submit a Pull Request

```bash
# Create feature branch
git checkout -b feature/add-featured-products

# Commit changes
git add .
git commit -m "feat: add featured flag to products

- Add is_featured column to products table
- Add validation for featured field
- Implement admin-only feature logic
- Add tests for feature logic
- Limit featured products to 10 maximum"

# Push and create PR
git push origin feature/add-featured-products
```

**PR Description Template**:
```markdown
## What?
Adds ability to mark products as "featured" for homepage display.

## Why?
Product managers need to highlight specific products on the homepage.

## How?
- Added `is_featured` boolean column to products table
- Created business logic to enforce admin-only access
- Limited featured products to 10 maximum
- Added comprehensive tests

## Testing
- [ ] Unit tests pass
- [ ] Manual testing completed
- [ ] Admin can feature products
- [ ] Non-admins are rejected
- [ ] Featured products appear in filtered list

## Screenshots
[Add screenshots if UI is involved]
```

### Common Pitfalls & How to Avoid Them

#### 1. **Forgetting to Update Schemas**

❌ **Wrong**:
```typescript
// Only updating database schema, forgetting validation
export const products = pgTable('products', {
    newField: varchar('new_field') // Added to DB
});
// Validation schema not updated!
```

✅ **Right**:
```typescript
// 1. Update database schema
export const products = pgTable('products', {
    newField: varchar('new_field')
});

// 2. Update validation schema
export const createProductsSchema = Joi.object({
    newField: Joi.string().max(255) // Add here too!
});
```

#### 2. **Not Handling Errors Properly**

❌ **Wrong**:
```typescript
async execute() {
    const product = await this.repository.read({ id: this.request.params.id });
    return product.data; // Crashes if product not found!
}
```

✅ **Right**:
```typescript
async execute() {
    const product = await this.repository.read({ id: this.request.params.id });
    
    if (!product.success) {
        throw this.errorFactory.create(
            'NotFoundError',
            `Product with ID ${this.request.params.id} not found`
        );
    }
    
    return product.data;
}
```

#### 3. **Skipping Authorization Checks**

❌ **Wrong**:
```typescript
async execute() {
    // Directly modifying data without permission check
    return await this.repository.update(data);
}
```

✅ **Right**:
```typescript
async execute() {
    // Check user permissions first
    if (!this.request.user?.roles.includes('admin')) {
        throw this.errorFactory.create(
            'ForbiddenError',
            'Insufficient permissions'
        );
    }
    
    return await this.repository.update(data);
}
```

#### 4. **Not Using Transactions for Multi-Step Operations**

❌ **Wrong**:
```typescript
async execute() {
    await this.productRepo.create(product);
    await this.categoryRepo.update(category); // If this fails, product is orphaned!
}
```

✅ **Right**:
```typescript
async execute() {
    return await this.repository.multiResourceTransact([
        { operation: 'create', resourceName: 'Products', data: product },
        { operation: 'update', resourceName: 'Categories', data: category }
    ]);
}
```

#### 5. **Hardcoding Values Instead of Using Config**

❌ **Wrong**:
```typescript
const apiUrl = 'http://localhost:3000'; // Hardcoded!
```

✅ **Right**:
```typescript
const config = AppConfig.getInstance().getConfig();
const apiUrl = config.API_URL;
```

#### 6. **Not Reading Existing Code First**

❌ **Wrong Approach**:
```
See task → Start coding immediately → Duplicate existing functionality
```

✅ **Right Approach**:
```
See task → Search codebase for similar features → Read existing implementation → Adapt pattern → Code
```

Use this search pattern:
```bash
# Find similar implementations
grep -r "similar-pattern" src/

# Or use the search_files tool to find patterns
```

#### 7. **Inconsistent Naming Conventions**

❌ **Wrong**:
```typescript
// Mixing conventions
class product_manager { } // snake_case
function GetProduct() { } // PascalCase
const user-data = {}; // kebab-case
```

✅ **Right**:
```typescript
// Follow project conventions
class ProductManager { } // PascalCase for classes
function getProduct() { } // camelCase for functions
const userData = {}; // camelCase for variables
```

#### 8. **Not Testing Edge Cases**

❌ **Wrong**:
```typescript
it('should create product', async () => {
    const result = await createProduct({ name: 'Test' });
    expect(result.success).toBe(true);
    // Only happy path tested!
});
```

✅ **Right**:
```typescript
describe('createProduct', () => {
    it('should create valid product', async () => { /* ... */ });
    
    it('should reject empty name', async () => { /* ... */ });
    
    it('should reject duplicate name', async () => { /* ... */ });
    
    it('should handle database errors', async () => { /* ... */ });
});
```

### Quick Wins for New Developers

These tasks are great for your first few PRs:

1. **Add a New Validation Rule** (1-2 hours)
   - Find a resource in [`src/core/schemas/resource/`](src/core/schemas/resource/)
   - Add a new validation rule (e.g., email format, min/max length)
   - Write tests

2. **Improve Error Messages** (2-3 hours)
   - Find generic error messages
   - Make them more descriptive
   - Add context about what went wrong

3. **Add Logging** (2-3 hours)
   - Find a complex logic class
   - Add structured logging at key points
   - Use the logger from CoreProvider

4. **Write Missing Tests** (3-4 hours)
   - Find logic classes without tests
   - Write comprehensive test coverage
   - Aim for edge cases

5. **Improve Documentation** (1-2 hours)
   - Add JSDoc comments to complex functions
   - Update README with missing information
   - Add code examples

### Learning Resources

#### Internal Resources
- **Architecture Decision Records**: `/docs/adr/` (if exists)
- **Team Wiki**: [Link to your team wiki]
- **Code Review Guidelines**: [Link to guidelines]

#### External Resources
- **Domain-Driven Design**: "Domain-Driven Design" by Eric Evans
- **Clean Architecture**: "Clean Architecture" by Robert C. Martin
- **TypeScript**: [TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/intro.html)
- **Drizzle ORM**: [Official Documentation](https://orm.drizzle.team/)
- **Design Patterns**: [Refactoring Guru](https://refactoring.guru/design-patterns)

#### Video Tutorials
- Clean Architecture with TypeScript
- Domain-Driven Design fundamentals
- Node.js best practices

### Getting Help

#### When You're Stuck

1. **Search the Codebase First**
   ```bash
   # Find similar patterns
   grep -r "pattern-you-need" src/
   
   # Search for specific error
   grep -r "error-message" src/
   ```

2. **Check the Handbook**
   - Use Ctrl+F to search this handbook
   - Check [Troubleshooting](#troubleshooting-handbook)
   - Review [Common Pitfalls](#common-pitfalls--how-to-avoid-them)

3. **Ask the Team**
   - **Quick Questions**: Team Slack channel
   - **Code Reviews**: Request review from senior developers
   - **Architecture Decisions**: Schedule time with tech lead
   - **Pair Programming**: Great for learning complex features

4. **Debug Systematically**
   ```typescript
   // Add strategic console.logs
   console.log('Step 1: Input', input);
   console.log('Step 2: After transformation', transformed);
   console.log('Step 3: Before save', toSave);
   ```

#### Office Hours

- **Daily Standup**: 9:30 AM - Quick sync
- **Code Review Sessions**: Tuesday & Thursday 2 PM
- **Architecture Office Hours**: Wednesday 3 PM
- **Pair Programming**: Book anytime

### Success Metrics

By the end of your onboarding, you should be able to:

✅ **Week 1**:
- [ ] Run the application locally
- [ ] Make a simple code change
- [ ] Submit your first PR

✅ **Week 4**:
- [ ] Implement a complete CRUD feature
- [ ] Write comprehensive tests
- [ ] Navigate the codebase confidently

✅ **Week 8**:
- [ ] Implement features with business logic
- [ ] Handle cross-resource operations
- [ ] Review others' code

✅ **Week 12**:
- [ ] Design new features independently
- [ ] Mentor new developers
- [ ] Contribute to architecture decisions

### Onboarding Checklist

Print this out and check off items as you complete them:

#### Setup
- [ ] Environment set up and working
- [ ] Access to all necessary tools (GitHub, Jira, etc.)
- [ ] Connected with the team

#### Knowledge
- [ ] Read Architecture Overview
- [ ] Understand request flow
- [ ] Familiar with key design patterns
- [ ] Know where to find things

#### Practical
- [ ] First PR merged
- [ ] Implemented CRUD feature
- [ ] Created logic class with business rules
- [ ] Written comprehensive tests

#### Integration
- [ ] Participated in code reviews
- [ ] Contributed to architecture discussions
- [ ] Helped another developer

**Welcome to the team! 🚀**

## Getting Started

### System Requirements

- **Node.js** (v18 or newer)  
- **Docker** & **Docker Compose** (for local services)  
- **PostgreSQL** (if running outside Docker)  
- **Redis** (optional, for caching & rate‑limiting)  

### Local Development Setup

1. **Clone the repository**  
   ```bash
   git clone https://github.com/your-org/byo-dpp-backend.git
   cd byo-dpp-backend
   ```

2. **Create an environment file**  
   ```bash
   cp .env.dev.example .env
   # Edit .env to match your local configuration (DB credentials, Keycloak URL, etc.)
   ```

3. **Start supporting services** (PostgreSQL, Redis, Nginx)  
   ```bash
   docker-compose up -d
   ```

4. **Install Node dependencies**  
   ```bash
   npm ci
   ```

5. **Run the development server**  
   ```bash
   npm run dev
   ```
   The API will be available at `http://localhost:3000`.

### Production Build & Run

```bash
# Build the multi‑stage Docker image
docker build -t byo-dpp:latest .

# Run the container (expose port 3000)
docker run -d -p 3000:3000 --env-file .env byo-dpp:latest
```

### Common Commands Cheat‑Sheet

| Command | Description |
|---------|-------------|
| `npm run dev` | Start the hot‑reloading development server |
| `npm run build` | Compile TypeScript sources to `dist/` |
| `npm start` | Run the compiled production server |
| `npm test` | Execute unit and integration tests |
| `npm run lint` | Run ESLint checks |
| `docker-compose up -d` | Spin up DB, Redis, Nginx containers |
| `docker-compose down` | Stop and remove containers |
| `docker logs <container>` | View logs for a specific container |

## Contribution Guide

1. **Fork the repository** and create a feature branch:  
   ```bash
   git checkout -b feature/your-feature
   ```

2. **Write tests** for any new functionality.  
3. **Run the full test suite** to ensure nothing breaks:  
   ```bash
   npm test
   ```

4. **Lint and format** the code:  
   ```bash
   npm run lint
   npm run format-fix
   ```

5. **Commit** with clear messages and **push** to your fork.  
6. Open a **Pull Request** against the `main` branch and reference the relevant issue.

> **Tip:** The project follows a **Clean Architecture** style, so keep new code within the appropriate layer (Application, Domain, or Infrastructure) and register any new services in `src/bootstrap.ts`.

--- 

*The above sections provide a concise, hands‑on guide for new developers to get up and running quickly, as well as clear instructions for contributing back to the project.*



### Key Technologies & Why They're Chosen

| Technology | Purpose | Problems Solved | Design Pattern Benefits |
|------------|---------|-----------------|-------------------------|
| **TypeScript** | Type-safe development | Runtime errors, better IDE support | **Static Typing Pattern**: Compile-time error detection prevents runtime issues |
| **Node.js + Express** | Web framework | Fast development, large ecosystem | **Middleware Pattern**: Request/response pipeline with composable handlers |
| **PostgreSQL + Drizzle ORM** | Data persistence | ACID transactions, complex queries | **Repository Pattern**: Abstract data access, enable testing with in-memory databases |
| **Clean Architecture + DDD** | System organization | Tight coupling, unclear boundaries | **Layered Architecture**: Dependency inversion, testability, maintainability |
| **Keycloak** | Identity management | Authentication complexity | **Single Sign-On Pattern**: Centralized auth, reduced security risks |
| **Server-Sent Events** | Real-time communication | Polling overhead, stale data | **Observer Pattern**: Push-based updates, reduced server load |
| **Joi + Zod** | Data validation | Invalid data, security vulnerabilities | **Validation Pattern**: Input sanitization, clear error messages |
| **Docker** | Containerization | Environment inconsistencies | **Infrastructure as Code**: Reproducible deployments, isolation |

## Architecture Overview

The BYO DPP backend follows a layered architecture pattern with clear separation of concerns:

```
┌─────────────────┐
│   Application   │  ← Controllers, Services, Adapters
├─────────────────┤
│     Domain      │  ← Business Logic, Entities, Value Objects
├─────────────────┤
│ Infrastructure  │  ← Database, External APIs, File Storage
└─────────────────┘
```

### Core Components

1. **Application Layer**: Handles HTTP requests, routing, and external communication
2. **Domain Layer**: Contains business logic, entities, and domain services
3. **Infrastructure Layer**: Manages data persistence, external services, and system concerns

### Key Patterns

- **Dependency Injection**: Using provider patterns for loose coupling
- **Repository Pattern**: Abstracting data access
- **Service Layer**: Orchestrating business operations
- **Adapter Pattern**: Adapting external interfaces
## Architecture Understanding

### System Architecture Philosophy

The BYO DPP architecture is built on several core principles that guide all design decisions:

#### 1. Separation of Concerns Principle

**Philosophy**: Each layer, module, and component should have a single, well-defined responsibility.

```typescript
// Anti-pattern: Mixed concerns
class ProductController {
    async createProduct(req, res) {
        // HTTP handling
        const data = req.body;
        
        // Business validation
        if (!data.name) throw new Error('Name required');
        
        // Database access
        const result = await db.query('INSERT INTO products...');
        
        // Email notification
        await sendEmail('product-created', result);
        
        // Logging
        console.log('Product created:', result);
        
        res.json(result);
    }
}

// Good pattern: Separated concerns
class ProductController {
    constructor(
        private productService: IProductService,
        private logger: ILogger
    ) {}
    
    async createProduct(req: IRequest): Promise<IResponse> {
        try {
            // Only HTTP concerns
            const result = await this.productService.createProduct(req);
            return this.formatResponse(result);
        } catch (error) {
            return this.handleError(error);
        }
    }
}

class ProductService {
    constructor(
        private repository: IProductRepository,
        private validator: IProductValidator,
        private notificationService: INotificationService,
        private logger: ILogger
    ) {}
    
    async createProduct(request: IRequest): Promise<Product> {
        // Only business orchestration
        const validatedData = await this.validator.validate(request.body);
        const product = await this.repository.create(validatedData);
        await this.notificationService.notify('product-created', product);
        this.logger.info('Product created', { productId: product.id });
        return product;
    }
}
```

**Benefits**:
- **Testability**: Each component can be tested in isolation
- **Maintainability**: Changes to one concern don't affect others
- **Reusability**: Components can be used in different contexts
- **Understanding**: Developers can focus on one concern at a time

#### 2. Dependency Inversion Principle

**Philosophy**: High-level modules should not depend on low-level modules. Both should depend on abstractions.

```typescript
// Anti-pattern: Direct dependency on concrete implementation
class ProductService {
    private repository: DrizzleProductRepository; // Concrete dependency
    
    constructor() {
        this.repository = new DrizzleProductRepository(); // Tightly coupled
    }
}

// Good pattern: Dependency on abstraction
interface IProductRepository {
    create(data: CreateProductData): Promise<Product>;
    findById(id: number): Promise<Product | null>;
}

class ProductService {
    constructor(private repository: IProductRepository) {} // Abstraction dependency
    
    async createProduct(data: CreateProductData): Promise<Product> {
        return await this.repository.create(data);
    }
}

// Can swap implementations without changing service
class DrizzleProductRepository implements IProductRepository { /* ... */ }
class MongoProductRepository implements IProductRepository { /* ... */ }
class InMemoryProductRepository implements IProductRepository { /* ... */ }
```

**Benefits**:
- **Flexibility**: Can swap implementations easily
- **Testability**: Can use mock implementations for testing
- **Decoupling**: Business logic independent of infrastructure
- **Evolution**: Can change technology without rewriting business logic

#### 3. Open/Closed Principle

**Philosophy**: Software entities should be open for extension but closed for modification.

```typescript
// Anti-pattern: Modifying existing code for new behavior
class ProductValidator {
    validate(product: Product): boolean {
        if (product.category === 'electronics') {
            return this.validateElectronics(product);
        } else if (product.category === 'clothing') {
            return this.validateClothing(product);
        } else if (product.category === 'food') { // Adding new category requires modification
            return this.validateFood(product);
        }
        return true;
    }
}

// Good pattern: Extension through strategy pattern
interface IValidationStrategy {
    validate(product: Product): boolean;
}

class ProductValidator {
    private strategies: Map<string, IValidationStrategy> = new Map();
    
    registerStrategy(category: string, strategy: IValidationStrategy): void {
        this.strategies.set(category, strategy);
    }
    
    validate(product: Product): boolean {
        const strategy = this.strategies.get(product.category);
        return strategy ? strategy.validate(product) : true;
    }
}

// Add new categories without modifying validator
class ElectronicsValidationStrategy implements IValidationStrategy { /* ... */ }
class ClothingValidationStrategy implements IValidationStrategy { /* ... */ }
class FoodValidationStrategy implements IValidationStrategy { /* ... */ }
```

**Benefits**:
- **Extensibility**: Add new features without changing existing code
- **Stability**: Existing code remains tested and stable
- **Safety**: Less risk of breaking existing functionality
- **Maintainability**: Easier to understand and modify

### Layered Architecture Deep Dive

The BYO DPP system implements a sophisticated layered architecture with clear boundaries and communication patterns.

#### Architecture Layers Visualization

```
┌─────────────────────────────────────────────────────────────┐
│                    PRESENTATION LAYER                        │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐            │
│  │   HTTP     │  │    SSE     │  │  GraphQL   │            │
│  │  Adapter   │  │  Adapter   │  │  (Future)  │            │
│  └────────────┘  └────────────┘  └────────────┘            │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                   APPLICATION LAYER                          │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐            │
│  │Controllers │→ │  Services  │→ │Middlewares │            │
│  └────────────┘  └────────────┘  └────────────┘            │
│         ↓              ↓                ↓                    │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐            │
│  │  Routers   │  │ Validators │  │   DTOs     │            │
│  └────────────┘  └────────────┘  └────────────┘            │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                     DOMAIN LAYER                             │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐            │
│  │  Entities  │  │   Value    │  │Aggregates  │            │
│  │            │  │  Objects   │  │            │            │
│  └────────────┘  └────────────┘  └────────────┘            │
│         ↓              ↓                ↓                    │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐            │
│  │   Logic    │  │   Domain   │  │   Domain   │            │
│  │  Classes   │  │  Services  │  │   Events   │            │
│  └────────────┘  └────────────┘  └────────────┘            │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                 INFRASTRUCTURE LAYER                         │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐            │
│  │Repository  │  │  Database  │  │   Cache    │            │
│  └────────────┘  └────────────┘  └────────────┘            │
│         ↓              ↓                ↓                    │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐            │
│  │   Email    │  │   File     │  │Blockchain  │            │
│  │  Service   │  │  Storage   │  │  Service   │            │
│  └────────────┘  └────────────┘  └────────────┘            │
└─────────────────────────────────────────────────────────────┘
```

#### Layer Communication Rules

**Rule 1: Unidirectional Dependencies**
Dependencies flow downward only. Upper layers can depend on lower layers, but never vice versa.

```typescript
// ✅ CORRECT: Application layer depends on domain layer
class ProductController {
    constructor(private productService: IProductService) {} // Domain interface
}

// ❌ WRONG: Domain layer depending on application layer
class Product {
    constructor(private httpClient: HttpClient) {} // Infrastructure in domain!
}
```

**Rule 2: Abstraction at Boundaries**
Layers communicate through interfaces, not concrete implementations.

```typescript
// Application Layer (Abstraction)
interface IProductRepository {
    create(data: CreateProductData): Promise<Product>;
}

// Domain Layer (Uses abstraction)
class ProductService {
    constructor(private repository: IProductRepository) {}
}

// Infrastructure Layer (Implements abstraction)
class DrizzleProductRepository implements IProductRepository {
    async create(data: CreateProductData): Promise<Product> {
        // Implementation details
    }
}
```

**Rule 3: No Layer Skipping**
Layers should communicate with adjacent layers only.

```typescript
// ❌ WRONG: Controller directly accessing repository (skips service layer)
class ProductController {
    constructor(private repository: IProductRepository) {
        // Skipping service layer
    }
}

// ✅ CORRECT: Controller → Service → Repository
class ProductController {
    constructor(private service: IProductService) {}
}

class ProductService {
    constructor(private repository: IProductRepository) {}
}
```

### Understanding the Architecture Through Real Examples

Let's trace a complete request through the system to understand how all pieces work together:

#### Example: Creating a Product with Complex Validation

**Step 1: HTTP Request Arrives**
```typescript
POST /api/v1/products
{
    "name": "Wireless Headphones",
    "categoryId": 5,
    "price": 99.99,
    "metadata": {
        "warranty": "2 years",
        "color": "black"
    }
}
```

**Step 2: Express Adapter (Presentation Layer)**
```typescript
// Converts HTTP to domain request
class ExpressAdapter {
    setupRoute(path: string, controller: IController) {
        this.app.post(path, async (req, res) => {
            const domainRequest: IRequest = {
                method: req.method,
                path: req.path,
                body: req.body,
                query: req.query,
                params: req.params,
                user: req.user,
                headers: req.headers
            };
            
            const response = await controller.create(domainRequest);
            res.status(response.success ? 201 : response.error.statusCode)
                .json(response);
        });
    }
}
```

**Step 3: Middleware Pipeline (Application Layer)**
```typescript
// Security Middleware
class SecurityMiddleware {
    execute(request: IRequest): IRequest {
        // Rate limiting check
        if (this.isRateLimited(request)) {
            throw new TooManyRequestsError();
        }
        return request;
    }
}

// Authentication Middleware
class AuthMiddleware {
    async execute(request: IRequest): Promise<IRequest> {
        const token = request.headers.authorization;
        request.user = await this.identityService.verifyToken(token);
        return request;
    }
}

// Validation Middleware
class ValidationMiddleware {
    async execute(request: IRequest): Promise<IRequest> {
        const { error, value } = await this.validator.validate(request.body);
        if (error) throw new ValidationError(error.message);
        request.body = value;
        return request;
    }
}
```

**Step 4: Controller (Application Layer)**
```typescript
class ProductController {
    async create(request: IRequest): Promise<IResponse> {
        try {
            // Delegate to service layer
            const result = await this.productService.create(request);
            return {
                success: true,
                data: result.data,
                message: 'Product created successfully'
            };
        } catch (error) {
            return this.handleError(error);
        }
    }
}
```

**Step 5: Service Layer (Application Layer)**
```typescript
class ProductService {
    async create(request: IRequest): Promise<any> {
        // Check if custom logic exists
        const LogicClass = this.logicMap.get('create');
        
        if (LogicClass) {
            // Use custom business logic
            const logic = new LogicClass(request, this.repository);
            return await logic.execute();
        }
        
        // Fallback to default repository create
        return await this.repository.create(request.body);
    }
}
```

**Step 6: Domain Logic (Domain Layer)**
```typescript
class ProductCreateLogic {
    async execute() {
        // Business Rule 1: Validate category exists and is active
        await this.validateCategory();
        
        // Business Rule 2: Check for duplicate product names in category
        await this.checkDuplicateName();
        
        // Business Rule 3: Validate price range for category
        await this.validatePriceRange();
        
        // Business Rule 4: Enrich metadata with defaults
        const enrichedData = this.enrichMetadata(this.request.body);
        
        // Create product
        const product = await this.repository.create(enrichedData);
        
        // Business Rule 5: Create audit log
        await this.createAuditLog(product);
        
        // Business Rule 6: Publish domain event
        await this.publishProductCreatedEvent(product);
        
        return product;
    }
    
    private async validateCategory() {
        const category = await this.categoryRepo.findById(
            this.request.body.categoryId
        );
        
        if (!category) {
            throw new BadRequestError('Category does not exist');
        }
        
        if (!category.isActive) {
            throw new BadRequestError('Category is not active');
        }
    }
}
```

**Step 7: Repository (Infrastructure Layer)**
```typescript
class DrizzleProductRepository {
    async create(data: CreateProductData): Promise<Product> {
        try {
            const result = await this.db
                .insert(products)
                .values({
                    name: data.name,
                    categoryId: data.categoryId,
                    price: data.price,
                    metadata: data.metadata,
                    isEnabled: true,
                    createdAt: new Date(),
                    updatedAt: new Date()
                })
                .returning();
                
            return { success: true, data: result[0] };
        } catch (error) {
            return this.handleDatabaseError(error);
        }
    }
}
```

**Step 8: Response Journey Back**
```typescript
// Repository → Logic → Service → Controller → Adapter → Client
{
    "success": true,
    "data": {
        "id": 123,
        "name": "Wireless Headphones",
        "categoryId": 5,
        "price": 99.99,
        "metadata": {
            "warranty": "2 years",
            "color": "black",
            "sku": "WH-2024-001" // Added by enrichMetadata
        },
        "isEnabled": true,
        "createdAt": "2024-01-15T10:30:00Z",
        "updatedAt": "2024-01-15T10:30:00Z"
    },
    "message": "Product created successfully"
}
```

### Key Architectural Insights

#### Insight 1: Flexibility Through Abstraction

The architecture allows you to:

1. **Change databases** without touching business logic
```typescript
// Before: Using Drizzle with PostgreSQL
const repository = new DrizzleProductRepository(pgDatabase);

// After: Switching to MongoDB
const repository = new MongoProductRepository(mongoDatabase);

// Business logic unchanged!
class ProductService {
    constructor(private repository: IProductRepository) {} // Still the same
}
```

2. **Add new features** without modifying existing code
```typescript
// Add new validation without changing ProductCreateLogic
class ProductCreateLogicV2 extends ProductCreateLogic {
    async execute() {
        await super.execute(); // Reuse existing logic
        await this.validateInventoryAvailability(); // Add new rule
    }
}

// Register new version in logic map
logicMap.set('create', ProductCreateLogicV2);
```

3. **Test in isolation** with mock implementations
```typescript
// Unit test with mocks
const mockRepository = {
    create: jest.fn().mockResolvedValue({ id: 1, name: 'Test' })
};

const logic = new ProductCreateLogic(request, mockRepository);
const result = await logic.execute();

expect(mockRepository.create).toHaveBeenCalledWith(expectedData);
```

#### Insight 2: Progressive Complexity

Start simple, add complexity only when needed:

```typescript
// Level 1: Simple CRUD (no custom logic)
// Service directly calls repository
await repository.create(data);

// Level 2: Add validation
class ProductCreateLogic {
    async execute() {
        this.validate(this.request.body);
        return await this.repository.create(this.request.body);
    }
}

// Level 3: Add business rules
class ProductCreateLogic {
    async execute() {
        await this.validateCategory();
        await this.checkDuplicate();
        return await this.repository.create(this.request.body);
    }
}

// Level 4: Add event sourcing
class ProductCreateLogic {
    async execute() {
        await this.validateCategory();
        const product = await this.repository.create(this.request.body);
        await this.eventStore.append(new ProductCreatedEvent(product));
        return product;
    }
}
```

#### Insight 3: Error Handling Strategy

Errors are handled at appropriate layers:

```typescript
// Infrastructure Layer: Technical errors
class DrizzleRepository {
    async create(data: any) {
        try {
            return await this.db.insert(this.table).values(data);
        } catch (error) {
            // Convert database errors to domain errors
            if (error.code === '23505') {
                throw new ConflictError('Duplicate entry');
            }
            throw new DatabaseError('Database operation failed');
        }
    }
}

// Domain Layer: Business errors
class ProductCreateLogic {
    async execute() {
        const category = await this.categoryRepo.findById(this.request.body.categoryId);
        if (!category) {
            throw new BadRequestError('Category not found'); // Business error
        }
    }
}

// Application Layer: HTTP errors
class ProductController {
    async create(request: IRequest) {
        try {
            return await this.service.create(request);
        } catch (error) {
            // Convert domain errors to HTTP responses
            if (error instanceof BadRequestError) {
                return { success: false, error: { statusCode: 400, message: error.message } };
            }
            return { success: false, error: { statusCode: 500, message: 'Internal error' } };
        }
    }
}
```

### Architecture Decision Records

#### ADR-001: Why Repository Pattern Over Direct ORM Access?

**Status**: Accepted

**Context**: We need to access data from PostgreSQL using Drizzle ORM.

**Decision**: Wrap Drizzle ORM behind a repository interface.

**Consequences**:

**Positive**:
- Can switch ORMs without changing business logic
- Can mock data access for testing
- Can add caching transparently
- Can standardize error handling

**Negative**:
- Additional abstraction layer
- Slightly more code to write
- Minor performance overhead

**Example**:
```typescript
// With Repository Pattern
interface IProductRepository {
    create(data: CreateProductData): Promise<Product>;
}

class DrizzleProductRepository implements IProductRepository {
    async create(data: CreateProductData): Promise<Product> {
        return await this.db.insert(products).values(data).returning();
    }
}

// Service depends on interface
class ProductService {
    constructor(private repository: IProductRepository) {}
}

// Can easily switch implementations
const repository = usePostgres 
    ? new DrizzleProductRepository(pgDb)
    : new MongoProductRepository(mongoDb);
```

#### ADR-002: Why Logic Classes Instead of Service Methods?

**Status**: Accepted

**Context**: Need a way to add custom business logic without modifying core service classes.

**Decision**: Use separate logic classes that can be dynamically registered.

**Consequences**:

**Positive**:
- Can add/modify business logic without touching service classes
- Each logic class has single responsibility
- Easy to test logic in isolation
- Can version logic classes (V1, V2, etc.)

**Negative**:
- More files to manage
- Needs logic registration mechanism
- Slightly more complex flow

**Example**:
```typescript
// Logic class for custom business rules
class ProductCreateLogic implements ILogic {
    async execute() {
        // Custom validation
        await this.validateProductRules();
        
        // Custom processing
        const enrichedData = this.enrichProductData(this.request.body);
        
        // Standard create
        return await this.repository.create(enrichedData);
    }
}

// Service remains generic
class ProductService {
    async create(request: IRequest) {
        const LogicClass = this.logicMap.get('create');
        if (LogicClass) {
            const logic = new LogicClass(request, this.repository);
            return await logic.execute();
        }
        return await this.repository.create(request.body);
    }
}
```

#### ADR-003: Why Middleware Pipeline Over Decorators?

**Status**: Accepted

**Context**: Need to handle cross-cutting concerns like authentication, validation, logging.

**Decision**: Use middleware pipeline pattern instead of decorators.

**Consequences**:

**Positive**:
- More explicit and visible in code
- Easier to understand request flow
- Can dynamically compose middleware
- Better TypeScript support

**Negative**:
- More verbose than decorators
- Order matters (must be explicit)

**Example**:
```typescript
// Middleware pipeline
const middlewares = [
    new SecurityMiddleware(),
    new AuthenticationMiddleware(),
    new ValidationMiddleware(schema),
    new LoggingMiddleware()
];

// Execute in order
for (const middleware of middlewares) {
    request = await middleware.execute(request);
}
```


### Advanced Testing Strategies

#### Testing Pyramid Implementation

The BYO DPP follows a comprehensive testing pyramid strategy:

```
                    /\
                   /  \
                  / E2E \
                 /______\
                /        \
               /Integration\
              /__________\
             /            \
            /     Unit     \
           /________________\
```

**Unit Tests (70%)**: Test individual components in isolation
**Integration Tests (20%)**: Test component interactions
**E2E Tests (10%)**: Test complete user workflows

#### Unit Testing Patterns

**Testing Domain Logic**:
```typescript
// src/core/logics/v1/products/__tests__/products-create.test.ts
import { ProductsCreateLogic } from '../crud/products-create.logic';

describe('ProductsCreateLogic', () => {
    let logic: ProductsCreateLogic;
    let mockRepository: jest.Mocked<ICoreRepository>;
    let mockCoreProvider: jest.Mocked<CoreProvider>;

    beforeEach(() => {
        // Setup mocks
        mockRepository = {
            create: jest.fn(),
            read: jest.fn(),
            list: jest.fn(),
            update: jest.fn(),
            delete: jest.fn()
        } as any;

        mockCoreProvider = {
            getRepoDiscovery: jest.fn().mockReturnValue({
                getRepository: jest.fn().mockReturnValue(mockRepository)
            }),
            getErrorFactory: jest.fn().mockReturnValue({
                create: jest.fn((type, message) => new Error(message))
            })
        } as any;

        // Mock CoreProvider singleton
        jest.spyOn(CoreProvider, 'getInstance').mockReturnValue(mockCoreProvider);
    });

    describe('Business Rule Validation', () => {
        it('should validate category exists before creating product', async () => {
            const request = {
                body: {
                    name: 'Test Product',
                    categoryId: 1,
                    price: 99.99
                },
                user: { id: 'user123' }
            };

            // Mock category exists
            mockRepository.read.mockResolvedValue({
                success: true,
                data: { id: 1, name: 'Electronics' }
            });

            // Mock product creation
            mockRepository.create.mockResolvedValue({
                success: true,
                data: { id: 1, ...request.body }
            });

            logic = new ProductsCreateLogic(request, mockRepository);
            const result = await logic.execute();

            expect(mockRepository.read).toHaveBeenCalledWith({
                filter: { id: request.body.categoryId }
            });
            expect(result.success).toBe(true);
        });

        it('should reject product creation if category does not exist', async () => {
            const request = {
                body: {
                    name: 'Test Product',
                    categoryId: 999,
                    price: 99.99
                },
                user: { id: 'user123' }
            };

            // Mock category not found
            mockRepository.read.mockResolvedValue({
                success: false,
                error: { statusCode: 404, message: 'Not found' }
            });

            logic = new ProductsCreateLogic(request, mockRepository);

            await expect(logic.execute()).rejects.toThrow(
                'Category with ID 999 does not exist'
            );
        });

        it('should check for duplicate product names within category', async () => {
            const request = {
                body: {
                    name: 'Duplicate Product',
                    categoryId: 1,
                    price: 99.99
                },
                user: { id: 'user123' }
            };

            // Mock category exists
            mockRepository.read.mockResolvedValue({
                success: true,
                data: { id: 1, name: 'Electronics' }
            });

            // Mock duplicate found
            mockRepository.list.mockResolvedValue({
                success: true,
                data: [{ id: 2, name: 'Duplicate Product' }],
                count: 1
            });

            logic = new ProductsCreateLogic(request, mockRepository);

            await expect(logic.execute()).rejects.toThrow(
                'Product with name "Duplicate Product" already exists in this category'
            );
        });
    });

    describe('Edge Cases', () => {
        it('should handle products with special characters in names', async () => {
            const request = {
                body: {
                    name: 'Product™ with © & special © characters',
                    categoryId: 1,
                    price: 99.99
                },
                user: { id: 'user123' }
            };

            mockRepository.read.mockResolvedValue({
                success: true,
                data: { id: 1, name: 'Electronics' }
            });

            mockRepository.list.mockResolvedValue({
                success: true,
                data: [],
                count: 0
            });

            mockRepository.create.mockResolvedValue({
                success: true,
                data: { id: 1, ...request.body }
            });

            logic = new ProductsCreateLogic(request, mockRepository);
            const result = await logic.execute();

            expect(result.success).toBe(true);
        });

        it('should handle empty metadata gracefully', async () => {
            const request = {
                body: {
                    name: 'Product',
                    categoryId: 1,
                    price: 99.99,
                    metadata: {}
                },
                user: { id: 'user123' }
            };

            mockRepository.read.mockResolvedValue({
                success: true,
                data: { id: 1, name: 'Electronics' }
            });

            mockRepository.list.mockResolvedValue({
                success: true,
                data: [],
                count: 0
            });

            mockRepository.create.mockResolvedValue({
                success: true,
                data: { id: 1, ...request.body }
            });

            logic = new ProductsCreateLogic(request, mockRepository);
            const result = await logic.execute();

            expect(result.data.metadata).toEqual({});
        });
    });

    describe('Performance Tests', () => {
        it('should complete within performance budget', async () => {
            const request = {
                body: {
                    name: 'Performance Test Product',
                    categoryId: 1,
                    price: 99.99
                },
                user: { id: 'user123' }
            };

            mockRepository.read.mockResolvedValue({
                success: true,
                data: { id: 1, name: 'Electronics' }
            });

            mockRepository.list.mockResolvedValue({
                success: true,
                data: [],
                count: 0
            });

            mockRepository.create.mockResolvedValue({
                success: true,
                data: { id: 1, ...request.body }
            });

            logic = new ProductsCreateLogic(request, mockRepository);

            const startTime = Date.now();
            await logic.execute();
            const duration = Date.now() - startTime;

            // Should complete within 100ms
            expect(duration).toBeLessThan(100);
        });
    });
});
```

#### Integration Testing Patterns

**Testing API Endpoints with Database**:
```typescript
// tests/integration/products.integration.test.ts
import request from 'supertest';
import { app } from '../../src/bootstrap';
import { cleanupDatabase, seedTestData } from '../helpers/database';

describe('Products API Integration Tests', () => {
    beforeAll(async () => {
        await seedTestData();
    });

    afterAll(async () => {
        await cleanupDatabase();
    });

    describe('POST /api/v1/products', () => {
        it('should create a new product with valid data', async () => {
            const productData = {
                name: 'Integration Test Product',
                categoryId: 1,
                price: 99.99,
                description: 'Test description'
            };

            const response = await request(app)
                .post('/api/v1/products')
                .set('Authorization', `Bearer ${testToken}`)
                .send(productData)
                .expect(201);

            expect(response.body).toMatchObject({
                success: true,
                data: {
                    name: productData.name,
                    categoryId: productData.categoryId,
                    price: productData.price,
                    description: productData.description
                }
            });

            expect(response.body.data.id).toBeDefined();
            expect(response.body.data.createdAt).toBeDefined();
        });

        it('should return 400 for invalid product data', async () => {
            const invalidData = {
                name: '', // Empty name should fail validation
                categoryId: 1
            };

            const response = await request(app)
                .post('/api/v1/products')
                .set('Authorization', `Bearer ${testToken}`)
                .send(invalidData)
                .expect(400);

            expect(response.body.success).toBe(false);
            expect(response.body.error.message).toContain('name');
        });

        it('should enforce unique product names within category', async () => {
            const productData = {
                name: 'Unique Product',
                categoryId: 1,
                price: 99.99
            };

            // Create first product
            await request(app)
                .post('/api/v1/products')
                .set('Authorization', `Bearer ${testToken}`)
                .send(productData)
                .expect(201);

            // Try to create duplicate
            const response = await request(app)
                .post('/api/v1/products')
                .set('Authorization', `Bearer ${testToken}`)
                .send(productData)
                .expect(409);

            expect(response.body.success).toBe(false);
            expect(response.body.error.message).toContain('already exists');
        });
    });

    describe('GET /api/v1/products', () => {
        it('should return paginated list of products', async () => {
            const response = await request(app)
                .get('/api/v1/products?limit=10&offset=0')
                .set('Authorization', `Bearer ${testToken}`)
                .expect(200);

            expect(response.body.success).toBe(true);
            expect(response.body.data).toBeInstanceOf(Array);
            expect(response.body.pagination).toMatchObject({
                page: expect.any(Number),
                limit: 10,
                total: expect.any(Number)
            });
        });

        it('should filter products by category', async () => {
            const response = await request(app)
                .get('/api/v1/products?filter[categoryId][eq]=1')
                .set('Authorization', `Bearer ${testToken}`)
                .expect(200);

            expect(response.body.success).toBe(true);
            expect(response.body.data.every(p => p.categoryId === 1)).toBe(true);
        });

        it('should include related category data when requested', async () => {
            const response = await request(app)
                .get('/api/v1/products?include[]=category')
                .set('Authorization', `Bearer ${testToken}`)
                .expect(200);

            expect(response.body.success).toBe(true);
            expect(response.body.data[0].category).toBeDefined();
            expect(response.body.data[0].category.name).toBeDefined();
        });
    });
});
```

#### E2E Testing Patterns

**Testing Complete User Workflows**:
```typescript
// tests/e2e/product-lifecycle.e2e.test.ts
import { test, expect } from '@playwright/test';

test.describe('Product Lifecycle E2E', () => {
```

### Scalability Architecture Patterns

#### Horizontal Scaling Strategy

**Stateless Service Design**:
```typescript
// ❌ ANTI-PATTERN: Stateful service
class StatefulProductService {
    private cache: Map<number, Product> = new Map(); // Instance state!
    private requestCount: number = 0;

    async getProduct(id: number): Promise<Product> {
        this.requestCount++; // Won't work with multiple instances
        
        if (this.cache.has(id)) {
            return this.cache.get(id)!; // Each instance has different cache
        }

        const product = await this.repository.findById(id);
        this.cache.set(id, product);
        return product;
    }
}

// ✅ CORRECT: Stateless service with shared cache
class StatelessProductService {
    constructor(
        private repository: IProductRepository,
        private redis: IRedis // Shared cache across instances
    ) {}

    async getProduct(id: number): Promise<Product> {
        const cacheKey = `product:${id}`;
        
        // Check shared cache
        const cached = await this.redis.get(cacheKey);
        if (cached) return JSON.parse(cached);

        // Fetch and cache
        const product = await this.repository.findById(id);
        await this.redis.set(cacheKey, JSON.stringify(product), 3600);
        
        return product;
    }
}
```

**Session Management for Horizontal Scaling**:
```typescript
class DistributedSessionManager {
    constructor(private redis: IRedis) {}

    async createSession(userId: string, data: SessionData): Promise<string> {
        const sessionId = generateSessionId();
        const sessionKey = `session:${sessionId}`;
        
        await this.redis.set(sessionKey, JSON.stringify({
            userId,
            ...data,
            createdAt: new Date(),
            lastAccessedAt: new Date()
        }), 86400); // 24 hours
        
        return sessionId;
    }

    async getSession(sessionId: string): Promise<SessionData | null> {
        const sessionKey = `session:${sessionId}`;
        const data = await this.redis.get(sessionKey);
        
        if (!data) return null;

        // Update last accessed time
        const session = JSON.parse(data);
        session.lastAccessedAt = new Date();
        await this.redis.set(sessionKey, JSON.stringify(session), 86400);
        
        return session;
    }

    async destroySession(sessionId: string): Promise<void> {
        await this.redis.del(`session:${sessionId}`);
    }
}
```

#### Vertical Scaling Optimization

**CPU-Intensive Operations**:
```typescript
import { Worker } from 'worker_threads';

class WorkerPoolManager {
    private workers: Worker[] = [];
    private taskQueue: Array<{ task: any; resolve: Function; reject: Function }> = [];

    constructor(private workerCount: number = 4) {
        for (let i = 0; i < workerCount; i++) {
            this.createWorker();
        }
    }

    private createWorker(): void {
        const worker = new Worker('./worker.js');
        
        worker.on('message', (result) => {
            const task = this.taskQueue.shift();
            if (task) {
                task.resolve(result);
            }
        });

        worker.on('error', (error) => {
            const task = this.taskQueue.shift();
            if (task) {
                task.reject(error);
            }
        });

        this.workers.push(worker);
    }

    async executeTask<T>(taskData: any): Promise<T> {
        return new Promise((resolve, reject) => {
            this.taskQueue.push({ task: taskData, resolve, reject });
            
            // Find available worker
            const availableWorker = this.workers.find(w => !w.isActive);
            if (availableWorker) {
                const task = this.taskQueue.shift();
                if (task) {
                    availableWorker.postMessage(task.task);
                }
            }
        });
    }
}

// worker.js
const { parentPort } = require('worker_threads');

parentPort.on('message', async (task) => {
    try {
        // Perform CPU-intensive work
        const result = await processCpuIntensiveTask(task);
        parentPort.postMessage(result);
    } catch (error) {
        parentPort.postMessage({ error: error.message });
    }
});
```

### Multi-Tenancy Architecture Deep Dive

#### Tenant Isolation Strategies

**Strategy 1: Shared Database with Tenant ID (Current Implementation)**:
```typescript
class TenantIsolationMiddleware implements IMiddleware {
    async execute(request: IRequest, response: IResponse, next: NextFunction) {
        // Extract tenant from request
        const tenantId = this.extractTenantId(request);
        
        // Set tenant context
        TenantContext.setCurrentTenant(tenantId);
        
        try {
            await next();
        } finally {
            TenantContext.clear();
        }
    }

    private extractTenantId(request: IRequest): string {
        // Priority 1: JWT claim
        if (request.user?.tenantId) {
            return request.user.tenantId;
        }

        // Priority 2: Custom header
        if (request.headers['x-tenant-id']) {
            return request.headers['x-tenant-id'] as string;
        }

        // Priority 3: Subdomain
        const host = request.headers.host;
        if (host) {
            const subdomain = host.split('.')[0];
            return this.resolveTenantFromSubdomain(subdomain);
        }

        throw new Error('No tenant context available');
    }
}

// Tenant-aware repository
class TenantAwareRepository implements IRepository {
    async list(query: any): Promise<any[]> {
        const tenantId = TenantContext.getCurrentTenant();
        
        // Automatically inject tenant filter
        const tenantQuery = {
            ...query,
            where: query.where 
                ? { ...query.where, tenantId: { eq: tenantId } }
                : { tenantId: { eq: tenantId } }
        };
        
        return await this.db.query(tenantQuery);
    }

    async create(data: any): Promise<any> {
        const tenantId = TenantContext.getCurrentTenant();
        
        // Automatically inject tenant ID
        const tenantData = {
            ...data,
            tenantId
        };
        
        return await this.db.insert(tenantData);
    }
}

// Tenant context using AsyncLocalStorage
import { AsyncLocalStorage } from 'async_hooks';

class TenantContext {
    private static storage = new AsyncLocalStorage<string>();

    static setCurrentTenant(tenantId: string): void {
        this.storage.enterWith(tenantId);
    }

    static getCurrentTenant(): string {
        const tenantId = this.storage.getStore();
        if (!tenantId) {
            throw new Error('No tenant context set');
        }
        return tenantId;
    }

    static clear(): void {
        this.storage.disable();
    }
}
```

**Strategy 2: Database Per Tenant (Alternative)**:
```typescript
class MultiTenantDatabaseManager {
    private connections: Map<string, IDatabase> = new Map();
    private config: DatabaseConfig;

    constructor(config: DatabaseConfig) {
        this.config = config;
    }

    async getConnection(tenantId: string): Promise<IDatabase> {
        // Check if connection exists
        if (this.connections.has(tenantId)) {
            return this.connections.get(tenantId)!;
        }

        // Create new connection for tenant
        const tenantDbName = `tenant_${tenantId}`;
        const connection = await this.createConnection({
            ...this.config,
            database: tenantDbName
        });

        this.connections.set(tenantId, connection);
        return connection;
    }

    private async createConnection(config: DatabaseConfig): Promise<IDatabase> {
        return new DrizzleDatabase({
            url: `postgresql://${config.user}:${config.password}@${config.host}:${config.port}/${config.database}`
        });
    }

    async closeConnection(tenantId: string): Promise<void> {
        const connection = this.connections.get(tenantId);
        if (connection) {
            await connection.close();
            this.connections.delete(tenantId);
        }
    }
}

// Usage in repository
class MultiTenantRepository implements IRepository {
    constructor(private dbManager: MultiTenantDatabaseManager) {}

    async create(data: any): Promise<any> {
        const tenantId = TenantContext.getCurrentTenant();
        const db = await this.dbManager.getConnection(tenantId);
        return await db.insert(data);
    }
}
```

**Comparison Matrix**:
| Aspect | Shared DB + Tenant ID | Database Per Tenant | Schema Per Tenant |
|--------|----------------------|---------------------|-------------------|
| **Isolation** | Low | High | Medium |
| **Cost** | Low | High | Medium |
| **Scalability** | Medium | High | Medium |
| **Maintenance** | Easy | Complex | Medium |
| **Performance** | Good (with indexes) | Excellent | Good |
| **Backup/Restore** | Simple | Complex | Medium |
| **Data Migration** | Easy | Difficult | Medium |

### API Versioning Strategies

#### URI Versioning (Current Implementation)

```typescript
class VersionedRouter {
    private routes: Map<string, Map<string, IRoute>> = new Map();

    registerRoute(version: string, path: string, route: IRoute): void {
        if (!this.routes.has(version)) {
            this.routes.set(version, new Map());
        }
        this.routes.get(version)!.set(path, route);
    }

    resolveRoute(version: string, path: string): IRoute | null {
        const versionRoutes = this.routes.get(version);
        if (!versionRoutes) {
            throw new Error(`API version ${version} not supported`);
        }
        
        return versionRoutes.get(path) || null;
    }
}

// Usage
router.registerRoute('v1', '/products', {
    get: ProductControllerV1.list,
    post: ProductControllerV1.create
});

router.registerRoute('v2', '/products', {
    get: ProductControllerV2.list, // Breaking changes
    post: ProductControllerV2.create
});

// Client requests: /api/v1/products or /api/v2/products
```

#### Header Versioning Alternative

```typescript
class HeaderVersioningMiddleware implements IMiddleware {
    async execute(request: IRequest, response: IResponse, next: NextFunction) {
        // Extract version from Accept header
        const acceptHeader = request.headers.accept;
        const versionMatch = acceptHeader?.match(/application\/vnd\.byo-dpp\.v(\d+)\+json/);
        
        if (!versionMatch) {
            request.apiVersion = 'v1'; // Default
        } else {
            request.apiVersion = `v${versionMatch[1]}`;
        }
        
        // Validate version is supported
        if (!this.isSupportedVersion(request.apiVersion)) {
            return response.status(400).json({
                success: false,
                error: {
                    statusCode: 400,
                    message: `API version ${request.apiVersion} is not supported`
                }
            });
        }
        
        next();
    }

    private isSupportedVersion(version: string): boolean {
        return ['v1', 'v2'].includes(version);
    }
}

// Client request header:
// Accept: application/vnd.byo-dpp.v2+json
```

### Advanced Monitoring and Observability

#### Distributed Tracing Implementation

```typescript
import { trace, SpanStatusCode } from '@opentelemetry/api';

class TracingService {
    private tracer = trace.getTracer('byo-dpp-backend');

    async traceOperation<T>(
        operationName: string,
        operation: (span: Span) => Promise<T>,
        attributes?: Record<string, any>
    ): Promise<T> {
        const span = this.tracer.startSpan(operationName, {
            attributes: {
                'service.name': 'byo-dpp',
                ...attributes
            }
        });

        try {
            const result = await operation(span);
            span.setStatus({ code: SpanStatusCode.OK });
            return result;
        } catch (error) {

name: string;
            price: number;
        }

        // After (v2) - backward compatible
        interface ProductV2 {
            id: number;
            name: string;
            price: number;
            sku?: string; // Optional field - backward compatible
        }
    }

    // Breaking Change: Rename Field
    static renameField() {
        // V1: Use old field name
        class ProductServiceV1 {
            async create(data: { productName: string }): Promise<Product> {
                return await this.repository.create({
                    name: data.productName // Map to internal field
                });
            }
        }

        // V2: Use new field name
        class ProductServiceV2 {
            async create(data: { name: string }): Promise<Product> {
                return await this.repository.create(data);
            }
        }

        // Support both versions simultaneously
        class VersionedProductService {
            async create(request: IRequest): Promise<Product> {
                const version = request.apiVersion;
                
                if (version === 'v1') {
                    return await this.serviceV1.create(request.body);
                } else {
                    return await this.serviceV2.create(request.body);
                }
            }
        }
    }
}
```

### Resilience Patterns Implementation

#### Bulkhead Pattern for Resource Isolation

```typescript
class BulkheadPattern {
    private pools: Map<string, TaskPool> = new Map();

    constructor() {
        // Create separate pools for different operations
        this.pools.set('critical', new TaskPool(10));   // 10 threads for critical ops
        this.pools.set('standard', new TaskPool(20));   // 20 threads for standard ops
        this.pools.set('background', new TaskPool(5));  // 5 threads for background jobs
    }

    async execute<T>(
        poolName: string,
        operation: () => Promise<T>
    ): Promise<T> {
        const pool = this.pools.get(poolName);
        if (!pool) {
            throw new Error(`Pool not found: ${poolName}`);
        }

        return await pool.execute(operation);
    }
}

class TaskPool {
    private activeCount: number = 0;
    private queue: Array<() => Promise<any>> = [];

    constructor(private maxConcurrent: number) {}

    async execute<T>(operation: () => Promise<T>): Promise<T> {
        // Wait if pool is full
        while (this.activeCount >= this.maxConcurrent) {
            await new Promise(resolve => setTimeout(resolve, 100));
        }

        this.activeCount++;

        try {
            return await operation();
        } finally {
            this.activeCount--;
            this.processQueue();
        }
    }

    private async processQueue(): void {
        if (this.queue.length > 0 && this.activeCount < this.maxConcurrent) {
            const task = this.queue.shift();
            if (task) {
                this.execute(task);
            }
        }
    }
}

// Usage
class ProductService {
    constructor(private bulkhead: BulkheadPattern) {}

    async createProduct(data: CreateProductData): Promise<Product> {
        // Critical operation - use dedicated pool
        return await this.bulkhead.execute('critical', async () => {
            return await this.repository.create(data);
        });
    }

    async generateReport(): Promise<Report> {
        // Background operation - use background pool
        return await this.bulkhead.execute('background', async () => {
            return await this.reportGenerator.generate();
        });
    }
}
```

#### Circuit Breaker with Health Monitoring

```typescript
class AdvancedCircuitBreaker {
    private state: 'CLOSED' | 'OPEN' | 'HALF_OPEN' = 'CLOSED';
    private failureCount: number = 0;
    private successCount: number = 0;
    private lastFailureTime: Date | null = null;
    private metrics: CircuitBreakerMetrics;

    constructor(
        private options: {
            failureThreshold: number;
            successThreshold: number;
            timeout: number;
            monitoringWindow: number;
        }
    ) {
        this.metrics = {
            totalRequests: 0,
            successfulRequests: 0,
            failedRequests: 0,
            rejectedRequests: 0,
            avgResponseTime: 0
        };
    }

    async execute<T>(operation: () => Promise<T>): Promise<T> {
        // Check circuit state
        if (this.state === 'OPEN') {
            if (this.shouldAttemptReset()) {
                this.state = 'HALF_OPEN';
                console.log('Circuit breaker: OPEN → HALF_OPEN');
            } else {
                this.metrics.rejectedRequests++;
                throw new Error('Circuit breaker is OPEN');
            }
        }

        const startTime = Date.now();
        this.metrics.totalRequests++;

        try {
            const result = await operation();
            const duration = Date.now() - startTime;
            
            this.onSuccess(duration);
            return result;
        } catch (error) {
            const duration = Date.now() - startTime;
            this.onFailure(error, duration);
            throw error;
        }
    }

    private onSuccess(duration: number): void {
        this.successCount++;
        this.metrics.successfulRequests++;
        
        // Update avg response time
        this.metrics.avgResponseTime = 
            (this.metrics.avgResponseTime * (this.metrics.successfulRequests - 1) + duration)
            / this.metrics.successfulRequests;

        // Reset if in HALF_OPEN and enough successes
        if (this.state === 'HALF_OPEN') {
            if (this.successCount >= this.options.successThreshold) {
                this.state = 'CLOSED';
                this.failureCount = 0;
                this.successCount = 0;
                console.log('Circuit breaker: HALF_OPEN → CLOSED');
            }
        }
    }

    private onFailure(error: any, duration: number): void {
        this.failureCount++;
        this.metrics.failedRequests++;
        this.lastFailureTime = new Date();

        // Open circuit if threshold exceeded
        if (this.failureCount >= this.options.failureThreshold) {
            this.state = 'OPEN';
            console.error('Circuit breaker: CLOSED → OPEN', {
                failureCount: this.failureCount,
                error: error.message
            });
        }
    }

    private shouldAttemptReset(): boolean {
        if (!this.lastFailureTime) return false;
        const timeSinceLastFailure = Date.now() - this.lastFailureTime.getTime();
        return timeSinceLastFailure >= this.options.timeout;
    }

    getMetrics(): CircuitBreakerMetrics {
        return { ...this.metrics, state: this.state };
    }
}
```

### Data Consistency Patterns

#### Eventual Consistency with Compensation

```typescript
class EventualConsistencyManager {
    constructor(
        private eventStore: IEventStore,
        private compensationService: ICompensationService
    ) {}

    async executeWithEventualConsistency<T>(
        operation: () => Promise<T>,
        compensationOps: CompensationOperation[]
    ): Promise<T> {
        const operationId = generateOperationId();
        
        try {
            // Execute main operation
            const result = await operation();
            
            // Record success event
            await this.eventStore.append(new OperationSucceededEvent(
                operationId,
                result,
                new Date()
            ));
            
            return result;
        } catch (error) {
            // Execute compensations in reverse order
            console.log(`Operation ${operationId} failed, executing compensations`);
            
            for (let i = compensationOps.length - 1; i >= 0; i--) {
                try {
                    await compensationOps[i].execute();
                    console.log(`Compensation ${i} succeeded`);
                } catch (compensationError) {
                    console.error(`Compensation ${i} failed:`, compensationError);
                    // Log for manual intervention
                    await this.compensationService.logFailedCompensation(
                        operationId,
                        compensationOps[i],
                        compensationError
                    );
                }
            }
            
            throw error;
        }
    }
}

// Usage in saga
class OrderSaga {
    async placeOrder(orderData: OrderData): Promise<Order> {
        const compensations: CompensationOperation[] = [];
        
        return await this.consistencyManager.executeWithEventualConsistency(
            async () => {
                // Step 1: Reserve inventory
                const reservation = await this.inventoryService.reserve(orderData.items);
                compensations.push({
                    name: 'release-inventory',
                    execute: () => this.inventoryService.release(reservation.id)
                });

                // Step 2: Charge payment
                const payment = await this.paymentService.charge(orderData.payment);
                compensations.push({
                    name: 'refund-payment',
                    execute: () => this.paymentService.refund(payment.id)
                });

                // Step 3: Create order
                const order = await this.orderService.create(orderData);
                compensations.push({
                    name: 'cancel-order',
                    execute: () => this.orderService.cancel(order.id)
                });

                return order;
            },
            compensations
        );
    }
}
```

#### Optimistic Locking for Concurrent Updates

```typescript
class OptimisticLockingRepository {
    async update(id: number, data: any, expectedVersion: number): Promise<any> {
        const result = await this.db.query(`
            UPDATE products
            SET 
                name = $1,
                price = $2,
                version = version + 1,
                updated_at = NOW()
            WHERE id = $3 AND version = $4
            RETURNING *
        `, [data.name, data.price, id, expectedVersion]);

        if (result.rowCount === 0) {
            // Check if product exists
            const existing = await this.db.query(`
                SELECT version FROM products WHERE id = $1
            `, [id]);

            if (existing.rowCount === 0) {
                throw new NotFoundError(`Product ${id} not found`);
            }

            // Product exists but version mismatch
            throw new ConcurrencyConflictError(
                `Product ${id} was modified by another transaction. ` +
                `Expected version ${expectedVersion}, current version ${existing.rows[0].version}`
            );
        }


// Keep Elasticsearch synced via events
    async syncToElasticsearch(event: ProductEvent): Promise<void> {
        if (event instanceof ProductCreatedEvent) {
            await this.elasticsearch.index({
                index: 'products',
                id: event.productId,
                body: {
                    name: event.name,
                    description: event.description,
                    category: event.categoryName,
                    price: event.price,
                    createdAt: event.timestamp
                }
            });
        } else if (event instanceof ProductUpdatedEvent) {
            await this.elasticsearch.update({
                index: 'products',
                id: event.productId,
                body: {
                    doc: event.changes
                }
            });
        } else if (event instanceof ProductDeletedEvent) {
            await this.elasticsearch.delete({
                index: 'products',
                id: event.productId
            });
        }
    }
}
// Performance: ~5ms avg (Elasticsearch), full-text search, fuzzy matching
// Improvement: 100x faster with advanced search features
```

**Performance Comparison Summary**:
| Version | Avg Response Time | Search Features | Cost |
|---------|------------------|-----------------|------|
| V1 (Basic) | 500ms | Basic LIKE | Low |
| V2 (Indexed) | 100ms | Similarity | Low |
| V3 (Cached) | 10ms (hit) / 100ms (miss) | Similarity + Cache | Medium |
| V4 (Elasticsearch) | 5ms | Full-text, Fuzzy, Ranking | High |

#### Case Study 2: Handling High-Volume Data Import

**Challenge**: Import 1 million product records efficiently.

**Solution Phases**:

```typescript
// Phase 1: Sequential Processing (Slow)
class SequentialImporter {
    async import(records: ProductRecord[]): Promise<void> {
        for (const record of records) {
            await this.validateRecord(record);
            await this.repository.create(record);
        }
    }
}
// Performance: ~2 hours for 1M records
// Problems: Sequential processing, no error recovery

// Phase 2: Batch Processing
class BatchImporter {
    async import(records: ProductRecord[]): Promise<void> {
        const batchSize = 1000;
        
        for (let i = 0; i < records.length; i += batchSize) {
            const batch = records.slice(i, i + batchSize);
            
            // Validate batch
            const validRecords = await this.validateBatch(batch);
            
            // Insert batch in single transaction
            await this.repository.createBatch(validRecords);
            
            console.log(`Processed ${i + batch.length} / ${records.length}`);
        }
    }
}
// Performance: ~20 minutes for 1M records
// Improvement: 6x faster

// Phase 3: Parallel Batch Processing
class ParallelBatchImporter {
    async import(records: ProductRecord[]): Promise<void> {
        const batchSize = 1000;
        const concurrency = 10;
        
        const batches = this.chunk(records, batchSize);
        
        // Process batches in parallel
        await this.processBatchesInParallel(batches, concurrency);
    }

    private async processBatchesInParallel(
        batches: ProductRecord[][],
        concurrency: number
    ): Promise<void> {
        for (let i = 0; i < batches.length; i += concurrency) {
            const batchGroup = batches.slice(i, i + concurrency);
            
            await Promise.all(
                batchGroup.map(batch => this.processBatch(batch))
            );
            
            console.log(`Processed ${Math.min(i + concurrency, batches.length)} / ${batches.length} batches`);
        }
    }

    private async processBatch(batch: ProductRecord[]): Promise<void> {
        const validRecords = await this.validateBatch(batch);
        await this.repository.createBatch(validRecords);
    }
}
// Performance: ~5 minutes for 1M records
// Improvement: 24x faster than sequential

// Phase 4: Stream Processing with Backpressure
class StreamImporter {
    async import(records: ProductRecord[]): Promise<void> {
        const readStream = Readable.from(records);
        const validationStream = new ValidationTransform();
        const importStream = new ImportTransform(this.repository);
        const writeStream = new MonitoringWritable();

        return new Promise((resolve, reject) => {
            pipeline(
                readStream,
                validationStream,
                importStream,
                writeStream,
                (error) => {
                    if (error) {
                        reject(error);
                    } else {
                        resolve();
                    }
                }
            );
        });
    }
}

class ImportTransform extends Transform {
    private batch: ProductRecord[] = [];
    private batchSize = 1000;

    constructor(private repository: IRepository) {
        super({ objectMode: true });
    }

    async _transform(record: ProductRecord, encoding: string, callback: Function) {
        this.batch.push(record);

        if (this.batch.length >= this.batchSize) {
            try {
                await this.repository.createBatch(this.batch);
                this.batch = [];
                callback();
            } catch (error) {
                callback(error);
            }
        } else {
            callback();
        }
    }

    async _flush(callback: Function) {
        if (this.batch.length > 0) {
            try {
                await this.repository.createBatch(this.batch);
                callback();
            } catch (error) {
                callback(error);
            }
        } else {
            callback();
        }
    }
}
// Performance: ~3 minutes for 1M records with memory-efficient streaming
// Improvement: 40x faster with controlled memory usage
```

### Microservices Communication Patterns

#### Service-to-Service Communication

**Synchronous Communication (HTTP)**:
```typescript
class ProductServiceClient {
    constructor(
        private httpClient: IHttpClient,
        private circuitBreaker: CircuitBreaker
    ) {}

    async getProduct(id: number): Promise<Product> {
        return await this.circuitBreaker.execute(async () => {
            const response = await this.httpClient.get(
                `http://product-service/api/v1/products/${id}`,
                {
                    timeout: 5000,
                    headers: {
                        'x-correlation-id': CorrelationIdManager.getContext().correlationId
                    }
                }
            );

            return response.data;
        });
    }
}
```

**Asynchronous Communication (Message Queue)**:
```typescript
class MessageBroker {
    constructor(private rabbitmq: IMessageQueue) {}

    async publishEvent(event: DomainEvent): Promise<void> {
        await this.rabbitmq.publish({
            exchange: 'domain-events',
            routingKey: event.type,
            message: JSON.stringify(event),
            options: {
                persistent: true,
                contentType: 'application/json',
                correlationId: CorrelationIdManager.getContext().correlationId
            }
        });
    }

    async subscribe(
        eventType: string,
        handler: (event: DomainEvent) => Promise<void>
    ): Promise<void> {
        await this.rabbitmq.subscribe({
            queue: `${eventType}-queue`,
            handler: async (message) => {
                try {
                    const event = JSON.parse(message.content);
                    await handler(event);
                    message.ack();
                } catch (error) {
                    console.error('Event handler failed:', error);
                    message.nack(true); // Requeue
                }
            }
        });
    }
}

// Service A: Publisher
class OrderService {
    async createOrder(orderData: OrderData): Promise<Order> {
        const order = await this.repository.create(orderData);
        
        // Publish event for other services
        await this.messageBroker.publishEvent(
            new OrderCreatedEvent(order.id, order.customerId, order.total)
        );
        
        return order;
    }
}

// Service B: Subscriber
class InventoryService {
    async initialize(): Promise<void> {
        await this.messageBroker.subscribe(
            'OrderCreated',
            async (event: OrderCreatedEvent) => {
                // React to order creation
                await this.reserveInventory(event.orderId, event.items);
            }
        );
    }
}
```

### Code Organization Best Practices

#### Feature-Based vs Layer-Based Organization

**Layer-Based (Current)**:
```
src/
├── application/
│   ├── controllers/
│   ├── services/
│   └── middlewares/
├── domain/
│   ├── entities/
│   ├── logics/
│   └── events/
└── infrastructure/
    ├── repositories/
    ├── adapters/
    └── schemas/
```

**Feature-Based (Alternative)**:
```
src/
├── products/
│   ├── application/
│   │   ├── product.controller.ts
│   │   └── product.service.ts
│   ├── domain/
│   │   ├── product.entity.ts
│   │   └── product.logic.ts
│   └── infrastructure/
│       ├── product.repository.ts
│       └── product.schema.ts
├── orders/
│   ├── application/
│   ├── domain/
│   └── infrastructure/
└── shared/
    ├── interfaces/
    └── utilities/
```

**Comparison**:
| Aspect | Layer-Based | Feature-Based |
|--------|------------|---------------|
| **Cohesion** | Low (scattered features) | High (feature together) |
| **Navigation** | By technical concern | By business feature |
| **Team Organization** | By technical role | By feature ownership |
| **Dependencies** | Easy to see layers | Easy to see features |
| **Scalability** | Can become large | Grows with features |

#### Module Organization Pattern

```typescript
// src/products/index.ts - Feature exports
export { ProductController } from './application/product.controller';
export { ProductService } from './application/product.service';
export { Product } from './domain/product.entity';
export { ProductRepository } from './infrastructure/product.repository';

// Public interface
export interface IProductModule {
    controller: IController;
    service: IService;
}

// Module factory
export function createProductModule(dependencies: ModuleDependencies): IProductModule {
    const repository = new ProductRepository(dependencies.database);
    const service = new ProductService(repository, dependencies.validator);
    const controller = new ProductController(service);

    return {
        controller,
        service
    };
}

// Usage in bootstrap
import { createProductModule } from './products';

const productModule = createProductModule({
    database,
    validator,
    eventBus
});

router.registerController('/products', productModule.controller);
```

### Disaster Recovery and High Availability

#### Backup Strategy Implementation

```typescript
class BackupManager {
    async performBackup(type: 'full' | 'incremental'): Promise<BackupResult> {
        const backupId = generateBackupId();
        const startTime = Date.now();

        console.log(`Starting ${type} backup: ${backupId}`);

        try {
            if (type === 'full') {
                await this.fullBackup(backupId);
            } else {
                await this.incrementalBackup(backupId);
            }

            const duration = Date.now() - startTime;
            
            return {
                backupId,
                type,
                status: 'completed',
                duration,
                timestamp: new Date()
            };
        } catch (error) {
            console.error('Backup failed:', error);
            
            return {
                backupId,
                type,
                status: 'failed',
                error: error.message,
                timestamp: new Date()
            };
        }
    }

    private async fullBackup(backupId: string): Promise<void> {
        // 1. Backup database
        await this.backupDatabase(backupId);
        
        // 2. Backup uploaded files
        await this.backupFiles(backupId);
        
        // 3. Backup configuration
        await this.backupConfiguration(backupId);
        
        // 4. Upload to remote storage
        await this.uploadToRemoteStorage(backupId);
    }

    private async incrementalBackup(backupId: string): Promise<void> {
        const lastBackup = await this.getLastBackup();
        
        // Backup only changes since last backup
        await this.backupChanges(lastBackup.timestamp, backupId);
        
        // Upload to remote storage
        await this.uploadToRemoteStorage(backupId);
    }

    async restore(backupId: string): Promise<void> {
        console.log(`Restoring from backup: ${backupId}`);
 {
                console.warn('Connection pool showing signs of saturation');
            }

            return {
                name: 'database-connections',
                passed: true,
                details: poolStats
            };
        } catch (error) {
            return {
                name: 'database-connections',
                passed: false,
                error: error.message
            };
        }
    }

    private async checkExternalServices(): Promise<CheckResult> {
        const services = [
            { name: 'keycloak', check: () => this.keycloak.healthCheck() },
            { name: 'redis', check: () => this.redis.ping() },
            { name: 'email-service', check: () => this.emailService.healthCheck() },
            { name: 'file-storage', check: () => this.fileStorage.healthCheck() }
        ];

        const results = await Promise.allSettled(
            services.map(s => s.check())
        );

        const failures = results.filter(r => r.status === 'rejected');
        
        return {
            name: 'external-services',
            passed: failures.length === 0,
            details: {
                total: services.length,
                healthy: services.length - failures.length,
                unhealthy: failures.length
            }
        };
    }

    private async checkSecurity(): Promise<CheckResult> {
        const securityChecks = [
            // Environment variables should not contain sensitive data in plain text
            this.checkEnvironmentSecrets(),
            // HTTPS should be enforced
            this.checkHttpsEnforcement(),
            // Rate limiting should be configured
            this.checkRateLimiting(),
            // Security headers should be set
            this.checkSecurityHeaders(),
            // CORS should be properly configured
            this.checkCorsConfiguration()
        ];

        const results = await Promise.all(securityChecks);
        const passed = results.every(r => r.passed);

        return {
            name: 'security',
            passed,
            details: results
        };
    }

    private async checkMonitoring(): Promise<CheckResult> {
        const checks = [
            // Metrics endpoint should be accessible
            await this.checkMetricsEndpoint(),
            // Logs should be properly configured
            await this.checkLoggingConfiguration(),
            // Health check endpoint should respond
            await this.checkHealthEndpoint(),
            // APM should be configured
            await this.checkAPMConfiguration()
        ];

        return {
            name: 'monitoring',
            passed: checks.every(c => c.passed),
            details: checks
        };
    }

    private async checkTests(): Promise<CheckResult> {
        try {
            // Run test suite
            const testResult = await this.runTests();
            
            return {
                name: 'tests',
                passed: testResult.passed,
                details: {
                    total: testResult.total,
                    passed: testResult.passed,
                    failed: testResult.failed,
                    coverage: testResult.coverage
                }
            };
        } catch (error) {
            return {
                name: 'tests',
                passed: false,
                error: error.message
            };
        }
    }
}
```

### Architecture Best Practices Summary

#### The Twelve-Factor App Principles Applied

**1. Codebase**: One codebase tracked in version control
```bash
# Single Git repository with multiple deployment environments
git clone https://github.com/org/byo-dpp-backend.git
git checkout main      # Production branch
git checkout develop   # Development branch
```

**2. Dependencies**: Explicitly declare and isolate dependencies
```json
{
  "dependencies": {
    "express": "^4.18.0",
    "drizzle-orm": "^0.29.0"
  },
  "devDependencies": {
    "typescript": "^5.0.0"
  }
}
```

**3. Config**: Store config in environment
```typescript
// ✅ Config from environment
const dbUrl = process.env.POSTGRES_DB_URI;

// ❌ Config in code
const dbUrl = 'postgresql://localhost:5432/byo_dpp';
```

**4. Backing Services**: Treat backing services as attached resources
```typescript
// Services accessed via URLs in environment
const services = {
    database: process.env.POSTGRES_DB_URI,
    cache: process.env.REDIS_URL,
    email: process.env.SMTP_HOST,
    storage: process.env.S3_ENDPOINT
};
```

**5. Build, Release, Run**: Strictly separate build and run stages
```dockerfile
# Build stage
FROM node:18 AS builder
COPY . .
RUN npm ci && npm run build

# Run stage
FROM node:18-alpine
COPY --from=builder /app/dist ./dist
CMD ["node", "dist/main.js"]
```

**6. Processes**: Execute as stateless processes
```typescript
// ✅ Stateless - no in-memory state
class ProductService {
    constructor(private redis: IRedis) {}
    
    async getProduct(id: number): Promise<Product> {
        return await this.redis.get(`product:${id}`);
    }
}

// ❌ Stateful - breaks in multi-instance deployment
class ProductService {
    private cache = new Map(); // In-memory state!
}
```

**7. Port Binding**: Export services via port binding
```typescript
const port = process.env.PORT || 3000;
app.listen(port, () => {
    console.log(`Server listening on port ${port}`);
});
```

**8. Concurrency**: Scale out via process model
```yaml
# Kubernetes deployment with replicas
apiVersion: apps/v1
kind: Deployment
metadata:
  name: byo-dpp-app
spec:
  replicas: 5  # Scale horizontally
```

**9. Disposability**: Maximize robustness with fast startup and graceful shutdown
```typescript
class GracefulShutdownManager {
    async shutdown(): Promise<void> {
        console.log('Shutting down gracefully...');
        
        // Stop accepting new requests
        this.server.close();
        
        // Wait for in-flight requests (max 30 seconds)
        await this.waitForInflightRequests(30000);
        
        // Close database connections
        await this.database.close();
        
        // Close other connections
        await this.redis.quit();
        
        console.log('Shutdown complete');
        process.exit(0);
    }
}

// Handle signals
process.on('SIGTERM', () => shutdownManager.shutdown());
process.on('SIGINT', () => shutdownManager.shutdown());
```

**10. Dev/Prod Parity**: Keep development, staging, and production as similar as possible
```typescript
// Same Docker images across environments
// Only configuration differs via environment variables
```

**11. Logs**: Treat logs as event streams
```typescript
// Write to stdout, let infrastructure handle routing
console.log(JSON.stringify({
    level: 'info',
    message: 'Product created',
    productId: product.id,
    timestamp: new Date().toISOString()
}));

// Infrastructure routes logs
// Development: Console
// Production: CloudWatch/ELK Stack
```

**12. Admin Processes**: Run admin tasks as one-off processes
```bash
# Database migrations
npm run db:migrate

# Data seeding
npm run db:seed

# Cache warming
npm run cache:warm
```

### Comprehensive Architecture Decision Framework

When making architectural decisions, consider these factors:

#### Decision Matrix Template

```typescript
interface ArchitectureDecision {
    context: string;
    problem: string;
    options: DecisionOption[];
    decision: string;
    rationale: string;
    consequences: {
        positive: string[];
        negative: string[];
        risks: string[];
    };
    revisitDate?: Date;
}

// Example decision
const cacheStrategyDecision: ArchitectureDecision = {
    context: 'Product catalog queries are slow (500ms average)',
    problem: 'Need to improve read performance without sacrificing data freshness',
    options: [
        {
            name: 'In-Memory Cache',
            pros: ['Fastest', 'Simple'],
            cons: ['Not shared across instances', 'Memory limited'],
            cost: 'Low'
        },
        {
            name: 'Redis Cache',
            pros: ['Shared', 'Persistent', 'Scalable'],
            cons: ['Network latency', 'Additional infrastructure'],
            cost: 'Medium'
        },
        {
            name: 'CDN Cache',
            pros: ['Global distribution', 'Very fast'],
            cons: ['Only for static content', 'Cache invalidation complex'],
            cost: 'High'
        }
    ],
    decision: 'Redis Cache',
    rationale: 'Need shared cache across multiple instances, acceptable network latency (~1ms), fits within budget',
    consequences: {
        positive: [
            '10x performance improvement',
            'Reduced database load by 80%',
            'Better scalability'
        ],
        negative: [
            'Additional infrastructure to maintain',
            'Cache invalidation complexity',
            'Increased deployment complexity'
        ],
        risks: [
            'Redis failure affects performance',
            'Cache stampede during cache miss',
            'Stale data if not invalidated properly'
        ]
    },
    revisitDate: new Date('2025-06-01') // Revisit in 6 months
};
```

### Architecture Anti-Patterns to Avoid

#### Anti-Pattern 1: God Object

```typescript
// ❌ ANTI-PATTERN: Single class does everything
class ProductManager {
    async handleProduct(action: string, data: any) {
        // Validation
        if (!data.name) throw new Error('Invalid');
        
        // Business logic
        if (action === 'create') {
            // Database access
            await db.insert(data);
            
            // Email notification
            await sendEmail();
```

## Architecture Understanding A2Z: Complete Guide with Actual Code References

This comprehensive guide walks through every aspect of the BYO DPP architecture with real code examples from the actual codebase.

### 1. Application Bootstrap and Initialization Flow


```mermaid
flowchart TD
    Start([Application Start])
    
    Start --> Config[Phase 1: Load Configuration<br/>setupConfig]
    Config --> JSON[Phase 2: Load Bootstrap JSON<br/>setupBootstrapJson]
    JSON --> Factory[Phase 3: Create Dependency Factory<br/>setupDependencyFactory]
    
    subgraph FactoryCreation["Dependency Factory Creates"]
        DB[(Database)]
        Router[Router]
        Server[Express Server]
        Graph[GraphAPI]
        Repo[RepoDiscovery]
        Event[EventStream]
        Mail[Mailer]
        BC[Blockchain]
        Identity[Identity Service]
        Auth[Authorization]
    end
    
    Factory --> FactoryCreation
    FactoryCreation --> CoreProv[Phase 4: Setup CoreProvider<br/>setupCoreProvider]
    CoreProv --> AppProv[Phase 5: Setup ApplicationProvider<br/>setupApplicationProvider]
    AppProv --> AppSetup[Phase 6: Setup Application<br/>setupApplication]
    AppSetup --> Build[Phase 7: Build Routes<br/>buildRoutes]
    
    subgraph BuildProcess["For Each Resource"]
        CreateRepo[Create Repository]
        RegisterRepo[Register with Discovery]
        CreateService[Create Service + Logic Map]
        CreateController[Create Controller]
        CreateMiddleware[Create Middleware Pipeline]
        RegisterRoutes[Register CRUD & Action Routes]
        
        CreateRepo --> RegisterRepo
        RegisterRepo --> CreateService
        CreateService --> CreateController
        CreateController --> CreateMiddleware
        CreateMiddleware --> RegisterRoutes
    end
    
    Build --> BuildProcess
    BuildProcess --> Boot[Phase 8: Boot Application<br/>boot]
    Boot --> Ready([Application Ready])
    
    style Start fill:#90EE90
    style Ready fill:#90EE90
    style FactoryCreation fill:#FFE4B5
    style BuildProcess fill:#E0FFFF
    style Config fill:#F0E68C
    style JSON fill:#F0E68C
    style Factory fill:#DDA0DD
    style CoreProv fill:#DDA0DD
    style AppProv fill:#DDA0DD
```

#### Complete Bootstrap Process Explained

The application starts in [`src/bootstrap.ts`](src/bootstrap.ts:1), following a carefully orchestrated initialization sequence:

**Phase 1: Configuration Loading** (Line 383-385)
```typescript
// From src/bootstrap.ts:383-385
setupConfig(): void {
    this.config = AppConfig.getInstance().getConfig();
}
```
**What happens**: Loads environment-specific configuration using singleton pattern
**Why important**: Ensures consistent configuration across the application
**Key decision**: Singleton pattern prevents multiple config instances

**Phase 2: Bootstrap Data Loading** (Line 387-389)
```typescript
// From src/bootstrap.ts:387-389
setupBootstrapJson(): void {
    this.byoDPPJson = JSON.parse(fs.readFileSync(BYO_DPP_DATA_FILE_PATH, 'utf8'));
}
```
**What happens**: Loads resource definitions from JSON file
**Why important**: Enables dynamic resource registration without code changes
**Key decision**: JSON-driven configuration allows runtime flexibility

**Phase 3: Dependency Factory Creation** (Line 391-440)
```typescript
// From src/bootstrap.ts:391-440
setupDependencyFactory(): void {
    const factoryConfig = {
        database: { url: this.config.POSTGRES_DB_URI! },
        server: { port: Number(this.config.PORT), host: getLocalIP() },
        swaggerData: this.byoDPPJson,
        // Keycloak configuration
        KEYCLOAK_HOST_URI: this.config.KEYCLOAK_HOST_URI!,
        // SMTP configuration
        SMTP_HOST: this.config.SMTP_HOST!,
        // Blockchain configuration
        blockchain: {
            network: this.config.POLYGON_NETWORK,
            rpcUrl: this.config.POLYGON_RPC_ENDPOINT,
            privateKey: this.config.POLYGON_PRIVATE_KEY
        }
    };
    this.dependencyFactory = new DependencyFactory(factoryConfig);
    this.dependencyFactory.createCoreDependencies();
}
```
**What happens**: Creates central factory for all dependencies
**Why important**: Single source of truth for dependency creation
**Key decision**: Factory pattern enables controlled instantiation

**Phase 4: Provider Initialization** (Line 442-448)
```typescript
// From src/bootstrap.ts:442-448
setupCoreProvider(): void {
    CoreProvider.initialize(new CoreDependencyFactoryProxy(this.dependencyFactory));
}

setupApplicationProvider(): void {
    ApplicationProvider.initialize(new ApplicationDependencyFactoryProxy(this.dependencyFactory));
}
```
**What happens**: Initializes singleton providers for core and application dependencies
**Why important**: Provides global access to dependencies throughout the application
**Key decision**: Provider pattern separates dependency creation from usage

#### Understanding the Dependency Factory

The [`DependencyFactory`](src/infra/factories/dependency-factory.ts:106) creates all infrastructure dependencies:

```typescript
// From src/infra/factories/dependency-factory.ts:333-348
async createCoreDependencies() {
    this.db = this.createDatabase();                    // PostgreSQL connection
    this.router = this.createRouter();                  // Express router
    this.server = this.createServer();                  // Express server
    this.graphAPI = this.createGraphAPI();              // Query builder
    this.repoDiscovery = this.createRepoDiscovery();    // Repository registry
    this.eventStream = this.createEventStream();        // SSE adapter
    this.mailer = this.createMailer();                  // Email service
    this.validator = JoiAdapter;                        // Validation adapter
    this.blockchain = this.createBlockchain();          // Blockchain service
    this.exportImporter = this.createExportImporterProvider(); // Import/export
    this.errorFactory = this.createErrorFactory();      // Error handling
    this.identityService = this.createIdentityService(); // Keycloak
    this.authorizationService = await this.createAuthorizationService(); // Casbin
    this.idGenerator = this.createIdGenerator();        // ULID generator
}
```

**Real-World Example**: Creating Database Connection
```typescript
// From src/infra/factories/dependency-factory.ts:136-141
createDatabase(): IDatabase {
    if (!this.db) {
        this.db = new PostgresDB({ url: this.config.database.url });
    }
    return this.db;
}
```
**Pattern**: Lazy initialization with caching
**Benefit**: Database connection created once and reused

### 2. Dynamic Resource Registration Architecture

#### How Resources Are Registered Dynamically

The [`buildRoutes()`](src/bootstrap.ts:456) method in bootstrap.ts implements the dynamic resource registration:

```typescript
// From src/bootstrap.ts:456-601
buildRoutes(): void {
    this.logger.info('📝 Building API routes');
    
    const graphQuery = this.dependencyFactory.getGraphAPI();
    const repoDiscovery = this.dependencyFactory.getRepoDiscovery();
    const database = this.dependencyFactory.getDB();
    const router = this.dependencyFactory.getRouter();
    
    this.byoDPPJson.forEach((service: Service) => {
        service.resources.forEach((resource: Resource) => {
            // Step 1: Create Repository
            let resourceBaseRepoAdapter: IAppRepository;
            if (resource.type === 'database') {
                const resourceORMRepository = new DrizzleORMRepository(
                    database,
                    resource.name,
                    ResourceModelMap[resource.name]
                );
                resourceBaseRepoAdapter = new BaseRepoAdapter(resourceORMRepository);
            } else {
                resourceBaseRepoAdapter = new NullRepoAdapter();
            }
            
            // Step 2: Register Repository
            repoDiscovery.registerRepository(resource.name, resourceBaseRepoAdapter);
            
            // Step 3: Create Service with Logic Map
            const resourceBaseService = new BaseService(
                resourceBaseRepoAdapter,
                resource.name,
                {
                    ...logicMap['v1'][resource.name]?.action,
                    ...logicMap['v1'][resource.name]?.crud
                }
            );
            
            // Step 4: Create Controller
            const resourceBaseController = new BaseController(
                resourceBaseService,
                graphQuery,
                resource.name
            );
            
            // Step 5: Setup Middleware Pipeline
            const authenticatorMiddleware = new AuthenticatorMiddleware(
                this.dependencyFactory.getIdentityService()
            );
            
            // Step 6: Register Action Routes
            resource.routes.actions.forEach((route) => {
                const middlewares: IMiddleware[] = [authenticatorMiddleware];
                
                const actionKey = `${route.url.split('/').pop()}:${route.method}`;
                
                if (logicMap['v1'][resource.name]?.action[actionKey]) {
                    router.addRoute({
                        path: route.url,
                        method: route.method,
                        action: 'action',
                        actionName: actionKey,
                        controller: resourceBaseController,
                        middlewares
                    });
                }
            });
            
            // Step 7: Register CRUD Routes
            resource.routes.crud.forEach((route) => {
                const middlewares: IMiddleware[] = [authenticatorMiddleware];
                
                // Add validation middleware if schema exists
                if (validationSchemaMap[resource.name]?.[httpMethod]) {
                    const schema = validationSchemaMap[resource.name][httpMethod];
                    middlewares.push(new ValidatorMiddleware(new JoiAdapter(schema)));
                }
                
                router.addRoute({
                    path: route.url,
                    method: httpMethodMap[route.method],
                    action: actionMap[route.method],
                    controller: resourceBaseController,
                    middlewares
                });
            });
        });
    });
}
```

**Key Architectural Insights**:
1. **Dynamic Registration**: Resources defined in JSON, not hardcoded
2. **Layered Creation**: Repository → Service → Controller (dependency flow)
3. **Logic Injection**: Custom business logic injected via logic map
4. **Middleware Composition**: Each route gets appropriate middleware chain

### 3. Provider Pattern Implementation with Actual Code

#### Core Provider: Global Access to Core Dependencies

The [`CoreProvider`](src/core/providers/core.provider.ts:17) implements singleton pattern for global dependency access:

```typescript
// From src/core/providers/core.provider.ts:17-50
export class CoreProvider implements ICoreDependencyFactory {
    private static instance: CoreProvider;
    private coreDependency: ICoreDependencyFactory;

    constructor(coreDependency: ICoreDependencyFactory) {
        this.coreDependency = coreDependency;
    }

    static initialize(coreDependency: ICoreDependencyFactory): void {
        if (CoreProvider.instance) {
            throw new Error('CoreProvider is already initialized.');
        }
        CoreProvider.instance = new CoreProvider(coreDependency);
    }

    public static getInstance(): CoreProvider {
        if (!CoreProvider.instance) {
            throw new Error('CoreProvider is not initialized. Call initialize() first.');
        }
        return CoreProvider.instance;
    }
    
    // Provides access to all core dependencies
    getRepoDiscovery(): IRepoDiscovery { /* ... */ }
    getDB(): IDatabase { /* ... */ }
    getGraphAPI(): IGraphAPI { /* ... */ }
    getErrorFactory(): IErrorFactory { /* ... */ }
    getFileStorage(): IFileStorage { /* ... */ }
    getMailer(): IMailer { /* ... */ }
    getBlockchain(): IBlockchain { /* ... */ }
    getIdentityService(): IIdentityService { /* ... */ }
    getAuthorizationService(): Promise<IAuthorizationService> { /* ... */ }
}
```

**Real Usage in Logic Classes**:
```typescript
// From src/core/logics/v2/products/crud/products-create.logic.ts:21-22
this.coreProvider = CoreProvider.getInstance();
this.errorFactory = this.coreProvider.getErrorFactory();
```

**Why This Pattern**:
- ✅ **Global Access**: Any logic class can access core dependencies
- ✅ **Singleton Safety**: Prevents multiple instances of critical services
- ✅ **Lazy Loading**: Dependencies created only when needed
- ✅ **Testability**: Can mock the entire provider in tests

#### Application Provider: Application-Level Dependencies

The [`ApplicationProvider`](src/application/providers/application.provider.ts:7) manages application-specific dependencies:

```typescript
// From src/application/providers/application.provider.ts:7-44
export class ApplicationProvider implements IApplicationDependencyFactory {
    private static instance: ApplicationProvider;
    private applicationDependency: IApplicationDependencyFactory;

    constructor(applicationDependency: IApplicationDependencyFactory) {
        this.applicationDependency = applicationDependency;
    }

    static initialize(applicationDependency: IApplicationDependencyFactory): void {
        if (ApplicationProvider.instance) {
            throw new Error('ApplicationProvider is already initialized.');
        }
        ApplicationProvider.instance = new ApplicationProvider(applicationDependency);
    }

    public static getInstance(): ApplicationProvider {
        if (!ApplicationProvider.instance) {
            throw new Error('ApplicationProvider is not initialized. Call initialize() first.');
        }
        return ApplicationProvider.instance;
    }

    getDB(): IDatabase { return this.applicationDependency.getDB(); }
    getServer(): IServer { return this.applicationDependency.getServer(); }
    getRouter(): IRouter { return this.applicationDependency.getRouter(); }
    getHttpClient(): IHttpClient { return this.applicationDependency.getHttpClient(); }
}
```

### 4. Request Flow with Actual Code References

#### Complete Request Journey: POST /api/v1/products

**Step 1: HTTP Request Enters Express**
```typescript
// From src/infra/adapters/express/express.adapter.ts (conceptual)
app.post('/api/v1/products', async (req, res) => {
    // Express converts HTTP to internal request format
    const request: IRequest = {
        method: req.method,
        path: req.path,
        body: req.body,
        query: req.query,
        headers: req.headers,
        user: req.user
    };
});
```

**Step 2: Authentication Middleware Validates Token**
```typescript
// From src/infra/adapters/middlewares/authenticator.middleware.ts (reference)
// Located at src/bootstrap.ts:496-498
const authenticatorMiddleware = new AuthenticatorMiddleware(
    this.dependencyFactory.getIdentityService()
);
```
**What it does**:
- Extracts JWT token from `Authorization: Bearer <token>` header
- Validates token with Keycloak service
- Attaches user object to request: `request.user = { id, roles, permissions }`

**Step 3: Validation Middleware Checks Schema**
```typescript
// From src/bootstrap.ts:573-580
if (validationSchemaMap[resource.name]?.[httpMethod]) {
    const schema = validationSchemaMap[resource.name][httpMethod];
    middlewares.push(new ValidatorMiddleware(new JoiAdapter(schema)));
}
```
**Uses schema from**: [`src/core/schemas/resource/products/post/create-products.schema.ts`](src/core/schemas/resource/products/post/create-products.schema.ts:1)

**Step 4: Controller Receives Request**
```typescript
// From src/application/adapters/controller/base-resource.controller.ts (reference)
// Created at src/bootstrap.ts:494
const resourceBaseController = new BaseController(
    resourceBaseService,
    graphQuery,
    resource.name
);

// Controller method (conceptual from codebase pattern)
async create(request: IRequest): Promise<IResponse> {
    try {
        const result = await this.service.create(request);
        return this.formatResponse(result);
    } catch (error) {
        return this.handleError(error);
    }
}
```

**Step 5: Service Layer with Logic Lookup**
```typescript
// From src/application/adapters/service/base-resource.service.ts (reference)
// Created at src/bootstrap.ts:484-492
const resourceBaseService = new BaseService(
    resourceBaseRepoAdapter,
    resource.name,
    {
        ...logicMap['v1'][resource.name]?.action,
        ...logicMap['v1'][resource.name]?.crud
    }
);

// Service checks logic map (conceptual pattern)
async create(request: IRequest) {
    const LogicClass = this.logicMap['create'];
    if (LogicClass) {
        // Custom business logic exists
        const logic = new LogicClass(request, this.repository);
        return await logic.execute();
    }
    // Fallback to repository
    return await this.repository.create(request.body);
}
```

**Step 6: Logic Class Executes Business Rules**
```typescript
// From src/core/logics/v2/products/crud/products-create.logic.ts:12-37
export class ProductsCreateLogic implements ILogic {
    request: IRequest;
    repository: ICoreRepository;
    errorFactory: IErrorFactory;
    coreProvider: CoreProvider;

    constructor(request: IRequest, repository: ICoreRepository) {
        this.request = request;
        this.repository = repository;
        this.coreProvider = CoreProvider.getInstance(); // Line 21
        this.errorFactory = this.coreProvider.getErrorFactory(); // Line 22
    }

    async execute() {
        const payload = this.request.body;

        // Business Rule: GFR451 - Product must be in Draft status
        if (!isValidStatusForANewProduct(payload.status)) {
            throw this.errorFactory.create(
                ErrorType.BAD_REQUEST,
                'Product status must be draft upon creation'
            );
        }

        // Create product via repository
        const productData = await this.repository.create(payload);
        
        if (!productData.success) {
            throw this.errorFactory.create(
                ErrorType.BAD_REQUEST,
                'Product not created!'
            );
        }
        
        return productData;
    }
}
```

**Key Insights from Actual Code**:
1. **CoreProvider Usage** (Line 21): Logic classes access dependencies via singleton
2. **Error Factory** (Line 22): Consistent error creation across application
3. **Business Rules** (Line 28-30): Domain validation before persistence
4. **Repository Abstraction** (Line 33): Logic doesn't know about database implementation

**Step 7: Repository Persists Data**
```typescript
// From src/infra/adapters/repository/drizzle-orm.repository.ts (reference)
// Created at src/bootstrap.ts:473-477
const resourceORMRepository = new DrizzleORMRepository(
    database,
    resource.name,
    ResourceModelMap[resource.name]
);

// Repository create method (conceptual from codebase pattern)
async create(data: object | object[]): Promise<RepositoryResult> {
    try {
        const isBulk = Array.isArray(data);
        const res = await this.dbClient.insert(this.table).values(data).returning();
        return { success: true, data: isBulk ? res : res[0] };
    } catch (error) {
        return handleDBError(error);
    }
}
```

### 5. Validation Schema System with Actual Examples

#### Centralized Validation Schema Map

```typescript
// From src/bootstrap.ts:170-318
const validationSchemaMap: ValidationSchemaMap = {
    Products: {
        post: createProductsSchema,  // Line 200
        put: updateProductsSchema    // Line 201
    },
    Categories: {
        post: createCategoriesSchema,  // Line 196
        put: updateCategoriesSchema    // Line 197
    },
    Templates: {
        post: createTemplatesSchema,   // Line 185
        put: updateTemplatesSchema     // Line 186
    },
    // ... 30+ more resources
};
```

**How Validation Is Applied**:
```typescript
// From src/bootstrap.ts:572-580
if (validationSchemaMap[resource.name]?.[httpMethod]) {
    const schema = validationSchemaMap[resource.name][httpMethod];
    if (schema) {
        middlewares.push(new ValidatorMiddleware(new JoiAdapter(schema)));
    }
}
```

**Real Validation Schema Example**:
```typescript
// Referenced from src/core/schemas/resource/products/post/create-products.schema.ts
export const createProductsSchema = Joi.object({
    name: Joi.string().required().min(1).max(255),
    description: Joi.string().allow('').max(1000),
    categoryId: Joi.number().integer().positive().required(),
    status: Joi.string().valid('draft', 'published'),
    metadata: Joi.object().default({})
});
```

### 6. Logic Map System: Custom Business Logic Registration

#### Understanding the Logic Map Architecture

The logic map connects resource operations to custom business logic classes:

```typescript
// From src/application/mapper/logic-map.ts (referenced at src/bootstrap.ts:78)
import { ProductsCreateLogic } from 'src/core/logics/v2/products/crud/products-create.logic';

export const logicMap = {
    'v1': {
        'products': {
            'crud': {
                'create': ProductsCreateLogic  // Maps 'create' to logic class
            },
            'action': {
                'feature:post': ProductsFeatureLogic  // Custom action
            }
        },
        'dpps': {
            'crud': {
                'create': DppsCreateLogic
            }
        }
        // ... more resources
    }
};
```

ts

↓ [AuthenticatorMiddleware validates JWT]
    File: src/infra/adapters/middlewares/authenticator.middleware.ts
    Created at: src/bootstrap.ts:496-498
    Action: Calls this.identityService.verifyToken(token)
    Result: request.user = { id: "user123", roles: ["admin"] }

↓ [ValidatorMiddleware validates schema]
    File: src/infra/adapters/middlewares/validator.middleware.ts
    Schema: src/core/schemas/resource/products/post/create-products.schema.ts
    Registered at: src/bootstrap.ts:170-318 (validationSchemaMap)
    Applied at: src/bootstrap.ts:573-580
    Action: Validates request.body against createProductsSchema
    Result: request.body validated and sanitized

↓ [Router matches route to controller]
    File: src/application/adapters/router/api.router.ts
    Route added at: src/bootstrap.ts:582-591
    Action: Matches POST /api/v1/products to BaseController.create()

↓ [BaseController.create() called]
    File: src/application/adapters/controller/base-resource.controller.ts
    Created at: src/bootstrap.ts:494
    Action: Delegates to service.create(request)

↓ [BaseService.create() looks up logic]
    File: src/application/adapters/service/base-resource.service.ts
    Created at: src/bootstrap.ts:484-492
    Logic Map: Injected with { create: ProductsCreateLogic }
    Action: Finds ProductsCreateLogic in logic map
    Decision: Custom logic exists, instantiate it

↓ [ProductsCreateLogic.execute() runs]
    File: src/core/logics/v2/products/crud/products-create.logic.ts:25-37
    Constructor: Lines 18-23
    - Gets CoreProvider.getInstance() (line 21)
    - Gets ErrorFactory (line 22)
    Execute: Lines 25-37
    - Validates business rule (line 28-32)
    - Calls repository.create() (line 33)
    - Handles errors (line 34-36)

↓ [Repository persists to database]
    File: src/infra/adapters/repository/drizzle-orm.repository.ts
    Created at: src/bootstrap.ts:473-477
    Action: Uses Drizzle ORM to insert into PostgreSQL
    Returns: { success: true, data: { id: 123, name: "Product", ... } }

↓ [Response flows back up]
    Logic → Service → Controller → Express → Client
    
FINAL RESPONSE
    Status: 201 Created
    Body: {
        success: true,
        data: { id: 123, name: "Product", categoryId: 1, status: "draft" },
        message: "Product created successfully"
    }
```

### 16. Infrastructure Adapters Deep Dive

#### Adapter Pattern Implementation Across Services

**Database Adapter**:
```typescript
// From src/infra/factories/dependency-factory.ts:136-141
createDatabase(): IDatabase {
    if (!this.db) {
        this.db = new PostgresDB({ url: this.config.database.url });
    }
    return this.db;
}
```
**Interface**: `IDatabase`
**Implementation**: `PostgresDB`
**ORM**: Drizzle
**Location**: src/infra/adapters/database/postgres.database.ts

**File Storage Adapter**:
```typescript
// From src/infra/factories/dependency-factory.ts:318-323
createFileStorage(): IFileStorage {
    if (!this.fileStorage) {
        this.fileStorage = new FileStorageAdapter(
            new AzureStorageAdapter(getAzureStorageConfig())
        );
    }
    return this.fileStorage;
}
```
**Interface**: `IFileStorage`
**Adapter Chain**: `FileStorageAdapter` → `AzureStorageAdapter`
**Provider**: Azure Blob Storage
**Location**: src/infra/adapters/file-storage/file-storage.adapter.ts

**Mailer Adapter**:
```typescript
// From src/infra/factories/dependency-factory.ts:309-316
createMailer(): IMailer {
    if (!this.mailer) {
        const nodemailerAdapter = new NodemailerAdapter(translations);
        this.mailer = new MailerAdapter(nodemailerAdapter);
    }
    return this.mailer;
}
```
**Interface**: `IMailer`
**Adapter Chain**: `MailerAdapter` → `NodemailerAdapter`
**Provider**: SMTP via Nodemailer
**Location**: src/infra/adapters/mailer/mailer.adapter.ts
**Translations**: src/core/resources/mailer/translations

### 17. Event Streaming Architecture

#### Server-Sent Events (SSE) Setup

```typescript
// From src/infra/factories/dependency-factory.ts:302-307
createEventStream(): IEventStream {
    if (!this.eventStream) {
        this.eventStream = new SseAdapter();
    }
    return this.eventStream;
}
```

**Special SSE Route Handling**:
```typescript
// From src/bootstrap.ts:503-520
if (route.url === '/api/events/connect') {
    const sseController = new SSEController(
        this.dependencyFactory.getEventStream(),
        resource.name
    );
    const _route = {
        path: route.url,
        method: route.method,
        action: 'handleConnection',
        actionName: 'connect:post',
        controller: sseController,
        middlewares: []
    };
    router.addRoute(_route as Route);
    return;  // Special handling, skip normal flow
}
```

**Why Special Handling**:
- SSE requires persistent HTTP connection (not request-response)
- Different controller type (SSEController vs BaseController)
- No standard middleware pipeline needed

### 18. Configuration Management System

#### Environment-Based Configuration

```typescript
// From src/bootstrap.ts:383-385
setupConfig(): void {
    this.config = AppConfig.getInstance().getConfig();
}
```

**Configuration Hierarchy**:
```
1. Environment Variables (process.env)
    ↓
2. AppConfig.getInstance() - Singleton pattern
    ↓
3. Environment-specific config files (dev/stage/prod)
    ↓
4. Merged final configuration object
```

**Used Throughout Bootstrap**:
```typescript
// From src/bootstrap.ts:392-437
const factoryConfig = {
    database: {
        url: this.config.POSTGRES_DB_URI!  // From environment
    },
    server: {
        port: Number(this.config.PORT),    // From environment
        host: getLocalIP()                 // Detected automatically
    },
    KEYCLOAK_HOST_URI: this.config.KEYCLOAK_HOST_URI!,
    SMTP_HOST: this.config.SMTP_HOST!,
    blockchain: {
        network: this.config.POLYGON_NETWORK,
        rpcUrl: this.config.POLYGON_RPC_ENDPOINT
    }
};
```

### 19. Logging System Architecture

#### Structured Logging Implementation

```typescript
// From src/bootstrap.ts:109-141
class ConsoleLogger implements ILogger {
    info(message: string): void {
        console.log(chalk.blue(`ℹ ${message}`));
    }

    error(message: string, error?: Error): void {
        console.error(chalk.red(`✖ ${message}`), error);
    }

    warn(message: string): void {
        console.warn(chalk.yellow(`⚠ ${message}`));
    }

    debug(message: string): void {
        if (process.env.NODE_ENV === 'dev') {
            // Formatted debug output with visual hierarchy
            const [action, ...rest] = message.split(':');
            if (action.includes('Processing')) {
                console.debug(chalk.cyan(`\n📦 ${message}`));
            } else if (action.includes('Adding')) {
                console.debug(chalk.green(`  └─ ${rest.join(':').trim()}`));
            } else {
                console.debug(chalk.gray(`  ${message}`));
            }
        }
    }
}
```

**Usage Throughout Bootstrap**:
```typescript
// From src/bootstrap.ts:338-342
this.logger.info('🚀 Starting BYO DPP Application');
this.setup();
this.build();
this.boot();
this.logger.info('✨ Application is ready');
```

**Logging at Different Stages**:
- **Info**: Major lifecycle events (startup, shutdown)
- **Debug**: Detailed route registration (line 517, 556, 590)
- **Error**: Failures with stack traces (line 345, 370, 599)
- **Warn**: Non-critical issues

### 20. Validation Schema System Explained

#### Schema Organization by Resource and Method

```typescript
// From src/bootstrap.ts:170-318
const validationSchemaMap: ValidationSchemaMap = {
    Products: {
        post: createProductsSchema,   // POST /products (create)
        put: updateProductsSchema     // PUT /products/:id (update)
    },
    Templates: {
        post: createTemplatesSchema,
        put: updateTemplatesSchema
    },
    Dpps: {
        post: createDppsSchema,
        put: updateDppsSchema
    }
    // ... 30+ more resources
};
```

**Import Structure**:
```typescript
// From src/bootstrap.ts:36-37
import { createProductsSchema } from './core/schemas/resource/products/post/create-products.schema';
import { updateProductsSchema } from './core/schemas/resource/products/put/update-products.schema';
```

**Schema File Organization**:
```
src/core/schemas/resource/
├── products/
│   ├── post/
│   │   └── create-products.schema.ts  → createProductsSchema
│   └── put/
│       └── update-products.schema.ts  → updateProductsSchema
├── categories/
│   ├── post/
│   │   └── create-categories.schema.ts
│   └── put/
│       └── update-categories.schema.ts
```

**How Validation Is Applied**:
```typescript
// From src/bootstrap.ts:572-580
if (route.method === 'batch') {
    middlewares.push(new BatchMiddleware(JoiAdapter, validationSchemaMap[resource.name]));
} else if (validationSchemaMap[resource.name]?.[httpMethod]) {
    const schema = validationSchemaMap[resource.name][httpMethod];
    if (schema) {
        middlewares.push(new ValidatorMiddleware(new JoiAdapter(schema)));
    }
}
```

### 21. HTTP Method Mapping System

#### Understanding Method Translation

```typescript
// From src/bootstrap.ts:145-166
const actionMap: Record<ValidMethod, string> = {
    list: 'list',
    post: 'create',
    get: 'read',
    put: 'update',
    delete: 'delete',
    patch: 'patch',
    batch: 'batch'
};

const httpMethodMap = {
    list: 'get',      // list operation → HTTP GET
    create: 'post',   // create operation → HTTP POST
    read: 'get',      // read operation → HTTP GET
    update: 'put',    // update operation → HTTP PUT
    delete: 'delete', // delete operation → HTTP DELETE
    patch: 'patch',   // patch operation → HTTP PATCH
    batch: 'post'     // batch operation → HTTP POST
} as const;
```

**Usage in Route Registration**:
```typescript
// From src/bootstrap.ts:582-591
const _route = {
    path: route.url,
    method: httpMethodMap[route.method],  // Translates 'create' → 'post'
    action: actionMap[route.method],      // Maps to service method name
    controller: resourceBaseController,
    middlewares: middlewares
};
router.addRoute(_route as Route);
```

**Why This Mapping**:
- ✅ **Semantic Clarity**: Business operations (create) vs HTTP verbs (POST)
- ✅ **Flexibility**: Can change HTTP mapping without changing business logic
- ✅ **Consistency**: Same mapping rules across all resources

### 22. Error Handling Flow with Code References

#### Error Flow Through Layers

```typescript
// LAYER 4: Logic Layer (Domain)
// From src/core/logics/v2/products/crud/products-create.logic.ts:28-32
if (!isValidStatusForANewProduct(payload.status)) {
    throw this.errorFactory.create(
        ErrorType.BAD_REQUEST,
        'Product status must be draft upon creation'
    );
}
```
**Error Type**: Business rule violation
**Handler**: Logic layer throws domain error

```typescript
// LAYER 3: Service Layer (Application)
// Conceptual from base-resource.service.ts pattern
async create(request: IRequest) {
    try {
        const LogicClass = this.logicMap['create'];
        const logic = new LogicClass(request, this.repository);
        return await logic.execute();
    } catch (error) {
        // Service may add context but typically passes error up
        throw error;
    }
}
```

```typescript
// LAYER 2: Controller Layer (Application)
// Conceptual from base-resource.controller.ts pattern
async create(request: IRequest): Promise<IResponse> {
    try {
        const result = await this.service.create(request);
        return this.formatResponse(result);
    } catch (error) {
22
│   Execute method: Lines 25-37
│   ├─→ Validates: isValidStatusForANewProduct(payload.status) - line 28
│   ├─→ Business Rule: Status must be "draft" for new products (GFR451)
│   ├─→ Calls: repository.create(payload) - line 33
│   └─→ Error Handling: Lines 34-36
│
├─→ DrizzleORMRepository.create() (src/infra/adapters/repository/drizzle-orm.repository.ts)
│   Created at: src/bootstrap.ts:473-477
│   ├─→ Uses: Drizzle ORM to build SQL INSERT
│   ├─→ Executes: INSERT INTO products (...) VALUES (...) RETURNING *
│   └─→ Returns: { success: true, data: { id, name, ... } }
│
└─→ Response Flows Back
    Logic → Service → Controller → Express → Client
    
    HTTP/1.1 201 Created
    {
        "success": true,
        "data": { "id": 123, "name": "Product", "categoryId": 1, "status": "draft" }
    }
```

### 33. Initialization Order and Why It Matters

#### Critical Dependencies First

```typescript
// From src/bootstrap.ts:360-373
setup(): void {
    this.setupConfig();              // 1️⃣ Must be first - everything needs config
    this.setupBootstrapJson();       // 2️⃣ Needs config to find JSON file
    this.setupDependencyFactory();   // 3️⃣ Needs config for service URLs
    this.setupResponseMessage();     // 4️⃣ Needs bootstrap JSON for messages
    this.setupCoreProvider();        // 5️⃣ Needs dependency factory
    this.setupApplicationProvider(); // 6️⃣ Needs dependency factory
    this.setupApplication();         // 7️⃣ Needs providers to be ready
}
```

**Why This Exact Order**:
1. **Config** → Contains all connection strings, credentials
2. **Bootstrap JSON** → Defines what resources exist
3. **Dependency Factory** → Creates database, services using config
4. **Response Messages** → Needs resource definitions from JSON
5. **Core Provider** → Wraps factory for global access
6. **Application Provider** → Wraps factory for app-level access
7. **Application** → Wires server and router (needs everything above)

**What Breaks If Order Changes**:
- Config after Factory → Factory can't connect to database (no URL)
- Providers before Factory → Providers have nothing to wrap
- Application before Providers → Controllers can't access dependencies

### 34. Complete Feature Implementation Checklist

Use this checklist when adding any new feature to BYO DPP:

#### Adding New Resource

**Step 1: Database Schema** 
```typescript
// File: src/infra/schemas/drizzle-pg-orm/byo-dpp/<resource>.schema.ts
export const invoices = pgTable('invoices', {
    id: serial('id').primaryKey(),
    // ... fields
});

// Add to exports in src/infra/schemas/drizzle-pg-orm/byo-dpp/index.ts
export { invoices } from './invoices.schema';
```

**Step 2: Resource Model Mapping**
```typescript
// File: src/infra/constants/byo-dpp/resource-model-map.ts
export const ResourceModelMap = {
    // ... existing
    Invoices: invoices  // Add new mapping
};
```

**Step 3: Validation Schemas**
```typescript
// File: src/core/schemas/resource/invoices/post/create-invoices.schema.ts
export const createInvoicesSchema = Joi.object({
    // ... validation rules
});

// File: src/core/schemas/resource/invoices/put/update-invoices.schema.ts
export const updateInvoicesSchema = Joi.object({
    // ... validation rules
});
```

**Step 4: Register in Bootstrap**
```typescript
// File: src/bootstrap.ts
import { createInvoicesSchema } from './core/schemas/resource/invoices/post/create-invoices.schema';
import { updateInvoicesSchema } from './core/schemas/resource/invoices/put/update-invoices.schema';

// Add to validationSchemaMap (around line 170-318)
const validationSchemaMap: ValidationSchemaMap = {
    // ... existing
    Invoices: {
        post: createInvoicesSchema,
        put: updateInvoicesSchema
    }
};
```

**Step 5: Update Bootstrap JSON**
```json
// File: input/latest/byo-dpp-data.json
{
    "resources": [
        {
            "name": "Invoices",
            "type": "database",
            "routes": {
                "crud": ["list", "create", "read", "update", "delete"],
                "actions": []
            }
        }
    ]
}
```

**Step 6: Create Logic (Optional)**
```typescript
// File: src/core/logics/v2/invoices/crud/invoices-create.logic.ts
export class InvoicesCreateLogic implements ILogic {
    request: IRequest;
    repository: ICoreRepository;
    coreProvider: CoreProvider;

    constructor(request: IRequest, repository: ICoreRepository) {
        this.request = request;
        this.repository = repository;
        this.coreProvider = CoreProvider.getInstance();
    }

    async execute() {
        // Custom business rules here
        return await this.repository.create(this.request.body);
    }
}

// File: src/application/mapper/logic-map.ts
import { InvoicesCreateLogic } from 'src/core/logics/v2/invoices/crud/invoices-create.logic';

export const logicMap = {
    'v1': {
        // ... existing
        'invoices': {
            'crud': {
                'create': InvoicesCreateLogic
            }
        }
    }
};
```

### 35. Debugging Architecture with Code References

#### Where to Look When Things Go Wrong

**Problem: Route Not Found (404)**
```
Check:
1. src/bootstrap.ts:465 - Is resource in byoDPPJson?
2. input/latest/byo-dpp-data.json - Is route defined?
3. src/bootstrap.ts:582-591 - Is route being registered?
4. Console logs - Look for "Adding route: POST /api/v1/<resource>"
```

**Problem: Validation Fails (400)**
```
Check:
1. src/bootstrap.ts:170-318 - Is schema in validationSchemaMap?
2. src/core/schemas/resource/<resource>/post/*.schema.ts - Schema definition
3. src/bootstrap.ts:573-580 - Is ValidationMiddleware being added?
4. Request body - Does it match schema requirements?
```

**Problem: Logic Not Executing**
```
Check:
1. src/application/mapper/logic-map.ts:78 - Is logic class imported?
2. Logic map structure - Is path correct? logicMap.v1.<resource>.crud.<operation>
3. src/bootstrap.ts:488-492 - Is logic map injected into service?
4. Service handleOperation - Is logic class being found?
```

**Problem: Repository Method Fails**
```
Check:
1. src/infra/constants/byo-dpp/resource-model-map.ts - Resource mapped?
2. src/bootstrap.ts:476 - ResourceModelMap[resource.name] exists?
3. Database schema - Does table exist?
4. Drizzle migrations - Are migrations applied?
```

**Problem: Authentication Fails (401)**
```
Check:
1. src/bootstrap.ts:496-498 - AuthenticatorMiddleware added?
2. Keycloak configuration - Lines 405-416 in bootstrap.ts
3. JWT token - Is it valid and not expired?
4. Identity service - src/infra/factories/dependency-factory.ts:207-244
```

### 36. Advanced: Multi-Version API Support (Future)

#### How to Add V2 API

```typescript
// Current limitation noted in code
// From src/bootstrap.ts:454
// FIXME: direct v1 access blocks scalability to more version and parallel versions
// TODO: allow multiple versions in parallel

// Future implementation pattern:
buildRoutes(): void {
    const apiVersion = 'v1';  // Make this dynamic
    
    this.byoDPPJson.forEach((service: Service) => {
        service.resources.forEach((resource: Resource) => {
            const resourceBaseService = new BaseService(
                resourceBaseRepoAdapter,
                resource.name,
                {
                    ...logicMap[apiVersion][resource.name]?.action,  // Dynamic version
                    ...logicMap[apiVersion][resource.name]?.crud
                }
            );
        });
    });
}

// Support multiple versions
export const logicMap = {
    'v1': {
        'products': {
            'crud': {
                'create': ProductsCreateLogicV1
            }
        }
    },
    'v2': {
        'products': {
            'crud': {
                'create': ProductsCreateLogicV2  // Breaking changes allowed
            }
        }
    }
};

// Routes become version-aware
router.addRoute({
    path: '/api/v1/products',  // Version in URL
    method: 'post',
    controller: controllerV1
});

router.addRoute({
    path: '/api/v2/products',  // Same resource, different version
    method: 'post',
    controller: controllerV2
});
```

### 37. Understanding Middleware Pipeline Execution

#### Middleware Chain with Actual References

```typescript
// From src/bootstrap.ts:522-538 (Action routes)
const middlewares: IMiddleware[] = [];
middlewares.push(authenticatorMiddleware);  // Always first

// From src/bootstrap.ts:563-580 (CRUD routes)
const middlewares: IMiddleware[] = [];
middlewares.push(authenticatorMiddleware);  // Authentication first

if (route.method === 'batch') {
    // Batch validation for bulk operations
    middlewares.push(new BatchMiddleware(JoiAdapter, validationSchemaMap[resource.name]));
} else if (validationSchemaMap[resource.name]?.[httpMethod]) {
    // Single item validation
    const schema = validationSchemaMap[resource.name][httpMethod];
    middlewares.push(new ValidatorMiddleware(new JoiAdapter(schema)));
}
```

**Execution Pattern** (from router/controller pattern):
```typescript
// Middleware chain execution
for (const middleware of middlewares) {
    await middleware.execute(request, response, next);
}

// If all middleware pass, controller is called
await controller[action](request);
```

**Pipeline Benefits**:
- ✅ **Composable**: Easy to add/remove middleware
- ✅ **Order Control**: Explicit ordering prevents issues
- ✅ **Reusable**: Same middleware across multiple routes
- ✅ **Testable**: Each middleware tested independently

### 38. Practical Architecture Patterns Reference

#### Pattern Catalog with File Locations

| Pattern | Implementation | File Reference | Line Numbers |
|---------|---------------|----------------|--------------|
| **Singleton** | CoreProvider | [`core.provider.ts`](src/core/providers/core.provider.ts:17) | 17-50 |
| **Singleton** | ApplicationProvider | [`application.provider.ts`](src/application/providers/application.provider.ts:7) | 7-28 |
| **Factory** | DependencyFactory | [`dependency-factory.ts`](src/infra/factories/dependency-factory.ts:106) | 106-418 |
| **Factory** | ErrorFactory | [`error.factory.ts`](src/infra/adapters/errors/error.factory.ts:1) | Referenced at dependency-factory.ts:296 |
| **Adapter** | ExpressAdapter | [`express.adapter.ts`](src/infra/adapters/express/express.adapter.ts:1) | Created at dependency-factory.ts:157 |
| **Adapter** | DrizzleORMRepository | [`drizzle-orm.repository.ts`](src/infra/adapters/repository/drizzle-orm.repository.ts:1) | Created at bootstrap.ts:473 |
| **Strategy** | Logic Classes | [`products-create.logic.ts`](src/core/logics/v2/products/crud/products-create.logic.ts:12) | 12-37 |
| **Registry** | RepoDiscovery | [`lite.repo-discovery.ts`](src/infra/adapters/repo-discovery/lite.repo-discovery.ts:1) | Registered at bootstrap.ts:482 |
| **Chain of Responsibility** | Middleware Pipeline | Various middleware files | Composed at bootstrap.ts:522-580 |
| **Template Method** | BaseController | [`base-resource.controller.ts`](src/application/adapters/controller/base-resource.controller.ts:1) | Created at bootstrap.ts:494 |

### 39. Configuration Values and Their Usage

#### Tracing Config from Environment to Usage

**Environment Variables** → **AppConfig** → **Bootstrap** → **Services**

```typescript
// 1. Environment (`.env` file)
POSTGRES_DB_URI=postgresql://user:pass@localhost:5432/byo_dpp
KEYCLOAK_HOST_URI=http://localhost:8080
SMTP_HOST=smtp.gmail.com

// 2. Loaded by AppConfig (referenced at bootstrap.ts:384)
this.config = AppConfig.getInstance().getConfig();

// 3. Used in DependencyFactory (bootstrap.ts:392-437)
const factoryConfig = {
    database: { url: this.config.POSTGRES_DB_URI! },  // Line 394
    KEYCLOAK_HOST_URI: this.config.KEYCLOAK_HOST_URI!,  // Line 403
    SMTP_HOST: this.config.SMTP_HOST!  // Line 420
};

// 4. Passed to Services (dependency-factory.ts)
createDatabase(): IDatabase {
    return new PostgresDB({ url: this.config.database.url });  // Uses from factoryConfig
}

createIdentityService(): IIdentityService {
    return new KeycloakService({
        baseURL: this.config.KEYCLOAK_HOST_URI  // Uses from factoryConfig
    });
}

createMailer(): IMailer {
    return new NodemailerAdapter({
        host: this.config.SMTP_HOST  // Uses from factoryConfig
    });
}
```

### 40. Understanding the Full Dependency Tree

```mermaid
graph TB
    Project[Project bootstrap.ts:321]
    
    subgraph Configuration["Phase 1: Configuration"]
        Config[AppConfig<br/>line 384]
        JSON[Bootstrap JSON<br/>line 388]
    end
    
    subgraph Factory["Phase 2: Dependency Factory"]
        DepFactory[DependencyFactory<br/>line 391-440]
        
        subgraph Core["Core Dependencies"]
            DB[(PostgresDB<br/>136-141)]
            Router[Router<br/>150-155]
            Server[Express Server<br/>157-177]
            Graph[GraphAPI<br/>179-184]
            RepoDisc[RepoDiscovery<br/>186-190]
            SSE[SseAdapter<br/>302-307]
            Mailer[MailerAdapter<br/>309-316]
            FileStore[FileStorage<br/>318-323]
            Blockchain[Blockchain<br/>143-148]
            ErrorFac[ErrorFactory<br/>296-300]
            Identity[Keycloak<br/>207-244]
            AuthSvc[Casbin<br/>247-256]
            IDGen[ULID Generator<br/>193-198]
            ExportImp[Export/Import<br/>325-330]
        end
    end
    
    subgraph Providers["Phase 3: Providers"]
        CoreProv[CoreProvider<br/>line 442-444]
        AppProv[ApplicationProvider<br/>line 446-448]
    end
    
    subgraph App["Phase 4: Application"]
        Application[Application<br/>line 350-358]
        Routes[Routes Builder<br/>line 456-601]
    end
    
    Project --> Config
    Project --> JSON
    Project --> DepFactory
    
    DepFactory --> DB
    DepFactory --> Router
    DepFactory --> Server
    DepFactory --> Graph
    DepFactory --> RepoDisc
    DepFactory --> SSE
    DepFactory --> Mailer
    DepFactory --> FileStore
    DepFactory --> Blockchain
    DepFactory --> ErrorFac
    DepFactory --> Identity
    DepFactory --> AuthSvc
    DepFactory --> IDGen
    DepFactory --> ExportImp
    
    DepFactory --> CoreProv
    DepFactory --> AppProv
    
    CoreProv --> Application
    AppProv --> Application
    Application --> Routes
    
    Routes --> Ready([Application Ready])
    
    style Project fill:#90EE90
    style Configuration fill:#F0E68C
    style Factory fill:#DDA0DD
    style Providers fill:#87CEEB
    style App fill:#FFB6C1
    style Ready fill:#90EE90
```


```
Project (bootstrap.ts:321)
    │
    ├─→ AppConfig (line 384)
    │   └─→ Environment Variables
    │
    ├─→ Bootstrap JSON (line 388)
    │   └─→ input/latest/byo-dpp-data.json
    │
    ├─→ DependencyFactory (line 391-440)
    │   ├─→ PostgresDB (dependency-factory.ts:136-141)
    │   │   └─→ Drizzle ORM
    │   │
    │   ├─→ Router (dependency-factory.ts:150-155)
    │   │   └─→ Express Router
    │   │
    │   ├─→ ExpressServer (dependency-factory.ts:157-177)
    │   │   ├─→ Express app
    │   │   ├─→ CORS middleware
    │   │   └─→ Swagger documentation
    │   │
    │   ├─→ GraphQueryORMAdapter (dependency-factory.ts:179-184)
    │   │   └─→ Query builder for complex filters
    │   │
    │   ├─→ RepoDiscoveryLite (dependency-factory.ts:186-190)
    │   │   └─→ Repository registry (Map<string, IRepository>)
    │   │
    │   ├─→ SseAdapter (dependency-factory.ts:302-307)
    │   │   └─→ Server-Sent Events for real-time
    │   │
    │   ├─→ MailerAdapter (dependency-factory.ts:309-316)
    │   │   └─→ NodemailerAdapter
    │   │       └─→ SMTP transport
    │   │
    │   ├─→ FileStorageAdapter (dependency-factory.ts:318-323)
    │   │   └─→ AzureStorageAdapter
    │   │       └─→ Azure Blob Storage
    │   │
    │   ├─→ BlockchainAdapter (dependency-factory.ts:143-148)
    │   │   └─→ PolygonAdapter
    │   │       └─→ Polygon (Amoy testnet)
    │   │
    │   ├─→ ErrorFactory (dependency-factory.ts:296-300)
    │   │   └─→ Creates typed errors
    │   │
    │   ├─→ IdentityServiceAdapter (dependency-factory.ts:207-244)
    │   │   └─→ KeycloakService
    │   │       ├─→ User management
    │   │       ├─→ Authentication (JWT)
    │   │       └─→ Authorization (RBAC)
    │   │
    │   ├─→ AuthorizationAdapter (dependency-factory.ts:247-256)
    │   │   └─→ CasbinService
    │   │       └─→ Policy-based access control
    │   │
    │   ├─→ UlidGeneratorService (dependency-factory.ts:193-198)
    │   │   └─→ Generates sortable unique IDs
    │   │
    │   └─→ ExportImporterProvider (dependency-factory.ts:325-330)
    │       └─→ CSV/JSON import/export strategies
    │
    ├─→ CoreProvider (line 442-444)
    │   └─→ Wraps DependencyFactory with CoreDependencyFactoryProxy
    │       Purpose: Global access to core dependencies
    │
    ├─→ ApplicationProvider (line 446-448)
    │   └─→ Wraps DependencyFactory with ApplicationDependencyFactoryProxy
    │       Purpose: Application-level dependency access
    │
    └─→ Application (line 350-358)
        └─→ new Application(server, router)
            Purpose: Wires server with routes and starts listening
```

### 41. Key Takeaways for Developers

#### Mental Model for Working with BYO DPP

**When Adding Business Logic**:
```
Question: "Do I need custom logic?"
├─ Simple CRUD? → NO
│  └─→ Service will use repository directly (automatic)
│
└─ Complex rules? → YES
   └─→ Create logic class
       1. File: src/core/logics/v2/<resource>/crud/<resource>-<operation>.logic.ts
       2. Implement ILogic interface
       3. Use CoreProvider.getInstance() for dependencies
       4. Register in logic-map.ts
       5. Service will automatically use it
```

**When Adding Validation**:
```
Question: "What level of validation?"
├─ Schema-level (type, format, required)?
│  └─→ Add to Joi schema
│      Location: src/core/schemas/resource/<resource>/post/create-<resource>.schema.ts
│      Register: src/bootstrap.ts validationSchemaMap (line 170-318)
│      Applied: Automatically by ValidatorMiddleware (line 573-580)
│
└─ Business-level (relationships, constraints)?
   └─→ Add to logic class
       Location: src/core/logics/v2/<resource>/crud/<resource>-<operation>.logic.ts
       Example: ProductsCreateLogic validates status (line 28-32)
```

**When Accessing Other Resources**:
```
In any logic class:

// Get repository for any resource
const otherRepo = this.coreProvider
    .getRepoDiscovery()
    .getRepository('ResourceName');

// Query it
const result = await otherRepo.read({ filter: { id: someId } });

// Process
if (result.success) {
    const data = result.data;
}
```

### 42. Bootstrap Performance Optimization

#### Understanding Lazy Initialization

All dependencies use lazy initialization pattern:

```typescript
// From src/infra/factories/dependency-factory.ts:136-141
createDatabase(): IDatabase {
    if (!this.db) {  // Check if already created
        this.db = new PostgresDB({ url: this.config.database.url });
    }
    return this.db;  // Return existing or newly created
}
```

**Benefits**:
- ✅ **Faster Startup**: Only create what's needed
- ✅ **Memory Efficient**: Don't load unused services
- ✅ **Initialization Control**: Services created in dependency order
- ✅ **Error Isolation**: Failed service doesn't block others

**Applied Throughout**:
- Database (line 136-141)
- Router (line 150-155)
- Server (line 157-177)
- All services follow this pattern

### 43. Architecture Compliance Checklist

Use this to ensure your code follows BYO DPP architecture:

#### Code Review Checklist

**Layer Compliance**:
- [ ] Controllers only handle HTTP concerns
- [ ] Services only orchestrate (no business rules)
- [ ] Logic classes contain business rules
- [ ] Repositories only handle data access
- [ ] No layer skipping (no controller → repository direct)

**Dependency Management**:
- [ ] Dependencies injected via constructor
- [ ] Interfaces used at layer boundaries
- [ ] CoreProvider used for global dependencies
- [ ] No `new` keyword for services (use factory/provider)

**Error Handling**:
- [ ] Use ErrorFactory.create() for errors
- [ ] Catch errors at appropriate layer
- [ ] Log errors with context
- [ ] Return consistent error format

**Code Organization**:
- [ ] Logic classes in src/core/logics/v2/<resource>/
- [ ] Schemas in src/core/schemas/resource/<resource>/
- [ ] Registered in appropriate map (logicMap, validationSchemaMap)
- [ ] Tests co-located with logic (\_\_tests\_\_ folder)

**Configuration**:
- [ ] No hardcoded values (use config)
- [ ] Environment variables for secrets
- [ ] Config accessed via AppConfig.getInstance()
- [ ] Defaults provided for optional config

### 44. Architecture Documentation Cross-Reference

#### Key Files and Their Purposes

| File | Purpose | Dependencies | Created By | Used By |
|------|---------|--------------|------------|---------|
| [`bootstrap.ts`](src/bootstrap.ts:321) | Application entry point | All | main | - |
| [`dependency-factory.ts`](src/infra/factories/dependency-factory.ts:106) | Creates all services | Config | Bootstrap:391 | Providers |
| [`core.provider.ts`](src/core/providers/core.provider.ts:17) | Global dependency access | DependencyFactory | Bootstrap:442 | All logic classes |
| [`application.provider.ts`](src/application/providers/application.provider.ts:7) | App-level dependencies | DependencyFactory | Bootstrap:446 | Controllers |
| [`products-create.logic.ts`](src/core/logics/v2/products/crud/products-create.logic.ts:12) | Product creation logic | CoreProvider | Service layer | BaseService |
| [`logic-map.ts`](src/application/mapper/logic-map.ts:1) | Logic class registry | Logic classes | Bootstrap:78 | Bootstrap:488 |
| [`resource-model-map.ts`](src/infra/constants/byo-dpp/resource-model-map.ts:1) | Resource-to-schema map | Drizzle schemas | Bootstrap:29 | Bootstrap:476 |

### 45. Complete Architecture Learning Path

#### Week-by-Week Deep Dive

**Week 1: Foundation**
```
Day 1-2: Read and understand bootstrap.ts
- Lines 336-348: run() method - entry point
- Lines 360-373: setup() method - initialization
- Lines 456-601: buildRoutes() - dynamic registration

Day 3-4: Understand providers
- core.provider.ts:17-50 - Singleton pattern
- application.provider.ts:7-44 - Provider pattern
- How logic classes use providers (products-create.logic.ts:21-22)

Day 5: Trace a complete request
- Follow product creation from HTTP to database
- Use line numbers provided in section 32
```

**Week 2: Implementation**
```
Day 1-2: Create a new logic class
- Copy products-create.logic.ts as template
- Implement custom business rules
- Register in logic-map.ts

Day 3-4: Add validation schema
- Create in src/core/schemas/resource/<resource>/post/
- Register in validationSchemaMap (bootstrap.ts:170-318)
- Test with actual HTTP requests

Day 5: Understand cross-resource access
- Use CoreProvider.getRepoDiscovery()
- Access other repositories
- Implement complex multi-resource logic
```

**Week 3: Advanced Patterns**
```
Day 1-2: Study dependency injection
- How DependencyFactory creates services
- How providers wrap factory
- Why lazy initialization matters

Day 3-4: Understand error flow
- ErrorFactory pattern
- Error handling at each layer
- Conversion to HTTP responses

Day 5: Master the architecture
- Explain architecture to another developer
- Review PRs with architectural lens
- Suggest improvements
```

### 46. Architecture Evolution History

#### Design Decisions Timeline

**V1.0: Initial Architecture**
- Layered architecture established
- Repository pattern implemented
- Basic DI with manual wiring

**V1.5: Provider Pattern**
- Added CoreProvider (core.provider.ts:33)
- Added ApplicationProvider (application.provider.ts:15)
- Centralized dependency access

**V2.0: Dynamic Registration**
- JSON-driven resource definition (bootstrap.ts:388)
- Logic map system (logic-map.ts, bootstrap.ts:78)
- Automatic route generation (bootstrap.ts:456-601)

**V2.5: Current State**
- Multiple validation strategies
- Batch processing support
- Event streaming (SSE)
- Blockchain integration

**Future (V3.0)**:
- Multi-version API support (bootstrap.ts:454 - TODO)
- Microservices decomposition
- GraphQL alongside REST
- Advanced caching strategies

### 47. Quick Reference: File Locations by Concern

#### Where to Find Things

**Adding Business Logic**:
- Logic Class: `src/core/logics/v2/<resource>/crud/<operation>.logic.ts`
- Logic Registration: `src/application/mapper/logic-map.ts:78`
- Logic Map Usage: `src/bootstrap.ts:488-492`

**Adding Validation**:
- Schema Definition: `src/core/schemas/resource/<resource>/post/create-<resource>.schema.ts`
- Schema Registration: `src/bootstrap.ts:170-318` (validationSchemaMap)
- Schema Application: `src/bootstrap.ts:573-580` (middleware)

**Adding New Resource**:
- Database Schema: `src/infra/schemas/drizzle-pg-orm/byo-dpp/<resource>.schema.ts`
- Model Mapping: `src/infra/constants/byo-dpp/resource-model-map.ts`
- Bootstrap JSON: `input/latest/byo-dpp-data.json`

**Configuring Services**:
- Environment: `.env` file
- Config Loading: `src/config/` (referenced at bootstrap.ts:384)
- Factory Config: `src/bootstrap.ts:392-437`
- Service Creation: `src/infra/factories/dependency-factory.ts:136-418`

**Understanding Flow**:
- Bootstrap Entry: `src/bootstrap.ts:336-348`
- Route Building: `src/bootstrap.ts:456-601`
- Middleware Chain: `src/bootstrap.ts:522-538, 563-580`
- Controller Logic: `src/application/adapters/controller/base-resource.controller.ts`
- Service Logic: `src/application/adapters/service/base-resource.service.ts`
- Domain Logic: `src/core/logics/v2/<resource>/`
- Repository: `src/infra/adapters/repository/drizzle-orm.repository.ts`

### 48. Final Architecture Wisdom

#### Principles Demonstrated in Code

**1. Fail Fast, Fail Loud**
```typescript
// From src/bootstrap.ts:336-348
run(): void {
    try {
        this.logger.info('🚀 Starting BYO DPP Application');
        this.setup();
        this.build();
        this.boot();
    } catch (error) {
        this.logger.error('Failed to start application', error as Error);
        process.exit(1);  // Exit immediately on failure
    }
}
```
**Why**: Better to fail at startup than run with broken state

**2. Single Responsibility Everywhere**
- **Bootstrap**: Only orchestrates initialization
- **DependencyFactory**: Only creates dependencies
- **Providers**: Only provide access
- **Services**: Only orchestrate
- **Logic**: Only business rules
- **Repository**: Only data access

**3. Explicit Dependencies**
```typescript
// From src/core/logics/v2/products/crud/products-create.logic.ts:18-23
constructor(request: IRequest, repository: ICoreRepository) {
    this.request = request;           // Explicitly injected
    this.repository = repository;     // Explicitly injected
    this.coreProvider = CoreProvider.getInstance();  // Explicit global access
    this.errorFactory = this.coreProvider.getErrorFactory();  // Explicit retrieval
}
```
**Why**: No hidden dependencies, clear what each class needs

**4. Consistent Patterns**
- All resources follow same registration pattern (bootstrap.ts:465-593)
- All logic classes implement ILogic interface
- All errors created through ErrorFactory
- All responses follow same format

### Summary: Architecture Understanding A2Z

This comprehensive guide has covered:

✅ **Complete Bootstrap Flow** with line-by-line explanations
✅ **Every Dependency** from creation to usage
✅ **Request Flow** with exact file and line references
✅ **Pattern Implementation** with actual code examples
✅ **Configuration Management** from environment to services
✅ **Error Handling** through all layers
✅ **Validation System** with schema organization
✅ **Logic Map System** for custom business rules
✅ **Repository Discovery** for cross-resource access
✅ **Provider Pattern** for global dependency access
✅ **Middleware Pipeline** composition and execution
✅ **External Services** integration patterns
✅ **Debugging Guide** with specific file locations
✅ **Development Workflow** with step-by-step guides
✅ **Architecture Decisions** with rationale

**Key Files to Master**:
1. [`bootstrap.ts`](src/bootstrap.ts:321) - Understand initialization (lines 336-601)
2. [`dependency-factory.ts`](src/infra/factories/dependency-factory.ts:106) - Know all services (lines 136-418)
3. [`core.provider.ts`](src/core/providers/core.provider.ts:17) - Global access pattern (lines 17-139)
4. [`products-create.logic.ts`](src/core/logics/v2/products/crud/products-create.logic.ts:12) - Logic template (lines 12-37)

**Next Steps for Mastery**:
1. Trace 3 different request types through the system
2. Create a new resource end-to-end
3. Add custom logic to existing resource
4. Debug an issue using architecture knowledge
5. Explain architecture to another developer

**Remember**: This architecture evolved over time based on real needs. Every pattern serves a purpose, every abstraction solves a problem. Understanding the "why" is as important as understanding the "how".

        return this.handleError(error);  // Converts to HTTP response
    }
}

private handleError(error: any): IResponse {
    if (error instanceof BaseError) {
        return {
            success: false,
            error: {
                statusCode: error.statusCode,  // 400 for BAD_REQUEST
                message: error.message
            }
        };
    }
    return { success: false, error: { statusCode: 500, message: 'Internal error' } };
}
```

```typescript
// LAYER 1: Express Adapter (Infrastructure)
// Returns HTTP response to client
HTTP/1.1 400 Bad Request
Content-Type: application/json

{
    "success": false,
    "error": {
        "statusCode": 400,
        "message": "Product status must be draft upon creation"
    }
}
```

### 23. Repository Discovery Pattern Explained

#### What is Repository Discovery Pattern?

The Repository Discovery Pattern is a **Service Locator** variant that enables **dynamic runtime lookup** of repositories by resource name. Instead of injecting every possible repository into a class constructor, logic classes can discover and access any repository on-demand through a central registry.

```mermaid
sequenceDiagram
    participant Bootstrap as Bootstrap Process
    participant RepoDiscovery as RepoDiscovery Registry
    participant Logic as Logic Class
    participant CoreProvider
    participant CategoryRepo as Category Repository
    participant ProductRepo as Product Repository
    
    rect rgb(255, 250, 205)
        Note over Bootstrap,RepoDiscovery: Registration Phase (Startup)
        Bootstrap->>RepoDiscovery: registerRepository('Products', productRepo)
        Bootstrap->>RepoDiscovery: registerRepository('Categories', categoryRepo)
        Bootstrap->>RepoDiscovery: registerRepository('Templates', templateRepo)
        Note over RepoDiscovery: Registry: Map<string, IRepository>
    end
    
    rect rgb(240, 248, 255)
        Note over Logic,ProductRepo: Usage Phase (Runtime)
        Logic->>CoreProvider: getRepoDiscovery()
        CoreProvider-->>Logic: RepoDiscovery instance
        
        Logic->>RepoDiscovery: getRepository('Categories')
        RepoDiscovery-->>Logic: CategoryRepository
        
        Logic->>CategoryRepo: read({filter: {id: categoryId}})
        CategoryRepo-->>Logic: Category data
        
        Note over Logic: Validate category exists<br/>Apply business rules
        
        Logic->>RepoDiscovery: getRepository('Products')
        RepoDiscovery-->>Logic: ProductRepository (own repo)
        
        Logic->>ProductRepo: create(productData)
        ProductRepo-->>Logic: Created product
    end
    
    style Bootstrap fill:#90EE90
    style RepoDiscovery fill:#FFE4B5
```
#### Why Repository Discovery Pattern?

**Problems Solved:**

1. **Constructor Injection Explosion**:
```typescript
// ❌ WITHOUT Repository Discovery - Constructor hell
class ProductCreateLogic {
    constructor(
        request: IRequest,
        productRepo: IProductRepository,
        categoryRepo: ICategoryRepository,  // Need to inject every repo
        templateRepo: ITemplateRepository,
        auditRepo: IAuditRepository,
        validationRepo: IValidationRepository,
        // ... potentially 10+ repositories
    ) {}
}
```

2. **Static Dependencies**:
```typescript
// ❌ WITHOUT Repository Discovery - Static, inflexible
class ProductCreateLogic {
    private categoryRepo: ICategoryRepository;
    
    constructor(request: IRequest, repository: ICoreRepository) {
        this.categoryRepo = new CategoryRepository(); // Hardcoded!
    }
}
```

3. **Cross-Resource Coupling**:
```typescript
// ❌ WITHOUT Repository Discovery - Tight coupling
class ProductCreateLogic {
    // Must know about Categories at compile time
    // Can't dynamically access other resources
    // Hard to test with mocks
}
```

**Solutions with Repository Discovery:**

```typescript
// ✅ WITH Repository Discovery - Clean, flexible
class ProductCreateLogic {
    constructor(
        request: IRequest,
        repository: ICoreRepository  // Only inject own repository
    ) {
        this.request = request;
        this.repository = repository;  // ProductRepository
        this.coreProvider = CoreProvider.getInstance();
    }
    
    async execute() {
        // Discover repositories at runtime as needed
        const categoryRepo = this.coreProvider
            .getRepoDiscovery()
            .getRepository('Categories');
        
        const templateRepo = this.coreProvider
            .getRepoDiscovery()
            .getRepository('Templates');
        
        // Use discovered repositories
        const category = await categoryRepo.read({filter: {id: this.request.body.categoryId}});
        const template = await templateRepo.read({filter: {id: this.request.body.templateId}});
    }
}
```

#### How It Works: Step-by-Step

**Step 1: Repository Registration (Bootstrap Phase)**

```typescript
// From src/bootstrap.ts:482
repoDiscovery.registerRepository(resource.name, resourceBaseRepoAdapter);

// What gets registered:
{
    'Products': ProductRepositoryInstance,
    'Categories': CategoryRepositoryInstance,
    'Templates': TemplateRepositoryInstance,
    'Users': UserRepositoryInstance,
    'Dpps': DppRepositoryInstance,
    // ... 30+ more repositories
}
```

**Step 2: Discovery Service Creation**

```typescript
// From src/infra/factories/dependency-factory.ts:186-190
createRepoDiscovery(): IRepoDiscovery {
    if (!this.repoDiscovery) {
        this.repoDiscovery = RepoDiscoveryLite.getInstance();  // Singleton
    }
    return this.repoDiscovery;
}

// RepoDiscoveryLite implementation (conceptual)
class RepoDiscoveryLite implements IRepoDiscovery {
    private static instance: RepoDiscoveryLite;
    private repositories: Map<string, IRepository> = new Map();
    
    static getInstance(): RepoDiscoveryLite {
        if (!RepoDiscoveryLite.instance) {
            RepoDiscoveryLite.instance = new RepoDiscoveryLite();
        }
        return RepoDiscoveryLite.instance;
    }
    
    registerRepository(name: string, repository: IRepository): void {
        this.repositories.set(name, repository);
    }
    
    getRepository(name: string): IRepository {
        const repo = this.repositories.get(name);
        if (!repo) {
            throw new Error(`Repository not found: ${name}`);
        }
        return repo;
    }
    
    hasRepository(name: string): boolean {
        return this.repositories.has(name);
    }
    
    getAllRepositoryNames(): string[] {
        return Array.from(this.repositories.keys());
    }
}
```

**Step 3: Access Through CoreProvider**

```typescript
// From src/core/providers/core.provider.ts
export class CoreProvider {
    getRepoDiscovery(): IRepoDiscovery {
        return this.coreDependency.getRepoDiscovery();
    }
}

// Usage in any logic class
const coreProvider = CoreProvider.getInstance();
const repoDiscovery = coreProvider.getRepoDiscovery();
const categoryRepo = repoDiscovery.getRepository('Categories');
```

#### Real-World Usage Examples from BYO DPP

**Example 1: TemplatesCreateLogic validates referenced fields**

```typescript
// From src/core/logics/v1/templates/crud/templates-create.logic.ts
export class TemplatesCreateLogic implements ILogic {
    async execute() {
        const { metadata } = this.request.body;
        
        // Need to validate that template field ULIDs exist
        if (metadata && typeof metadata === 'object') {
            const ulids = this.extractUlidsFromTemplateMetadata(metadata);
            
            if (ulids.size > 0) {
                // ✅ Discover TemplateFields repository at runtime
                const templateFieldsRepo = this.coreProvider
                    .getRepoDiscovery()
                    .getRepository('TemplateFields');
                
                // Validate ULIDs exist
                const query = {
                    fields: ['ulid'],
                    filter: { ulid: { in: Array.from(ulids) } }
                };
                
                const result = await templateFieldsRepo.list(query);
                const returnedUlids = new Set(
                    (result?.data || []).map((r: { ulid: string }) => r.ulid)
                );
                
                if (returnedUlids.size !== ulids.size) {
                    const missing = Array.from(ulids).filter(u => !returnedUlids.has(u));
                    throw this.errorFactory.create(
                        'BadRequestError',
                        `Unknown template field ULID(s): ${missing.join(', ')}`
                    );
                }
            }
        }
        
        return await this.repository.create(this.request.body);
    }
}
```

**Example 2: ProductCreateLogic validates category**

```typescript
// Conceptual example based on codebase pattern
export class ProductCreateLogic implements ILogic {
    async execute() {
        // ✅ Discover CategoryRepository at runtime
        const categoryRepo = this.coreProvider
            .getRepoDiscovery()
            .getRepository('Categories');
        
        // Validate category exists
        const category = await categoryRepo.read({
            filter: { id: { eq: this.request.body.categoryId } }
        });
        
        if (!category.success) {
            throw this.errorFactory.create(
                'BadRequestError',
                `Category with ID ${this.request.body.categoryId} does not exist`
            );
        }
        
        // Validate category is active
        if (!category.data.isActive) {
            throw this.errorFactory.create(
                'BadRequestError',
                'Cannot create product in inactive category'
            );
        }
        
        // Check product limit for category
        const categoryProductCount = await this.repository.list({
            filter: { categoryId: { eq: this.request.body.categoryId } }
        });
        
        if (categoryProductCount.count >= category.data.maxProducts) {
            throw this.errorFactory.create(
                'BadRequestError',
                `Category has reached maximum product limit of ${category.data.maxProducts}`
            );
        }
        
        // Create product
        return await this.repository.create(this.request.body);
    }
}
```

**Example 3: Cross-Resource Transaction with Discovery**

```typescript
class DppCreateLogic implements ILogic {
    async execute() {
        // Discover multiple repositories for cross-resource operation
        const productRepo = this.coreProvider.getRepoDiscovery().getRepository('Products');
        const templateRepo = this.coreProvider.getRepoDiscovery().getRepository('Templates');
        const auditRepo = this.coreProvider.getRepoDiscovery().getRepository('AuditLogs');
        
        // Validate product exists
        const product = await productRepo.read({
            filter: { id: { eq: this.request.body.productId } }
        });
        
        // Validate template is compatible
        const template = await templateRepo.read({
            filter: { id: { eq: this.request.body.templateId } }
        });
        
        if (product.data.categoryId !== template.data.categoryId) {
            throw this.errorFactory.create(
                'BadRequestError',
                'Template not compatible with product category'
            );
        }
        
        // Create DPP
        const dpp = await this.repository.create(this.request.body);
        
        // Create audit log
        await auditRepo.create({
            action: 'CREATE_DPP',
            resourceType: 'Dpp',
            resourceId: dpp.data.id,
            userId: this.request.user?.id,
            details: {
                productId: product.data.id,
                templateId: template.data.id
            }
        });
        
        return dpp;
    }
}
```

#### Benefits of Repository Discovery Pattern

**1. Reduced Constructor Complexity**
```typescript
// Before (Constructor Injection)
constructor(
    request: IRequest,
    repository: ICoreRepository,
    categoryRepo: ICategoryRepository,      // Explicit injection
    templateRepo: ITemplateRepository,      // Explicit injection
    auditRepo: IAuditRepository,            // Explicit injection
    validationRepo: IValidationRepository   // Explicit injection
) {}

// After (Repository Discovery)
constructor(
    request: IRequest,
    repository: ICoreRepository  // Only inject own repository
) {
    this.coreProvider = CoreProvider.getInstance();  // Access discovery via provider
}
```

**2. Dynamic Resource Access**
```typescript
// Can access any registered repository at runtime
async validateReferences(resourceName: string, ids: number[]) {
    const repo = this.coreProvider.getRepoDiscovery().getRepository(resourceName);
    const results = await repo.list({ filter: { id: { in: ids } } });
    return results.count === ids.length;
}
```

**3. Loose Coupling**
```typescript
// Logic class doesn't depend on specific repositories
// Dependencies resolved at runtime, not compile time
// Easy to add new resources without changing existing logic
```

**4. Testing Flexibility**
```typescript
// Can mock entire discovery service
const mockRepoDiscovery = {
    getRepository: jest.fn((name) => {
        if (name === 'Categories') return mockCategoryRepo;
        if (name === 'Templates') return mockTemplateRepo;
        return mockGenericRepo;
    })
};

const mockCoreProvider = {
    getRepoDiscovery: () => mockRepoDiscovery
};

jest.spyOn(CoreProvider, 'getInstance').mockReturnValue(mockCoreProvider);
```

#### When to Use Repository Discovery

**✅ Use Repository Discovery When:**
- Logic class needs to access multiple resource repositories
- Repository requirements are dynamic or conditional
- Want to avoid constructor injection explosion
- Need runtime flexibility for which repositories to access
- Want to maintain loose coupling between logic classes

**❌ Don't Use Repository Discovery When:**
- Only accessing own repository (use injected `this.repository`)
- Dependencies are known and fixed at compile time
- Want explicit dependency declaration for clarity
- Testing with specific mock implementations

#### Architecture Comparison

**Pattern 1: Constructor Injection (Traditional)**
```typescript
class ProductCreateLogic {
    constructor(
        request: IRequest,
        private productRepo: IProductRepository,
        private categoryRepo: ICategoryRepository,
        private auditRepo: IAuditRepository
    ) {}
    
    async execute() {
        const category = await this.categoryRepo.findById(this.request.body.categoryId);
        // ...
    }
}
```
**Pros**: Explicit dependencies, easy to see what's needed
**Cons**: Constructor bloat, tight coupling, hard to extend

**Pattern 2: Service Locator (Repository Discovery)**
```typescript
class ProductCreateLogic {
    constructor(
        request: IRequest,
        repository: ICoreRepository
    ) {
        this.coreProvider = CoreProvider.getInstance();
    }
    
    async execute() {
        const categoryRepo = this.coreProvider.getRepoDiscovery().getRepository('Categories');
        const category = await categoryRepo.findById(this.request.body.categoryId);
        // ...
    }
}
```
**Pros**: Clean constructor, flexible, loose coupling, easy to extend
**Cons**: Hidden dependencies, potential runtime errors if repository not registered

**BYO DPP Choice**: Hybrid approach
- Inject own repository (explicit dependency)
- Discover other repositories as needed (flexible access)
- Best of both worlds!

#### Common Usage Patterns

**Pattern 1: Reference Validation**
```typescript
async validateReferences() {
    // Validate category exists
    const categoryRepo = this.coreProvider.getRepoDiscovery().getRepository('Categories');
    const category = await categoryRepo.read({ filter: { id: this.request.body.categoryId } });
    
    if (!category.success) {
        throw this.errorFactory.create('BadRequestError', 'Category not found');
    }
}
```

**Pattern 2: Cross-Resource Business Rules**
```typescript
async enforceBusinessRules() {
    // Get related resources
    const userRepo = this.coreProvider.getRepoDiscovery().getRepository('Users');
    const tenantRepo = this.coreProvider.getRepoDiscovery().getRepository('Tenants');
    
    // Check tenant quota
    const user = await userRepo.read({ filter: { id: this.request.user.id } });
    const tenant = await tenantRepo.read({ filter: { id: user.data.tenantId } });
    
    if (tenant.data.productQuota >= tenant.data.maxProducts) {
        throw this.errorFactory.create('ForbiddenError', 'Tenant product quota exceeded');
    }
}
```

**Pattern 3: Audit Trail Creation**
```typescript
async createAuditLog(action: string, resourceId: number) {
    const auditRepo = this.coreProvider.getRepoDiscovery().getRepository('AuditLogs');
    
    await auditRepo.create({
        action,
        resourceType: this.repository.modelName,
        resourceId,
        userId: this.request.user?.id,
        timestamp: new Date(),
        details: { /* ... */ }
    });
}
```

#### Advanced Usage: Dynamic Resource Access

**Scenario**: Generic validation logic for any resource

```typescript
class GenericReferenceValidator {
    async validateForeignKeys(
        references: Array<{resourceName: string; fieldName: string; id: number}>
    ): Promise<void> {
        for (const ref of references) {
            // Dynamically access repository by name
            const repo = this.coreProvider
                .getRepoDiscovery()
                .getRepository(ref.resourceName);
            
            const exists = await repo.read({
                filter: { id: { eq: ref.id } }
            });
            
            if (!exists.success) {
                throw this.errorFactory.create(
                    'BadRequestError',
                    `Referenced ${ref.resourceName} with ID ${ref.id} does not exist`
                );
            }
        }
    }
}

// Usage
await this.validateForeignKeys([
    { resourceName: 'Categories', fieldName: 'categoryId', id: payload.categoryId },
    { resourceName: 'Templates', fieldName: 'templateId', id: payload.templateId },
    { resourceName: 'Units', fieldName: 'unitId', id: payload.unitId }
]);
```

#### Performance Considerations

**Repository Instance Reuse:**
```typescript
// RepoDiscovery returns same instance (singleton per resource)
const repo1 = repoDiscovery.getRepository('Products');
const repo2 = repoDiscovery.getRepository('Products');
// repo1 === repo2 (same instance, no re-creation)
```

**Lazy Loading:**
```typescript
// Repositories only accessed when needed
// No upfront cost for unused repositories
async execute() {
    if (this.needsCategoryValidation()) {
        // Only retrieve when needed
        const categoryRepo = this.coreProvider.getRepoDiscovery().getRepository('Categories');
        await this.validateCategory(categoryRepo);
    }
}
```

#### Testing with Repository Discovery

**Mock Setup:**
```typescript
describe('ProductCreateLogic', () => {
    let mockRepoDiscovery: jest.Mocked<IRepoDiscovery>;
    let mockCategoryRepo: jest.Mocked<ICoreRepository>;
    let mockAuditRepo: jest.Mocked<ICoreRepository>;
    
    beforeEach(() => {
        // Mock repositories
        mockCategoryRepo = {
            read: jest.fn(),
            list: jest.fn(),
            create: jest.fn()
        } as any;
        
        mockAuditRepo = {
            create: jest.fn()
        } as any;
        
        // Mock discovery to return mocks
        mockRepoDiscovery = {
            getRepository: jest.fn((name) => {
                if (name === 'Categories') return mockCategoryRepo;
                if (name === 'AuditLogs') return mockAuditRepo;
                throw new Error(`Unexpected repository: ${name}`);
            })
        } as any;
        
        // Mock CoreProvider
        const mockCoreProvider = {
            getRepoDiscovery: jest.fn(() => mockRepoDiscovery),
            getErrorFactory: jest.fn(() => errorFactory)
        } as any;
        
        jest.spyOn(CoreProvider, 'getInstance').mockReturnValue(mockCoreProvider);
    });
    
    it('should validate category exists', async () => {
        // Arrange
        mockCategoryRepo.read.mockResolvedValue({
            success: true,
            data: { id: 1, name: 'Electronics', isActive: true }
        });
        
        // Act
        const logic = new ProductCreateLogic(request, productRepository);
        await logic.execute();
        
        // Assert
        expect(mockRepoDiscovery.getRepository).toHaveBeenCalledWith('Categories');
        expect(mockCategoryRepo.read).toHaveBeenCalledWith({
            filter: { id: { eq: request.body.categoryId } }
        });
    });
});
```

#### Comparison: Repository Discovery vs Alternatives

| Aspect | Constructor Injection | Service Locator | Repository Discovery (BYO DPP) |
|--------|----------------------|----------------|-------------------------------|
| **Dependency Visibility** | ✅ Explicit | ❌ Hidden | ⚠️ Semi-hidden (via provider) |
| **Constructor Complexity** | ❌ Can explode | ✅ Clean | ✅ Clean |
| **Runtime Flexibility** | ❌ Fixed | ✅ Dynamic | ✅ Dynamic |
| **Compile-Time Safety** | ✅ Type-safe | ❌ Runtime errors | ⚠️ String-based lookup |
| **Testing** | ✅ Easy to mock | ⚠️ Need setup | ✅ Easy with CoreProvider mock |
| **Loose Coupling** | ⚠️ Medium | ✅ High | ✅ High |
| **Discoverability** | ✅ Easy to see deps | ❌ Hidden | ⚠️ Must check code |

#### Best Practices

**1. Always Check Repository Exists**
```typescript
// ✅ GOOD: Handle missing repository
const repoDiscovery = this.coreProvider.getRepoDiscovery();
if (!repoDiscovery.hasRepository('OptionalResource')) {
    // Handle gracefully
    return;
}
const repo = repoDiscovery.getRepository('OptionalResource');
```

**2. Cache Repository References**
```typescript
// ✅ GOOD: Cache for multiple uses
async execute() {
    const categoryRepo = this.coreProvider.getRepoDiscovery().getRepository('Categories');
    
    // Use multiple times without re-fetching
    const category1 = await categoryRepo.read({filter: {id: 1}});
    const category2 = await categoryRepo.read({filter: {id: 2}});
}
```

**3. Document Dependencies**
```typescript
/**
 * ProductCreateLogic
 * 
 * @description Creates a new product with validation
 * 
 * @repositories
 * - Products: Direct injection (this.repository)
 * - Categories: Via discovery (validates category exists)
 * - AuditLogs: Via discovery (creates audit trail)
 * 
 * @businessRules
 * - Category must exist and be active
 * - Product name must be unique within category
 * - Tenant must not exceed product quota
 */
export class ProductCreateLogic implements ILogic {
    // Implementation
}
```

#### Summary: Repository Discovery in BYO DPP

**Core Concept**: Centralized registry + dynamic lookup = flexible, loosely-coupled architecture

**Key Files**:
- **Registry**: `src/infra/adapters/repo-discovery/lite.repo-discovery.ts`
- **Registration**: `src/bootstrap.ts:482`
- **Access Point**: `src/core/providers/core.provider.ts`
- **Usage Example**: `src/core/logics/v1/templates/crud/templates-create.logic.ts`

**When Logic Class Needs Another Repository**:
```
Logic needs CategoryRepository
    ↓
Access CoreProvider.getInstance()
    ↓
Get RepoDiscovery service
    ↓
Call getRepository('Categories')
    ↓
Receive CategoryRepository instance
    ↓
Use repository methods (read, list, create, etc.)
```

**Benefits in BYO DPP**:
- ✅ **Scalability**: Add new resources without changing logic classes
- ✅ **Flexibility**: Logic determines which repositories to access
- ✅ **Maintainability**: Clear separation between resource logic
- ✅ **Testability**: Easy to mock entire discovery mechanism
- ✅ **Consistency**: Same pattern across all logic classes

The Repository Discovery Pattern is a core architectural decision in BYO DPP that enables the dynamic, flexible, and maintainable resource system.

    style Logic fill:#E0FFFF
```


#### Cross-Resource Repository Access

**Registration During Bootstrap**:
```typescript
// From src/bootstrap.ts:482
repoDiscovery.registerRepository(resource.name, resourceBaseRepoAdapter);
```

**What Gets Registered**:
- Resource Name: "Products", "Categories", "Templates", etc.
- Repository Instance: Configured DrizzleORMRepository wrapped in BaseRepoAdapter

**Usage in Logic Classes**:
```typescript
// Common pattern from various logic files
export class TemplatesCreateLogic implements ILogic {
    async execute() {
        // Access different resource repository
        const templateFieldsRepo = this.coreProvider
            .getRepoDiscovery()
            .getRepository('TemplateFields');
        
        // Use it to validate references
        const result = await templateFieldsRepo.list({
            filter: { ulid: { in: ulidArray } }
        });
    }
}
```

**Architecture Benefits**:
1. **Loose Coupling**: Logic doesn't need direct repository injection
2. **Dynamic Resolution**: Repositories resolved by name at runtime
3. **Cross-Resource Operations**: Easy to access related resources
4. **Single Registry**: One place to manage all repositories

### 24. Complete Dependency Graph Visualization

```
┌─────────────────────────────────────────────────────────────┐
│                         Bootstrap.ts                         │
│                      (src/bootstrap.ts:336)                  │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ├─── setupConfig() (line 383)
                         │    └─→ AppConfig.getInstance()
                         │
                         ├─── setupBootstrapJson() (line 387)
                         │    └─→ Load JSON from input/latest/
                         │
                         ├─── setupDependencyFactory() (line 391)
                         │    └─→ DependencyFactory (dependency-factory.ts:106)
                         │        ├─→ createDatabase() (line 136)
                         │        ├─→ createRouter() (line 150)
                         │        ├─→ createServer() (line 157)
                         │        ├─→ createGraphAPI() (line 179)
                         │        ├─→ createRepoDiscovery() (line 186)
                         │        ├─→ createEventStream() (line 302)
                         │        ├─→ createMailer() (line 309)
                         │        ├─→ createBlockchain() (line 143)
                         │        ├─→ createErrorFactory() (line 296)
                         │        ├─→ createIdentityService() (line 207)
                         │        └─→ createAuthorizationService() (line 247)
                         │
                         ├─── setupCoreProvider() (line 442)
                         │    └─→ CoreProvider.initialize() (core.provider.ts:33)
                         │
                         ├─── setupApplicationProvider() (line 446)
                         │    └─→ ApplicationProvider.initialize() (application.provider.ts:15)
                         │
                         └─── buildRoutes() (line 456)
                              └─→ For each resource:
                                  ├─→ Create Repository (line 473)
                                  ├─→ Register with Discovery (line 482)
                                  ├─→ Create Service (line 484)
                                  ├─→ Create Controller (line 494)
                                  ├─→ Create Middleware (line 496)
                                  └─→ Register Routes (line 582, 501)
```

### 25. Advanced Pattern: Logic Map Versioning

#### Supporting Multiple API Versions

```typescript
// From src/bootstrap.ts:488-492
const resourceBaseService = new BaseService(
    resourceBaseRepoAdapter,
    resource.name,
    {
        ...logicMap['v1'][resource.name]?.action,  // Version 1 logics
        ...logicMap['v1'][resource.name]?.crud
    } as Record<string, LogicConstructor>
);
```

**Logic Map Structure**:
```typescript
// From src/application/mapper/logic-map.ts
export const logicMap = {
    'v1': {  // API Version
        'products': {  // Resource name
            'crud': {  // Operation category
                'create': ProductsCreateLogic  // Operation → Logic class
            },
            'action': {  // Custom actions
                'feature:post': ProductsFeatureLogic
            }
        }
    }
    // Can add v2, v3, etc.
};
```

**Future Extensibility**:
```typescript
// Adding v2 with breaking changes
export const logicMap = {
    'v1': {
        'products': {
            'crud': {
                'create': ProductsCreateLogicV1  // Old behavior
            }
        }
    },
    'v2': {
        'products': {
            'crud': {
                'create': ProductsCreateLogicV2  // New behavior
            }
        }
    }
};
```

### 26. Batch Processing Architecture

#### Batch Middleware for Bulk Operations

```typescript
// From src/bootstrap.ts:569-572
if (route.method === 'batch') {
    middlewares.push(new BatchMiddleware(JoiAdapter, validationSchemaMap[resource.name]));
}
```

**What Batch Processing Does**:
1. **Array Validation**: Ensures request.body is an array
2. **Individual Validation**: Validates each item against schema
3. **Atomic Transaction**: All items succeed or all fail
4. **Efficient Insertion**: Single database round-trip

**Usage Pattern**:
```
POST /api/v1/products/batch
[
    { name: "Product 1", categoryId: 1, status: "draft" },
    { name: "Product 2", categoryId: 1, status: "draft" },
    { name: "Product 3", categoryId: 2, status: "draft" }
]

↓ [BatchMiddleware validates each]
↓ [Repository.createBatch() inserts all]

Response: {
    success: true,
    data: [
        { id: 1, name: "Product 1", ... },
        { id: 2, name: "Product 2", ... },
        { id: 3, name: "Product 3", ... }
    ]
}
```

### 27. Graph Query API Architecture

#### Complex Query Building System

```typescript
// From src/infra/factories/dependency-factory.ts:179-184
createGraphAPI(): IGraphAPI {
    if (!this.graphAPI) {
        this.graphAPI = new GraphQueryORMAdapter();
    }
    return this.graphAPI;
}
```

**Used in Controller**:
```typescript
// From src/bootstrap.ts:494 - Controller creation
const resourceBaseController = new BaseController(
    resourceBaseService,
    graphQuery,  // GraphAPI injected into controller
    resource.name
);

// Controller uses it (conceptual from base-resource.controller.ts)
async list(request: IRequest): Promise<IResponse> {
    // Parse complex query parameters
    const query = this.graphQuery.parse(request.query);
    const result = await this.service.list({ ...request, query });
}
```

**Query Capabilities**:
```
GET /api/v1/products?filter[categoryId][eq]=1&include=category&sort=name:asc&limit=10

Parsed to:
{
    where: (table, operators) => operators.eq(table.categoryId, 1),
    include: { category: true },
    orderBy: { name: 'asc' },
    limit: 10,
    offset: 0
}
```

### 28. Resource Model Mapping System

#### Connecting Resources to Database Schemas

```typescript
// From src/bootstrap.ts:29
import { ResourceModelMap } from './infra/constants/byo-dpp/resource-model-map';

// Used at line 476
ResourceModelMap[resource.name as keyof typeof ResourceModelMap]
```

**ResourceModelMap Structure**:
```typescript
// From src/infra/constants/byo-dpp/resource-model-map.ts
export const ResourceModelMap = {
    Products: products,        // Drizzle schema from products.schema.ts
    Categories: categories,    // Drizzle schema from categories.schema.ts
    Dpps: dpps,               // Drizzle schema from dpps.schema.ts
    // Maps logical resource name to physical database schema
};
```

**Why This Mapping**:
- ✅ **Type Safety**: Compile-time validation of resource names
- ✅ **Single Source**: One place to define resource-to-schema mapping
- ✅ **Flexibility**: Can change schema without changing resource logic
- ✅ **Consistency**: Same pattern for all resources

### 29. Action vs CRUD Route Distinction

#### Two Types of Routes in BYO DPP

**CRUD Routes** (Standard Operations):
```typescript
// From src/bootstrap.ts:562-593
resource.routes.crud.forEach((route) => {
    // Standard CRUD: list, create, read, update, delete, patch
    const _route = {
        path: route.url,                              // e.g., /api/v1/products
        method: httpMethodMap[route.method],          // 'create' → 'post'
        action: actionMap[route.method],              // 'create' (service method)
        controller: resourceBaseController,
        middlewares: middlewares
    };
    router.addRoute(_route as Route);
});
```

**Action Routes** (Custom Business Operations):
```typescript
// From src/bootstrap.ts:501-559
resource.routes.actions.forEach((route) => {
    // Custom actions: feature, approve, publish, etc.
    const action = route.url.split('/').pop();  // e.g., 'feature'
    const actionKey = `${action}:${route.method}`;  // e.g., 'feature:post'
    
    if (logicMap['v1'][resource.name]?.action[actionKey]) {
        const _route = {
            path: route.url,                    // e.g., /api/v1/products/1/feature
            method: route.method,               // 'post'
            action: 'action',                   // Calls controller.action()
            actionName: actionKey,              // Passed to service
            controller: resourceBaseController,
            middlewares: middlewares
        };
        router.addRoute(_route as Route);
    }
});
```

**Examples**:
| Route Type | Example | Controller Method | Logic Lookup |
|------------|---------|-------------------|--------------|
| CRUD | `POST /products` | `controller.create()` | `logicMap.v1.products.crud.create` |
| CRUD | `GET /products` | `controller.list()` | `logicMap.v1.products.crud.list` |
| Action | `POST /products/1/feature` | `controller.action('feature:post')` | `logicMap.v1.products.action['feature:post']` |
| Action | `POST /dpps/1/publish` | `controller.action('publish:post')` | `logicMap.v1.dpps.action['publish:post']` |

### 30. Understanding CoreProvider Usage Pattern

#### How Logic Classes Access Dependencies

```typescript
// From src/core/logics/v2/products/crud/products-create.logic.ts:12-23
export class ProductsCreateLogic implements ILogic {
    request: IRequest;
    repository: ICoreRepository;
    errorFactory: IErrorFactory;
    coreProvider: CoreProvider;

    constructor(request: IRequest, repository: ICoreRepository) {
        this.request = request;           // Injected by service
        this.repository = repository;     // Injected by service
        this.coreProvider = CoreProvider.getInstance();  // Global access
        this.errorFactory = this.coreProvider.getErrorFactory();  // From provider
    }
}
```

**What CoreProvider Provides** (from core.provider.ts):
```typescript
// From src/core/providers/core.provider.ts:56-138
getValidatorClass(): new (schema: SchemaDefinition) => IValidator
getRepoDiscovery(): IRepoDiscovery  // Access any repository
getDB(): IDatabase                   // Direct database access
getGraphAPI(): IGraphAPI             // Query builder
getEventStream(): IEventStream       // SSE for real-time events
getFileStorage(): IFileStorage       // File upload/download
getErrorFactory(): IErrorFactory     // Error creation
getIdentityService(): IIdentityService  // User management
getAuthorizationService(): Promise<IAuthorizationService>  // Permissions
getBlockchain(): IBlockchain         // Blockchain operations
getMailer(): IMailer                 // Email sending
getExportImporterProvider(): IExportImporterProvider  // Import/export
getIdGenerator(): IdGenerator        // ULID generation
```

**Common Patterns**:
```typescript
// Pattern 1: Access other repositories
const categoryRepo = this.coreProvider.getRepoDiscovery().getRepository('Categories');

// Pattern 2: Send emails
const mailer = this.coreProvider.getMailer();
await mailer.send({ to: user.email, subject: 'Welcome', body: html });

// Pattern 3: Store files
const storage = this.coreProvider.getFileStorage();
const url = await storage.upload(fileBuffer, filename, mimeType);

// Pattern 4: Generate IDs
const idGen = this.coreProvider.getIdGenerator();
const ulid = idGen.generate();
```

### 31. Bootstrap Lifecycle and Error Handling

#### Complete Startup Sequence

```typescript
// From src/bootstrap.ts:336-348
run(): void {
    try {
        this.logger.info('🚀 Starting BYO DPP Application');
        this.setup();   // Line 360-373: Initialize all dependencies
        this.build();   // Line 375-377: Build routes
        this.boot();    // Line 379-381: Start server
        this.logger.info('✨ Application is ready');
        console.log(chalk.green(`Server running at http://${getLocalIP()}:${this.config.PORT}`));
    } catch (error) {
        this.logger.error('Failed to start application', error as Error);
        process.exit(1);  // Exit on startup failure
    }
}
```

**Setup Phase Details**:
```typescript
// From src/bootstrap.ts:360-373
setup(): void {
    try {
        this.setupConfig();              // Load environment config
        this.setupBootstrapJson();       // Load resource definitions
        this.setupDependencyFactory();   // Create all dependencies
        this.setupResponseMessage();     // Initialize response templates
        this.setupCoreProvider();        // Setup core dependency provider
        this.setupApplicationProvider(); // Setup application provider
        this.setupApplication();         // Create Express application
    } catch (error) {
        this.logger.error('Setup failed', error as Error);
        throw error;
    }
}
```

**Robust Error Handling**:
- Each setup method wrapped in try-catch
- Errors logged with context
- Application exits gracefully on failure
- Prevents partially initialized state

### 32. Real-World Architecture Flow Diagrams

#### Product Creation Flow with Line Numbers

```
User: POST /api/v1/products { name, categoryId, status }
│
├─→ Express Server (src/infra/adapters/express/express.adapter.ts)
│   └─→ Route matched by Router (src/application/adapters/router/api.router.ts)
│       Registered at: src/bootstrap.ts:582-591
│
├─→ AuthenticatorMiddleware (src/infra/adapters/middlewares/authenticator.middleware.ts)
│   Created at: src/bootstrap.ts:496-498
│   ├─→ Calls: identityService.verifyToken(token)
│   └─→ Sets: request.user = { id, roles, permissions }
│
├─→ ValidatorMiddleware (src/infra/adapters/middlewares/validator.middleware.ts)
│   Schema from: src/bootstrap.ts:170-318 (validationSchemaMap)
│   Added at: src/bootstrap.ts:573-580
│   ├─→ Validates: request.body against createProductsSchema
│   └─→ Result: Clean, validated data in request.body
│
├─→ BaseController.create() (src/application/adapters/controller/base-resource.controller.ts)
│   Created at: src/bootstrap.ts:494
│   └─→ Calls: service.create(request)
│
├─→ BaseService.create() (src/application/adapters/service/base-resource.service.ts)
│   Created at: src/bootstrap.ts:484-492
│   Logic Map injected from: src/application/mapper/logic-map.ts (line 78)
│   ├─→ Looks up: logicMap['v1']['products']['crud']['create']
│   ├─→ Finds: ProductsCreateLogic class
│   ├─→ Instantiates: new ProductsCreateLogic(request, repository)
│   └─→ Calls: logic.execute()
│
├─→ ProductsCreateLogic.execute() (src/core/logics/v2/products/crud/products-create.logic.ts:25)
│   Constructor: Lines 18-23
│   ├─→ Gets CoreProvider: line 21
│   ├─→ Gets ErrorFactory: line
**How Logic Is Retrieved and Executed**:
```typescript
// From src/bootstrap.ts:488-492 - Service creation with logic injection
const resourceBaseService = new BaseService(
    resourceBaseRepoAdapter,
    resource.name,
    {
        ...logicMap['v1'][resource.name]?.action,  // Action logics
        ...logicMap['v1'][resource.name]?.crud     // CRUD logics
    } as Record<string, LogicConstructor>
);
```

**Service Uses Logic** (conceptual from base-resource.service.ts):
```typescript
private handleOperation(name: string, request: IRequest, fallback: () => unknown) {
    const LogicClass = this.getLogic(name);
    if (LogicClass) {
        // Execute custom logic
        const instance = new LogicClass(request, this.repository);
        return instance.execute();
    }
    // Fallback to repository
    return fallback();
}
```

### 7. Error Handling with Factory Pattern

#### Error Factory from Actual Code

```typescript
// From src/infra/adapters/errors/error.factory.ts (referenced at dependency-factory.ts:296-300)
createErrorFactory(): IErrorFactory {
    if (!this.errorFactory) {
        this.errorFactory = new ErrorFactory();
    }
    return this.errorFactory;
}

// ErrorFactory implementation (conceptual from error.factory.ts)
export class ErrorFactory implements IErrorFactory {
    create(errorType: string, message: string): BaseError {
        switch (errorType) {
            case ErrorType.BAD_REQUEST:
                return new BadRequestError(message);
            case ErrorType.NOT_FOUND:
                return new NotFoundError(message);
            case ErrorType.FORBIDDEN:
                return new ForbiddenError(message);
            // ... more error types
        }
    }
}
```

**Real Usage in Logic Class**:
```typescript
// From src/core/logics/v2/products/crud/products-create.logic.ts:29-30
throw this.errorFactory.create(
    ErrorType.BAD_REQUEST,
    'Product status must be draft upon creation'
);
```

**Why This Pattern**:
- ✅ **Consistency**: All errors created through factory
- ✅ **Type Safety**: ErrorType enum prevents typos
- ✅ **Centralized**: Error logic in one place
- ✅ **Testability**: Easy to mock error factory

### 8. Repository Discovery System

#### How Repositories Are Registered and Retrieved

```typescript
// From src/infra/factories/dependency-factory.ts:186-190
createRepoDiscovery(): IRepoDiscovery {
    if (!this.repoDiscovery) {
        this.repoDiscovery = RepoDiscoveryLite.getInstance();
    }
    return this.repoDiscovery;
}
```

**Registration During Bootstrap**:
```typescript
// From src/bootstrap.ts:482
repoDiscovery.registerRepository(resource.name, resourceBaseRepoAdapter);
```

**Usage in Logic Classes** (conceptual pattern):
```typescript
// Logic classes can access any repository
const categoryRepo = this.coreProvider.getRepoDiscovery().getRepository('Categories');
const product = await categoryRepo.read({ filter: { id: categoryId } });
```

**Benefits**:
- ✅ **Cross-Resource Access**: Logic can access any resource's repository
- ✅ **Loose Coupling**: Logic doesn't need direct repository injection
- ✅ **Dynamic Resolution**: Repositories resolved at runtime
- ✅ **Testing**: Can register mock repositories for testing

### 9. Identity and Authorization Flow

#### Keycloak Integration Setup

```typescript
// From src/infra/factories/dependency-factory.ts:207-244
createIdentityService(): IIdentityService {
    this.identityService = new IdentityServiceAdapter(
        new KeycloakService({
            baseURL: this.config.KEYCLOAK_HOST_URI,
            adminRealmId: this.config.KEYCLOAK_ADMIN_REALM_ID,
            adminClientId: this.config.KEYCLOAK_ADMIN_CLIENT_ID,
            adminClientSecret: this.config.KEYCLOAK_ADMIN_CLIENT_SECRET,
            frontendClientId: this.config.KEYCLOAK_FRONTEND_CLIENT_ID,
            frontendClientSecret: this.config.KEYCLOAK_FRONTEND_CLIENT_SECRET,
            frontendRedirectURI: this.config.KEYCLOAK_FRONTEND_REDIRECT_URI,
            emailConfig: {
                host: this.config.SMTP_HOST,
                port: this.config.SMTP_PORT,
                from: this.config.SMTP_FROM,
                // ... more email config
            }
        })
    );
    return this.identityService;
}
```

**Usage in Authentication Middleware**:
```typescript
// From src/bootstrap.ts:496-498
const authenticatorMiddleware = new AuthenticatorMiddleware(
    this.dependencyFactory.getIdentityService()
);
```

**How It Works**:
1. **Token Extraction**: Middleware extracts JWT from `Authorization` header
2. **Token Validation**: Keycloak service validates token signature and expiry
3. **User Extraction**: User details (id, roles, permissions) extracted from token
4. **Request Enrichment**: User object attached to request for downstream use

### 10. Middleware Pipeline Architecture

#### Middleware Composition in Action

```typescript
// From src/bootstrap.ts:522-538
const middlewares: IMiddleware[] = [];

// Always add authentication
middlewares.push(authenticatorMiddleware);

// Add validation if schema exists
if (validationSchemaMap[resource.name]?.[route.method]) {
    const schema = validationSchemaMap[resource.name][route.method];
    middlewares.push(new ValidatorMiddleware(new JoiAdapter(schema)));
}

// Add batch processing if needed
if (route.method === 'batch') {
    middlewares.push(new BatchMiddleware(JoiAdapter, validationSchemaMap[resource.name]));
}
```

**Middleware Execution Order**:
```
1. AuthenticatorMiddleware  → Validates JWT, sets request.user
2. ValidatorMiddleware       → Validates request.body against schema
3. BatchMiddleware          → Handles batch operations (if applicable)
4. Controller               → Business logic execution
```

### 11. Service Layer: The Orchestration Hub

#### Service Layer Responsibilities from Actual Code

```typescript
// From src/bootstrap.ts:484-492
const resourceBaseService = new BaseService(
    resourceBaseRepoAdapter,    // Repository for data access
    resource.name,              // Resource identifier
    {
        ...logicMap['v1'][resource.name]?.action,  // Custom action logics
        ...logicMap['v1'][resource.name]?.crud     // CRUD logics
    } as Record<string, LogicConstructor>
);
```

**What Service Layer Does**:
1. **Logic Dispatch**: Routes to custom logic or default repository
2. **Transaction Coordination**: Manages complex multi-step operations
3. **Error Translation**: Converts repository errors to domain errors
4. **Event Publishing**: Triggers domain events after operations

**Decision Flow**:
```
Request arrives at Service
    ↓
Is there custom logic for this operation?
    ├─ YES → Instantiate logic class
    │        Call logic.execute()
    │        Return result
    │
    └─ NO → Call repository directly
             Return result
```

### 12. Repository Pattern with ORM Abstraction

#### Repository Creation from Bootstrap

```typescript
// From src/bootstrap.ts:473-479
const resourceORMRepository = new DrizzleORMRepository(
    database,                    // Database connection from factory
    resource.name,               // e.g., "Products"
    ResourceModelMap[resource.name]  // Drizzle schema mapping
);

resourceBaseRepoAdapter = new BaseRepoAdapter(resourceORMRepository);
```

**ResourceModelMap Connection**:
```typescript
// From src/infra/constants/byo-dpp/resource-model-map.ts (referenced)
export const ResourceModelMap = {
    Products: products,
    Categories: categories,
    Dpps: dpps,
    Templates: templates,
    // Maps resource name to Drizzle schema
};
```

**Benefits of This Architecture**:
- ✅ **Type Safety**: ResourceModelMap provides compile-time checking
- ✅ **Consistency**: All resources use same repository pattern
- ✅ **Testability**: Can inject mock repositories
- ✅ **Flexibility**: Can swap ORM without changing business logic

### 13. External Service Integration Architecture

#### Blockchain Integration

```typescript
// From src/infra/factories/dependency-factory.ts:143-148
createBlockchain(): IBlockchain {
    if (!this.blockchain) {
        this.blockchain = new BlockchainAdapter(
            new PolygonAdapter(this.config.blockchain)
        );
    }
    return this.blockchain;
}
```

**Configuration from Bootstrap**:
```typescript
// From src/bootstrap.ts:432-436
blockchain: {
    network: this.config.POLYGON_NETWORK,
    rpcUrl: this.config.POLYGON_RPC_ENDPOINT,
    privateKey: this.config.POLYGON_PRIVATE_KEY
}
```

**Adapter Pattern Benefits**:
- ✅ **Technology Independence**: Can swap blockchain providers
- ✅ **Interface Consistency**: Business logic uses IBlockchain interface
- ✅ **Testing**: Can mock blockchain calls

#### Email Service Integration

```typescript
// From src/infra/factories/dependency-factory.ts:309-316
createMailer(): IMailer {
    if (!this.mailer) {
        const nodemailerAdapter = new NodemailerAdapter(translations);
        this.mailer = new MailerAdapter(nodemailerAdapter);
    }
    return this.mailer;
}
```

**Email Configuration**:
```typescript
// From src/bootstrap.ts:420-428 (passed to dependency factory)
SMTP_HOST: this.config.SMTP_HOST!,
SMTP_PORT: this.config.SMTP_PORT!,
SMTP_FROM: this.config.SMTP_FROM!,
SMTP_AUTH: this.config.SMTP_AUTH!,
SMTP_USERNAME: this.config.SMTP_USERNAME!,
SMTP_PASSWORD: this.config.SMTP_PASSWORD!,
```

### 14. Complete Architecture Decision Map

#### Decision Points in Bootstrap Process

```typescript
// From src/bootstrap.ts:360-373
setup(): void {
    try {
        this.setupConfig();              // Decision: Singleton config
        this.setupBootstrapJson();       // Decision: JSON-driven resources
        this.setupDependencyFactory();   // Decision: Factory pattern
        this.setupResponseMessage();     // Decision: Centralized messages
        this.setupCoreProvider();        // Decision: Provider pattern
        this.setupApplicationProvider(); // Decision: Layered providers
        this.setupApplication();         // Decision: Separation of concerns
    } catch (error) {
        this.logger.error('Setup failed', error as Error);
        throw error;
    }
}
```

**Why This Order**:
1. **Config First**: Everything depends on configuration
2. **Bootstrap JSON Second**: Resource definitions needed for routing
3. **Dependency Factory Third**: Creates all infrastructure
4. **Providers Fourth**: Wrap factory for global access
5. **Application Last**: Wire everything together

### 15. Real Code Path Tracing Examples

#### Example 1: Product Creation Complete Path

```
CLIENT REQUEST
    POST /api/v1/products
    Body: { name: "Product", categoryId: 1, status: "draft" }
    Header: Authorization: Bearer <JWT>

↓ [Express Server receives request]
    File: src/infra/adapters/express/express.adapter.
            
            // Cache update
            await redis.set(`product:${data.id}`, data);
            
            // Search indexing
            await elasticsearch.index(data);
            
            // Logging
            console.log('Product created');
        }
        // ... hundreds more lines
    }
}

// ✅ CORRECT: Separated responsibilities
class ProductController { /* HTTP handling */ }
class ProductService { /* Business orchestration */ }
class ProductLogic { /* Business rules */ }
class ProductRepository { /* Data access */ }
class ProductNotifier { /* Notifications */ }
class ProductSearchIndexer { /* Search indexing */ }
```

#### Anti-Pattern 2: Circular Dependencies

```typescript
// ❌ ANTI-PATTERN: Circular dependency
class ProductService {
    constructor(private categoryService: CategoryService) {}
    
    async create(data: CreateProductData): Promise<Product> {
        // Calls CategoryService
        await this.categoryService.updateProductCount(data.categoryId);
    }
}

class CategoryService {
    constructor(private productService: ProductService) {} // Circular!
    
    async delete(id: number): Promise<void> {
        // Calls ProductService
        await this.productService.deleteByCategory(id);
    }
}

// ✅ CORRECT: Use domain events to break cycle
class ProductService {
    constructor(private eventBus: IEventBus) {}
    
    async create(data: CreateProductData): Promise<Product> {
        const product = await this.repository.create(data);
        
        // Publish event instead of direct call
        await this.eventBus.publish(new ProductCreatedEvent(product));
        
        return product;
    }
}

class CategoryService {
    async initialize(): Promise<void> {
        // Subscribe to events
        this.eventBus.subscribe('ProductCreated', async (event) => {
            await this.updateProductCount(event.categoryId);
        });
    }
}
```

#### Anti-Pattern 3: Leaky Abstractions

```typescript
// ❌ ANTI-PATTERN: Domain layer knows about HTTP
class Product {
    constructor(
        public id: number,
        private httpClient: HttpClient // Infrastructure leak!
    ) {}

    async updatePrice(newPrice: number): Promise<void> {
        // Domain object shouldn't make HTTP calls
        await this.httpClient.post('/api/prices', { price: newPrice });
    }
}

// ✅ CORRECT: Clean separation
class Product {
    private domainEvents: DomainEvent[] = [];

    updatePrice(newPrice: number): void {
        // Pure domain logic
        if (newPrice <= 0) {
            throw new Error('Price must be positive');
        }
        
        this.price = newPrice;
        this.domainEvents.push(new PriceChangedEvent(this.id, newPrice));
    }

    getDomainEvents(): DomainEvent[] {
        return [...this.domainEvents];
    }
}

// Infrastructure handles HTTP
class PriceChangedHandler {
    constructor(private httpClient: HttpClient) {}
    
    async handle(event: PriceChangedEvent): Promise<void> {
        await this.httpClient.post('/api/external/prices', {
            productId: event.productId,
            price: event.newPrice
        });
    }
}
```

### Final Architecture Principles Summary

#### Core Tenets of BYO DPP Architecture

1. **Simplicity First, Complexity When Needed**
   - Start with simple CRUD
   - Add business logic layer only when rules are complex
   - Add caching only when performance requires it
   - Add events only when decoupling is necessary

2. **Explicit Over Implicit**
   - Dependencies injected, not hidden
   - Configuration explicit, not scattered
   - Data flow clear and traceable
   - Errors handled explicitly at each layer

3. **Consistency Through Conventions**
   - All resources follow same patterns
   - All errors use error factory
   - All logs use structured format
   - All responses use standard format

4. **Flexibility Through Abstraction**
   - Interfaces at layer boundaries
   - Adapters for external services
   - Strategies for varying algorithms
   - Factories for object creation

5. **Observability Built In**
   - Structured logging throughout
   - Metrics at critical points
   - Tracing across boundaries
   - Health checks for all components

6. **Security By Default**
   - Authentication required unless explicitly public
   - Authorization checked at service layer
   - Input validation at every entry point
   - Audit logging for sensitive operations

### Understanding the Complete Picture

#### Mental Model for Navigation

When working with the BYO DPP architecture, think in terms of these questions:

**"Where do I add business logic?"**
```typescript
// Simple validation → Joi schema
// Complex business rules → Logic class
// Cross-resource operations → Domain service
// Side effects → Event handlers
```

**"Where do I handle errors?"**
```typescript
// Database errors → Repository layer
// Business errors → Domain layer
// HTTP errors → Controller layer
// Security errors → Middleware layer
```

**"Where do I add caching?"**
```typescript
// Repository layer → Data caching
// Service layer → Business logic results
// Controller layer → HTTP responses (rare)
```

**"Where do I add logging?"**
```typescript
// Critical business events → Domain layer
// Request/response → Controller layer
// External service calls → Infrastructure layer
// Errors → All layers
```

#### Architecture Maturity Model

**Level 1: Basic CRUD**
- Simple repository pattern
- Minimal business logic
- Basic error handling
- No caching
- No events

**Level 2: Business Rules**
- Logic classes added
- Custom validation
- Domain events
- Basic caching
- Structured logging

**Level 3: Advanced Patterns**
- Event sourcing
- CQRS implementation
- Multi-level caching
- Circuit breakers
- Distributed tracing

**Level 4: Microservices**
- Service decomposition
- Message queues
- API gateway
- Service mesh
- Advanced monitoring

**Current BYO DPP Level**: Primarily Level 2, with some Level 3 features.

### Architecture Evolution Roadmap

#### Short-Term (Next 3 Months)
- [ ] Implement comprehensive event sourcing for audit trail
- [ ] Add Elasticsearch for advanced product search
- [ ] Implement Redis cluster for high-availability caching
- [ ] Add distributed tracing with OpenTelemetry
- [ ] Improve test coverage to 80%+

#### Medium-Term (6-12 Months)
- [ ] Migrate to microservices architecture
- [ ] Implement CQRS pattern for read/write separation
- [ ] Add message queue (RabbitMQ/Kafka) for event processing
- [ ] Implement GraphQL alongside REST
- [ ] Add real-time WebSocket support

#### Long-Term (12+ Months)
- [ ] Implement service mesh (Istio/Linkerd)
- [ ] Add machine learning for recommendations
- [ ] Implement blockchain for immutable audit trail
- [ ] Add multi-region deployment
- [ ] Implement advanced analytics platform

### Developer Decision Tree

Use this decision tree when implementing new features:

```
Is it a new resource?
├─ YES → Follow resource creation checklist
│  ├─ Create schema (validation + database)
│  ├─ Register in bootstrap.ts
│  └─ Test CRUD operations
│
└─ NO → Is it custom business logic?
   ├─ YES → Create logic class
   │  ├─ Extend ILogic interface
   │  ├─ Implement business rules
   │  ├─ Register in logic map
   │  └─ Write comprehensive tests
   │
   └─ NO → Is it cross-cutting concern?
      ├─ YES → Create middleware
      │  ├─ Implement IMiddleware
      │  ├─ Add to middleware pipeline
      │  └─ Test middleware behavior
      │
      └─ NO → Is it external integration?
         ├─ YES → Create adapter
         │  ├─ Define interface
         │  ├─ Implement adapter
         │  ├─ Register in DI container
         │  └─ Handle errors gracefully
         │
         └─ NO → Is it infrastructure concern?
            └─ YES → Add to infrastructure layer
               ├─ Create service/adapter
               ├─ Define interface
               ├─ Register in bootstrap
               └─ Document configuration
```

### Quick Architecture Reference Card

**Print this for your desk:**

```
┌─────────────────────────────────────────────────────────┐
│            BYO DPP ARCHITECTURE QUICK REFERENCE         │
├─────────────────────────────────────────────────────────┤
│                                                          │
│ LAYER RULES:                                            │
│ • Dependencies flow downward only                       │
│ • Communicate through interfaces                        │
│ • No layer skipping                                     │
│                                                          │
│ FILE LOCATIONS:                                         │
│ • Controllers:    src/application/adapters/controller/  │
│ • Services:       src/application/adapters/service/     │
│ • Logic Classes:  src/core/logics/v1/<resource>/       │
│ • Repositories:   src/infra/adapters/repository/        │
│ • Schemas:        src/core/schemas/resource/            │
│                                                          │
│ COMMON TASKS:                                           │
│ • Add resource:   bootstrap.ts → buildRoutes()          │
│ • Add validation: src/core/schemas/resource/<name>/     │
│ • Add logic:      src/core/logics/v1/<resource>/        │
│ • Add middleware: src/infra/middlewares/                │
│                                                          │
│ ERROR HANDLING:                                         │
│ • Use ErrorFactory for consistent errors                │
│ • Catch at appropriate layer                            │
│ • Log with context                                      │
│ • Return structured response                            │
│                                                          │
│ TESTING:                                                │
│ • Unit: Test logic in isolation                         │
│ • Integration: Test with database                       │
│ • E2E: Test complete workflows                          │
│                                                          │
│ BEFORE COMMITTING:                                      │
│ npm run lint && npm test && npm run format-fix          │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### Architecture Documentation Standards

Every new architectural component should include:

```typescript
/**
 * ProductCreateLogic
 * 
 * @description Implements business rules for product creation
 * 
 * @businessRules
 * - Category must exist and be active
 * - Product name must be unique within category
 * - Price must be within category price range
 * - Metadata must conform to category schema
 * 
 * @dependencies
 * - ProductRepository: Data persistence
 * - CategoryRepository: Category validation
 * - AuditRepository: Audit trail creation
 * 
 * @events
 * - ProductCreated: Published after successful creation
 * 
 * @errors
 * - BadRequestError: Invalid input or category not found
 * - ConflictError: Duplicate product name
 * - ValidationError: Metadata validation fails
 * 
 * @performance
 * - Average: 50ms
 * - P95: 150ms
 * - Database queries: 3 (1 category check, 1 duplicate check, 1 create)
 * 
 * @testing
 * - Unit tests: 15 test cases covering all business rules
 * - Integration tests: 5 scenarios with actual database
 * 
 * @author Development Team
 * @since v1.0.0
 * @lastModified 2024-01-15
 */
export class ProductCreateLogic implements ILogic {
    // Implementation
}
```

### Conclusion: Architecture as Living System

The BYO DPP architecture is not a fixed structure but a living system that evolves with:

**1. Business Needs**
- New features require new patterns
- Changing requirements drive refactoring
- Performance needs influence design

**2. Technology Advancements**
- New tools and frameworks emerge
- Better practices are discovered
- Performance characteristics change

**3. Team Growth**
- More developers need clearer boundaries
- Distributed teams need better documentation
- Knowledge transfer becomes critical

**4. Scale Requirements**
- Traffic growth demands optimization
- Data growth requires new strategies
- Geographic expansion needs distribution

**Key Takeaways**:
- ✅ Architecture serves the business, not the other way around
- ✅ Start simple, add complexity only when justified
- ✅ Document decisions and rationale
- ✅ Review and refactor regularly
- ✅ Measure impact of architectural changes
- ✅ Learn from production issues
- ✅ Share knowledge across the team

**Remember**: Good architecture makes the easy things easy and the hard things possible. The BYO DPP architecture achieves this through:
- **Clear layer separation** for understanding
- **Flexible abstractions** for adaptability
- **Rich patterns** for solving complex problems
- **Comprehensive tooling** for productivity
- **Strong conventions** for consistency
- **Extensive documentation** for onboarding

        
        // 1. Stop application
        await this.stopApplication();
        
        // 2. Restore database
        await this.restoreDatabase(backupId);
        
        // 3. Restore files
        await this.restoreFiles(backupId);
        
        // 4. Restore configuration
        await this.restoreConfiguration(backupId);
        
        // 5. Verify restore
        await this.verifyRestore(backupId);
        
        // 6. Start application
        await this.startApplication();
        
        console.log('Restore completed successfully');
    }
}

// Automated backup schedule
class BackupScheduler {
    schedule(): void {
        // Full backup daily at 2 AM
        cron.schedule('0 2 * * *', async () => {
            await backupManager.performBackup('full');
        });

        // Incremental backup every 6 hours
        cron.schedule('0 */6 * * *', async () => {
            await backupManager.performBackup('incremental');
        });
    }
}
```

#### Failover and Recovery

```typescript
class HighAvailabilityManager {
    private primary: IDatabase;
    private replica: IDatabase;
    private isFailedOver: boolean = false;

    constructor(
        primary: IDatabase,
        replica: IDatabase
    ) {
        this.primary = primary;
        this.replica = replica;
        this.startHealthCheck();
    }

    async query<T>(sql: string, params?: any[]): Promise<T> {
        const db = this.isFailedOver ? this.replica : this.primary;
        
        try {
            return await db.query(sql, params);
        } catch (error) {
            if (!this.isFailedOver) {
                console.error('Primary database failed, failing over to replica');
                this.isFailedOver = true;
                return await this.replica.query(sql, params);
            }
            throw error;
        }
    }

    private startHealthCheck(): void {
        setInterval(async () => {
            const primaryHealthy = await this.checkHealth(this.primary);
            const replicaHealthy = await this.checkHealth(this.replica);

            if (!primaryHealthy && replicaHealthy && !this.isFailedOver) {
                console.warn('Primary database unhealthy, failing over to replica');
                this.isFailedOver = true;
                await this.notifyAdmins('Database failover occurred');
            } else if (primaryHealthy && this.isFailedOver) {
                console.log('Primary database recovered, failing back');
                this.isFailedOver = false;
                await this.notifyAdmins('Database failback completed');
            }
        }, 10000); // Check every 10 seconds
    }

    private async checkHealth(db: IDatabase): Promise<boolean> {
        try {
            await db.query('SELECT 1');
            return true;
        } catch (error) {
            return false;
        }
    }
}
```

### Capacity Planning and Auto-Scaling

#### Resource Usage Prediction

```typescript
class CapacityPlanner {
    private historicalData: ResourceUsage[] = [];

    recordUsage(usage: ResourceUsage): void {
        this.historicalData.push(usage);
        
        // Keep last 7 days
        const sevenDaysAgo = Date.now() - (7 * 24 * 60 * 60 * 1000);
        this.historicalData = this.historicalData.filter(
            u => u.timestamp.getTime() > sevenDaysAgo
        );
    }

    predictUsage(hoursAhead: number): PredictedUsage {
        // Simple linear regression
        const recentData = this.historicalData.slice(-168); // Last week
        
        const avgCpu = recentData.reduce((sum, u) => sum + u.cpu, 0) / recentData.length;
        const avgMemory = recentData.reduce((sum, u) => sum + u.memory, 0) / recentData.length;
        const avgRequests = recentData.reduce((sum, u) => sum + u.requests, 0) / recentData.length;

        // Calculate trend
        const cpuTrend = this.calculateTrend(recentData.map(u => u.cpu));
        const memoryTrend = this.calculateTrend(recentData.map(u => u.memory));

        return {
            predictedCpu: avgCpu + (cpuTrend * hoursAhead),
            predictedMemory: avgMemory + (memoryTrend * hoursAhead),
            predictedRequests: avgRequests * (1 + (cpuTrend / avgCpu)),
            confidence: this.calculateConfidence(recentData),
            recommendedReplicas: this.calculateRecommendedReplicas(avgCpu, avgMemory)
        };
    }

    private calculateTrend(data: number[]): number {
        // Simple moving average
        if (data.length < 2) return 0;
        
        const recent = data.slice(-24); // Last 24 hours
        const older = data.slice(-48, -24); // Previous 24 hours
        
        const recentAvg = recent.reduce((a, b) => a + b) / recent.length;
        const olderAvg = older.reduce((a, b) => a + b) / older.length;
        
        return recentAvg - olderAvg;
    }

    private calculateRecommendedReplicas(cpu: number, memory: number): number {
        // Scale when CPU > 70% or Memory > 80%
        if (cpu > 0.7 || memory > 0.8) {
            return Math.ceil(cpu / 0.5); // Target 50% CPU per replica
        }
        return 1;
    }
}
```

#### Auto-Scaling Controller

```typescript
class AutoScalingController {
    constructor(
        private capacityPlanner: CapacityPlanner,
        private kubernetesClient: IKubernetesClient
    ) {}

    async evaluateScaling(): Promise<void> {
        const currentUsage = await this.getCurrentUsage();
        this.capacityPlanner.recordUsage(currentUsage);

        const prediction = this.capacityPlanner.predictUsage(1);
        const currentReplicas = await this.getCurrentReplicas();

        if (prediction.recommendedReplicas > currentReplicas) {
            // Scale up
            await this.scaleUp(prediction.recommendedReplicas);
        } else if (prediction.recommendedReplicas < currentReplicas) {
            // Scale down (with grace period)
            await this.scaleDown(prediction.recommendedReplicas);
        }
    }

    private async scaleUp(targetReplicas: number): Promise<void> {
        console.log(`Scaling up to ${targetReplicas} replicas`);
        
        await this.kubernetesClient.scale('byo-dpp-app', targetReplicas);
        
        // Wait for new replicas to be healthy
        await this.waitForHealthy(targetReplicas);
        
        console.log('Scale up completed');
    }

    private async scaleDown(targetReplicas: number): Promise<void> {
        const current = await this.getCurrentReplicas();
        
        // Don't scale down too aggressively
        const safeTarget = Math.max(targetReplicas, Math.ceil(current * 0.7));
        
        console.log(`Scaling down to ${safeTarget} replicas`);
        
        await this.kubernetesClient.scale('byo-dpp-app', safeTarget);
        
        console.log('Scale down completed');
    }
}
```

### Advanced Monitoring and Alerting

#### Custom Metrics and Alerts

```typescript
class BusinessMetricsCollector {
    constructor(private metrics: IMetrics) {}

    // Business-specific metrics
    recordProductCreation(product: Product): void {
        this.metrics.incrementCounter('products_created_total', {
            category: product.categoryId.toString(),
            tenant: product.tenantId
        });

        this.metrics.observeHistogram('product_price_distribution', product.price, {
            category: product.categoryId.toString()
        });
    }

    recordDppProcessing(dpp: Dpp, duration: number): void {
        this.metrics.incrementCounter('dpp_processing_total', {
            status: dpp.status,
            template: dpp.templateId.toString()
        });

        this.metrics.observeHistogram('dpp_processing_duration_seconds', duration / 1000, {
            template: dpp.templateId.toString()
        });
    }

    recordUserActivity(userId: string, action: string): void {
        this.metrics.incrementCounter('user_actions_total', {
            action,
            userId
        });
    }
}

// Alert Manager
class AlertManager {
    private rules: AlertRule[] = [
        {
            name: 'high-error-rate',
            condition: (metrics) => metrics.errorRate > 0.05,
            severity: 'critical',
            message: 'Error rate exceeds 5%'
        },
        {
            name: 'slow-response-time',
            condition: (metrics) => metrics.p95ResponseTime > 2000,
            severity: 'warning',
            message: 'P95 response time exceeds 2 seconds'
        },
        {
            name: 'high-memory-usage',
            condition: (metrics) => metrics.memoryUsage > 0.9,
            severity: 'critical',
            message: 'Memory usage exceeds 90%'
        },
        {
            name: 'database-connection-pool-exhausted',
            condition: (metrics) => metrics.dbPoolWaiting > 10,
            severity: 'critical',
            message: 'Database connection pool has 10+ waiting requests'
        }
    ];

    async evaluateAlerts(metrics: SystemMetrics): Promise<void> {
        for (const rule of this.rules) {
            if (rule.condition(metrics)) {
                await this.triggerAlert({
                    rule: rule.name,
                    severity: rule.severity,
                    message: rule.message,
                    metrics,
                    timestamp: new Date()
                });
            }
        }
    }

    private async triggerAlert(alert: Alert): Promise<void> {
        console.error('ALERT:', alert);

        // Send to PagerDuty
        if (alert.severity === 'critical') {
            await this.pagerDuty.createIncident(alert);
        }

        // Send to Slack
        await this.slack.sendMessage('#alerts', this.formatAlert(alert));

        // Store alert for tracking
        await this.alertStore.save(alert);
    }

    private formatAlert(alert: Alert): string {
        return `
🚨 *${alert.severity.toUpperCase()}*: ${alert.message}

*Rule*: ${alert.rule}
*Timestamp*: ${alert.timestamp.toISOString()}

*Metrics*:
- Error Rate: ${(alert.metrics.errorRate * 100).toFixed(2)}%
- Response Time (P95): ${alert.metrics.p95ResponseTime}ms
- Memory Usage: ${(alert.metrics.memoryUsage * 100).toFixed(2)}%
- CPU Usage: ${(alert.metrics.cpuUsage * 100).toFixed(2)}%
        `;
    }
}
```

### Documentation as Code

#### Architecture Documentation Generation

```typescript
class ArchitectureDocGenerator {
    async generateDocumentation(): Promise<void> {
        const doc = {
            components: await this.discoverComponents(),
            dependencies: await this.analyzeDependencies(),
            apiEndpoints: await this.documentEndpoints(),
            databaseSchema: await this.documentSchema(),
            eventFlows: await this.documentEvents()
        };

        await this.generateMarkdown(doc);
        await this.generateDiagrams(doc);
    }

    private async discoverComponents(): Promise<Component[]> {
        const components: Component[] = [];
        
        // Scan source code for components
        const files = await this.getAllTypeScriptFiles();
        
        for (const file of files) {
            const classes = this.extractClasses(file);
            
            for (const cls of classes) {
                components.push({
                    name: cls.name,
                    type: this.inferComponentType(cls),
                    file: file.path,
                    dependencies: cls.constructorParams.map(p => p.type),
                    methods: cls.methods.map(m => m.name)
                });
            }
        }
        
        return components;
    }

    private async analyzeDependencies(): Promise<DependencyGraph> {
        const graph: DependencyGraph = {
            nodes: [],
            edges: []
        };

        const components = await this.discoverComponents();

        for (const component of components) {
            graph.nodes.push({
                id: component.name,
                type: component.type
            });

            for (const dep of component.dependencies) {
                graph.edges.push({
                    from: component.name,
                    to: dep,
                    type: 'depends-on'
                });
            }
        }

        return graph;
    }

    private async generateMarkdown(doc: ArchitectureDoc): Promise<void> {
        const markdown = `
# Architecture Documentation

## Components

${doc.components.map(c => `
### ${c.name}
- **Type**: ${c.type}
- **File**: ${c.file}
- **Dependencies**: ${c.dependencies.join(', ')}
- **Methods**: ${c.methods.join(', ')}
`).join('\n')}

## Dependency Graph

\`\`\`mermaid
graph TD
${doc.dependencies.edges.map(e => `    ${e.from} --> ${e.to}`).join('\n')}
\`\`\`

## API Endpoints

${doc.apiEndpoints.map(e => `
### ${e.method} ${e.path}
- **Description**: ${e.description}
- **Controller**: ${e.controller}
- **Auth Required**: ${e.authRequired ? 'Yes' : 'No'}
`).join('\n')}
        `;

        await fs.writeFile('ARCHITECTURE.md', markdown);
    }
}
```

### Architecture Evolution Timeline

#### Evolution Pattern: Strangler Fig

```typescript
// Gradually replace legacy system with new architecture

// Step 1: Facade Pattern - Create interface
interface IProductService {
    create(data: CreateProductData): Promise<Product>;
    getById(id: number): Promise<Product>;
}

// Step 2: Implement new service
class NewProductService implements IProductService {
    async create(data: CreateProductData): Promise<Product> {
        // New clean architecture implementation
        return await this.repository.create(data);
    }

    async getById(id: number): Promise<Product> {
        return await this.repository.findById(id);
    }
}

// Step 3: Adapter for legacy service
class LegacyProductServiceAdapter implements IProductService {
    constructor(private legacyService: LegacyProductService) {}

    async create(data: CreateProductData): Promise<Product> {
        // Adapt to legacy interface
        const legacyData = this.convertToLegacyFormat(data);
        const result = await this.legacyService.createProduct(legacyData);
        return this.convertFromLegacyFormat(result);
    }

    async getById(id: number): Promise<Product> {
        const result = await this.legacyService.getProduct(id);
        return this.convertFromLegacyFormat(result);
    }
}

// Step 4: Router to gradually migrate
class MigrationRouter {
    constructor(
        private newService: IProductService,
        private legacyService: IProductService,
        private migrationConfig: MigrationConfig
    ) {}

    async route(request: IRequest): Promise<Product> {
        // Use feature flag to gradually migrate
        if (this.shouldUseNewService(request)) {
            try {
                return await this.newService.create(request.body);
            } catch (error) {
                console.error('New service failed, falling back to legacy', error);
                return await this.legacyService.create(request.body);
            }
        } else {
            return await this.legacyService.create(request.body);
        }
    }

    private shouldUseNewService(request: IRequest): boolean {
        // Gradual rollout strategies
        
        // Strategy 1: Percentage-based
        if (this.migrationConfig.rolloutPercentage) {
            return Math.random() < this.migrationConfig.rolloutPercentage;
        }

        // Strategy 2: User-based (test users first)
        if (this.migrationConfig.testUsers?.includes(request.user?.id)) {
            return true;
        }

        // Strategy 3: Tenant-based
        if (this.migrationConfig.migrated Tenants?.includes(request.tenantId)) {
            return true;
        }

        return false;
    }
}
```

### Production Readiness Checklist

#### Comprehensive Production Checklist

```typescript
class ProductionReadinessChecker {
    async performChecks(): Promise<ReadinessReport> {
        const checks = [
            this.checkDatabaseConnections(),
            this.checkExternalServices(),
            this.checkConfiguration(),
            this.checkSecurity(),
            this.checkMonitoring(),
            this.checkBackups(),
            this.checkDocumentation(),
            this.checkTests()
        ];

        const results = await Promise.all(checks);
        const passed = results.every(r => r.passed);

        return {
            ready: passed,
            checks: results,
            timestamp: new Date()
        };
    }

    private async checkDatabaseConnections(): Promise<CheckResult> {
        try {
            // Check primary database
            await this.primary.query('SELECT 1');
            
            // Check replica if configured
            if (this.replica) {
                await this.replica.query('SELECT 1');
            }

            // Check connection pool health
            const poolStats = this.connectionPool.getStats();
            if (poolStats.waitingConnections > 5)
        return result.rows[0];
    }
}

// Usage in service with retry
class ProductService {
    async updateProduct(
        id: number,
        data: UpdateProductData,
        maxRetries: number = 3
    ): Promise<Product> {
        let attempt = 0;

        while (attempt < maxRetries) {
            try {
                // Get current version
                const current = await this.repository.findById(id);
                
                // Try to update with optimistic lock
                return await this.repository.update(id, data, current.version);
            } catch (error) {
                if (error instanceof ConcurrencyConflictError) {
                    attempt++;
                    if (attempt >= maxRetries) {
                        throw new Error('Max retries exceeded due to concurrent modifications');
                    }
                    // Exponential backoff
                    await new Promise(resolve => 
                        setTimeout(resolve, Math.pow(2, attempt) * 100)
                    );
                } else {
                    throw error;
                }
            }
        }

        throw new Error('Update failed after retries');
    }
}
```

### Advanced Debugging Tools and Techniques

#### Request Correlation and Tracing

```typescript
class CorrelationIdManager {
    private static storage = new AsyncLocalStorage<CorrelationContext>();

    static initializeContext(correlationId?: string): void {
        const context: CorrelationContext = {
            correlationId: correlationId || generateCorrelationId(),
            requestId: generateRequestId(),
            startTime: Date.now(),
            metadata: {}
        };

        this.storage.enterWith(context);
    }

    static getContext(): CorrelationContext {
        const context = this.storage.getStore();
        if (!context) {
            throw new Error('No correlation context initialized');
        }
        return context;
    }

    static addMetadata(key: string, value: any): void {
        const context = this.getContext();
        context.metadata[key] = value;
    }
}

// Middleware to initialize correlation
class CorrelationMiddleware implements IMiddleware {
    async execute(request: IRequest, response: IResponse, next: NextFunction) {
        // Extract or generate correlation ID
        const correlationId = request.headers['x-correlation-id'] || generateCorrelationId();
        
        // Initialize context
        CorrelationIdManager.initializeContext(correlationId);
        
        // Add to response headers
        response.setHeader('x-correlation-id', correlationId);
        
        try {
            await next();
        } catch (error) {
            // Log with correlation context
            const context = CorrelationIdManager.getContext();
            console.error('Request failed', {
                correlationId: context.correlationId,
                requestId: context.requestId,
                duration: Date.now() - context.startTime,
                error: error.message,
                stack: error.stack,
                metadata: context.metadata
            });
            throw error;
        }
    }
}

// Usage in services
class ProductService {
    async createProduct(data: CreateProductData): Promise<Product> {
        const context = CorrelationIdManager.getContext();
        
        // All logs will include correlation ID
        console.log('Creating product', {
            correlationId: context.correlationId,
            productName: data.name
        });

        CorrelationIdManager.addMetadata('operation', 'create_product');
        CorrelationIdManager.addMetadata('productName', data.name);

        const product = await this.repository.create(data);

        console.log('Product created', {
            correlationId: context.correlationId,
            productId: product.id
        });

        return product;
    }
}
```

#### Query Performance Analyzer

```typescript
class QueryPerformanceAnalyzer {
    private slowQueries: SlowQuery[] = [];
    private queryStats: Map<string, QueryStats> = new Map();

    async analyzeQuery<T>(
        queryName: string,
        query: () => Promise<T>
    ): Promise<T> {
        const startTime = Date.now();
        const startMemory = process.memoryUsage().heapUsed;

        try {
            const result = await query();
            const duration = Date.now() - startTime;
            const memoryUsed = process.memoryUsage().heapUsed - startMemory;

            this.recordQueryStats(queryName, {
                duration,
                memoryUsed,
                success: true
            });

            // Log slow queries
            if (duration > 1000) {
                this.slowQueries.push({
                    queryName,
                    duration,
                    timestamp: new Date(),
                    stackTrace: new Error().stack
                });

                console.warn('Slow query detected', {
                    queryName,
                    duration,
                    memoryUsed
                });
            }

            return result;
        } catch (error) {
            const duration = Date.now() - startTime;
            
            this.recordQueryStats(queryName, {
                duration,
                memoryUsed: process.memoryUsage().heapUsed - startMemory,
                success: false,
                error: error.message
            });

            throw error;
        }
    }

    private recordQueryStats(queryName: string, stats: QueryStats): void {
        if (!this.queryStats.has(queryName)) {
            this.queryStats.set(queryName, {
                queryName,
                totalExecutions: 0,
                successfulExecutions: 0,
                failedExecutions: 0,
                avgDuration: 0,
                maxDuration: 0,
                minDuration: Infinity,
                avgMemoryUsed: 0
            });
        }

        const current = this.queryStats.get(queryName)!;
        current.totalExecutions++;

        if (stats.success) {
            current.successfulExecutions++;
        } else {
            current.failedExecutions++;
        }

        current.avgDuration = 
            (current.avgDuration * (current.totalExecutions - 1) + stats.duration) 
            / current.totalExecutions;

        current.maxDuration = Math.max(current.maxDuration, stats.duration);
        current.minDuration = Math.min(current.minDuration, stats.duration);

        current.avgMemoryUsed = 
            (current.avgMemoryUsed * (current.totalExecutions - 1) + stats.memoryUsed) 
            / current.totalExecutions;
    }

    getSlowQueries(threshold: number = 1000): SlowQuery[] {
        return this.slowQueries.filter(q => q.duration > threshold);
    }

    getQueryStats(): QueryStats[] {
        return Array.from(this.queryStats.values());
    }

    generatePerformanceReport(): string {
        const stats = this.getQueryStats();
        const slowQueries = this.getSlowQueries();

        return `
Performance Report
==================

Total Queries: ${stats.reduce((sum, s) => sum + s.totalExecutions, 0)}
Slow Queries: ${slowQueries.length}

Top 10 Slowest Queries:
${this.getSlowQueries()
    .sort((a, b) => b.duration - a.duration)
    .slice(0, 10)
    .map((q, i) => `${i + 1}. ${q.queryName}: ${q.duration}ms`)
    .join('\n')}

Query Statistics:
${stats
    .sort((a, b) => b.avgDuration - a.avgDuration)
    .slice(0, 10)
    .map(s => `${s.queryName}:
    - Executions: ${s.totalExecutions}
    - Avg Duration: ${s.avgDuration.toFixed(2)}ms
    - Max Duration: ${s.maxDuration}ms
    - Success Rate: ${((s.successfulExecutions / s.totalExecutions) * 100).toFixed(2)}%`)
    .join('\n\n')}
        `;
    }
}
```

### Architecture Trade-offs Analysis

#### Trade-off 1: Consistency vs Availability (CAP Theorem)

```typescript
// Scenario: Product inventory update during high traffic

// Option A: Strong Consistency (CP)
class StronglyConsistentInventoryService {
    async updateInventory(productId: string, quantity: number): Promise<void> {
        // Use distributed lock
        const lock = await this.redis.acquireLock(`inventory:${productId}`, 5000);
        
        try {
            // Read current inventory
            const current = await this.repository.getInventory(productId);
            
            // Update with transaction
            await this.db.transaction(async (tx) => {
                await tx.update('inventory', {
                    quantity: current.quantity + quantity,
                    version: current.version + 1
                }).where({ id: productId, version: current.version });
            });
        } finally {
            await lock.release();
        }
    }
}

// Option B: Eventual Consistency (AP)
class EventuallyConsistentInventoryService {
    async updateInventory(productId: string, quantity: number): Promise<void> {
        // No locks - accept all updates immediately
        const event = new InventoryUpdatedEvent(productId, quantity, new Date());
        
        // Publish event (async)
        await this.eventBus.publish(event);
        
        // Return immediately
        return;
    }

    // Background process reconciles inventory
    async reconcileInventory(): Promise<void> {
        const events = await this.eventStore.getUnprocessedEvents('InventoryUpdated');
        
        for (const event of events) {
            try {
                // Apply event to inventory
                await this.applyInventoryUpdate(event);
                await this.eventStore.markProcessed(event.id);
            } catch (error) {
                console.error('Failed to process inventory event:', error);
                // Retry later
            }
        }
    }
}
```

**Decision Matrix**:
| Requirement | Strong Consistency | Eventual Consistency |
|-------------|-------------------|---------------------|
| Financial transactions | ✅ Required | ❌ Not acceptable |
| Inventory updates | ✅ Preferred | ⚠️ Acceptable with reconciliation |
| Social media likes | ❌ Overkill | ✅ Perfect fit |
| User profiles | ✅ Preferred | ⚠️ Acceptable |
| Analytics data | ❌ Overkill | ✅ Perfect fit |

#### Trade-off 2: Normalization vs Denormalization

```typescript
// Normalized Design (3NF)
// Products Table
interface Product {
    id: number;
    name: string;
    categoryId: number; // Foreign key
}

// Categories Table
interface Category {
    id: number;
    name: string;
    description: string;
}

class NormalizedProductQuery {
    async getProductsWithCategories(): Promise<ProductWithCategory[]> {
        // Requires JOIN
        return await this.db.query(`
            SELECT 
                p.*,
                c.name as category_name,
                c.description as category_description
            FROM products p
            JOIN categories c ON p.category_id = c.id
        `);
    }
}

// Denormalized Design
interface DenormalizedProduct {
    id: number;
    name: string;
    categoryId: number;
    categoryName: string; // Duplicated data
    categoryDescription: string; // Duplicated data
}

class DenormalizedProductQuery {
    async getProducts(): Promise<DenormalizedProduct[]> {
        // No JOIN needed - faster reads
        return await this.db.query(`
            SELECT * FROM products
        `);
    }

    // But updates are more complex
    async updateCategory(categoryId: number, newName: string): Promise<void> {
        // Must update all products in that category
        await this.db.transaction(async (tx) => {
            // Update category
            await tx.query(`
                UPDATE categories
                SET name = $1
                WHERE id = $2
            `, [newName, categoryId]);

            // Update denormalized data in products
            await tx.query(`
                UPDATE products
                SET category_name = $1
                WHERE category_id = $2
            `, [newName, categoryId]);
        });
    }
}
```

**When to Use Each**:
| Use Case | Normalization | Denormalization |
|----------|--------------|-----------------|
| **Read-heavy workload** | ⚠️ Can be slow | ✅ Fast reads |
| **Write-heavy workload** | ✅ Simple writes | ❌ Complex writes |
| **Data integrity critical** | ✅ Single source of truth | ⚠️ Risk of inconsistency |
| **Real-time analytics** | ❌ Too slow | ✅ Pre-aggregated data |
| **Storage cost sensitive** | ✅ Minimal redundancy | ❌ Data duplication |

### Performance Benchmarking Framework

#### Load Testing Framework

```typescript
class LoadTester {
    async runLoadTest(config: LoadTestConfig): Promise<LoadTestResult> {
        const results: RequestResult[] = [];
        const concurrency = config.concurrency || 10;
        const duration = config.duration || 60000; // 60 seconds
        const endpoint = config.endpoint;

        console.log(`Starting load test: ${endpoint}`);
        console.log(`Concurrency: ${concurrency}, Duration: ${duration}ms`);

        const startTime = Date.now();
        const workers: Promise<void>[] = [];

        // Spawn concurrent workers
        for (let i = 0; i < concurrency; i++) {
            workers.push(this.runWorker(endpoint, duration, startTime, results));
        }

        await Promise.all(workers);

        return this.analyzeResults(results, Date.now() - startTime);
    }

    private async runWorker(
        endpoint: string,
        duration: number,
        startTime: number,
        results: RequestResult[]
    ): Promise<void> {
        while (Date.now() - startTime < duration) {
            const requestStart = Date.now();
            
            try {
                const response = await fetch(endpoint, {
                    method: 'GET',
                    headers: {
                        'Authorization': `Bearer ${testToken}`
                    }
                });

                results.push({
                    duration: Date.now() - requestStart,
                    statusCode: response.status,
                    success: response.ok,
                    timestamp: new Date()
                });
            } catch (error) {
                results.push({
                    duration: Date.now() - requestStart,
                    statusCode: 0,
                    success: false,
                    error: error.message,
                    timestamp: new Date()
                });
            }
        }
    }

    private analyzeResults(results: RequestResult[], totalDuration: number): LoadTestResult {
        const successful = results.filter(r => r.success);
        const failed = results.filter(r => !r.success);

        const durations = results.map(r => r.duration).sort((a, b) => a - b);
        const p50 = durations[Math.floor(durations.length * 0.5)];
        const p95 = durations[Math.floor(durations.length * 0.95)];
        const p99 = durations[Math.floor(durations.length * 0.99)];

        return {
            totalRequests: results.length,
            successfulRequests: successful.length,
            failedRequests: failed.length,
            successRate: (successful.length / results.length) * 100,
            avgDuration: durations.reduce((a, b) => a + b, 0) / durations.length,
            minDuration: Math.min(...durations),
            maxDuration: Math.max(...durations),
            p50Duration: p50,
            p95Duration: p95,
            p99Duration: p99,
            requestsPerSecond: (results.length / totalDuration) * 1000,
            errors: this.groupErrors(failed)
        };
    }

    private groupErrors(failed: RequestResult[]): Record<string, number> {
        return failed.reduce((acc, result) => {
            const key = result.error || `HTTP ${result.statusCode}`;
            acc[key] = (acc[key] || 0) + 1;
            return acc;
        }, {} as Record<string, number>);
    }
}

// Usage
const loadTester = new LoadTester();

const result = await loadTester.runLoadTest({
    endpoint: 'http://localhost:3000/api/v1/products',
    concurrency: 50,
    duration: 60000
});

console.log(`
Load Test Results:
==================
Total Requests: ${result.totalRequests}
Success Rate: ${result.successRate.toFixed(2)}%
Requests/sec: ${result.requestsPerSecond.toFixed(2)}

Response Times:
- Average: ${result.avgDuration.toFixed(2)}ms
- P50: ${result.p50Duration}ms
- P95: ${result.p95Duration}ms
- P99: ${result.p99Duration}ms
- Min: ${result.minDuration}ms
- Max: ${result.maxDuration}ms

Errors:
${Object.entries(result.errors).map(([error, count]) => `- ${error}: ${count}`).join('\n')}
`);
```

### Real-World Architecture Case Studies

#### Case Study 1: Scaling Product Search

**Challenge**: Product search becomes slow with 100,000+ products.

**Solution Evolution**:

**Phase 1: Basic Database Query**
```typescript
class ProductSearchV1 {
    async search(query: string): Promise<Product[]> {
        return await this.db.query(`
            SELECT * FROM products
            WHERE name ILIKE $1
            LIMIT 20
        `, [`%${query}%`]);
    }
}
// Performance: ~500ms avg
// Problems: Full table scan, slow LIKE queries
```

**Phase 2: Add Database Indexes**
```sql
CREATE INDEX idx_products_name_trgm ON products 
USING gin (name gin_trgm_ops);
```
```typescript
class ProductSearchV2 {
    async search(query: string): Promise<Product[]> {
        return await this.db.query(`
            SELECT * FROM products
            WHERE name ILIKE $1
            ORDER BY similarity(name, $2) DESC
            LIMIT 20
        `, [`%${query}%`, query]);
    }
}
// Performance: ~100ms avg
// Improvement: 5x faster
```

**Phase 3: Add Redis Cache**
```typescript
class ProductSearchV3 {
    async search(query: string): Promise<Product[]> {
        const cacheKey = `search:${query.toLowerCase()}`;
        
        // Check cache
        const cached = await this.redis.get(cacheKey);
        if (cached) {
            return JSON.parse(cached);
        }

        // Query database
        const results = await this.db.query(`
            SELECT * FROM products
            WHERE name ILIKE $1
            LIMIT 20
        `, [`%${query}%`]);

        // Cache for 5 minutes
        await this.redis.set(cacheKey, JSON.stringify(results), 300);

        return results;
    }
}
// Performance: ~10ms avg (cache hit), ~100ms (cache miss)
// Improvement: 10x faster for repeated queries
```

**Phase 4: Elasticsearch Integration**
```typescript
class ProductSearchV4 {
    constructor(
        private elasticsearch: Client,
        private db: IDatabase
    ) {}

    async search(query: string): Promise<Product[]> {
        // Search in Elasticsearch
        const esResults = await this.elasticsearch.search({
            index: 'products',
            body: {
                query: {
                    multi_match: {
                        query,
                        fields: ['name^3', 'description', 'category'],
                        fuzziness: 'AUTO'
                    }
                },
                size: 20
            }
        });

        const productIds = esResults.hits.hits.map(hit => hit._id);

        // Fetch full data from database
        return await this.db.query(`
            SELECT * FROM products
            WHERE id = ANY($1)
        `, [productIds]);
    }

    
            span.setStatus({
                code: SpanStatusCode.ERROR,
                message: error.message
            });
            span.recordException(error);
            throw error;
        } finally {
            span.end();
        }
    }
}

// Usage in service
class ProductService {
    constructor(
        private repository: IProductRepository,
        private tracing: TracingService
    ) {}

    async createProduct(data: CreateProductData): Promise<Product> {
        return await this.tracing.traceOperation(
            'ProductService.createProduct',
            async (span) => {
                span.setAttribute('product.name', data.name);
                span.setAttribute('product.categoryId', data.categoryId);

                // Trace validation
                const validatedData = await this.tracing.traceOperation(
                    'validate-product',
                    async () => await this.validator.validate(data)
                );

                // Trace database operation
                const product = await this.tracing.traceOperation(
                    'create-product-db',
                    async () => await this.repository.create(validatedData)
                );

                span.setAttribute('product.id', product.id);
                return product;
            },
            { 'http.method': 'POST', 'http.route': '/products' }
        );
    }
}
```

#### Application Performance Monitoring (APM)

```typescript
class PerformanceMonitor {
    private metrics: Map<string, PerformanceMetric> = new Map();

    async measureOperation<T>(
        operationName: string,
        operation: () => Promise<T>
    ): Promise<T> {
        const startTime = Date.now();
        const startMemory = process.memoryUsage().heapUsed;
        
        try {
            const result = await operation();
            const duration = Date.now() - startTime;
            const memoryDelta = process.memoryUsage().heapUsed - startMemory;
            
            this.recordMetric(operationName, {
                duration,
                memoryDelta,
                success: true
            });
            
            return result;
        } catch (error) {
            const duration = Date.now() - startTime;
            
            this.recordMetric(operationName, {
                duration,
                memoryDelta: process.memoryUsage().heapUsed - startMemory,
                success: false,
                error: error.message
            });
            
            throw error;
        }
    }

    private recordMetric(operationName: string, metric: PerformanceMetric): void {
        if (!this.metrics.has(operationName)) {
            this.metrics.set(operationName, {
                operationName,
                totalCalls: 0,
                successfulCalls: 0,
                failedCalls: 0,
                avgDuration: 0,
                maxDuration: 0,
                minDuration: Infinity,
                avgMemoryDelta: 0
            });
        }

        const current = this.metrics.get(operationName)!;
        current.totalCalls++;
        
        if (metric.success) {
            current.successfulCalls++;
        } else {
            current.failedCalls++;
        }

        // Update duration stats
        current.avgDuration = 
            (current.avgDuration * (current.totalCalls - 1) + metric.duration) 
            / current.totalCalls;
        current.maxDuration = Math.max(current.maxDuration, metric.duration);
        current.minDuration = Math.min(current.minDuration, metric.duration);

        // Update memory stats
        current.avgMemoryDelta = 
            (current.avgMemoryDelta * (current.totalCalls - 1) + metric.memoryDelta) 
            / current.totalCalls;
    }

    getMetrics(): PerformanceMetric[] {
        return Array.from(this.metrics.values());
    }

    getSlowOperations(threshold: number = 1000): PerformanceMetric[] {
        return this.getMetrics().filter(m => m.avgDuration > threshold);
    }
}

// Usage
const monitor = new PerformanceMonitor();

class ProductService {
    async createProduct(data: CreateProductData): Promise<Product> {
        return await monitor.measureOperation('ProductService.createProduct', async () => {
            return await this.repository.create(data);
        });
    }
}
```

### Event Sourcing Deep Dive

#### Complete Event Sourcing Implementation

```typescript
// Event Store Interface
interface IEventStore {
    append(event: DomainEvent): Promise<void>;
    getEvents(aggregateId: string): Promise<DomainEvent[]>;
    getEventsAfter(aggregateId: string, version: number): Promise<DomainEvent[]>;
}

// Event Store Implementation
class PostgresEventStore implements IEventStore {
    constructor(private db: IDatabase) {}

    async append(event: DomainEvent): Promise<void> {
        await this.db.query(`
            INSERT INTO event_store (
                aggregate_id,
                aggregate_type,
                event_type,
                event_data,
                version,
                occurred_at
            ) VALUES ($1, $2, $3, $4, $5, $6)
        `, [
            event.aggregateId,
            event.aggregateType,
            event.type,
            JSON.stringify(event.data),
            event.version,
            event.occurredAt
        ]);
    }

    async getEvents(aggregateId: string): Promise<DomainEvent[]> {
        const result = await this.db.query(`
            SELECT * FROM event_store
            WHERE aggregate_id = $1
            ORDER BY version ASC
        `, [aggregateId]);

        return result.rows.map(row => this.deserializeEvent(row));
    }

    async getEventsAfter(aggregateId: string, version: number): Promise<DomainEvent[]> {
        const result = await this.db.query(`
            SELECT * FROM event_store
            WHERE aggregate_id = $1 AND version > $2
            ORDER BY version ASC
        `, [aggregateId, version]);

        return result.rows.map(row => this.deserializeEvent(row));
    }

    private deserializeEvent(row: any): DomainEvent {
        return {
            aggregateId: row.aggregate_id,
            aggregateType: row.aggregate_type,
            type: row.event_type,
            data: JSON.parse(row.event_data),
            version: row.version,
            occurredAt: row.occurred_at
        };
    }
}

// Aggregate with Event Sourcing
class ProductAggregate {
    private events: DomainEvent[] = [];
    private version: number = 0;

    constructor(
        private id: string,
        private name: string,
        private price: Money,
        private categoryId: string
    ) {}

    // Command: Change price
    changePrice(newPrice: Money): void {
        // Business rule validation
        if (newPrice.amount <= 0) {
            throw new Error('Price must be positive');
        }

        const oldPrice = this.price;
        this.price = newPrice;
        this.version++;

        // Append event
        this.appendEvent(new ProductPriceChangedEvent(
            this.id,
            oldPrice,
            newPrice,
            this.version,
            new Date()
        ));
    }

    // Command: Change category
    changeCategory(newCategoryId: string): void {
        if (this.categoryId === newCategoryId) {
            throw new Error('Product already in this category');
        }

        const oldCategoryId = this.categoryId;
        this.categoryId = newCategoryId;
        this.version++;

        this.appendEvent(new ProductCategoryChangedEvent(
            this.id,
            oldCategoryId,
            newCategoryId,
            this.version,
            new Date()
        ));
    }

    private appendEvent(event: DomainEvent): void {
        this.events.push(event);
    }

    getUncommittedEvents(): DomainEvent[] {
        return [...this.events];
    }

    clearUncommittedEvents(): void {
        this.events = [];
    }

    // Reconstruct from events
    static fromEvents(events: DomainEvent[]): ProductAggregate {
        if (events.length === 0) {
            throw new Error('Cannot create aggregate from empty event stream');
        }

        const firstEvent = events[0];
        if (!(firstEvent instanceof ProductCreatedEvent)) {
            throw new Error('First event must be ProductCreatedEvent');
        }

        const aggregate = new ProductAggregate(
            firstEvent.data.id,
            firstEvent.data.name,
            firstEvent.data.price,
            firstEvent.data.categoryId
        );

        // Replay events
        for (let i = 1; i < events.length; i++) {
            aggregate.applyEvent(events[i]);
        }

        aggregate.version = events.length;
        return aggregate;
    }

    private applyEvent(event: DomainEvent): void {
        if (event instanceof ProductPriceChangedEvent) {
            this.price = event.data.newPrice;
        } else if (event instanceof ProductCategoryChangedEvent) {
            this.categoryId = event.data.newCategoryId;
        }
        // Handle other events...
    }
}

// Repository with Event Sourcing
class EventSourcedProductRepository {
    constructor(
        private eventStore: IEventStore,
        private snapshotStore: ISnapshotStore
    ) {}

    async save(aggregate: ProductAggregate): Promise<void> {
        const events = aggregate.getUncommittedEvents();
        
        // Append events to event store
        for (const event of events) {
            await this.eventStore.append(event);
        }

        // Clear uncommitted events
        aggregate.clearUncommittedEvents();

        // Create snapshot every 10 events for performance
        if (aggregate.version % 10 === 0) {
            await this.snapshotStore.save(aggregate);
        }
    }

    async load(aggregateId: string): Promise<ProductAggregate> {
        // Try to load from snapshot first
        const snapshot = await this.snapshotStore.load(aggregateId);
        
        if (snapshot) {
            // Load events after snapshot
            const events = await this.eventStore.getEventsAfter(
                aggregateId,
                snapshot.version
            );
            
            // Replay events on snapshot
            for (const event of events) {
                snapshot.applyEvent(event);
            }
            
            return snapshot;
        }

        // No snapshot, load all events
        const events = await this.eventStore.getEvents(aggregateId);
        return ProductAggregate.fromEvents(events);
    }
}
```

**Benefits of Event Sourcing**:
- **Complete Audit Trail**: Every change is recorded
- **Temporal Queries**: Can query state at any point in time
- **Event Replay**: Can rebuild state or fix bugs by replaying events
- **Analytics**: Rich data for business intelligence
- **Debugging**: Can trace exactly what happened

### Security Architecture Deep Dive

#### Defense in Depth Implementation

```typescript
class SecurityLayerManager {
    private layers: ISecurityLayer[] = [
        new NetworkSecurityLayer(),
        new ApplicationSecurityLayer(),
        new DataSecurityLayer(),
        new AuditSecurityLayer()
    ];

    async validateRequest(request: IRequest): Promise<SecurityResult> {
        const results: SecurityResult[] = [];

        for (const layer of this.layers) {
            const result = await layer.validate(request);
            results.push(result);

            if (!result.passed) {
                return {
                    passed: false,
                    layer: layer.name,
                    reason: result.reason,
                    remediation: result.remediation
                };
            }
        }

        return {
            passed: true,
            layers: results.map(r => r.layer)
        };
    }
}

// Layer 1: Network Security
class NetworkSecurityLayer implements ISecurityLayer {
    name = 'network';

    async validate(request: IRequest): Promise<SecurityResult> {
        // IP whitelist check
        if (!this.isIpWhitelisted(request.ip)) {
            return {
                passed: false,
                reason: 'IP not whitelisted',
                remediation: 'Contact admin to whitelist your IP'
            };
        }

        // Rate limiting
        const rateLimit = await this.checkRateLimit(request.ip);
        if (!rateLimit.allowed) {
            return {
                passed: false,
                reason: 'Rate limit exceeded',
                remediation: `Retry after ${rateLimit.retryAfter} seconds`
            };
        }

        // DDoS protection
        const requestPattern = await this.analyzeRequestPattern(request.ip);
        if (requestPattern.suspicious) {
            return {
                passed: false,
                reason: 'Suspicious request pattern detected',
                remediation: 'Complete CAPTCHA challenge'
            };
        }

        return { passed: true };
    }
}

// Layer 2: Application Security
class ApplicationSecurityLayer implements ISecurityLayer {
    name = 'application';

    async validate(request: IRequest): Promise<SecurityResult> {
        // JWT validation
        const tokenResult = await this.validateToken(request.headers.authorization);
        if (!tokenResult.valid) {
            return {
                passed: false,
                reason: tokenResult.reason,
                remediation: 'Obtain a new access token'
            };
        }

        // Permission check
        const hasPermission = await this.checkPermission(
            tokenResult.user,
            request.method,
            request.path
        );

        if (!hasPermission) {
            return {
                passed: false,
                reason: 'Insufficient permissions',
                remediation: 'Contact admin for permission grant'
            };
        }

        // Input validation
        const inputResult = this.validateInput(request.body);
        if (!inputResult.valid) {
            return {
                passed: false,
                reason: inputResult.reason,
                remediation: 'Fix input validation errors'
            };
        }

        return { passed: true };
    }
}

// Layer 3: Data Security
class DataSecurityLayer implements ISecurityLayer {
    name = 'data';

    async validate(request: IRequest): Promise<SecurityResult> {
        // Tenant isolation check
        const tenantId = TenantContext.getCurrentTenant();
        if (request.body.tenantId && request.body.tenantId !== tenantId) {
            return {
                passed: false,
                reason: 'Tenant boundary violation',
                remediation: 'Cannot access data from other tenants'
            };
        }

        // Data sensitivity check
        const sensitivity = this.classifyDataSensitivity(request.body);
        const userClearance = request.user.securityClearance;

        if (sensitivity > userClearance) {
            return {
                passed: false,
                reason: 'Insufficient security clearance',
                remediation: 'Elevated privileges required'
            };
        }

        // PII detection
        const containsPII = this.detectPII(request.body);
        if (containsPII && !request.secure) {
            return {
                passed: false,
                reason: 'PII transmission over insecure connection',
                remediation: 'Use HTTPS for sensitive data'
            };
        }

        return { passed: true };
    }

    private detectPII(data: any): boolean {
        const piiPatterns = [
            /\b\d{3}-\d{2}-\d{4}\b/, // SSN
            /\b\d{16}\b/, // Credit card
            /\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b/ // Email
        ];

        const dataString = JSON.stringify(data);
        return piiPatterns.some(pattern => pattern.test(dataString));
    }
}
```

### Migration and Evolution Strategies

#### Zero-Downtime Database Migration

```typescript
class ZeroDowntimeMigrationManager {
    async executeMigration(migration: Migration): Promise<void> {
        console.log(`Starting migration: ${migration.name}`);

        // Phase 1: Add new column with nullable
        await this.db.query(`
            ALTER TABLE products
            ADD COLUMN new_field VARCHAR(255) NULL;
        `);

        console.log('Phase 1: New column added (nullable)');

        // Phase 2: Backfill data (in batches to avoid locks)
        await this.backfillData('products', 'new_field', 
            (row) => this.calculateNewFieldValue(row)
        );

        console.log('Phase 2: Data backfilled');

        // Phase 3: Update application code to use new field
        // (This is deployed separately)

        // Phase 4: Make column NOT NULL
        await this.db.query(`
            ALTER TABLE products
            ALTER COLUMN new_field SET NOT NULL;
        `);

        console.log('Phase 4: Column made NOT NULL');

        // Phase 5: Remove old column (after verifying new field works)
        await this.db.query(`
            ALTER TABLE products
            DROP COLUMN old_field;
        `);

        console.log('Phase 5: Old column removed');
    }

    private async backfillData(
        table: string,
        column: string,
        calculator: (row: any) => any
    ): Promise<void> {
        const batchSize = 1000;
        let offset = 0;

        while (true) {
            const rows = await this.db.query(`
                SELECT * FROM ${table}
                WHERE ${column} IS NULL
                LIMIT ${batchSize} OFFSET ${offset}
            `);

            if (rows.length === 0) break;

            // Process in transaction
            await this.db.transaction(async (tx) => {
                for (const row of rows) {
                    const newValue = calculator(row);
                    await tx.query(`
                        UPDATE ${table}
                        SET ${column} = $1
                        WHERE id = $2
                    `, [newValue, row.id]);
                }
            });

            offset += batchSize;
            console.log(`Backfilled ${offset} rows`);
        }
    }
}
```

#### API Contract Evolution

```typescript
class ApiContractEvolution {
    // Backward Compatible Change: Add Optional Field
    static addOptionalField() {
        // Before (v1)
        interface ProductV1 {
            id: number;
            
    test('complete product management workflow', async ({ page }) => {
        // 1. Login
        await page.goto('http://localhost:3000/login');
        await page.fill('[name="username"]', 'testuser');
        await page.fill('[name="password"]', 'testpass');
        await page.click('button[type="submit"]');
        await expect(page).toHaveURL('http://localhost:3000/dashboard');

        // 2. Navigate to products page
        await page.click('a[href="/products"]');
        await expect(page).toHaveURL('http://localhost:3000/products');

        // 3. Create new product
        await page.click('button:has-text("Create Product")');
        await page.fill('[name="name"]', 'E2E Test Product');
        await page.selectOption('[name="categoryId"]', '1');
        await page.fill('[name="price"]', '99.99');
        await page.fill('[name="description"]', 'Test description');
        await page.click('button:has-text("Save")');

        // 4. Verify product appears in list
        await expect(page.locator('text=E2E Test Product')).toBeVisible();

        // 5. Edit product
        await page.click('button[aria-label="Edit E2E Test Product"]');
        await page.fill('[name="price"]', '149.99');
        await page.click('button:has-text("Save")');

        // 6. Verify price update
        await expect(page.locator('text=$149.99')).toBeVisible();

        // 7. Delete product
        await page.click('button[aria-label="Delete E2E Test Product"]');
        await page.click('button:has-text("Confirm")');

        // 8. Verify product is removed
        await expect(page.locator('text=E2E Test Product')).not.toBeVisible();
    });
});
```

### Performance Optimization Patterns

#### Database Query Optimization

**Problem**: N+1 Query Problem
```typescript
// ❌ PROBLEM: N+1 queries
class BadProductService {
    async getProductsWithCategories() {
        const products = await this.repository.list();
        
        for (const product of products) {
            // This creates N additional queries!
            product.category = await this.categoryRepo.findById(product.categoryId);
        }
        
        return products;
    }
}

// ✅ SOLUTION: Use eager loading
class OptimizedProductService {
    async getProductsWithCategories() {
        return await this.repository.list({
            include: [{
                resource: 'category',
                fields: ['id', 'name', 'description']
            }]
        });
    }
}
```

**Query Indexing Strategy**:
```sql
-- Analyze slow queries
EXPLAIN ANALYZE
SELECT p.*, c.name as category_name
FROM products p
JOIN categories c ON p.category_id = c.id
WHERE p.is_enabled = true
AND p.price BETWEEN 100 AND 500;

-- Add appropriate indexes
CREATE INDEX idx_products_enabled_price ON products(is_enabled, price);
CREATE INDEX idx_products_category_id ON products(category_id);

-- Verify index usage
EXPLAIN ANALYZE
SELECT p.*, c.name as category_name
FROM products p
JOIN categories c ON p.category_id = c.id
WHERE p.is_enabled = true
AND p.price BETWEEN 100 AND 500;
```

#### Caching Strategies

**Multi-Level Caching Implementation**:
```typescript
class AdvancedCacheManager {
    constructor(
        private l1Cache: NodeCache,      // Memory cache (fast, small)
        private l2Cache: Redis,           // Redis cache (medium, medium)
        private l3Cache: IDatabase        // Database (slow, large)
    ) {}

    async get<T>(key: string): Promise<T | null> {
        // L1: Memory cache (fastest)
        const l1Value = this.l1Cache.get<T>(key);
        if (l1Value !== undefined) {
            console.log(`Cache hit: L1 (${key})`);
            return l1Value;
        }

        // L2: Redis cache
        const l2Value = await this.l2Cache.get<T>(key);
        if (l2Value !== null) {
            console.log(`Cache hit: L2 (${key})`);
            // Populate L1 for future requests
            this.l1Cache.set(key, l2Value, 60);
            return l2Value;
        }

        // L3: Database
        const l3Value = await this.l3Cache.get<T>(key);
        if (l3Value !== null) {
            console.log(`Cache hit: L3 (${key})`);
            // Populate L2 and L1
            await this.l2Cache.set(key, l3Value, 3600);
            this.l1Cache.set(key, l3Value, 60);
            return l3Value;
        }

        return null;
    }

    async set<T>(key: string, value: T, ttl?: number): Promise<void> {
        // Write to all cache levels
        const l1Ttl = Math.min(ttl || 60, 60);
        const l2Ttl = ttl || 3600;

        this.l1Cache.set(key, value, l1Ttl);
        await this.l2Cache.set(key, value, l2Ttl);
    }

    async invalidate(pattern: string): Promise<void> {
        // Invalidate all cache levels
        this.l1Cache.keys().forEach(key => {
            if (new RegExp(pattern).test(key)) {
                this.l1Cache.del(key);
            }
        });

        await this.l2Cache.keys(pattern).then(keys => {
            return Promise.all(keys.map(key => this.l2Cache.del(key)));
        });
    }
}

// Usage with cache-aside pattern
class CachedProductService {
    async getProduct(id: number): Promise<Product> {
        const cacheKey = `product:${id}`;
        
        // Try to get from cache
        const cached = await this.cacheManager.get<Product>(cacheKey);
        if (cached) return cached;

        // Fetch from database
        const product = await this.repository.findById(id);
        
        // Store in cache
        await this.cacheManager.set(cacheKey, product, 3600);
        
        return product;
    }

    async updateProduct(id: number, data: UpdateProductData): Promise<Product> {
        // Update database
        const product = await this.repository.update(id, data);
        
        // Invalidate cache
        await this.cacheManager.invalidate(`product:${id}`);
        
        // Also invalidate related caches
        await this.cacheManager.invalidate(`category:${product.categoryId}:products`);
        
        return product;
    }
}
```

#### Connection Pooling Optimization

```typescript
class OptimizedDatabasePool {
    private pool: Pool;
    private metrics: {
        activeConnections: number;
        waitingConnections: number;
        idleConnections: number;
        totalQueries: number;
        avgQueryTime: number;
    };

    constructor(config: DatabaseConfig) {
        // Optimal pool size calculation
        const cpuCount = require('os').cpus().length;
        const maxConnections = cpuCount * 2 + 1; // Common formula

        this.pool = new Pool({
            ...config,
            max: maxConnections,
            min: Math.ceil(maxConnections / 4),
            idleTimeoutMillis: 30000,
            connectionTimeoutMillis: 2000,
        });

        this.setupMonitoring();
    }

    private setupMonitoring() {
        setInterval(() => {
            this.metrics = {
                activeConnections: this.pool.totalCount - this.pool.idleCount,
                waitingConnections: this.pool.waitingCount,
                idleConnections: this.pool.idleCount,
                totalQueries: this.metrics?.totalQueries || 0,
                avgQueryTime: this.metrics?.avgQueryTime || 0
            };

            // Alert if pool is saturated
            if (this.metrics.waitingConnections > 5) {
                console.warn('Database pool saturation detected', this.metrics);
            }
        }, 10000);
    }

    async query<T>(sql: string, params?: any[]): Promise<T> {
        const startTime = Date.now();
        const client = await this.pool.connect();
        
        try {
            const result = await client.query(sql, params);
            
            // Update metrics
            this.metrics.totalQueries++;
            const queryTime = Date.now() - startTime;
            this.metrics.avgQueryTime = 
                (this.metrics.avgQueryTime * (this.metrics.totalQueries - 1) + queryTime) 
                / this.metrics.totalQueries;
            
            return result as T;
        } finally {
            client.release();
        }
    }

    getMetrics() {
        return { ...this.metrics };
    }
}
```

### Real-World Debugging Scenarios

#### Scenario 1: Memory Leak Detection

```typescript
// Memory leak detector
class MemoryLeakDetector {
    private heapSnapshots: any[] = [];
    
    takeSnapshot(): void {
        if (global.gc) {
            global.gc(); // Force garbage collection
        }
        const used = process.memoryUsage();
        this.heapSnapshots.push({
            timestamp: new Date(),
            heapUsed: used.heapUsed,
            heapTotal: used.heapTotal,
            external: used.external,
            rss: used.rss
        });
    }

    analyzeLeaks(): void {
        if (this.heapSnapshots.length < 2) return;

        const first = this.heapSnapshots[0];
        const last = this.heapSnapshots[this.heapSnapshots.length - 1];
        
        const heapGrowth = last.heapUsed - first.heapUsed;
        const timeElapsed = last.timestamp - first.timestamp;
        const growthRate = heapGrowth / (timeElapsed / 1000); // bytes per second

        console.log('Memory Analysis:', {
            initialHeap: `${(first.heapUsed / 1024 / 1024).toFixed(2)} MB`,
            currentHeap: `${(last.heapUsed / 1024 / 1024).toFixed(2)} MB`,
            growth: `${(heapGrowth / 1024 / 1024).toFixed(2)} MB`,
            growthRate: `${(growthRate / 1024).toFixed(2)} KB/s`,
            possibleLeak: growthRate > 1024 // More than 1KB/s growth
        });
    }
}

// Usage in production
const leakDetector = new MemoryLeakDetector();
setInterval(() => {
    leakDetector.takeSnapshot();
    leakDetector.analyzeLeaks();
}, 60000); // Every minute
```

#### Scenario 2: Transaction Deadlock Detection

```typescript
class DeadlockDetector {
    async detectAndResolve() {
        try {
            // Query to detect deadlocks in PostgreSQL
            const deadlocks = await this.db.query(`
                SELECT
                    blocked_locks.pid AS blocked_pid,
                    blocked_activity.usename AS blocked_user,
                    blocking_locks.pid AS blocking_pid,
                    blocking_activity.usename AS blocking_user,
                    blocked_activity.query AS blocked_statement,
                    blocking_activity.query AS blocking_statement
                FROM pg_catalog.pg_locks blocked_locks
                JOIN pg_catalog.pg_stat_activity blocked_activity 
                    ON blocked_activity.pid = blocked_locks.pid
                JOIN pg_catalog.pg_locks blocking_locks 
                    ON blocking_locks.locktype = blocked_locks.locktype
                    AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
                    AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
                    AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
                    AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
                    AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
                    AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
                    AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
                    AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
                    AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
                    AND blocking_locks.pid != blocked_locks.pid
                JOIN pg_catalog.pg_stat_activity blocking_activity 
                    ON blocking_activity.pid = blocking_locks.pid
                WHERE NOT blocked_locks.granted;
            `);

            if (deadlocks.length > 0) {
                console.error('Deadlocks detected:', deadlocks);
                
                // Log for analysis
                await this.logDeadlock(deadlocks);
                
                // Optionally terminate long-running blocking queries
                for (const deadlock of deadlocks) {
                    const blockingDuration = await this.getQueryDuration(deadlock.blocking_pid);
                    if (blockingDuration > 30000) { // 30 seconds
                        await this.db.query(`SELECT pg_terminate_backend($1)`, [deadlock.blocking_pid]);
                        console.log(`Terminated blocking query ${deadlock.blocking_pid}`);
                    }
                }
            }
        } catch (error) {
            console.error('Error detecting deadlocks:', error);
        }
    }
}
```

### Advanced Architecture Patterns

#### Microkernel Architecture Pattern

```typescript
// Plugin system for extensible architecture
interface IPlugin {
    name: string;
    version: string;
    initialize(context: PluginContext): Promise<void>;
    execute(input: any): Promise<any>;
    shutdown(): Promise<void>;
}

class PluginManager {
    private plugins: Map<string, IPlugin> = new Map();
    private context: PluginContext;

    async registerPlugin(plugin: IPlugin): Promise<void> {
        await plugin.initialize(this.context);
        this.plugins.set(plugin.name, plugin);
        console.log(`Plugin registered: ${plugin.name} v${plugin.version}`);
    }

    async executePlugin(pluginName: string, input: any): Promise<any> {
        const plugin = this.plugins.get(pluginName);
        if (!plugin) {
            throw new Error(`Plugin not found: ${pluginName}`);
        }
        return await plugin.execute(input);
    }

    async shutdownAll(): Promise<void> {
        for (const [name, plugin] of this.plugins) {
            await plugin.shutdown();
            console.log(`Plugin shutdown: ${name}`);
        }
    }
}

// Example plugin implementation
class ValidationPlugin implements IPlugin {
    name = 'validation-plugin';
    version = '1.0.0';
    private validators: Map<string, Function> = new Map();

    async initialize(context: PluginContext): Promise<void> {
        // Load validators
        this.validators.set('email', this.validateEmail);
        this.validators.set('phone', this.validatePhone);
    }

    async execute(input: { type: string; value: any }): Promise<any> {
        const validator = this.validators.get(input.type);
        if (!validator) {
            throw new Error(`Validator not found: ${input.type}`);
        }
        return validator(input.value);
    }

    async shutdown(): Promise<void> {
        this.validators.clear();
    }

    private validateEmail(email: string): boolean {
        return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
    }

    private validatePhone(phone: string): boolean {
        return /^\d{10}$/.test(phone);
    }
}
```

#### Feature Flag System

```typescript
class FeatureFlagManager {
    private flags: Map<string, boolean> = new Map();
    private remoteConfig: IRemoteConfig;

    async initialize(): Promise<void> {
        // Load flags from remote config
        const remoteFlags = await this.remoteConfig.getFlags();
        Object.entries(remoteFlags).forEach(([key, value]) => {
            this.flags.set(key, value);
        });

        // Set up real-time updates
        this.remoteConfig.onFlagChange((key, value) => {
            this.flags.set(key, value);
            console.log(`Feature flag updated: ${key} = ${value}`);
        });
    }

    isEnabled(flagName: string, context?: any): boolean {
        // Check user-specific overrides
        if (context?.userId) {
            const userOverride = this.flags.get(`${flagName}:${context.userId}`);
            if (userOverride !== undefined) return userOverride;
        }

        // Check tenant-specific overrides
        if (context?.tenantId) {
            const tenantOverride = this.flags.get(`${flagName}:tenant:${context.tenantId}`);
            if (tenantOverride !== undefined) return tenantOverride;
        }

        // Default flag value
        return this.flags.get(flagName) || false;
    }

    async setFlag(flagName: string, value: boolean): Promise<void> {
        this.flags.set(flagName, value);
        await this.remoteConfig.setFlag(flagName, value);
    }
}

// Usage in business logic
class ProductService {
    async createProduct(data: CreateProductData, context: any): Promise<Product> {
        // New feature: AI-powered product description
        if (this.featureFlags.isEnabled('ai-descriptions', context)) {
            data.description = await this.aiService.generateDescription(data);
        }

        // New feature: Blockchain verification
        if (this.featureFlags.isEnabled('blockchain-verification', context)) {
            data.verificationHash = await this.blockchainService.store(data);
        }

        return await this.repository.create(data);
    }
}
```

This substantial addition provides advanced testing strategies, performance optimization patterns, real-world debugging scenarios, and sophisticated architecture patterns, making the Architecture Understanding chapter significantly more comprehensive and practical for developers.

This Architecture Understanding chapter provides deep insights into the design philosophy, patterns, and decision-making process behind the BYO DPP system. It helps developers understand not just "how" things work, but "why" they were designed that way.


## Domain Driven Design Alignment

### Why Domain-Driven Design? Problems Solved

**Traditional Development Problems**:
- **Ubiquitous Language Missing**: Developers and domain experts use different terms
- **Anemic Domain Models**: Business logic scattered across layers
- **Boundless Contexts**: Large models that try to do everything
- **Technical Over Business Focus**: Implementation details drive design decisions

**DDD Solutions**:
- **Ubiquitous Language**: Shared vocabulary between technical and business teams
- **Rich Domain Models**: Business logic encapsulated in domain objects
- **Bounded Contexts**: Clear boundaries around domain areas
- **Business-Centric Design**: Business needs drive technical decisions

### Bounded Contexts & Their Responsibilities

The system is organized around the following bounded contexts, each with clear responsibilities and boundaries:

#### 1. Product Management Context
**Purpose**: DPP creation, configuration, and lifecycle management
**Problems Solved**:
- **Product Complexity**: Manages complex product hierarchies and relationships
- **Configuration Management**: Handles product-specific settings and validations
- **Lifecycle Tracking**: Manages product states and transitions

**Entities**: `Product`, `Category`, `ProductData`, `ProductDataSource`
**Aggregates**: `ProductAggregate` (Product + associated data)

#### 2. User Management Context
**Purpose**: Authentication, authorization, and user lifecycle
**Problems Solved**:
- **Identity Management**: Centralized user identity and profile management
- **Access Control**: Role-based and permission-based security
- **Multi-Tenancy**: User isolation across different organizations

**Entities**: `User`, `Role`, `UserRole`, `Tenant`, `TenantUser`
**Aggregates**: `UserAggregate` (User + roles + tenant associations)

#### 3. Data Processing Context
**Purpose**: DPP execution, monitoring, and pipeline management
**Problems Solved**:
- **Pipeline Orchestration**: Coordinates complex data processing workflows
- **State Management**: Tracks processing status and progress
- **Error Recovery**: Handles failures and retries in processing pipelines

**Entities**: `Dpp`, `DppSection`, `DppData`, `AuditLog`
**Aggregates**: `DppAggregate` (DPP + sections + data)

#### 4. Template Management Context
**Purpose**: DPP template definition and validation rules
**Problems Solved**:
- **Template Consistency**: Ensures templates follow business rules
- **Field Validation**: Manages complex validation logic for template fields
- **Version Control**: Handles template evolution and compatibility

**Entities**: `Template`, `TemplateSection`, `TemplateField`, `Validation`
**Aggregates**: `TemplateAggregate` (Template + sections + fields)

#### 5. Export/Import Context
**Purpose**: Data exchange between systems and formats
**Problems Solved**:
- **Format Translation**: Converts between different data formats (CSV, JSON, XML)
- **Bulk Operations**: Handles large-scale data import/export
- **Data Integrity**: Ensures data consistency during transfers

**Entities**: `ExportImporter`, `File`, `BackupStorage`, `BackupStorageLog`
**Aggregates**: `ExportAggregate` (Export job + files + logs)

### Domain Entities & Their Behaviors

#### Entity Pattern: Identity + Behavior
```typescript
// Problem: Anemic entities (just data containers)
class AnemicProduct {
    id: number;
    name: string;
    isEnabled: boolean;
    // No behavior, just data
}

// Solution: Rich domain entities with behavior
export class Product {
    constructor(
        public id: number,
        public name: string,
        private isEnabled: boolean = true
    ) {}

    // Domain behavior encapsulated in entity
    enable(): void {
        this.isEnabled = true;
        this.updatedAt = new Date();
        // Could trigger domain events
    }

    disable(): void {
        this.isEnabled = false;
        this.updatedAt = new Date();
        // Could trigger domain events
    }

    updateName(newName: string): void {
        if (!newName || newName.trim().length === 0) {
            throw new Error('Product name cannot be empty');
        }
        this.name = newName.trim();
        this.updatedAt = new Date();
    }

    canBeDeleted(): boolean {
        // Business rule: can't delete if referenced elsewhere
        return !this.hasActiveReferences();
    }
}
```
**Benefits**:
- **Encapsulation**: Business rules kept with data
- **Consistency**: Entity ensures its own invariants
- **Testability**: Entity behavior can be tested in isolation

#### Value Object Pattern: Immutable Values
```typescript
// Problem: Primitive obsession leads to bugs
function processOrder(productId: number, quantity: number, price: number) {
    // Easy to pass wrong parameter order
}

// Solution: Value objects with behavior
export class Money {
    constructor(
        private readonly amount: number,
        private readonly currency: string
    ) {}

    add(other: Money): Money {
        if (this.currency !== other.currency) {
            throw new Error('Cannot add different currencies');
        }
        return new Money(this.amount + other.amount, this.currency);
    }

    multiply(factor: number): Money {
        return new Money(this.amount * factor, this.currency);
    }

    toString(): string {
        return `${this.currency} ${this.amount.toFixed(2)}`;
    }
}

export class Email {
    constructor(private readonly value: string) {
        if (!this.isValid(value)) {
            throw new Error('Invalid email format');
        }
    }

    private isValid(email: string): boolean {
        const regex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
        return regex.test(email);
    }

    toString(): string {
        return this.value;
    }
}
```
**Benefits**:
- **Immutability**: Prevents accidental changes
- **Validation**: Ensures value correctness at creation
- **Behavior**: Values can have domain-specific operations

### Aggregate Pattern: Consistency Boundaries

#### Aggregate Design Principles
```typescript
// Problem: Inconsistent object graphs
class Order {
    constructor(
        public id: number,
        public items: OrderItem[], // Could be modified externally
        public customer: Customer   // Could be modified externally
    ) {}
}

// Solution: Aggregate with root entity controlling access
export class OrderAggregate {
    private order: Order;
    private items: OrderItem[];

    constructor(order: Order, items: OrderItem[]) {
        this.order = order;
        this.items = [...items]; // Defensive copy
    }

    // Aggregate root controls all access
    addItem(productId: number, quantity: number, price: Money): void {
        // Business rule: check inventory
        // Business rule: validate price
        // Business rule: check order limits

        const item = new OrderItem(productId, quantity, price);
        this.items.push(item);

        // Update order total
        this.order.total = this.calculateTotal();
    }

    removeItem(itemId: number): void {
        const index = this.items.findIndex(item => item.id === itemId);
        if (index === -1) {
            throw new Error('Item not found in order');
        }

        this.items.splice(index, 1);
        this.order.total = this.calculateTotal();
    }

    // Only aggregate root can modify internal state
    private calculateTotal(): Money {
        return this.items.reduce(
            (total, item) => total.add(item.getSubtotal()),
            new Money(0, 'USD')
        );
    }
}
```
**Benefits**:
- **Consistency**: Aggregate root ensures invariants
- **Encapsulation**: Internal structure hidden from outside
- **Transaction Boundaries**: Aggregates define consistency scope

### Domain Services: Cross-Aggregate Operations

#### Domain Service Pattern
```typescript
// Problem: Business logic doesn't fit in single entity
class Product {
    transferToCategory(newCategoryId: number): void {
        // Can't check if category exists - that's another aggregate
        // Can't check business rules across products
    }
}

// Solution: Domain service for cross-entity operations
export class ProductTransferService {
    constructor(
        private productRepo: IProductRepository,
        private categoryRepo: ICategoryRepository
    ) {}

    async transferProduct(productId: number, newCategoryId: number): Promise<void> {
        // Load aggregates
        const product = await this.productRepo.findById(productId);
        const newCategory = await this.categoryRepo.findById(newCategoryId);

        // Business rules across aggregates
        if (!newCategory.canAcceptProducts()) {
            throw new Error('Category cannot accept new products');
        }

        if (product.getCategoryId() === newCategoryId) {
            throw new Error('Product already in target category');
        }

        // Check category capacity
        const categoryProductCount = await this.productRepo.countByCategory(newCategoryId);
        if (categoryProductCount >= newCategory.getMaxProducts()) {
            throw new Error('Category is at maximum capacity');
        }

        // Perform transfer
        product.changeCategory(newCategoryId);
        await this.productRepo.save(product);

        // Domain event (could trigger notifications, audit logs, etc.)
        this.publishEvent(new ProductTransferredEvent(productId, newCategoryId));
    }
}
```
**Benefits**:
- **Cross-Aggregate Logic**: Operations spanning multiple aggregates
- **Business Rule Centralization**: Complex rules in dedicated services
- **Testability**: Services can be tested with mock repositories

### Domain Events: Decoupling Business Logic

#### Domain Event Pattern
```typescript
// Problem: Tight coupling between business operations
export class Product {
    changePrice(newPrice: Money): void {
        this.price = newPrice;
        // Directly coupled to notification logic
        this.sendPriceChangeNotification();
        this.updateSearchIndex();
        this.logAuditTrail();
    }
}

// Solution: Domain events for loose coupling
export class Product {
    private domainEvents: DomainEvent[] = [];

    changePrice(newPrice: Money): void {
        if (newPrice.lessThan(new Money(0, this.price.currency))) {
            throw new Error('Price cannot be negative');
        }

        const oldPrice = this.price;
        this.price = newPrice;
        this.updatedAt = new Date();

        // Raise domain event instead of direct coupling
        this.addDomainEvent(new ProductPriceChangedEvent(
            this.id, oldPrice, newPrice, new Date()
        ));
    }

    private addDomainEvent(event: DomainEvent): void {
        this.domainEvents.push(event);
    }

    getDomainEvents(): DomainEvent[] {
        return [...this.domainEvents];
    }

    clearDomainEvents(): void {
        this.domainEvents = [];
    }
}

// Event handlers registered separately
export class ProductPriceChangedHandler {
    constructor(
        private notificationService: INotificationService,
        private searchService: ISearchService,
        private auditService: IAuditService
    ) {}

    async handle(event: ProductPriceChangedEvent): Promise<void> {
        // Each concern handled separately
        await Promise.all([
            this.notificationService.sendPriceChangeAlert(event),
            this.searchService.updateProductPrice(event.productId, event.newPrice),
            this.auditService.logPriceChange(event)
        ]);
    }
}
```
**Benefits**:
- **Loose Coupling**: Business logic doesn't know about side effects
- **Separation of Concerns**: Each handler focuses on one responsibility
- **Testability**: Events and handlers can be tested independently
- **Flexibility**: New handlers can be added without changing business logic

## Module Lifecycle Management

### Application Bootstrap Process

The BYO DPP application follows a structured, layered bootstrap process ensuring proper initialization order:

#### Phase 1: Configuration & Environment Setup
```typescript
setupConfig(): void {
    this.config = AppConfig.getInstance().getConfig();
}

setupBootstrapJson(): void {
    this.byoDPPJson = JSON.parse(fs.readFileSync(BYO_DPP_DATA_FILE_PATH, 'utf8'));
}
```

#### Phase 2: Dependency Injection Container
```typescript
setupDependencyFactory(): void {
    const factoryConfig = {
        database: { url: this.config.POSTGRES_DB_URI! },
        server: { port: Number(this.config.PORT), host: getLocalIP() },
        swaggerData: this.byoDPPJson,
        // ... other configurations
    };
    this.dependencyFactory = new DependencyFactory(factoryConfig);
    this.dependencyFactory.createCoreDependencies();
}
```

#### Phase 3: Provider Initialization
```typescript
setupCoreProvider(): void {
    CoreProvider.initialize(new CoreDependencyFactoryProxy(this.dependencyFactory));
}

setupApplicationProvider(): void {
    ApplicationProvider.initialize(new ApplicationDependencyFactoryProxy(this.dependencyFactory));
}
```

#### Phase 4: Dynamic Route Registration
```typescript
buildRoutes(): void {
    this.byoDPPJson.forEach((service: Service) => {
        service.resources.forEach((resource: Resource) => {
            // Dynamic repository, service, and controller creation
            const resourceBaseRepoAdapter = this.createRepository(resource);
            const resourceBaseService = this.createService(resource, resourceBaseRepoAdapter);
            const resourceBaseController = this.createController(resource, resourceBaseService);

            // Register CRUD and action routes
            this.registerCrudRoutes(resource, resourceBaseController);
            this.registerActionRoutes(resource, resourceBaseController);
        });
    });
}
```

### Provider Pattern Implementation

#### Core Provider (Singleton)
```typescript
export class CoreProvider implements ICoreDependencyFactory {
    private static instance: CoreProvider;

    static initialize(coreDependency: ICoreDependencyFactory): void {
        if (CoreProvider.instance) {
            throw new Error('CoreProvider is already initialized.');
        }
        CoreProvider.instance = new CoreProvider(coreDependency);
    }

    static getInstance(): CoreProvider {
        if (!CoreProvider.instance) {
            throw new Error('CoreProvider is not initialized. Call initialize() first.');
        }
        return CoreProvider.instance;
    }
}
```

#### Application Provider
```typescript
export class ApplicationProvider implements IApplicationDependencyFactory {
    private static instance: ApplicationProvider;

    // Similar singleton pattern with application-specific dependencies
    getRouter(): IRouter;
    getGraphAPI(): IGraphAPI;
    getEventStream(): IEventStream;
}
```

### Dependency Factory Pattern

The system uses a factory pattern for complex dependency creation:

```typescript
export class DependencyFactory {
    createCoreDependencies(): void {
        // Create database connection
        // Initialize repositories
        // Setup external services (Keycloak, Email, etc.)
        // Configure event streaming
    }
}
```

### Module Loading Strategy

#### Dynamic Resource Registration
Resources are registered dynamically from JSON configuration:

```typescript
// From bootstrap.ts
this.byoDPPJson.forEach((service: Service) => {
    service.resources.forEach((resource: Resource) => {
        // 1. Create repository based on resource type
        if (resource.type === 'database') {
            const resourceORMRepository = new DrizzleORMRepository(
                database, resource.name, ResourceModelMap[resource.name]
            );
            resourceBaseRepoAdapter = new BaseRepoAdapter(resourceORMRepository);
        }

        // 2. Register with repository discovery
        repoDiscovery.registerRepository(resource.name, resourceBaseRepoAdapter);

        // 3. Create service with logic mapping
        const resourceBaseService = new BaseService(
            resourceBaseRepoAdapter, resource.name,
            logicMap['v1'][resource.name]?.action
        );

        // 4. Create controller
        const resourceBaseController = new BaseController(
            resourceBaseService, graphQuery, resource.name
        );
    });
});
```

#### Logic Mapping System
Business logic is mapped dynamically:

```typescript
// Logic map structure
const logicMap = {
    'v1': {
        'dpps': {
            'action': {
                'create:post': DppsCreateLogic,
                'update:put': DppsUpdateLogic
            },
            'crud': {
                'create': DppsCreateLogic,
                'list': DppsListLogic
            }
        }
    }
};
```

### Lifecycle Hooks

The system supports lifecycle hooks for modules:

```typescript
interface IModuleLifecycle {
    onInit?(): Promise<void>;
    onDestroy?(): Promise<void>;
    onHealthCheck?(): Promise<HealthStatus>;
}
```

### Error Handling in Bootstrap

Bootstrap process includes comprehensive error handling:

```typescript
run(): void {
    try {
        this.logger.info('🚀 Starting BYO DPP Application');
        this.setup();
        this.build();
        this.boot();
        this.logger.info('✨ Application is ready');
    } catch (error) {
        this.logger.error('Failed to start application', error as Error);
        process.exit(1);
    }
}
```

## Event Orchestration and Pipeline Patterns

### Why Event-Driven Architecture? Problems Solved

**Traditional Request-Response Problems**:
- **Tight Coupling**: Components directly call each other, creating rigid dependencies
- **Synchronous Processing**: Slow operations block user interactions and system responsiveness
- **Scalability Issues**: Monolithic processing patterns can't scale horizontally across services
- **Error Propagation**: Single component failure brings down entire request flows
- **State Management**: Complex state synchronization required across distributed components

**Event-Driven Solutions in BYO DPP**:
- **Loose Coupling**: Components communicate through events, enabling independent evolution
- **Asynchronous Processing**: Non-blocking operations improve user experience and system throughput
- **Horizontal Scaling**: Event processing can scale independently based on load patterns
- **Fault Tolerance**: Failed components don't break the entire system (circuit breaker patterns)
- **State Synchronization**: Events provide eventual consistency with guaranteed delivery

#### Event-Driven Patterns in BYO DPP

**Observer Pattern for Real-Time Updates**:
```typescript
// Subject (Event Source)
export class DppService {
    constructor(private eventPublisher: IEventPublisher) {}

    async createDpp(input: CreateDppInput): Promise<Dpp> {
        const dpp = await this.repository.create(input);

        // Notify observers asynchronously
        await this.eventPublisher.publish(new DppCreatedEvent(dpp));

        return dpp;
    }
}

// Observer (Event Handler)
export class DppCreatedHandler {
    constructor(
        private searchService: ISearchService,
        private notificationService: INotificationService
    ) {}

    async handle(event: DppCreatedEvent): Promise<void> {
        // Update search index
        await this.searchService.indexDpp(event.dpp);

        // Send notifications
        await this.notificationService.notifyStakeholders(event.dpp);
    }
}
```
**Benefits**:
- **Decoupling**: Business logic doesn't know about side effects
- **Extensibility**: New handlers can be added without changing core logic
- **Testability**: Events and handlers tested independently

**Event Sourcing for Audit Trails**:
```typescript
// Instead of updating state directly
export class Product {
    changePrice(newPrice: Money): void {
        // Store event, not just state
        this.events.push(new PriceChangedEvent(
            this.id, this.price, newPrice, new Date()
        ));
        this.price = newPrice;
    }
}

// Rebuild state from events
export class ProductEventSourced {
    static fromEvents(events: DomainEvent[]): Product {
        let product: Product | null = null;

        for (const event of events) {
            if (event instanceof ProductCreatedEvent) {
                product = new Product(event.id, event.name, event.price);
            } else if (event instanceof PriceChangedEvent) {
                product!.changePrice(event.newPrice);
            }
        }

        return product!;
    }
}
```
**Benefits**:
- **Complete Audit Trail**: Every state change is recorded
- **Temporal Queries**: Can query state at any point in time
- **Event Replay**: Can rebuild state from scratch

**Saga Pattern for Distributed Transactions**:
```typescript
// Problem: Distributed transactions are hard
export class OrderSaga {
    constructor(
        private orderService: IOrderService,
        private inventoryService: IInventoryService,
        private paymentService: IPaymentService
    ) {}

    async processOrder(orderData: OrderData): Promise<void> {
        // Traditional approach - hard to coordinate
        await this.inventoryService.reserveItems(orderData.items);
        await this.paymentService.charge(orderData.payment);
        await this.orderService.create(orderData);
    }
}

// Solution: Event-driven saga
export class OrderSagaOrchestrator {
    private steps: SagaStep[] = [
        { action: 'reserve_inventory', compensation: 'release_inventory' },
        { action: 'charge_payment', compensation: 'refund_payment' },
        { action: 'create_order', compensation: 'cancel_order' }
    ];

    async execute(orderData: OrderData): Promise<void> {
        const sagaId = generateSagaId();
        let completedSteps = 0;

        try {
            for (const step of this.steps) {
                await this.executeStep(step.action, orderData, sagaId);
                completedSteps++;
            }
        } catch (error) {
            // Compensate completed steps in reverse order
            for (let i = completedSteps - 1; i >= 0; i--) {
                await this.compensateStep(this.steps[i].compensation, orderData, sagaId);
            }
            throw error;
        }
    }
}
```
**Benefits**:
- **Consistency**: Either all operations succeed or all are compensated
- **Resilience**: Partial failures can be recovered
- **Observability**: Each step can be monitored and logged

#### CQRS Pattern for Read/Write Optimization

**Command Side (Write Model)**:
```typescript
// Focused on business logic and validation
export class CreateProductCommand {
    constructor(
        public readonly name: string,
        public readonly price: Money,
        public readonly categoryId: string
    ) {}
}

export class ProductCommandHandler {
    constructor(private repository: IProductRepository) {}

    async handle(command: CreateProductCommand): Promise<string> {
        // Business validation
        await this.validateProduct(command);

        // Create aggregate
        const product = Product.create(command.name, command.price);

        // Persist
        await this.repository.save(product);

        // Publish event
        await this.eventPublisher.publish(new ProductCreatedEvent(product.id));

        return product.id;
    }
}
```

**Query Side (Read Model)**:
```typescript
// Optimized for reading, possibly denormalized
export class ProductReadModel {
    constructor(
        public readonly id: string,
        public readonly name: string,
        public readonly price: string,
        public readonly categoryName: string,
        public readonly createdAt: Date
    ) {}
}

export class ProductQueryHandler {
    constructor(private readModelStore: IReadModelStore) {}

    async getProduct(id: string): Promise<ProductReadModel> {
        return await this.readModelStore.getProduct(id);
    }

    async getProductsByCategory(categoryId: string): Promise<ProductReadModel[]> {
        return await this.readModelStore.getProductsByCategory(categoryId);
    }
}
```

**Event Handler Updates Read Model**:
```typescript
export class ProductReadModelUpdater {
    async handle(event: ProductCreatedEvent): Promise<void> {
        await this.readModelStore.insert({
            id: event.productId,
            name: event.name,
            price: event.price.toString(),
            categoryName: await this.getCategoryName(event.categoryId),
            createdAt: event.timestamp
        });
    }
}
```
**Benefits**:
- **Performance**: Read and write models can be optimized separately
- **Scalability**: Read models can be scaled independently
- **Flexibility**: Different data structures for different use cases
- **Consistency**: Eventual consistency between models

#### Event Storming for System Design

BYO DPP uses Event Storming methodology for system design:

**Domain Events Identified**:
- `ProductCreated`
- `ProductPriceChanged`
- `DppProcessingStarted`
- `DppProcessingCompleted`
- `UserOnboarded`
- `TenantQuotaExceeded`

**Commands Identified**:
- `CreateProduct`
- `ChangeProductPrice`
- `StartDppProcessing`
- `OnboardUser`
- `CreateTenant`

**Aggregates Identified**:
- `Product` (handles product lifecycle)
- `Dpp` (handles DPP processing)
- `User` (handles user management)
- `Tenant` (handles multi-tenancy)

**Read Models Identified**:
- `ProductListView`
- `DppStatusDashboard`
- `UserProfileView`
- `TenantUsageReport`

This event-driven approach provides:
- **Scalability**: Components can scale independently
- **Reliability**: System continues functioning despite partial failures
- **Maintainability**: Loose coupling enables easier changes
- **Observability**: Events provide comprehensive audit trails
- **Flexibility**: New features can be added by subscribing to existing events

### Event Streaming: Observer Pattern Implementation

The system uses Server-Sent Events (SSE) for real-time communication, implementing the Observer pattern:

#### SSE Controller: Subject in Observer Pattern
```typescript
// SSE Controller - Manages event subscriptions
export class SSEController {
    constructor(
        private eventStream: IEventStream,
        private resourceName: string
    ) {}

    async handleConnection(request: IRequest, response: IResponse) {
        // Set up SSE headers for persistent connection
        response.writeHead(200, {
            'Content-Type': 'text/event-stream',
            'Cache-Control': 'no-cache',
            'Connection': 'keep-alive',
            'Access-Control-Allow-Origin': '*'
        });

        // Register client as observer
        const connectionId = `conn_${Date.now()}_${Math.random()}`;
        this.eventStream.addConnection(connectionId, response);

        // Send initial connection event
        this.eventStream.initConnection(connectionId);

        // Handle client disconnect (unsubscribe)
        request.on('close', () => {
            this.eventStream.closeConnection(connectionId);
        });
    }
}
```
**Observer Pattern Benefits**:
- **Decoupling**: Publishers don't know about subscribers
- **Dynamic Subscriptions**: Clients can subscribe/unsubscribe at runtime
- **Broadcast Capability**: Single event can reach multiple observers
- **Memory Efficiency**: Events processed only by interested parties

#### SSE Adapter: Concrete Subject Implementation
```typescript
// SSE Event Stream - Manages event distribution
export class SseAdapter implements IEventStream {
    private connections: Map<string, Response> = new Map();

    addConnection(connectionId: string, res: Response): void {
        this.setupSseHeaders(res);
        this.connections.set(connectionId, res);
        this.sendHeartbeat(connectionId); // Keep connection alive
    }

    // Broadcast to all observers
    sendEvent(eventName: string, data: any, connectionId?: string): void {
        if (connectionId) {
            // Unicast to specific observer
            this.sendEventToClient(connectionId, eventName, data);
        } else {
            // Broadcast to all observers
            this.connections.forEach((res, id) => {
                this.sendEventToClient(id, eventName, data);
            });
        }
    }

    private sendEventToClient(connectionId: string, eventName: string, data: any): void {
        const res = this.connections.get(connectionId);
        if (res) {
            const eventData = {
                event: eventName,
                data: data,
                timestamp: Date.now(),
                id: connectionId
            };
            res.write(`data: ${JSON.stringify(eventData)}\n\n`);
        }
    }

    private sendHeartbeat(connectionId: string): void {
        setInterval(() => {
            this.sendEventToClient(connectionId, 'heartbeat', { timestamp: Date.now() });
        }, 30000); // Prevent timeout every 30 seconds
    }
}
```
**Problems Solved**:
- **Real-time Updates**: Instant notifications without polling
- **Connection Management**: Automatic cleanup of disconnected clients
- **Heartbeat Mechanism**: Prevents proxy/server timeouts
- **Scalability**: Can handle thousands of concurrent connections

### Pipeline Execution: Chain of Responsibility Pattern

Data processing pipelines follow a Chain of Responsibility pattern with clear stages:

#### Pipeline Stage Interface
```typescript
interface IPipelineStage<TInput, TOutput> {
    name: string;
    execute(input: TInput, context: PipelineContext): Promise<TOutput>;
    canExecute?(input: TInput, context: PipelineContext): boolean;
}

interface PipelineContext {
    correlationId: string;
    userId?: string;
    tenantId?: string;
    metadata: Record<string, any>;
    startTime: Date;
}
```

#### DPP Processing Pipeline Implementation
```typescript
export class DppProcessingPipeline {
    private stages: IPipelineStage<any, any>[] = [
        new ValidationStage(),
        new EnrichmentStage(),
        new TransformationStage(),
        new BusinessRuleStage(),
        new PersistenceStage(),
        new NotificationStage()
    ];

    async execute(input: DppInput): Promise<DppOutput> {
        const context: PipelineContext = {
            correlationId: generateCorrelationId(),
            userId: input.userId,
            tenantId: input.tenantId,
            metadata: {},
            startTime: new Date()
        };

        let currentInput = input;

        for (const stage of this.stages) {
            try {
                // Pre-execution checks
                if (stage.canExecute && !stage.canExecute(currentInput, context)) {
                    console.log(`Skipping stage ${stage.name}`);
                    continue;
                }

                console.log(`Executing stage: ${stage.name}`);
                const startTime = Date.now();

                // Execute stage
                currentInput = await stage.execute(currentInput, context);

                const duration = Date.now() - startTime;
                console.log(`Stage ${stage.name} completed in ${duration}ms`);

                // Add stage result to context
                context.metadata[`${stage.name}_duration`] = duration;

            } catch (error) {
                console.error(`Stage ${stage.name} failed:`, error);

                // Stage-specific error handling
                await this.handleStageError(stage, error, context);

                // Decide whether to continue or fail pipeline
                if (this.shouldFailPipeline(stage, error)) {
                    throw new PipelineExecutionError(
                        `Pipeline failed at stage ${stage.name}`,
                        stage.name,
                        error
                    );
                }
            }
        }

        return currentInput as DppOutput;
    }
}
```
**Chain of Responsibility Benefits**:
- **Modular Processing**: Each stage has single responsibility
- **Error Isolation**: Stage failures don't necessarily stop the pipeline
- **Monitoring**: Each stage can be monitored independently
- **Testing**: Individual stages can be tested in isolation

#### Pipeline Stages Implementation

##### 1. Validation Stage: Guardian Pattern
```typescript
export class ValidationStage implements IPipelineStage<DppInput, ValidatedDppInput> {
    name = 'validation';

    async execute(input: DppInput, context: PipelineContext): Promise<ValidatedDppInput> {
        // Schema validation
        const { error, value } = dppInputSchema.validate(input);
        if (error) {
            throw new ValidationError(`Input validation failed: ${error.details[0].message}`);
        }

        // Business rule validation
        await this.validateBusinessRules(value, context);

        return { ...value, validatedAt: new Date() };
    }

    private async validateBusinessRules(input: DppInput, context: PipelineContext): Promise<void> {
        // Check tenant permissions
        if (context.tenantId) {
            const hasPermission = await this.checkTenantPermission(context.tenantId, 'create_dpp');
            if (!hasPermission) {
                throw new ForbiddenError('Tenant does not have permission to create DPPs');
            }
        }

        // Check DPP name uniqueness
        const existingDpp = await this.dppRepository.findByName(input.name);
        if (existingDpp) {
            throw new ConflictError(`DPP with name "${input.name}" already exists`);
        }
    }
}
```
**Guardian Pattern Benefits**:
- **Input Sanitization**: Prevents invalid data from entering the system
- **Early Failure**: Catches problems before expensive processing
- **Security**: Validates permissions and business rules upfront

##### 2. Enrichment Stage: Decorator Pattern
```typescript
export class EnrichmentStage implements IPipelineStage<ValidatedDppInput, EnrichedDppInput> {
    name = 'enrichment';

    async execute(input: ValidatedDppInput, context: PipelineContext): Promise<EnrichedDppInput> {
        // Add metadata
        const enriched = {
            ...input,
            metadata: {
                ...input.metadata,
                createdBy: context.userId,
                tenantId: context.tenantId,
                correlationId: context.correlationId,
                createdAt: new Date()
            }
        };

        // Enrich with related data
        enriched.category = await this.categoryRepository.findById(input.categoryId);
        enriched.template = await this.templateRepository.findById(input.templateId);

        // Add computed fields
        enriched.computedFields = await this.computeFields(input, context);

        return enriched;
    }

    private async computeFields(input: ValidatedDppInput, context: PipelineContext) {
        // Generate DPP UUID
        const uuid = await this.uuidGenerator.generate();

        // Calculate estimated processing time
        const processingTime = await this.estimateProcessingTime(input);

        return {
            uuid,
            estimatedProcessingTime: processingTime,
            priority: this.calculatePriority(input)
        };
    }
}
```
**Decorator Pattern Benefits**:
- **Data Augmentation**: Adds useful information without changing core data
- **Computed Fields**: Derives values from existing data
- **Context Awareness**: Enriches data based on execution context

##### 3. Business Rule Stage: Specification Pattern
```typescript
export class BusinessRuleStage implements IPipelineStage<EnrichedDppInput, ProcessedDppInput> {
    name = 'business_rules';

    async execute(input: EnrichedDppInput, context: PipelineContext): Promise<ProcessedDppInput> {
        const rules = [
            new DppNameFormatRule(),
            new CategoryCapacityRule(),
            new TemplateCompatibilityRule(),
            new TenantQuotaRule()
        ];

        for (const rule of rules) {
            const result = await rule.evaluate(input, context);
            if (!result.passed) {
                throw new BusinessRuleViolationError(
                    `Business rule "${rule.name}" failed: ${result.message}`,
                    rule.name,
                    result.details
                );
            }
        }

        return {
            ...input,
            businessRulesValidated: true,
            validationTimestamp: new Date()
        };
    }
}

// Specification Pattern implementation
interface IBusinessRule {
    name: string;
    evaluate(input: any, context: PipelineContext): Promise<RuleResult>;
}

export class DppNameFormatRule implements IBusinessRule {
    name = 'dpp_name_format';

    async evaluate(input: EnrichedDppInput, context: PipelineContext): Promise<RuleResult> {
        const nameRegex = /^[A-Z][A-Z0-9_]{2,49}$/;
        if (!nameRegex.test(input.name)) {
            return {
                passed: false,
                message: 'DPP name must start with uppercase letter and contain only uppercase letters, numbers, and underscores (3-50 characters)',
                details: { providedName: input.name }
            };
        }
        return { passed: true };
    }
}
```
**Specification Pattern Benefits**:
- **Declarative Rules**: Business rules expressed as specifications
- **Composability**: Rules can be combined and reused
- **Testability**: Each rule can be tested independently

##### 4. Persistence Stage: Unit of Work Pattern
```typescript
export class PersistenceStage implements IPipelineStage<ProcessedDppInput, PersistedDppInput> {
    name = 'persistence';

    async execute(input: ProcessedDppInput, context: PipelineContext): Promise<PersistedDppInput> {
        const unitOfWork = new DppUnitOfWork(this.transactionManager);

        try {
            await unitOfWork.begin();

            // Create DPP
            const dpp = await unitOfWork.dppRepository.create({
                name: input.name,
                description: input.description,
                categoryId: input.categoryId,
                templateId: input.templateId,
                metadata: input.metadata,
                uuid: input.computedFields.uuid
            });

            // Create DPP sections
            for (const section of input.sections) {
                await unitOfWork.dppSectionRepository.create({
                    dppId: dpp.id,
                    name: section.name,
                    content: section.content,
                    order: section.order
                });
            }

            // Create audit log
            await unitOfWork.auditRepository.create({
                action: 'CREATE',
                resourceType: 'DPP',
                resourceId: dpp.id,
                userId: context.userId,
                details: {
                    correlationId: context.correlationId,
                    sectionsCreated: input.sections.length
                }
            });

            await unitOfWork.commit();

            return {
                ...input,
                id: dpp.id,
                persistedAt: new Date()
            };

        } catch (error) {
            await unitOfWork.rollback();
            throw error;
        }
    }
}
```
**Unit of Work Pattern Benefits**:
- **Transactional Consistency**: All changes succeed or all fail
- **Performance**: Batch operations reduce database round trips
- **Error Recovery**: Automatic rollback on failures

##### 5. Notification Stage: Event Sourcing Pattern
```typescript
export class NotificationStage implements IPipelineStage<PersistedDppInput, FinalizedDppOutput> {
    name = 'notification';

    async execute(input: PersistedDppInput, context: PipelineContext): Promise<FinalizedDppOutput> {
        // Create domain event
        const event = new DppCreatedEvent({
            dppId: input.id,
            name: input.name,
            tenantId: context.tenantId,
            userId: context.userId,
            correlationId: context.correlationId,
            timestamp: new Date()
        });

        // Store event (Event Sourcing)
        await this.eventStore.store(event);

        // Publish to event bus (asynchronous processing)
        await this.eventBus.publish('dpp.created', event);

        // Immediate notifications (synchronous)
        await Promise.all([
            this.notifyUser(context.userId, 'dpp_created', { dppName: input.name }),
            this.notifyTenantAdmins(context.tenantId, 'new_dpp_created', {
                dppId: input.id,
                dppName: input.name,
                createdBy: context.userId
            })
        ]);

        return {
            ...input,
            eventsPublished: true,
            notificationsSent: true,
            completedAt: new Date()
        };
    }
}
```
**Event Sourcing Benefits**:
- **Audit Trail**: Complete history of all changes
- **Eventual Consistency**: Asynchronous processing of side effects
- **Replay Capability**: Can rebuild state from events
- **Decoupling**: Components react to events, not direct calls

### Event Orchestration: Mediator Pattern

The system uses a central event orchestrator to coordinate complex workflows:

```typescript
export class EventOrchestrator {
    private handlers: Map<string, IEventHandler[]> = new Map();

    registerHandler(eventType: string, handler: IEventHandler): void {
        if (!this.handlers.has(eventType)) {
            this.handlers.set(eventType, []);
        }
        this.handlers.get(eventType)!.push(handler);
    }

    async publish(event: DomainEvent): Promise<void> {
        const handlers = this.handlers.get(event.type) || [];

        // Execute all handlers concurrently
        const results = await Promise.allSettled(
            handlers.map(handler => handler.handle(event))
        );

        // Log results
        results.forEach((result, index) => {
            if (result.status === 'rejected') {
                console.error(`Handler ${index} failed for event ${event.type}:`, result.reason);
            }
        });
    }
}

// Usage
const orchestrator = new EventOrchestrator();

// Register event handlers
orchestrator.registerHandler('dpp.created', new AuditLogHandler());
orchestrator.registerHandler('dpp.created', new NotificationHandler());
orchestrator.registerHandler('dpp.created', new SearchIndexHandler());
orchestrator.registerHandler('dpp.created', new AnalyticsHandler());

// Publish event
await orchestrator.publish(dppCreatedEvent);
```
**Mediator Pattern Benefits**:
- **Centralized Communication**: Single point for event routing
- **Loose Coupling**: Publishers and subscribers don't know each other
- **Monitoring**: Easy to track event flow and performance
- **Error Isolation**: Handler failures don't affect other handlers

## Error Handling and Retry Architecture

### Comprehensive Error Hierarchy

The system implements a structured error handling system with specific error types and factory pattern:

#### Error Factory Implementation
```typescript
// From src/infra/adapters/errors/error.factory.ts
export class ErrorFactory implements IErrorFactory {
    create(errorType: string, message: string): BaseError {
        switch (errorType) {
            case 'BadRequestError':
                return new BadRequestError(message);
            case 'NotFoundError':
                return new NotFoundError(message);
            case 'UnauthorizedError':
                return new UnauthorizedError(message);
            case 'ForbiddenError':
                return new ForbiddenError(message);
            case 'ConflictError':
                return new ConflictError(message);
            case 'UnprocessableEntityError':
                return new UnprocessableEntityError(message);
            case 'TooManyRequestsError':
                return new TooManyRequestsError(message);
            case 'MethodNotAllowedError':
                return new MethodNotAllowedError(message);
            case 'BadGatewayError':
                return new BadGatewayError(message);
            case 'ServiceUnavailableError':
                return new ServiceUnavailableError(message);
            case 'InternalServerError':
                return new InternalServerError(message);
            default:
                return new InternalServerError(message);
        }
    }
}
```

#### Base Error Classes
```typescript
// From src/core/types/error.types.ts
export abstract class BaseError extends Error {
    public readonly statusCode: number;
    public readonly isOperational: boolean;

    constructor(message: string, statusCode: number, isOperational = true) {
        super(message);
        this.statusCode = statusCode;
        this.isOperational = isOperational;
        this.name = this.constructor.name;

        Error.captureStackTrace(this, this.constructor);
    }
}

export class ValidationError extends BaseError {
    constructor(message: string) {
        super(message, 400);
    }
}

export class DatabaseError extends BaseError {
    constructor(message: string) {
        super(message, 500);
    }
}
```

### Database Error Mapping

PostgreSQL errors are mapped to HTTP status codes:

```typescript
// From drizzle-orm.repository.ts
const errorCodeMap: Record<string, { statusCode: number; message: string }> = {
    '23505': { statusCode: 409, message: 'Duplicate key error' },
    '23503': { statusCode: 400, message: 'Foreign key constraint error' },
    '23502': { statusCode: 400, message: 'Not null violation' },
    '23514': { statusCode: 400, message: 'Check constraint violation' },
    '23501': { statusCode: 400, message: 'Invalid foreign key value' },
    // ... additional mappings
};

const handleDBError = (res: any) => {
    if (res?.code && errorCodeMap[res.code]) {
        return { success: false, error: { ...errorCodeMap[res.code] } };
    }
    return { success: false, error: { statusCode: 500, message: 'Internal server error' } };
};
```

### Repository Error Handling

All repository operations include comprehensive error handling:

```typescript
// From drizzle-orm.repository.ts
async read(query: any) {
    try {
        const data = await this.dbClient.query[this.modelName].findFirst(query);
        if (!data) {
            return { success: false, error: { statusCode: 404, message: 'Item not found' } };
        }
        return { success: true, data: data };
    } catch (error) {
        return handleDBError(error);
    }
}
```

### Transaction Error Handling

Multi-resource transactions include rollback mechanisms:

```typescript
async multiResourceTransact(operations: MultiResourceTransactionOperation[]): Promise<MultiResourceTransactionResult> {
    try {
        return await this.dbClient.transaction(async (tx) => {
            const results = [];
            for (const operation of operations) {
                // Execute operations with error handling
                const result = await this.executeOperation(tx, operation);
                if (!result) {
                    throw new Error(`Operation ${operation.operation} on table ${operation.resourceName} failed`);
                }
                results.push(result);
            }
            return { success: true, data: results };
        });
    } catch (error) {
        console.error('Multi-resource transaction error:', error);
        return handleDBError(error);
    }
}
```

### Logic Layer Error Handling

Business logic classes use error factory for consistent error responses:

```typescript
// From templates-create.logic.ts
export class TemplatesCreateLogic implements ILogic {
    async execute() {
        try {
            // Business logic with validation
            this.validateMetadataShape(metadata);

            const ulids = this.extractUlidsFromTemplateMetadata(metadata);
            if (ulids.size > 0) {
                await this.assertTemplateFieldUlidsExist(ulids);
            }

            return await this.repository.create(this.request.body);
        } catch (error) {
            // Re-throw with proper error factory
            if (error instanceof BaseError) {
                throw error;
            }
            throw this.errorFactory.create('InternalServerError', 'Template creation failed');
        }
    }

    private async assertTemplateFieldUlidsExist(ulids: Set<string>) {
        // ... validation logic
        if (returned.size !== ulids.size) {
            const missing = ulidArray.filter((u) => !returned.has(u));
            throw this.errorFactory.create(
                'BadRequestError',
                `Unknown template field ULID(s): ${missing.join(', ')}`
            );
        }
    }
}
```

### Retry and Circuit Breaker Patterns

#### Exponential Backoff Implementation
```typescript
class RetryService {
    async executeWithRetry<T>(
        operation: () => Promise<T>,
        options: RetryOptions = { maxRetries: 3, baseDelay: 1000, maxDelay: 10000 }
    ): Promise<T> {
        let lastError: Error;

        for (let attempt = 0; attempt <= options.maxRetries; attempt++) {
            try {
                return await operation();
            } catch (error) {
                lastError = error as Error;

                if (attempt === options.maxRetries) {
                    break;
                }

                // Exponential backoff with jitter
                const delay = Math.min(
                    options.baseDelay * Math.pow(2, attempt) + Math.random() * 1000,
                    options.maxDelay
                );

                await new Promise(resolve => setTimeout(resolve, delay));
            }
        }

        throw lastError;
    }
}
```

#### Circuit Breaker Pattern
```typescript
class CircuitBreaker {
    private failures = 0;
    private state: 'CLOSED' | 'OPEN' | 'HALF_OPEN' = 'CLOSED';

    async execute<T>(operation: () => Promise<T>): Promise<T> {
        if (this.state === 'OPEN') {
            throw new Error('Circuit breaker is OPEN');
        }

        try {
            const result = await operation();
            this.onSuccess();
            return result;
        } catch (error) {
            this.onFailure();
            throw error;
        }
    }

    private onSuccess() {
        this.failures = 0;
        this.state = 'CLOSED';
    }

    private onFailure() {
        this.failures++;
        if (this.failures >= 5) { // threshold
            this.state = 'OPEN';
            setTimeout(() => this.state = 'HALF_OPEN', 60000); // 1 minute
        }
    }
}
```

### Error Response Standardization

All errors follow a consistent response format:

```typescript
interface ErrorResponse {
    success: false;
    error: {
        statusCode: number;
        message: string;
        details?: unknown;
        timestamp?: string;
    };
}
```

### Bootstrap Error Handling

Application startup includes comprehensive error handling:

```typescript
// From bootstrap.ts
run(): void {
    try {
        this.logger.info('🚀 Starting BYO DPP Application');
        this.setup();
        this.build();
        this.boot();
        this.logger.info('✨ Application is ready');
    } catch (error) {
        this.logger.error('Failed to start application', error as Error);
        process.exit(1);
    }
}
```

### External Service Error Handling

Integration with external services includes timeout and retry logic:

```typescript
class ExternalServiceClient {
    async callWithTimeout<T>(
        operation: () => Promise<T>,
        timeoutMs: number = 5000
    ): Promise<T> {
        const timeoutPromise = new Promise<never>((_, reject) =>
            setTimeout(() => reject(new Error('Operation timed out')), timeoutMs)
        );

        return Promise.race([operation(), timeoutPromise]);
    }
}
```
## Security Best Practices

### Defense-in-Depth Strategy
The BYO DPP implements multiple layers of security controls:

```typescript
// src/infra/middlewares/security.middleware.ts
export class SecurityMiddleware implements IMiddleware {
    constructor(config: SecurityConfig) {
        this.helmet = helmet({
            contentSecurityPolicy: {
                directives: {
                    defaultSrc: ["'self'"],
                    scriptSrc: ["'self'", "'unsafe-inline'"],
                    styleSrc: ["'self'", "'unsafe-inline'"],
                    imgSrc: ["'self'", "data:", "https:"]
                }
            },
            hsts: { maxAge: 31536000, includeSubDomains: true }
        });
        
        this.rateLimiter = rateLimit({
            windowMs: config.rateLimitWindow,
            max: config.rateLimitMax,
            message: 'Too many requests from this IP'
        });
    }
}
```

### Authentication & Authorization
#### JWT Validation Flow:
```typescript
// src/infra/middlewares/authenticator.middleware.ts
export class AuthenticatorMiddleware implements IMiddleware {
    async execute(req: Request, res: Response, next: NextFunction) {
        const token = this.extractToken(req);
        const decoded = await this.identityService.verifyToken(token);
        
        req.user = {
            id: decoded.sub,
            roles: decoded.roles,
            permissions: decoded.permissions
        };
        
        next();
    }
}
```

### Input Validation
#### Schema-Based Validation:
```typescript
// src/core/schemas/resource/users/post/create-users.schema.ts
export const createUserSchema = Joi.object({
    email: Joi.string().email().required(),
    password: Joi.string()
        .min(12)
        .pattern(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d).*$/)
        .required()
});
```

### Security Controls Matrix
| Control Type | Implementation | Technical Details |
|--------------|----------------|-------------------|
| Encryption | TLS 1.3 | HTTPS enforced via HSTS |
| Authentication | OAuth2/OIDC | Integration with Keycloak |
| Authorization | RBAC | Role-based access control |
| Input Validation | Joi/Zod | Schema validation middleware |
| Rate Limiting | Express Rate Limit | 100 req/min per IP |
| Headers | Helmet | 15+ security headers |
| Logging | Winston | PII redaction |

### Audit Logging Implementation
```typescript
// src/core/functionals/gfr779.determineNewVersion.ts
export const auditSecurityEvent = (event: SecurityEvent) => {
    logger.info('Security event', {
        eventType: event.type,
        userId: event.userId,
        ip: event.ip,
        userAgent: event.userAgent,
        resource: event.resource,
        metadata: redactPII(event.metadata)
    });
    
    // Send to SIEM system
    siemService.logEvent(event);
};
```

### Vulnerability Management
1. **Dependency Scanning**:
```bash
npm audit --production
```
2. **Container Scanning**:
```bash
docker scan byo-dpp-app
```
3. **DAST**:
```bash
zap-baseline.py -t https://api.example.com
```

### Security Headers Configuration
```typescript
// Security middleware configuration
app.use(helmet({
    contentSecurityPolicy: {
        directives: {
            defaultSrc: ["'self'"],
            scriptSrc: ["'self'", "trusted-cdn.com"],
            objectSrc: ["'none'"],
            upgradeInsecureRequests: []
        }
    },
    referrerPolicy: { policy: 'strict-origin-when-cross-origin' }
}));
```

### Incident Response Protocol
1. **Detection**: Monitor security logs and metrics
2. **Containment**: Isolate affected systems
3. **Eradication**: Remove root cause
4. **Recovery**: Restore from clean backups
5. **Post-Mortem**: Document lessons learned

### Data Protection Measures
```typescript
// src/infra/services/encryption.service.ts
export class EncryptionService {
    async encryptData(data: string): Promise<string> {
        const iv = crypto.randomBytes(12);
        const cipher = crypto.createCipheriv('aes-256-gcm', this.key, iv);
        return iv.toString('hex') + ':' + cipher.update(data, 'utf8', 'hex');
    }

    async decryptData(encrypted: string): Promise<string> {
        const [ivHex, cipherText] = encrypted.split(':');
        const iv = Buffer.from(ivHex, 'hex');
        const decipher = crypto.createDecipheriv('aes-256-gcm', this.key, iv);
        return decipher.update(cipherText, 'hex', 'utf8');
    }
}
```

### Logging and Monitoring

Errors are logged with appropriate levels:

```typescript
class Logger {
    error(message: string, error?: Error): void {
        console.error(chalk.red(`✖ ${message}`), error);
        // Send to monitoring service
        this.sendToMonitoring('error', message, error);
    }

    warn(message: string): void {
        console.warn(chalk.yellow(`⚠ ${message}`));
    }
}
```

## Deployment Architecture and Runtime Concerns

### Containerization Strategy

The BYO DPP system uses a comprehensive containerization approach with multi-stage builds and orchestration:

#### Multi-Stage Dockerfile Implementation
```dockerfile
# Dockerfile
FROM node:18-alpine AS base

# Install dependencies for native modules
RUN apk add --no-cache python3 make g++

WORKDIR /app

# Dependencies stage
FROM base AS dependencies
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

# Build stage
FROM base AS build
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Runtime stage
FROM node:18-alpine AS runtime

# Install runtime dependencies
RUN apk add --no-cache dumb-init curl

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nextjs -u 1001

WORKDIR /app

# Copy production dependencies
COPY --from=dependencies --chown=nextjs:nodejs /app/node_modules ./node_modules

# Copy built application
COPY --from=build --chown=nextjs:nodejs /app/dist ./dist
COPY --from=build --chown=nextjs:nodejs /app/package*.json ./
COPY --from=build --chown=nextjs:nodejs /app/scripts ./scripts
COPY --from=build --chown=nextjs:nodejs /app/migrations ./migrations

# Switch to non-root user
USER nextjs

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:3000/health || exit 1

# Use dumb-init for proper signal handling
ENTRYPOINT ["dumb-init", "--"]
CMD ["npm", "run", "start:prod"]
```

#### Development Dockerfile
```dockerfile
# Dockerfile.dev
FROM node:18-alpine

# Install development dependencies
RUN apk add --no-cache python3 make g++ git curl

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install all dependencies (including dev)
RUN npm install

# Copy source code
COPY . .

# Expose port
EXPOSE 3000

# Development command with hot reload
CMD ["npm", "run", "dev"]
```

### Docker Compose Orchestration

#### Production docker-compose.yml
```yaml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: byo-dpp-app
    environment:
      - NODE_ENV=production
      - POSTGRES_DB_URI=${POSTGRES_DB_URI}
      - REDIS_URL=${REDIS_URL}
    ports:
      - "3000:3000"
    depends_on:
      - db
      - redis
    networks:
      - byo-dpp-network
    restart: unless-stopped
    volumes:
      - uploads:/app/uploads
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  db:
    image: postgres:15-alpine
    container_name: byo-dpp-db
    environment:
      - POSTGRES_DB=byo_dpp
      - POSTGRES_USER=${DB_USER}
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./migrations:/docker-entrypoint-initdb.d
    ports:
      - "5432:5432"
    networks:
      - byo-dpp-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER} -d byo_dpp"]
      interval: 30s
      timeout: 10s
      retries: 3

  redis:
    image: redis:7-alpine
    container_name: byo-dpp-redis
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"
    networks:
      - byo-dpp-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 30s
      timeout: 10s
      retries: 3

  nginx:
    image: nginx:alpine
    container_name: byo-dpp-nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/nginx/ssl:ro
    depends_on:
      - app
    networks:
      - byo-dpp-network
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:
  uploads:

networks:
  byo-dpp-network:
    driver: bridge
```

#### Development docker-compose.yml
```yaml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile.dev
    container_name: byo-dpp-app-dev
    environment:
      - NODE_ENV=development
      - POSTGRES_DB_URI=postgresql://user:password@db:5432/byo_dpp
    ports:
      - "3000:3000"
    volumes:
      - .:/app
      - /app/node_modules
    depends_on:
      - db
    networks:
      - byo-dpp-dev-network
    command: npm run dev

  db:
    image: postgres:15-alpine
    container_name: byo-dpp-db-dev
    environment:
      - POSTGRES_DB=byo_dpp
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=password
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    networks:
      - byo-dpp-dev-network

volumes:
  postgres_data:

networks:
  byo-dpp-dev-network:
    driver: bridge
```

### Environment Management

#### Environment Configuration System
```typescript
// src/config/app-config.ts
export class AppConfig {
    private static instance: AppConfig;
    private config: IAppConfigOptions;

    private constructor() {
        this.config = this.loadConfig();
    }

    static getInstance(): AppConfig {
        if (!AppConfig.instance) {
            AppConfig.instance = new AppConfig();
        }
        return AppConfig.instance;
    }

    private loadConfig(): IAppConfigOptions {
        const nodeEnv = process.env.NODE_ENV || 'dev';

        // Load base config
        const baseConfig = this.loadBaseConfig();

        // Load environment-specific config
        const envConfig = this.loadEnvironmentConfig(nodeEnv);

        // Merge configurations
        return { ...baseConfig, ...envConfig };
    }

    private loadBaseConfig(): Partial<IAppConfigOptions> {
        return {
            PORT: parseInt(process.env.PORT || '3000'),
            NODE_ENV: process.env.NODE_ENV || 'dev',
            LOG_LEVEL: process.env.LOG_LEVEL || 'info',
        };
    }

    private loadEnvironmentConfig(nodeEnv: string): Partial<IAppConfigOptions> {
        const configPath = path.resolve(__dirname, `../../config/${nodeEnv}.config.ts`);

        if (fs.existsSync(configPath)) {
            const envConfig = require(configPath);
            return envConfig.default || envConfig;
        }

        throw new Error(`Configuration file not found for environment: ${nodeEnv}`);
    }

    getConfig(): IAppConfigOptions {
        return this.config;
    }
}

// config/dev.config.ts
export default {
    POSTGRES_DB_URI: process.env.POSTGRES_DB_URI || 'postgresql://user:password@localhost:5432/byo_dpp',
    KEYCLOAK_HOST_URI: process.env.KEYCLOAK_HOST_URI || 'http://localhost:8080',
    REDIS_URL: process.env.REDIS_URL || 'redis://localhost:6379',
    LOG_LEVEL: 'debug',
    ENABLE_SWAGGER: true,
    CORS_ORIGIN: '*',
};

// config/prod.config.ts
export default {
    POSTGRES_DB_URI: process.env.POSTGRES_DB_URI!,
    KEYCLOAK_HOST_URI: process.env.KEYCLOAK_HOST_URI!,
    REDIS_URL: process.env.REDIS_URL!,
    LOG_LEVEL: 'warn',
    ENABLE_SWAGGER: false,
    CORS_ORIGIN: process.env.CORS_ORIGIN || 'https://yourdomain.com',
};
```

### Health Checks and Monitoring

#### Application Health Endpoints
```typescript
// src/application/controllers/health.controller.ts
export class HealthController {
    constructor(
        private database: IDatabase,
        private redis: IRedis,
        private externalServices: IExternalService[]
    ) {}

    async healthCheck(): Promise<HealthStatus> {
        const checks = await Promise.all([
            this.checkDatabase(),
            this.checkRedis(),
            ...this.externalServices.map(service => this.checkExternalService(service))
        ]);

        const isHealthy = checks.every(check => check.healthy);

        return {
            status: isHealthy ? 'healthy' : 'unhealthy',
            timestamp: new Date().toISOString(),
            checks,
            uptime: process.uptime()
        };
    }

    private async checkDatabase(): Promise<HealthCheck> {
        try {
            await this.database.query('SELECT 1');
            return { name: 'database', healthy: true };
        } catch (error) {
            return { name: 'database', healthy: false, error: error.message };
        }
    }

    private async checkRedis(): Promise<HealthCheck> {
        try {
            await this.redis.ping();
            return { name: 'redis', healthy: true };
        } catch (error) {
            return { name: 'redis', healthy: false, error: error.message };
        }
    }

    private async checkExternalService(service: IExternalService): Promise<HealthCheck> {
        try {
            await service.healthCheck();
            return { name: service.name, healthy: true };
        } catch (error) {
            return { name: service.name, healthy: false, error: error.message };
        }
    }
}

interface HealthStatus {
    status: 'healthy' | 'unhealthy';
    timestamp: string;
    checks: HealthCheck[];
    uptime: number;
}

interface HealthCheck {
    name: string;
    healthy: boolean;
    error?: string;
    responseTime?: number;
}
```

#### Comprehensive Monitoring System
```typescript
// src/infra/monitoring/monitoring.service.ts
export class MonitoringService {
    constructor(
        private metrics: IMetrics,
        private logger: ILogger,
        private alertManager: IAlertManager
    ) {}

    async recordRequestMetrics(req: Request, res: Response, duration: number) {
        const labels = {
            method: req.method,
            route: req.route?.path || req.path,
            statusCode: res.statusCode
        };

        this.metrics.incrementCounter('http_requests_total', labels);
        this.metrics.observeHistogram('http_request_duration_seconds', duration / 1000, labels);

        if (res.statusCode >= 400) {
            this.metrics.incrementCounter('http_requests_errors_total', labels);
        }
    }

    async recordDatabaseMetrics(operation: string, duration: number, success: boolean) {
        const labels = { operation, success: success.toString() };

        this.metrics.incrementCounter('db_operations_total', labels);
        this.metrics.observeHistogram('db_operation_duration_seconds', duration / 1000, labels);
    }

    async recordBusinessMetrics(event: BusinessEvent) {
        this.metrics.incrementCounter('business_events_total', {
            event_type: event.type,
            resource: event.resource
        });
    }

    async checkSystemHealth(): Promise<SystemHealth> {
        const metrics = await this.metrics.getSystemMetrics();

        if (metrics.memoryUsage > 0.9) {
            await this.alertManager.sendAlert('High memory usage detected');
        }

        if (metrics.cpuUsage > 0.8) {
            await this.alertManager.sendAlert('High CPU usage detected');
        }

        return metrics;
    }
}

// Metrics collection
export class PrometheusMetrics implements IMetrics {
    private register: Registry;

    constructor() {
        this.register = new Registry();

        // HTTP metrics
        this.register.registerMetric(new Counter({
            name: 'http_requests_total',
            help: 'Total number of HTTP requests',
            labelNames: ['method', 'route', 'status_code']
        }));

        // Database metrics
        this.register.registerMetric(new Histogram({
            name: 'db_operation_duration_seconds',
            help: 'Duration of database operations',
            labelNames: ['operation', 'success']
        }));

        // Business metrics
        this.register.registerMetric(new Counter({
            name: 'business_events_total',
            help: 'Total number of business events',
            labelNames: ['event_type', 'resource']
        }));
    }

    incrementCounter(name: string, labels: Record<string, string>) {
        const metric = this.register.getSingleMetric(name) as Counter;
        metric.inc(labels);
    }

    observeHistogram(name: string, value: number, labels: Record<string, string>) {
        const metric = this.register.getSingleMetric(name) as Histogram;
        metric.observe(labels, value);
    }

    async getSystemMetrics(): Promise<SystemHealth> {
        const memUsage = process.memoryUsage();
        const cpuUsage = process.cpuUsage();

        return {
            memoryUsage: memUsage.heapUsed / memUsage.heapTotal,
            cpuUsage: cpuUsage.user + cpuUsage.system,
            uptime: process.uptime(),
            activeConnections: await this.getActiveConnections()
        };
    }
}
```

### Load Balancing and Scaling

#### Nginx Configuration for Load Balancing
```nginx
# nginx.conf
upstream byo_dpp_app {
    least_conn;
    server app1:3000;
    server app2:3000;
    server app3:3000;
    keepalive 32;
}

server {
    listen 80;
    server_name yourdomain.com;

    # SSL configuration
    listen 443 ssl http2;
    ssl_certificate /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;

    # Security headers
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";
    add_header Strict-Transport-Security "max-age=31536000; includeSubdomains";

    # API routes
    location /api/ {
        proxy_pass http://byo_dpp_app;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
        proxy_read_timeout 86400;
    }

    # Static files
    location /static/ {
        alias /app/static/;
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # Health check
    location /health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }
}
```

#### Horizontal Pod Autoscaling (Kubernetes)
```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: byo-dpp-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: byo-dpp-app
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
```

### Database Optimization and Connection Pooling

#### Connection Pool Configuration
```typescript
// src/infra/database/connection-pool.ts
export class DatabaseConnectionPool {
    private pool: Pool;

    constructor(config: DatabaseConfig) {
        this.pool = new Pool({
            host: config.host,
            port: config.port,
            database: config.database,
            user: config.user,
            password: config.password,
            max: config.maxConnections || 20,
            min: config.minConnections || 5,
            idleTimeoutMillis: config.idleTimeout || 30000,
            connectionTimeoutMillis: config.connectionTimeout || 2000,
            acquireTimeoutMillis: config.acquireTimeout || 60000,
        });

        this.setupPoolEvents();
    }

    private setupPoolEvents() {
        this.pool.on('connect', (client) => {
            console.log('New client connected to database');
        });

        this.pool.on('error', (err, client) => {
            console.error('Unexpected error on idle client', err);
        });

        this.pool.on('remove', (client) => {
            console.log('Client removed from pool');
        });
    }

    async getClient(): Promise<PoolClient> {
        return await this.pool.connect();
    }

    async query(text: string, params?: any[]): Promise<QueryResult> {
        const client = await this.getClient();
        try {
            return await client.query(text, params);
        } finally {
            client.release();
        }
    }

    async healthCheck(): Promise<boolean> {
        try {
            await this.query('SELECT 1');
            return true;
        } catch (error) {
            return false;
        }
    }

    async close(): Promise<void> {
        await this.pool.end();
    }

    getPoolStats() {
        return {
            totalCount: this.pool.totalCount,
            idleCount: this.pool.idleCount,
            waitingCount: this.pool.waitingCount
        };
    }
}
```

### Caching Strategy

#### Multi-Level Caching Implementation
```typescript
// src/infra/cache/cache-manager.ts
export class CacheManager implements ICache {
    constructor(
        private redis: IRedis,
        private memoryCache: NodeCache,
        private config: CacheConfig
    ) {}

    async get<T>(key: string): Promise<T | null> {
        // Try memory cache first
        let value = this.memoryCache.get<T>(key);
        if (value !== undefined) {
            return value;
        }

        // Try Redis cache
        value = await this.redis.get<T>(key);
        if (value !== null) {
            // Populate memory cache
            this.memoryCache.set(key, value, this.config.memoryTTL);
            return value;
        }

        return null;
    }

    async set<T>(key: string, value: T, ttl?: number): Promise<void> {
        const expiry = ttl || this.config.defaultTTL;

        // Set in both caches
        this.memoryCache.set(key, value, expiry);
        await this.redis.set(key, value, expiry);
    }

    async delete(key: string): Promise<void> {
        this.memoryCache.del(key);
        await this.redis.del(key);
    }

    async clear(): Promise<void> {
        this.memoryCache.flushAll();
        await this.redis.flushall();
    }

    // Cache warming for frequently accessed data
    async warmCache(): Promise<void> {
        const keys = await this.getFrequentKeys();
        for (const key of keys) {
            const value = await this.redis.get(key);
            if (value) {
                this.memoryCache.set(key, value, this.config.memoryTTL);
            }
        }
    }
}
```

### Security Hardening

#### Security Middleware Stack
```typescript
// src/infra/middlewares/security.middleware.ts
export class SecurityMiddleware implements IMiddleware {
    private rateLimiter: RateLimiter;
    private helmet: Helmet;

    constructor(config: SecurityConfig) {
        this.rateLimiter = new RateLimiter({
            windowMs: config.rateLimitWindow,
            max: config.rateLimitMax,
            message: 'Too many requests from this IP, please try again later.'
        });

        this.helmet = helmet({
            contentSecurityPolicy: {
                directives: {
                    defaultSrc: ["'self'"],
                    styleSrc: ["'self'", "'unsafe-inline'"],
                    scriptSrc: ["'self'"],
                    imgSrc: ["'self'", "data:", "https:"],
                },
            },
            hsts: {
                maxAge: 31536000,
                includeSubDomains: true,
                preload: true
            }
        });
    }

    async execute(req: Request, res: Response, next: NextFunction) {
        // Apply Helmet security headers
        this.helmet(req, res, (err) => {
            if (err) return next(err);
        });

        // Rate limiting
        this.rateLimiter(req, res, (err) => {
            if (err) return next(err);
        });

        // CORS
        res.header('Access-Control-Allow-Origin', process.env.CORS_ORIGIN || '*');
        res.header('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE, PATCH, OPTIONS');
        res.header('Access-Control-Allow-Headers', 'Origin, X-Requested-With, Content-Type, Accept, Authorization');

        if (req.method === 'OPTIONS') {
            res.sendStatus(200);
            return;
        }

        next();
    }
}
```

### Logging and Observability

#### Structured Logging System
```typescript
// src/infra/logging/logger.ts
export class StructuredLogger implements ILogger {
    constructor(
        private winstonLogger: winston.Logger,
        private context: string
    ) {}

    info(message: string, meta?: any) {
        this.winstonLogger.info(message, {
            context: this.context,
            ...meta,
            timestamp: new Date().toISOString()
        });
    }

    error(message: string, error?: Error, meta?: any) {
        this.winstonLogger.error(message, {
            context: this.context,
            error: error?.message,
            stack: error?.stack,
            ...meta,
            timestamp: new Date().toISOString()
        });
    }

    warn(message: string, meta?: any) {
        this.winstonLogger.warn(message, {
            context: this.context,
            ...meta,
            timestamp: new Date().toISOString()
        });
    }

    debug(message: string, meta?: any) {
        this.winstonLogger.debug(message, {
            context: this.context,
            ...meta,
            timestamp: new Date().toISOString()
        });
    }
}

// Winston configuration
const logger = winston.createLogger({
    level: process.env.LOG_LEVEL || 'info',
    format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.errors({ stack: true }),
        winston.format.json()
    ),
    defaultMeta: { service: 'byo-dpp-backend' },
    transports: [
        new winston.transports.Console({
            format: winston.format.combine(
                winston.format.colorize(),
                winston.format.simple()
            )
        }),
        new winston.transports.File({
            filename: 'logs/error.log',
            level: 'error'
        }),
        new winston.transports.File({
            filename: 'logs/combined.log'
        })
    ]
});
```

This deployment architecture ensures:
- **Scalability**: Horizontal scaling with load balancing
- **Reliability**: Health checks, monitoring, and failover
- **Security**: Comprehensive security hardening
- **Performance**: Caching, connection pooling, and optimization
- **Observability**: Structured logging and metrics collection
- **Maintainability**: Containerization and configuration management

## API → Service → Domain → Infra Implementation Walk-through

### Complete Request Flow Architecture

The BYO DPP system implements a layered architecture with clear separation of concerns, following Clean Architecture principles:

#### Layer Overview
```
┌─────────────────────────────────────┐
│         Application Layer           │ ← HTTP, Routing, Controllers
├─────────────────────────────────────┤
│          Domain Layer               │ ← Business Logic, Entities
├─────────────────────────────────────┤
│       Infrastructure Layer          │ ← Database, External Services
└─────────────────────────────────────┘
```

### Detailed Request Flow Implementation

#### 1. HTTP Request Entry Point
```typescript
// From src/infra/adapters/express/express.adapter.ts
export class ExpressAdapter implements IServer {
    private app: Express;

    constructor(private config: IServerConfig) {
        this.app = express();
        this.setupMiddleware();
        this.setupRoutes();
    }

    private setupMiddleware() {
        this.app.use(cors(this.config.corsOptions));
        this.app.use(express.json({ limit: '50mb' }));
        this.app.use(express.urlencoded({ extended: true }));
        this.app.use(this.loggerMiddleware);
    }

    listen(port: number, host: string): void {
        this.app.listen(port, host, () => {
            console.log(`Server running on http://${host}:${port}`);
        });
    }
}
```

#### 2. Route Registration and Middleware Pipeline
```typescript
// From src/bootstrap.ts - Route Building
buildRoutes(): void {
    this.byoDPPJson.forEach((service: Service) => {
        service.resources.forEach((resource: Resource) => {
            // Create layered components
            const resourceBaseRepoAdapter = this.createRepository(resource);
            const resourceBaseService = this.createService(resource, resourceBaseRepoAdapter);
            const resourceBaseController = this.createController(resource, resourceBaseService);

            // Setup middleware pipeline
            const middlewares = this.createMiddlewarePipeline(resource);

            // Register routes for each HTTP method
            this.registerResourceRoutes(resource, resourceBaseController, middlewares);
        });
    });
}

private createMiddlewarePipeline(resource: Resource): IMiddleware[] {
    const middlewares: IMiddleware[] = [];

    // Authentication middleware
    middlewares.push(new AuthenticatorMiddleware(this.dependencyFactory.getIdentityService()));

    // Validation middleware (for CRUD operations)
    if (this.validationSchemaMap[resource.name]) {
        // Add appropriate validators based on HTTP method
    }

    // Batch processing middleware
    if (resource.supportsBatch) {
        middlewares.push(new BatchMiddleware(JoiAdapter, this.validationSchemaMap[resource.name]));
    }

    return middlewares;
}
```

#### 3. Controller Layer Implementation
```typescript
// From src/application/adapters/controller/base-resource.controller.ts
export class BaseController {
    constructor(
        private service: IResourceService,
        private graphQuery: IGraphAPI,
        private resourceName: string
    ) {}

    async create(request: IRequest): Promise<IResponse> {
        try {
            const result = await this.service.create(request);
            return this.formatResponse(result);
        } catch (error) {
            return this.handleError(error);
        }
    }

    async list(request: IRequest): Promise<IResponse> {
        try {
            // Parse complex query parameters
            const query = this.graphQuery.parse(request.query);
            const result = await this.service.list({ ...request, query });

            return {
                success: true,
                data: result.data,
                count: result.count,
                pagination: this.buildPaginationInfo(request, result.count)
            };
        } catch (error) {
            return this.handleError(error);
        }
    }

    async read(request: IRequest): Promise<IResponse> {
        try {
            const query = this.graphQuery.parse(request.query);
            const result = await this.service.read({ ...request, query });
            return this.formatResponse(result);
        } catch (error) {
            return this.handleError(error);
        }
    }

    private formatResponse(result: any): IResponse {
        if (result.success) {
            return {
                success: true,
                data: result.data,
                message: this.getSuccessMessage('create')
            };
        } else {
            return {
                success: false,
                error: result.error
            };
        }
    }

    private handleError(error: any): IResponse {
        if (error instanceof BaseError) {
            return {
                success: false,
                error: {
                    statusCode: error.statusCode,
                    message: error.message
                }
            };
        }

        return {
            success: false,
            error: {
                statusCode: 500,
                message: 'Internal server error'
            }
        };
    }
}
```

#### 4. Service Layer with Logic Injection
```typescript
// From src/application/adapters/service/base-resource.service.ts
export class BaseService implements IResourceService {
    constructor(
        private repository: IAppRepository,
        private resourceName: string,
        private logicMap: Record<string, LogicConstructor> = {}
    ) {}

    // Generic operation handler with logic fallback
    private handleOperation(
        name: string,
        request: IRequest,
        fallback: () => unknown
    ) {
        const LogicClass = this.getLogic(name);
        if (LogicClass) {
            // Use custom business logic
            const instance = new LogicClass(request, this.repository);
            return instance.execute();
        }
        // Fallback to repository method
        return fallback();
    }

    async create(request: IRequest) {
        return this.handleOperation('create', request,
            () => this.repository.create(request.body)
        );
    }

    async list(request: IRequest) {
        return this.handleOperation('list', request,
            () => this.repository.list(request.query)
        );
    }

    async read(request: IRequest) {
        return this.handleOperation('read', request,
            () => this.repository.read(request.query)
        );
    }

    async update(request: IRequest) {
        return this.handleOperation('update', request,
            () => this.repository.update(request.query, request.body)
        );
    }

    async patch(request: IRequest) {
        return this.handleOperation('patch', request,
            () => this.repository.patch(request.query, request.body)
        );
    }

    async delete(request: IRequest) {
        return this.handleOperation('delete', request,
            () => this.repository.delete(request.query)
        );
    }

    // Action methods for custom business operations
    async action(request: IRequest, actionName: string) {
        return this.handleOperation(actionName, request, () => {
            return {
                success: false,
                error: 'Logic not found'
            };
        });
    }

    private getLogic(methodName: string): LogicConstructor | undefined {
        return this.logicMap[methodName];
    }
}
```

#### 5. Domain Logic Implementation
```typescript
// From src/core/logics/v1/dpps/crud/dpps-create.logic.ts
export class DppsCreateLogic implements ILogic {
    request: IRequest;
    repository: ICoreRepository;
    coreProvider: CoreProvider;
    errorFactory: IErrorFactory;

    constructor(request: IRequest, repository: ICoreRepository) {
        this.request = request;
        this.repository = repository;
        this.coreProvider = CoreProvider.getInstance();
        this.errorFactory = this.coreProvider.getErrorFactory();
    }

    async execute() {
        // Domain logic: Prepare payload with metadata
        const payload = {
            ...this.request.body,
            metadata: {},
            uuid: null
        };

        // Business rule: DPP ID must be unique (handled by DB constraints)
        // Additional domain validations can be added here

        return await this.repository.create(payload);
    }
}

// From src/core/logics/v1/templates/crud/templates-create.logic.ts
export class TemplatesCreateLogic implements ILogic {
    async execute() {
        const { metadata } = this.request.body;

        // Domain validation: Metadata structure
        if (metadata !== undefined) {
            this.validateMetadataShape(metadata);
        }

        // Domain logic: Validate template field references
        if (metadata && typeof metadata === 'object') {
            const ulids = this.extractUlidsFromTemplateMetadata(metadata);
            if (ulids.size > 0) {
                await this.assertTemplateFieldUlidsExist(ulids);
            }
        }

        return await this.repository.create(this.request.body);
    }

    private async assertTemplateFieldUlidsExist(ulids: Set<string>) {
        const repo = this.coreProvider.getRepoDiscovery().getRepository('TemplateFields');
        const ulidArray = Array.from(ulids);
        const query = {
            fields: ['ulid'],
            filter: { ulid: { in: ulidArray } }
        };
        const parsed = this.coreProvider.getGraphAPI().parse(query);
        const result = await repo.list(parsed);
        const returned = new Set<string>((result?.data || []).map((r: { ulid: string }) => r.ulid));

        if (returned.size !== ulids.size) {
            const missing = ulidArray.filter((u) => !returned.has(u));
            throw this.errorFactory.create(
                'BadRequestError',
                `Unknown template field ULID(s): ${missing.join(', ')}`
            );
        }
    }
}
```

#### 6. Repository Layer with ORM Abstraction
```typescript
// From src/infra/adapters/repository/drizzle-orm.repository.ts
export class DrizzleORMRepository<T extends string, M extends PgTable> implements IRepository {
    constructor(
        database: IDatabase,
        modelName: T,
        model: M
    ) {
        this.db = database;
        this.modelName = modelName;
        this.model = model;
        this.operators = this.getOperators();
        this.table = ResourceModelMap[this.modelName as keyof typeof ResourceModelMap];
        this.dbClient = this.getClient();
    }

    async create(data: object | object[]): Promise<RepositoryResult> {
        try {
            const isBulk = Array.isArray(data);
            const res = await this.dbClient.insert(this.table).values(data).returning();
            return { success: true, data: isBulk ? res : res[0] };
        } catch (error) {
            return handleDBError(error);
        }
    }

    async read(query: any): Promise<RepositoryResult> {
        try {
            const data = await this.dbClient.query[this.modelName].findFirst(query);
            if (!data) {
                return { success: false, error: { statusCode: 404, message: 'Item not found' } };
            }
            return { success: true, data: data };
        } catch (error) {
            return handleDBError(error);
        }
    }

    async list(query: any): Promise<RepositoryResult & { count?: number }> {
        try {
            // Build where condition
            let whereCondition;
            if (query.where) {
                whereCondition = query.where(this.table, this.operators);
            }

            // Execute data query
            const dataQuery = { ...query };
            if (whereCondition) {
                dataQuery.where = whereCondition;
            }

            const data = await this.dbClient.query[this.modelName].findMany(dataQuery);

            // Execute count query
            const countQuery = this.dbClient.select({ count: sql<number>`count(*)` }).from(this.table);
            if (whereCondition) {
                countQuery.where(whereCondition);
            }
            const countResult = await countQuery;

            return {
                success: true,
                data: data,
                count: Number(countResult[0].count)
            };
        } catch (error) {
            return handleDBError(error);
        }
    }

    async update(query: any, data: object): Promise<RepositoryResult> {
        try {
            // Find existing item
            const item = await this.getItem(query);
            if (!item) {
                return { success: false, error: { statusCode: 404, message: 'Item not found' } };
            }

            // Update item
            const res = await this.dbClient
                .update(this.model)
                .set(data)
                .where(eq(this.table.id, Number(item.id)))
                .returning();

            return { success: true, data: res[0] };
        } catch (error) {
            return handleDBError(error);
        }
    }

    // Transaction support for complex operations
    async transact(operations: [TransactionMethod, any][]): Promise<RepositoryResult> {
        try {
            return await this.dbClient.transaction(async (tx) => {
                const results = [];

                for (const [operation, data] of operations) {
                    let result;

                    switch (operation.toLowerCase()) {
                        case 'create':
                            result = await tx.insert(this.table).values(data).returning();
                            break;
                        case 'update':
                            result = await tx
                                .update(this.table)
                                .set(data)
                                .where(eq(this.table.id, Number(data.id)))
                                .returning();
                            break;
                        case 'delete':
                            result = await tx
                                .delete(this.table)
                                .where(eq(this.table.id, Number(data.id)))
                                .returning();
                            break;
                    }

                    if (!result?.[0]) {
                        return { success: false, error: { statusCode: 400, message: `Operation ${operation} failed` } };
                    }

                    results.push(result[0]);
                }

                return { success: true, data: results };
            });
        } catch (error) {
            return handleDBError(error);
        }
    }

    // Multi-resource transactions
    async multiResourceTransact(operations: MultiResourceTransactionOperation[]): Promise<MultiResourceTransactionResult> {
        try {
            return await this.dbClient.transaction(async (tx) => {
                const results = [];

                for (const operation of operations) {
                    const { resourceName, operation: op, data, where } = operation;
                    const table = ResourceModelMap[resourceName as keyof typeof ResourceModelMap];

                    const resolvedData = this.resolveResultStrings(data, results);
                    const resolvedWhere = where ? this.resolveResultStrings(where, results) : where;

                    let result;
                    switch (op.toLowerCase()) {
                        case 'create':
                            result = await tx.insert(table).values(resolvedData).returning();
                            break;
                        case 'update':
                            result = await tx
                                .update(table)
                                .set(resolvedData)
                                .where(eq(table.id, Number(resolvedData.id)))
                                .returning();
                            break;
                        case 'delete':
                            result = await tx
                                .delete(table)
                                .where(eq(table.id, Number(resolvedData.id)))
                                .returning();
                            break;
                    }

                    results.push(result[0]);
                }

                return { success: true, data: results };
            });
        } catch (error) {
            return handleDBError(error);
        }
    }
}
```

#### 7. Infrastructure Services Integration
```typescript
// From src/infra/adapters/file-storage/file-storage.adapter.ts
export class FileStorageAdapter implements IFileStorage {
    async upload(file: Buffer, filename: string, mimeType: string): Promise<string> {
        // Upload to cloud storage (AWS S3, etc.)
        // Return public URL
    }

    async download(url: string): Promise<Buffer> {
        // Download file from storage
    }

    async delete(url: string): Promise<void> {
        // Delete file from storage
    }
}

// From src/infra/adapters/mailer/mailer.adapter.ts
export class MailerAdapter implements IMailer {
    async sendEmail(options: EmailOptions): Promise<void> {
        // Send email using SMTP or service (SendGrid, etc.)
    }
}

// From src/infra/adapters/blockchains/blockchain.adapter.ts
export class BlockchainAdapter implements IBlockchain {
    async storeHash(data: BlockchainData): Promise<string> {
        // Store hash on blockchain
        // Return transaction hash
    }

    async verifyHash(hash: string): Promise<boolean> {
        // Verify hash exists on blockchain
    }
}
```

### Middleware Pipeline Implementation

#### Authentication Middleware
```typescript
// From src/infra/adapters/middlewares/authenticator.middleware.ts
export class AuthenticatorMiddleware implements IMiddleware {
    constructor(private identityService: IIdentityService) {}

    async execute(request: IRequest, response: IResponse, next: NextFunction) {
        try {
            const token = this.extractToken(request);
            const user = await this.identityService.verifyToken(token);

            request.user = user;
            next();
        } catch (error) {
            response.status(401).json({
                success: false,
                error: { statusCode: 401, message: 'Unauthorized' }
            });
        }
    }
}
```

#### Validation Middleware
```typescript
// From src/infra/adapters/middlewares/validator.middleware.ts
export class ValidatorMiddleware implements IMiddleware {
    constructor(private validator: IValidator, private schema: SchemaDefinition) {}

    async execute(request: IRequest, response: IResponse, next: NextFunction) {
        try {
            const { error, value } = await this.validator.validate(request.body, this.schema);

            if (error) {
                response.status(400).json({
                    success: false,
                    error: { statusCode: 400, message: error.details[0].message }
                });
                return;
            }

            request.body = value;
            next();
        } catch (error) {
            response.status(500).json({
                success: false,
                error: { statusCode: 500, message: 'Validation error' }
            });
        }
    }
}
```

### Graph Query API for Complex Relationships

#### Graph Query Implementation
```typescript
// From src/infra/adapters/graph-query/drizzle-graph-query.adapter.ts
export class DrizzleGraphQueryAdapter implements IGraphAPI {
    parse(query: any): ParsedQuery {
        // Parse include, filter, sort, pagination parameters
        // Build complex SQL queries with joins
        return {
            where: this.buildWhereClause(query.filter),
            include: this.buildIncludeClause(query.include),
            orderBy: this.buildOrderByClause(query.sort),
            limit: query.limit,
            offset: query.offset
        };
    }

    private buildWhereClause(filter: any) {
        // Build complex where conditions with operators
        return (table: any, operators: any) => {
            // Implement filter parsing logic
        };
    }

    private buildIncludeClause(include: any) {
        // Build joins for related data
        return (query: any) => {
            // Implement include parsing logic
        };
    }
}
```

### Response Formatting and Content Negotiation

#### Response Builder
```typescript
class ResponseBuilder {
    static buildSuccess(data: any, message?: string, meta?: any): IResponse {
        return {
            success: true,
            data,
            message: message || 'Operation successful',
            meta
        };
    }

    static buildError(error: BaseError | string, statusCode?: number): IResponse {
        const err = error instanceof BaseError ? error : new InternalServerError(error);
        return {
            success: false,
            error: {
                statusCode: err.statusCode,
                message: err.message
            }
        };
    }

    static buildPaginated(data: any[], count: number, page: number, limit: number): IResponse {
        return {
            success: true,
            data,
            pagination: {
                page,
                limit,
                total: count,
                totalPages: Math.ceil(count / limit)
            }
        };
    }
}
```

### Complete Flow Example: Creating a DPP

#### Step-by-Step Execution with Design Pattern Analysis

1. **HTTP Request**: `POST /api/v1/dpps`
   - **Pattern**: **Front Controller Pattern**
   - **Purpose**: Single entry point for all HTTP requests
   - **Benefits**: Centralized request handling, consistent preprocessing

2. **Express Adapter**: Receives request, applies global middleware
   - **Pattern**: **Adapter Pattern** (adapts Express to our interface)
   - **Purpose**: Technology abstraction, framework independence
   - **Benefits**: Can swap web frameworks without changing business logic

3. **Router**: Matches route, applies authentication middleware
   - **Pattern**: **Chain of Responsibility** (middleware pipeline)
   - **Purpose**: Route resolution and security enforcement
   - **Benefits**: Composable request processing, security as aspect

4. **Controller**: `BaseController.create()` called
   - **Pattern**: **Application Controller Pattern**
   - **Purpose**: Orchestrates use case execution
   - **Benefits**: Thin controllers, focused on HTTP concerns

5. **Service**: `BaseService.create()` delegates to logic or repository
   - **Pattern**: **Service Layer Pattern**
   - **Purpose**: Coordinates domain operations
   - **Benefits**: Transaction management, cross-cutting concerns

6. **Logic**: `DppsCreateLogic.execute()` applies business rules
   - **Pattern**: **Domain Service Pattern** + **Strategy Pattern**
   - **Purpose**: Encapsulates complex business logic
   - **Benefits**: Business rules centralized, easily testable

7. **Repository**: `DrizzleORMRepository.create()` persists data
   - **Pattern**: **Repository Pattern** + **Data Mapper**
   - **Purpose**: Abstracts data persistence
   - **Benefits**: Technology agnostic, test-friendly

8. **Database**: PostgreSQL stores the DPP record
   - **Pattern**: **Gateway Pattern** (database as external service)
   - **Purpose**: Data persistence and retrieval
   - **Benefits**: ACID transactions, data integrity

9. **Response**: Formatted success/error response returned
   - **Pattern**: **Data Transfer Object (DTO)**
   - **Purpose**: Structured response formatting
   - **Benefits**: API contract consistency, client expectations

### Design Pattern Analysis by Layer

#### Application Layer Patterns

**Controller Pattern**: Thin layer handling HTTP concerns
```typescript
// Problem: Controllers with business logic
class BadController {
    async create(req, res) {
        // Validation, business logic, data access ALL HERE
        if (!req.body.name) return res.status(400).send('Name required');
        const user = await db.query('SELECT * FROM users WHERE id = ?', [req.user.id]);
        if (!user.canCreateDpp) return res.status(403).send('Forbidden');
        const dpp = await db.query('INSERT INTO dpps ...');
        // Email sending, logging, etc.
    }
}

// Solution: Thin controller delegating to services
export class BaseController {
    async create(request: IRequest): Promise<IResponse> {
        try {
            const result = await this.service.create(request);
            return this.formatResponse(result);
        } catch (error) {
            return this.handleError(error);
        }
    }
}
```
**Benefits**:
- **Single Responsibility**: Controllers only handle HTTP
- **Testability**: Business logic tested separately
- **Reusability**: Same logic works for REST, GraphQL, etc.

**Middleware Pattern**: Composable request processing pipeline
```typescript
// Chain of Responsibility for cross-cutting concerns
const middlewarePipeline = [
    corsMiddleware,           // CORS headers
    securityMiddleware,       // Security headers
    loggerMiddleware,         // Request logging
    rateLimitMiddleware,      // Rate limiting
    authMiddleware,           // Authentication
    validationMiddleware,     // Input validation
    controller                // Business logic
];
```
**Benefits**:
- **Separation of Concerns**: Each middleware handles one aspect
- **Composability**: Easy to add/remove/reorder middleware
- **Reusability**: Same middleware used across routes

#### Domain Layer Patterns

**Entity Pattern**: Rich domain objects with behavior
```typescript
// Problem: Anemic entities
class ProductData {
    id: number;
    name: string;
    price: number;
}

// Solution: Rich entities with business rules
export class Product {
    constructor(
        private id: number,
        private name: string,
        private price: Money
    ) {}

    changePrice(newPrice: Money): void {
        if (newPrice.lessThanOrEqual(new Money(0, this.price.currency))) {
            throw new Error('Price must be positive');
        }

        const oldPrice = this.price;
        this.price = newPrice;

        // Domain event
        this.addDomainEvent(new ProductPriceChangedEvent(
            this.id, oldPrice, newPrice
        ));
    }
}
```
**Benefits**:
- **Business Rules**: Logic encapsulated with data
- **Consistency**: Entity ensures its own invariants
- **Ubiquitous Language**: Code reflects business terminology

**Value Object Pattern**: Immutable values with behavior
```typescript
// Problem: Primitive obsession
function calculateTotal(items: { price: number, quantity: number }[]) {
    return items.reduce((sum, item) => sum + (item.price * item.quantity), 0);
}

// Solution: Rich value objects
export class Money {
    constructor(
        private readonly amount: number,
        private readonly currency: string
    ) {}

    add(other: Money): Money {
        this.ensureSameCurrency(other);
        return new Money(this.amount + other.amount, this.currency);
    }

    multiply(factor: number): Money {
        return new Money(this.amount * factor, this.currency);
    }

    private ensureSameCurrency(other: Money): void {
        if (this.currency !== other.currency) {
            throw new Error('Cannot operate on different currencies');
        }
    }
}
```
**Benefits**:
- **Immutability**: Prevents accidental changes
- **Type Safety**: Compile-time currency checking
- **Behavior**: Values can validate and operate on themselves

**Aggregate Pattern**: Consistency boundaries
```typescript
// Problem: Inconsistent object graphs
class Order {
    constructor(
        public items: OrderItem[], // Can be modified externally
        public customer: Customer   // Can be modified externally
    ) {}
}

// Solution: Aggregate with root controlling access
export class OrderAggregate {
    private order: Order;
    private items: OrderItem[];

    addItem(productId: number, quantity: number): void {
        // Business rules checked here
        this.ensureOrderNotShipped();
        this.ensureProductExists(productId);
        this.ensureQuantityAvailable(productId, quantity);

        const item = new OrderItem(productId, quantity);
        this.items.push(item);

        this.order.total = this.calculateTotal();
    }

    private ensureOrderNotShipped(): void {
        if (this.order.status === 'shipped') {
            throw new Error('Cannot modify shipped order');
        }
    }
}
```
**Benefits**:
- **Consistency**: Aggregate root ensures invariants
- **Encapsulation**: Internal structure hidden
- **Transaction Boundaries**: Defines consistency scope

#### Infrastructure Layer Patterns

**Repository Pattern**: Data access abstraction
```typescript
// Problem: Direct database coupling
class ProductService {
    async getProduct(id: number) {
        return await db.query('SELECT * FROM products WHERE id = ?', [id]);
    }
}

// Solution: Repository abstraction
interface IProductRepository {
    findById(id: number): Promise<Product>;
    findByCategory(categoryId: number): Promise<Product[]>;
    save(product: Product): Promise<void>;
}

export class DrizzleProductRepository implements IProductRepository {
    async findById(id: number): Promise<Product> {
        const result = await this.db.query.products.findFirst({
            where: eq(products.id, id)
        });
        return this.mapToDomain(result);
    }
}
```
**Benefits**:
- **Technology Agnostic**: Can switch databases
- **Testability**: Easy to mock for testing
- **Domain Focus**: Queries expressed in domain terms

**Unit of Work Pattern**: Transaction management
```typescript
// Problem: Transaction leaks
class OrderService {
    async createOrder(orderData) {
        const tx = await db.beginTransaction();
        try {
            const order = await tx.insert('orders', orderData);
            for (const item of orderData.items) {
                await tx.insert('order_items', { ...item, orderId: order.id });
            }
            await tx.commit();
            return order;
        } catch (error) {
            await tx.rollback();
            throw error;
        }
    }
}

// Solution: Unit of Work pattern
export class OrderUnitOfWork {
    constructor(private db: Database) {}

    async execute(work: (uow: IUnitOfWork) => Promise<any>): Promise<any> {
        const transaction = await this.db.beginTransaction();
        try {
            const result = await work({
                orders: new OrderRepository(transaction),
                orderItems: new OrderItemRepository(transaction)
            });
            await transaction.commit();
            return result;
        } catch (error) {
            await transaction.rollback();
            throw error;
        }
    }
}
```
**Benefits**:
- **Atomicity**: All changes succeed or all fail
- **Consistency**: Maintains data integrity
- **Isolation**: Changes isolated until commit

**Adapter Pattern**: External service abstraction
```typescript
// Problem: Direct external service coupling
class NotificationService {
    async sendEmail(email: Email) {
        // Direct coupling to email service
        return await sendgrid.send(email);
    }
}

// Solution: Adapter pattern
interface IEmailService {
    sendEmail(email: Email): Promise<void>;
}

export class SendGridAdapter implements IEmailService {
    async sendEmail(email: Email): Promise<void> {
        await this.sendgridClient.send({
            to: email.to,
            from: email.from,
            subject: email.subject,
            html: email.body
        });
    }
}

export class SESAdapter implements IEmailService {
    async sendEmail(email: Email): Promise<void> {
        await this.sesClient.sendEmail({
            Destination: { ToAddresses: [email.to] },
            Message: {
                Subject: { Data: email.subject },
                Body: { Html: { Data: email.body } }
            }
        });
    }
}
```
**Benefits**:
- **Technology Independence**: Can switch email providers
- **Testability**: Easy to mock external services
- **Interface Segregation**: Clean contracts between systems

### Cross-Cutting Concerns Patterns

**Aspect-Oriented Programming**: Separating cross-cutting concerns
```typescript
// Problem: Scattered logging
class ProductService {
    async createProduct(data) {
        console.log('Creating product:', data);
        try {
            const result = await this.repository.create(data);
            console.log('Product created successfully');
            return result;
        } catch (error) {
            console.error('Product creation failed:', error);
            throw error;
        }
    }
}

// Solution: AOP with decorators
@LogExecutionTime()
@HandleErrors()
export class ProductService {
    async createProduct(data: CreateProductData): Promise<Product> {
        // Only business logic here
        return await this.repository.create(data);
    }
}

// Decorators handle cross-cutting concerns
function LogExecutionTime() {
    return function (target, propertyKey, descriptor) {
        const originalMethod = descriptor.value;
        descriptor.value = async function (...args) {
            const start = Date.now();
            console.log(`Starting ${propertyKey}`);
            try {
                const result = await originalMethod.apply(this, args);
                console.log(`${propertyKey} completed in ${Date.now() - start}ms`);
                return result;
            } catch (error) {
                console.error(`${propertyKey} failed:`, error);
                throw error;
            }
        };
    };
}
```
**Benefits**:
- **Separation of Concerns**: Business logic separate from infrastructure
- **DRY Principle**: Common behavior defined once
- **Maintainability**: Changes to logging don't affect business logic

**Dependency Injection Pattern**: Loose coupling through injection
```typescript
// Problem: Tight coupling with new
class ProductService {
    constructor() {
        this.repository = new ProductRepository(); // Hardcoded dependency
        this.validator = new ProductValidator();
    }
}

// Solution: Dependency injection
export class ProductService {
    constructor(
        private repository: IProductRepository, // Injected interface
        private validator: IProductValidator,
        private eventPublisher: IEventPublisher
    ) {}

    async createProduct(data: CreateProductData): Promise<Product> {
        // Dependencies provided externally
        const validatedData = await this.validator.validate(data);
        const product = await this.repository.create(validatedData);
        await this.eventPublisher.publish(new ProductCreatedEvent(product));
        return product;
    }
}

// Composition root handles wiring
const productService = new ProductService(
    new DrizzleProductRepository(database),
    new JoiProductValidator(),
    new EventBusPublisher()
);
```
**Benefits**:
- **Testability**: Dependencies can be mocked
- **Flexibility**: Different implementations for different environments
- **Maintainability**: Changes to dependencies don't affect service logic

This layered approach with design patterns ensures:
- **Separation of Concerns**: Each layer and pattern has a specific responsibility
- **Testability**: Patterns enable comprehensive testing strategies
- **Maintainability**: Changes isolated through appropriate patterns
- **Scalability**: Patterns support horizontal and vertical scaling
- **Flexibility**: Business logic independent of infrastructure choices

## Code Samples & Implementation Scenarios

### Scenario 1: Product Transfer Between Categories
**Problem**: Need to safely transfer products between categories while enforcing business rules

```typescript
// From src/core/logics/v1/products/transfer/product-transfer.logic.ts
export class ProductTransferLogic implements ILogic {
  async execute() {
    const { productId, newCategoryId } = this.request.body;
    
    // 1. Validate category existence
    const newCategory = await this.categoryRepo.getById(newCategoryId);
    if (!newCategory) throw new Error(`Category ${newCategoryId} not found`);

    // 2. Check category compatibility
    if (newCategory.productType !== this.product.type) {
      throw new Error(`Category ${newCategoryId} doesn't support this product type`);
    }

    // 3. Verify category capacity
    const currentCount = await this.productRepo.getCategoryProductCount(newCategoryId);
    if (currentCount >= newCategory.maxProducts) {
      throw new Error(`Category ${newCategoryId} has reached maximum capacity`);
    }

    // 4. Execute transfer
    await this.productRepo.transferProduct(productId, newCategoryId);

    // 5. Publish domain event
    await this.eventPublisher.publish(new ProductTransferredEvent(
      productId,
      this.product.currentCategoryId,
      newCategoryId
    ));

    return { success: true };
  }
}
```

**Key Decisions**:
1. Atomic operation with multiple validation checks
2. Domain event emission for downstream processing
3. Repository pattern abstraction

### Scenario 2: Bulk Import with Transaction Handling
**Challenge**: Maintain data consistency during large imports

```typescript
// From src/application/adapters/export-import/strategies/csv-import.strategy.ts
export class CSVImportStrategy implements ImportStrategy {
  async import(data: Buffer) {
    const parsed = parseCSV(data);
    
    return this.db.multiResourceTransact([
      ['create', 'AuditLog', { action: 'IMPORT_START' }],
      ...parsed.map(record => ['create', 'Product', record]),
      ['create', 'AuditLog', { action: 'IMPORT_COMPLETE' }]
    ]);
  }
}
```

**Patterns Used**:
- Transactional batch processing
- Audit trail creation
- Strategy pattern for different formats

### Creating a New Resource (From Repository)

#### 1. Define Validation Schema
```typescript
// From src/core/schemas/resource/products/post/create-products.schema.ts
export const createProductsSchema = Joi.object({
    name: Joi.string().required().min(1).max(255),
    description: Joi.string().allow('').max(1000),
    categoryId: Joi.number().integer().positive().required(),
    isEnabled: Joi.boolean().default(true),
    metadata: Joi.object().default({})
});
```

#### 2. Create Entity Model
```typescript
// From src/core/entities/byo-dpp/Products.entity.ts
export class Products {
    constructor(
        public id: number,
        public name: string,
        public description?: string,
        public categoryId: number,
        public isEnabled: boolean = true,
        public metadata: Record<string, unknown> = {},
        public createdAt: Date,
        public updatedAt: Date
    ) {}

    // Domain methods
    enable(): void {
        this.isEnabled = true;
        this.updatedAt = new Date();
    }

    disable(): void {
        this.isEnabled = false;
        this.updatedAt = new Date();
    }

    updateName(newName: string): void {
        if (!newName || newName.trim().length === 0) {
            throw new Error('Product name cannot be empty');
        }
        this.name = newName.trim();
        this.updatedAt = new Date();
    }
}
```

#### 3. Database Schema Definition
```typescript
// From src/infra/schemas/drizzle-pg-orm/byo-dpp/products.schema.ts
export const products = pgTable('products', {
    id: serial('id').primaryKey(),
    name: varchar('name', { length: 255 }).notNull(),
    description: text('description'),
    categoryId: integer('category_id').references(() => categories.id).notNull(),
    isEnabled: boolean('is_enabled').default(true).notNull(),
    metadata: jsonb('metadata').default({}).notNull(),
    createdAt: timestamp('created_at').defaultNow().notNull(),
    updatedAt: timestamp('updated_at').defaultNow().notNull(),
});

// Relations
export const productsRelations = relations(products, ({ one, many }) => ({
    category: one(categories, {
        fields: [products.categoryId],
        references: [categories.id],
    }),
    productData: many(productData),
    productDataSources: many(productDataSources),
}));
```

#### 4. Custom Business Logic Implementation
```typescript
// From src/core/logics/v1/products/crud/products-create.logic.ts
import { IRequest } from 'src/core/interfaces';
import { ILogic } from 'src/core/interfaces/logic.interface';
import { ICoreRepository } from 'src/core/interfaces/repository.interface';
import { CoreProvider } from 'src/core/providers/core.provider';
import { ErrorType, IErrorFactory } from 'src/core/types/error.types';

export class ProductsCreateLogic implements ILogic {
    request: IRequest;
    repository: ICoreRepository;
    coreProvider: CoreProvider;
    errorFactory: IErrorFactory;

    constructor(request: IRequest, repository: ICoreRepository) {
        this.request = request;
        this.repository = repository;
        this.coreProvider = CoreProvider.getInstance();
        this.errorFactory = this.coreProvider.getErrorFactory();
    }

    async execute() {
        // Business rule: Validate category exists
        await this.validateCategoryExists();

        // Business rule: Check for duplicate product names within category
        await this.validateUniqueNameInCategory();

        // Prepare payload with defaults
        const payload = {
            ...this.request.body,
            metadata: this.request.body.metadata || {},
            isEnabled: this.request.body.isEnabled !== undefined ? this.request.body.isEnabled : true
        };

        // Create product
        const product = await this.repository.create(payload);

        // Business rule: Create audit log
        await this.createAuditLog('CREATE', product);

        // Business rule: Send notification to category owner
        await this.notifyCategoryOwner(product);

        return product;
    }

    private async validateCategoryExists() {
        const categoryRepo = this.coreProvider.getRepoDiscovery().getRepository('Categories');
        const category = await categoryRepo.read({
            filter: { id: this.request.body.categoryId }
        });

        if (!category.success) {
            throw this.errorFactory.create(
                ErrorType.BAD_REQUEST,
                `Category with ID ${this.request.body.categoryId} does not exist`
            );
        }
    }

    private async validateUniqueNameInCategory() {
        const existingProduct = await this.repository.list({
            filter: {
                name: { eq: this.request.body.name },
                categoryId: { eq: this.request.body.categoryId }
            },
            limit: 1
        });

        if (existingProduct.data && existingProduct.data.length > 0) {
            throw this.errorFactory.create(
                ErrorType.CONFLICT,
                `Product with name "${this.request.body.name}" already exists in this category`
            );
        }
    }

    private async createAuditLog(action: string, product: any) {
        const auditRepo = this.coreProvider.getRepoDiscovery().getRepository('AuditLogs');
        await auditRepo.create({
            action,
            resourceType: 'Product',
            resourceId: product.data.id,
            userId: this.request.user?.id,
            details: {
                productName: product.data.name,
                categoryId: this.request.body.categoryId
            },
            timestamp: new Date()
        });
    }

    private async notifyCategoryOwner(product: any) {
        // Implementation for sending notifications
        // This would integrate with the notification system
    }
}
```

### Event-Driven Architecture Implementation

#### Server-Sent Events Controller
```typescript
// From src/application/adapters/controller/sse.controller.ts
export class SSEController {
    constructor(
        private eventStream: IEventStream,
        private resourceName: string
    ) {}

    async handleConnection(request: IRequest, response: IResponse) {
        // Set SSE headers
        response.writeHead(200, {
            'Content-Type': 'text/event-stream',
            'Cache-Control': 'no-cache',
            'Connection': 'keep-alive',
            'Access-Control-Allow-Origin': '*',
            'Access-Control-Allow-Headers': 'Cache-Control',
        });

        // Add connection to stream
        const connectionId = `conn_${Date.now()}_${Math.random()}`;
        this.eventStream.addConnection(connectionId, response);

        // Send initial connection event
        this.eventStream.initConnection(connectionId);

        // Handle client disconnect
        request.on('close', () => {
            this.eventStream.closeConnection(connectionId);
        });
    }
}
```

#### SSE Adapter Implementation
```typescript
// From src/infra/adapters/event-stream/sse-event-stream.adapter.ts
export class SseAdapter implements IEventStream {
    private connections: Map<string, Response> = new Map();

    addConnection(connectionId: string, res: Response) {
        this.setupSseHeaders(res);
        this.connections.set(connectionId, res);
        this.sendHeartbeat(connectionId);
    }

    sendEvent<T>(eventName: string, data: T, connectionId?: string) {
        if (connectionId) {
            this.sendEventToClient(connectionId, eventName, data);
        } else {
            // Broadcast to all connections
            this.connections.forEach((res, id) => {
                this.sendEventToClient(id, eventName, data);
            });
        }
    }

    private sendEventToClient<T>(connectionId: string, eventName: string, data: T) {
        const res = this.connections.get(connectionId);
        if (res) {
            const eventData = {
                event: eventName,
                data: data,
                timestamp: Date.now(),
                id: connectionId
            };
            res.write(`data: ${JSON.stringify(eventData)}\n\n`);
        }
    }

    private sendHeartbeat(connectionId: string) {
        setInterval(() => {
            this.sendEventToClient(connectionId, 'heartbeat', { timestamp: Date.now() });
        }, 30000); // 30 seconds
    }
}
```

### Export/Import Strategy Pattern

#### CSV Export Strategy
```typescript
// From src/application/adapters/export-import/strategies/csv-export.strategy.ts
import { IExportStrategy } from 'src/core/interfaces/export-importer-strategy.interface';
import { stringify } from 'csv-stringify/sync';

export class CSVExporterStrategy implements IExportStrategy {
    export(data: Record<string, unknown>[], resource: string): string {
        if (!data || data.length === 0) {
            return '';
        }

        // Get headers from first row
        const headers = Object.keys(data[0]);

        // Convert data to CSV format
        const csvData = data.map(row =>
            headers.map(header => {
                const value = row[header];
                // Handle different data types
                if (value === null || value === undefined) {
                    return '';
                }
                if (typeof value === 'object') {
                    return JSON.stringify(value);
                }
                return String(value);
            })
        );

        // Add headers as first row
        csvData.unshift(headers);

        // Generate CSV string
        return stringify(csvData);
    }
}
```

#### JSON Export Strategy
```typescript
// From src/application/adapters/export-import/strategies/json-export.strategy.ts
import { IExportStrategy } from 'src/core/interfaces/export-importer-strategy.interface';

export class JSONExporterStrategy implements IExportStrategy {
    export(data: Record<string, unknown>[], resource: string): string {
        return JSON.stringify({
            resource,
            exportDate: new Date().toISOString(),
            count: data.length,
            data
        }, null, 2);
    }
}
```

### Authentication and Authorization

#### Authenticator Middleware
```typescript
// From src/infra/adapters/middlewares/authenticator.middleware.ts
export class AuthenticatorMiddleware implements IMiddleware {
    constructor(private identityService: IIdentityService) {}

    async execute(request: IRequest, response: IResponse, next: NextFunction) {
        try {
            const token = this.extractToken(request);
            if (!token) {
                return response.status(401).json({
                    success: false,
                    error: { statusCode: 401, message: 'No token provided' }
                });
            }

            const user = await this.identityService.verifyToken(token);
            if (!user) {
                return response.status(401).json({
                    success: false,
                    error: { statusCode: 401, message: 'Invalid token' }
                });
            }

            request.user = user;
            next();
        } catch (error) {
            response.status(401).json({
                success: false,
                error: { statusCode: 401, message: 'Authentication failed' }
            });
        }
    }

    private extractToken(request: IRequest): string | null {
        const authHeader = request.headers.authorization;
        if (authHeader && authHeader.startsWith('Bearer ')) {
            return authHeader.substring(7);
        }

        // Check query parameter as fallback
        return request.query.token as string || null;
    }
}
```

### Validation Middleware

#### Batch Validation Middleware
```typescript
// From src/infra/adapters/middlewares/batch.middleware.ts
export class BatchMiddleware implements IMiddleware {
    constructor(
        private validatorAdapter: new (schema: SchemaDefinition) => IValidator,
        private validationSchemas: ValidationSchemaMap
    ) {}

    async execute(request: IRequest, response: IResponse, next: NextFunction) {
        try {
            if (!Array.isArray(request.body)) {
                return response.status(400).json({
                    success: false,
                    error: { statusCode: 400, message: 'Batch request body must be an array' }
                });
            }

            const schema = this.validationSchemas.post;
            if (!schema) {
                return next(); // No validation schema, continue
            }

            const validator = new this.validatorAdapter(schema);

            // Validate each item in the batch
            const validationPromises = request.body.map(async (item, index) => {
                const { error, value } = await validator.validate(item);
                if (error) {
                    throw new Error(`Item ${index}: ${error.details[0].message}`);
                }
                return value;
            });

            const validatedItems = await Promise.all(validationPromises);
            request.body = validatedItems;

            next();
        } catch (error) {
            response.status(400).json({
                success: false,
                error: {
                    statusCode: 400,
                    message: `Batch validation failed: ${error.message}`
                }
            });
        }
    }
}
```

### Graph Query API Implementation

#### Complex Query Building
```typescript
// From src/infra/adapters/graph-query/drizzle-graph-query.adapter.ts
export class DrizzleGraphQueryAdapter implements IGraphAPI {
    parse(query: any): ParsedQuery {
        return {
            where: query.where ? this.buildWhereClause(query.where) : undefined,
            include: query.include ? this.buildIncludeClause(query.include) : undefined,
            orderBy: query.sort ? this.buildOrderByClause(query.sort) : undefined,
            limit: query.limit,
            offset: query.offset,
            select: query.fields
        };
    }

    private buildWhereClause(whereCondition: any) {
        return (table: any, operators: any) => {
            const conditions = [];

            for (const [field, condition] of Object.entries(whereCondition)) {
                if (typeof condition === 'object' && condition !== null) {
                    const cond = condition as any;

                    if (cond.eq !== undefined) {
                        conditions.push(operators.eq(table[field], cond.eq));
                    } else if (cond.ne !== undefined) {
                        conditions.push(operators.ne(table[field], cond.ne));
                    } else if (cond.gt !== undefined) {
                        conditions.push(operators.gt(table[field], cond.gt));
                    } else if (cond.gte !== undefined) {
                        conditions.push(operators.gte(table[field], cond.gte));
                    } else if (cond.lt !== undefined) {
                        conditions.push(operators.lt(table[field], cond.lt));
                    } else if (cond.lte !== undefined) {
                        conditions.push(operators.lte(table[field], cond.lte));
                    } else if (cond.like !== undefined) {
                        conditions.push(operators.like(table[field], `%${cond.like}%`));
                    } else if (cond.in !== undefined) {
                        conditions.push(operators.inArray(table[field], cond.in));
                    }
                }
            }

            return conditions.length > 1 ? operators.and(...conditions) : conditions[0];
        };
    }

    private buildIncludeClause(includeSpec: any) {
        // Build complex joins and relations
        return (baseQuery: any) => {
            // Implementation for building relational queries
            // This would handle nested includes like categories.subcategories.products
        };
    }
}
```

### Transaction Management

#### Multi-Resource Transactions
```typescript
// From drizzle-orm.repository.ts
async multiResourceTransact(operations: MultiResourceTransactionOperation[]): Promise<MultiResourceTransactionResult> {
    try {
        return await this.dbClient.transaction(async (tx) => {
            const results = [];

            for (const operation of operations) {
                const { resourceName, operation: op, data, where } = operation;
                const table = ResourceModelMap[resourceName as keyof typeof ResourceModelMap];

                const resolvedData = this.resolveResultStrings(data, results);
                const resolvedWhere = where ? this.resolveResultStrings(where, results) : where;

                let result;
                switch (op.toLowerCase()) {
                    case 'create':
                        result = await tx.insert(table).values(resolvedData).returning();
                        break;
                    case 'update':
                        result = await tx
                            .update(table)
                            .set(resolvedData)
                            .where(eq(table.id, Number(resolvedData.id)))
                            .returning();
                        break;
                    case 'delete':
                        result = await tx
                            .delete(table)
                            .where(eq(table.id, Number(resolvedData.id)))
                            .returning();
                        break;
                }

                results.push(result[0]);
            }

            return { success: true, data: results };
        });
    } catch (error) {
        return handleDBError(error);
    }
}

// Template string resolution for cross-resource references
private resolveResultStrings(data: any, results: any[]): any {
    if (!data || typeof data !== 'object') {
        return data;
    }

    if (Array.isArray(data)) {
        return data.map((item) => this.resolveResultStrings(item, results));
    }

    const resolved: any = {};
    for (const [key, value] of Object.entries(data)) {
        if (typeof value === 'string' && value.startsWith('{{') && value.endsWith('}}')) {
            const expression = value.slice(2, -2).trim();
            const match = expression.match(/^result\[(\d+)\]\.(.+)$/);

            if (match) {
                const resultIndex = parseInt(match[1], 10);
                const propertyPath = match[2];

                if (resultIndex < results.length) {
                    resolved[key] = this.getNestedProperty(results[resultIndex], propertyPath);
                } else {
                    throw new Error(`Template reference to result[${resultIndex}] is invalid`);
                }
            }
        } else if (typeof value === 'object' && value !== null) {
            resolved[key] = this.resolveResultStrings(value, results);
        } else {
            resolved[key] = value;
        }
    }
    return resolved;
}
```

### Configuration Management

#### Environment-Based Configuration
```typescript
// From src/config/app-config.ts
export class AppConfig {
    private static instance: AppConfig;
    private config: IAppConfigOptions;

    private constructor() {
        this.config = this.loadConfig();
    }

    static getInstance(): AppConfig {
        if (!AppConfig.instance) {
            AppConfig.instance = new AppConfig();
        }
        return AppConfig.instance;
    }

    private loadConfig(): IAppConfigOptions {
        const nodeEnv = process.env.NODE_ENV || 'dev';

        const baseConfig = {
            PORT: parseInt(process.env.PORT || '3000'),
            NODE_ENV: nodeEnv,
            LOG_LEVEL: process.env.LOG_LEVEL || 'info',
        };

        const envConfig = this.loadEnvironmentConfig(nodeEnv);
        return { ...baseConfig, ...envConfig };
    }

    private loadEnvironmentConfig(nodeEnv: string): Partial<IAppConfigOptions> {
        const configPath = path.resolve(__dirname, `../../config/${nodeEnv}.config.ts`);

        if (fs.existsSync(configPath)) {
            const envConfig = require(configPath);
            return envConfig.default || envConfig;
        }

        throw new Error(`Configuration file not found for environment: ${nodeEnv}`);
    }

    getConfig(): IAppConfigOptions {
        return this.config;
    }
}
```

These code samples demonstrate the actual implementation patterns used throughout the BYO DPP codebase, showing how the layered architecture, dependency injection, and domain-driven design principles are applied in practice.

## Diagrams

### System Architecture Diagram

```
[Client] → [API Gateway] → [Application Layer]
                    ↓
[Domain Layer] ←→ [Infrastructure Layer]
                    ↓
[Database] ←→ [External Services]
```

### Data Flow Diagram

```
Input Data → Validation → Transformation → Business Logic → Persistence → Output
     ↓            ↓            ↓            ↓            ↓            ↓
   Schema     Joi/Zod     Domain     Services     Repository    Response
Validation   Validation   Objects     Layer       Layer        Formatting
```

### Component Interaction Diagram

```
Controller → Service → Repository → Database
     ↑         ↑         ↑         ↑
   Routes   Business  Data Access  ORM
           Logic      Layer       Layer
```

## Tables and References

### HTTP Status Codes

| Code | Meaning | Usage |
|------|---------|-------|
| 200 | OK | Successful operation |
| 201 | Created | Resource created |
| 400 | Bad Request | Invalid input |
| 401 | Unauthorized | Authentication required |
| 403 | Forbidden | Insufficient permissions |
| 404 | Not Found | Resource not found |
| 409 | Conflict | Resource conflict |
| 500 | Internal Server Error | Server error |

### Database Error Codes

| Code | Description | HTTP Status |
|------|-------------|-------------|
| 23505 | Unique violation | 409 |
| 23503 | Foreign key violation | 400 |
| 23502 | Not null violation | 400 |
| 23514 | Check constraint violation | 400 |

### Environment Variables

| Variable | Description | Required |
|----------|-------------|----------|
| POSTGRES_DB_URI | Database connection string | Yes |
| KEYCLOAK_HOST_URI | Keycloak server URL | Yes |
| PORT | Application port | No (default: 3000) |
| NODE_ENV | Environment (dev/stage/prod) | No |

## Glossary

### Domain-Driven Design Terms

- **Aggregate**: A cluster of domain objects that can be treated as a single unit. An aggregate will have one of its component objects be the aggregate root.
- **Aggregate Root**: An entity that is the root of an aggregate, responsible for enforcing business rules and maintaining consistency.
- **Bounded Context**: A boundary within which a particular domain model applies. Each bounded context has its own ubiquitous language.
- **Domain Event**: An event that represents something that happened in the domain and is of interest to domain experts.
- **Entity**: An object with identity that changes over time. Entities are defined by their identity, not their attributes.
- **Value Object**: An immutable object without identity, defined only by its attributes. Examples: Money, Address, DateRange.
- **Repository**: An abstraction over data persistence that provides a collection-like interface for accessing domain objects.
- **Service**: A stateless operation that performs business logic that doesn't naturally fit within an entity or value object.
- **Domain Service**: A service that belongs to the domain layer and encapsulates domain logic that involves multiple entities.

### Architecture Terms

- **Clean Architecture**: An architectural pattern that separates concerns into layers with dependency inversion.
- **Dependency Injection**: A design pattern where dependencies are injected into objects rather than created internally.
- **Provider Pattern**: A pattern for managing dependencies and configuration through provider classes.
- **Adapter Pattern**: A pattern that allows incompatible interfaces to work together through adaptation.
- **Middleware**: Software that acts as a bridge between applications, handling communication and data transformation.

### Technical Terms

- **ORM (Object-Relational Mapping)**: A technique for converting data between incompatible type systems in object-oriented programming languages.
- **Drizzle ORM**: A TypeScript ORM for SQL databases that provides type safety and compile-time SQL validation.
- **Server-Sent Events (SSE)**: A server push technology enabling a client to receive automatic updates from a server via HTTP connection.
- **JWT (JSON Web Token)**: An open standard for securely transmitting information between parties as a JSON object.
- **Keycloak**: An open-source identity and access management solution for modern applications and services.
- **PostgreSQL**: An advanced open-source relational database with support for SQL and JSON querying.
- **Docker**: A platform for developing, shipping, and running applications in containers.
- **Nginx**: A web server that can also be used as a reverse proxy, load balancer, and HTTP cache.

### Business Terms

- **DPP (Data Processing Pipeline)**: A structured workflow for processing and transforming data according to business rules.
- **Template**: A reusable structure defining the format and validation rules for DPPs.
- **Tenant**: An organization or customer that uses the system with isolated data and configuration.
- **Resource**: A RESTful API endpoint representing a business entity (Products, Categories, Templates, etc.).
- **Batch Operation**: A single API call that performs multiple operations atomically.
- **Export/Import**: Functionality to transfer data between systems in various formats (CSV, JSON, XML).

### Development Terms

- **Logic Class**: A custom business logic implementation that overrides default CRUD behavior.
- **Validation Schema**: A Joi or Zod schema defining the structure and constraints for API request/response data.
- **Migration**: A database schema change script that can be applied incrementally to update the database.
- **Seed Data**: Initial data loaded into the database for development, testing, or production setup.
- **Health Check**: An endpoint that returns the status of application components (database, cache, external services).
- **Circuit Breaker**: A pattern that prevents cascading failures by temporarily stopping calls to a failing service.
- **Rate Limiting**: A technique to control the rate of requests sent or received by a network interface.

## Key Files for New Developers

| File | Purpose |
|------|---------|
| [`src/bootstrap.ts`](src/bootstrap.ts) | Application start‑up, DI container wiring |
| [`src/application/adapters/controller/base-resource.controller.ts`](src/application/adapters/controller/base-resource.controller.ts) | Thin controllers handling HTTP requests |
| [`src/application/adapters/service/base-resource.service.ts`](src/application/adapters/service/base-resource.service.ts) | Service layer coordinating use‑cases |
| [`src/core/logics/v1/dpps/crud/dpps-create.logic.ts`](src/core/logics/v1/dpps/crud/dpps-create.logic.ts) | Example domain logic with business rules |
| [`src/infra/adapters/repository/drizzle-orm.repository.ts`](src/infra/adapters/repository/drizzle-orm.repository.ts) | Repository abstraction over Drizzle ORM |
| [`src/infra/adapters/event-stream/sse-event-stream.adapter.ts`](src/infra/adapters/event-stream/sse-event-stream.adapter.ts) | Server‑Sent Events implementation |
| [`src/infra/middlewares/security.middleware.ts`](src/infra/middlewares/security.middleware.ts) | Security headers, rate limiting, CORS |
| [`docker-compose.yml`](docker-compose.yml) | Local development and production orchestration |
| [`Dockerfile`](Dockerfile) | Multi‑stage production image |
| [`README.md`](README.md) | Project overview and quick start |

These links open the actual source files in the repository, allowing new developers to jump directly to the implementation details they need to understand.
## Troubleshooting Handbook

### Common Issues

#### Database Connection Issues

**Problem**: Application fails to connect to PostgreSQL

**Solution**:
1. Check POSTGRES_DB_URI environment variable
2. Verify database is running: `docker ps`
3. Check database logs: `docker logs <container_id>`
4. Test connection: `psql $POSTGRES_DB_URI`

#### Authentication Failures

**Problem**: Keycloak authentication not working

**Solution**:
1. Verify Keycloak configuration in environment variables
2. Check Keycloak server status
3. Validate client credentials
4. Review JWT token expiration

#### Build Failures

**Problem**: TypeScript compilation errors

**Solution**:
1. Run `npm run format-fix` to fix formatting
2. Check for missing dependencies: `npm install`
3. Clear node_modules and reinstall: `rm -rf node_modules && npm install`
4. Check TypeScript version compatibility

### Debug Commands

```bash
# Check application logs
docker logs byo-dpp-backend

# Access database
docker exec -it byo-dpp-db psql -U postgres -d byo_dpp

# Run tests
npm run test

# Check health endpoint
curl http://localhost:3000/health
```

### Performance Tuning

- Enable database connection pooling
- Implement caching for frequently accessed data
- Use pagination for large datasets
- Monitor memory usage and garbage collection
- Optimize database queries with proper indexing

---

*This handbook is continuously updated. Please contribute improvements and corrections.*


---

## 🏢 Multi-Tenancy Implementation: Complete Architecture & Planning Guide

### Table of Contents
- [Overview & Current Implementation](#overview-current-implementation)
- [Multi-Tenancy Patterns Analysis](#multi-tenancy-patterns-analysis)
- [BYO DPP Implementation Deep Dive](#byo-dpp-implementation-deep-dive)
- [Data Isolation Strategy](#data-isolation-strategy)
- [Security & Authorization Architecture](#security-authorization-architecture)
- [Tenant Lifecycle Management](#tenant-lifecycle-management)
- [Scaling Strategy](#scaling-strategy)
- [Migration & Evolution Roadmap](#migration-evolution-roadmap)
- [Performance Optimization](#performance-optimization)
- [Monitoring & Observability](#monitoring-observability)
- [Implementation Checklist](#implementation-checklist)

---

### Overview & Current Implementation

#### What is Multi-Tenancy?

Multi-tenancy is an architectural pattern where a single application instance serves multiple customers (tenants), with each tenant's data logically isolated from others. In BYO DPP, multi-tenancy enables multiple organizations to use the Digital Product Passport platform while maintaining complete data privacy and customization capabilities.

#### Current BYO DPP Multi-Tenancy Model

**Pattern**: Hybrid Multi-Tenancy (Shared Database + Logical Isolation + Separate Identity Realms)

```mermaid
graph TB
    subgraph Platform["BYO DPP Platform - Single Application Instance"]
        App[Application Layer]
        
        subgraph Data["Data Layer"]
            DB[(Shared PostgreSQL<br/>tenant_id discriminator)]
        end
        
        subgraph Identity["Identity Layer"]
            KC[Keycloak Multi-Realm<br/>realm per tenant]
        end
        
        subgraph Authorization["Authorization Layer"]
            Casbin[Casbin ABAC<br/>tenant-scoped policies]
        end
        
        subgraph Shared["Shared Infrastructure"]
            Redis[(Redis Cache)]
            Storage[(File Storage)]
            Other[Other Services]
        end
    end
    
    subgraph TenantA["Tenant A"]
        RealmA[Keycloak Realm: tenant-a]
        DataA[Data: tenant_id = A]
        UsersA[Users & Roles]
    end
    
    subgraph TenantB["Tenant B"]
        RealmB[Keycloak Realm: tenant-b]
        DataB[Data: tenant_id = B]
        UsersB[Users & Roles]
    end
    
    subgraph TenantC["Tenant C"]
        RealmC[Keycloak Realm: tenant-c]
        DataC[Data: tenant_id = C]
        UsersC[Users & Roles]
    end
    
    App --> DB
    App --> KC
    App --> Casbin
    App --> Redis
    App --> Storage
    
    KC --> RealmA
    KC --> RealmB
    KC --> RealmC
    
    DB --> DataA
    DB --> DataB
    DB --> DataC
    
    RealmA --> UsersA
    RealmB --> UsersB
    RealmC --> UsersC
    
    style Platform fill:#e3f2fd
    style TenantA fill:#fff3e0
    style TenantB fill:#f3e5f5
    style TenantC fill:#e8f5e9
```

**Key Implementation Files:**
- [`Tenants` Schema](src/infra/schemas/drizzle-pg-orm/byo-dpp/tenants.schema.ts:39-72) - Tenant entity definition
- [`TenantClasses` Schema](src/infra/schemas/drizzle-pg-orm/byo-dpp/tenantClasses.schema.ts:5-21) - Tenant categorization
- [`TenantUsers` Schema](src/infra/schemas/drizzle-pg-orm/byo-dpp/tenantUsers.schema.ts:6-36) - User-tenant mapping
- [`TenantTemplates` Schema](src/infra/schemas/drizzle-pg-orm/byo-dpp/tenantTemplates.schema.ts:6-36) - Template assignments
- [`KeycloakService`](src/infra/lib/identity/keycloak/src/keycloak-service.adapter.ts:11-362) - Identity management
- [`CasbinService`](src/infra/lib/authorization/casbin/src/casbin-service.ts:8-36) - Authorization enforcement
- [`Policies` Schema](src/infra/schemas/drizzle-pg-orm/byo-dpp/policies.schema.ts:7-30) - Tenant-specific policies

---

### Multi-Tenancy Patterns Analysis

#### Pattern 1: Separate Databases (Database-per-Tenant)

**Architecture:**
```
Application Layer
    ├── Tenant A → Database A
    ├── Tenant B → Database B
    └── Tenant C → Database C
```

**Pros:**
- ✅ **Strongest data isolation** - Physical separation
- ✅ **Easy to backup/restore per tenant**
- ✅ **Compliance-friendly** - GDPR, HIPAA requirements
- ✅ **Performance isolation** - One tenant can't affect others
- ✅ **Easy to customize schema per tenant**
- ✅ **Simple tenant migration** - Just move database

**Cons:**
- ❌ **High infrastructure cost** - N databases for N tenants
- ❌ **Complex management** - Multiple schema versions
- ❌ **Difficult cross-tenant analytics**
- ❌ **Resource waste** - Each DB has overhead
- ❌ **Scaling complexity** - Must provision per tenant

**When to Use:**
- Enterprise customers with strict compliance
- High-value customers willing to pay premium
- Regulatory requirements mandate physical isolation
- Tenants need complete schema customization

**Cost Example:**
```
10 Tenants × $50/month (managed DB) = $500/month
vs
1 Shared DB × $100/month = $100/month
```

---

#### Pattern 2: Shared Database with Tenant Discriminator (BYO DPP Current)

**Architecture:**
```
Single Database
    └── All Tables with tenant_id column
        ├── WHERE tenant_id = 'A' → Tenant A data
        ├── WHERE tenant_id = 'B' → Tenant B data
        └── WHERE tenant_id = 'C' → Tenant C data
```

**BYO DPP Implementation Example:**
```sql
-- Tenants table with hierarchical support
CREATE TABLE "Tenants" (
    id SERIAL PRIMARY KEY,
    tenant_id INTEGER REFERENCES "Tenants"(id),  -- Self-referential for hierarchy
    industry_id INTEGER NOT NULL REFERENCES "Industries"(id),
    tenant_class_id INTEGER NOT NULL REFERENCES "TenantClasses"(id),
    name VARCHAR(128) NOT NULL,
    ulid VARCHAR(32) UNIQUE NOT NULL,
    type tenants_type NOT NULL,
    status tenants_status NOT NULL DEFAULT 'invited',
    metadata JSON,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Roles table (tenant-scoped)
CREATE TABLE "Roles" (
    id SERIAL PRIMARY KEY,
    tenant_id INTEGER NOT NULL REFERENCES "Tenants"(id) ON DELETE CASCADE,
    name VARCHAR(32) NOT NULL
);

-- Policies table (tenant-scoped)
CREATE TABLE "Policies" (
    id SERIAL PRIMARY KEY,
    tenant_id INTEGER REFERENCES "Tenants"(id) ON DELETE CASCADE,
    ptype VARCHAR(128) NOT NULL,
    subject VARCHAR(128) NOT NULL,
    object VARCHAR(128) NOT NULL,
    action VARCHAR(128) NOT NULL,
    effect policies_effect NOT NULL DEFAULT 'allow'
);
```

**Pros:**
- ✅ **Cost-effective** - Single database for all tenants
- ✅ **Easy maintenance** - One schema to manage
- ✅ **Simple deployment** - No per-tenant provisioning
- ✅ **Cross-tenant analytics** - Query across tenants
- ✅ **Efficient resource utilization**
- ✅ **Fast onboarding** - Just create tenant record

**Cons:**
- ❌ **Risk of data leakage** - Forgot WHERE tenant_id clause
- ❌ **Performance contention** - One tenant affects others
- ❌ **Limited customization** - Shared schema
- ❌ **Backup complexity** - Can't backup one tenant easily
- ❌ **Noisy neighbor problem** - Large tenant impacts small ones

**BYO DPP Mitigation Strategies:**
1. **Row-Level Security (RLS)** - PostgreSQL policy enforcement
2. **Repository Pattern** - Automatic tenant_id injection
3. **Middleware Validation** - Verify tenant context
4. **Query Auditing** - Log all queries for compliance

**When to Use:**
- **✅ Current BYO DPP use case** - B2B SaaS with 10-1000 tenants
- Small to medium-sized tenants
- Cost-sensitive deployment
- Need fast tenant provisioning

---

#### Pattern 3: Shared Database with Schema-per-Tenant

**Architecture:**
```
Single Database
    ├── Schema tenant_a (all tables)
    ├── Schema tenant_b (all tables)
    └── Schema tenant_c (all tables)
```

**PostgreSQL Example:**
```sql
-- Create tenant schema
CREATE SCHEMA tenant_a;

-- Create tables in tenant schema
CREATE TABLE tenant_a."Products" (...);
CREATE TABLE tenant_a."Users" (...);

-- Switch schema per request
SET search_path = tenant_a;
SELECT * FROM "Products";  -- Queries tenant_a."Products"
```

**Pros:**
- ✅ **Better isolation** than tenant_id discriminator
- ✅ **Schema customization** per tenant
- ✅ **Cost-effective** - One database
- ✅ **Logical separation** - Clear boundaries
- ✅ **Easier backup** - Can dump specific schema

**Cons:**
- ❌ **Connection pooling complexity** - Must set search_path
- ❌ **Cross-tenant queries harder**
- ❌ **Schema migration complexity** - N schemas to migrate
- ❌ **Database connection overhead**

**When to Use:**
- Need stronger isolation than Pattern 2
- Can't afford Pattern 1 costs
- Tenants need schema customization
- 10-100 tenants (not thousands)

---

#### Pattern 4: Hybrid (BYO DPP Enhanced)

**Architecture:**
```
┌─────────────────────────────────────────────────────────────────┐
│                    Tenant Classification                         │
├─────────────────────────────────────────────────────────────────┤
│  Enterprise Tier      │  Professional Tier  │  Standard Tier     │
│  (Pattern 1)          │  (Pattern 3)        │  (Pattern 2)       │
│  Dedicated DB         │  Dedicated Schema   │  Shared Tables     │
├───────────────────────┼─────────────────────┼────────────────────┤
│  • Large customers    │  • Medium customers │  • Small customers │
│  • Compliance needs   │  • Custom fields    │  • Cost-sensitive  │
│  • $10K+/month        │  • $1K-10K/month    │  • <$1K/month      │
└─────────────────────────────────────────────────────────────────┘
```

**BYO DPP TenantClasses Implementation:**
```typescript
// From tenantClasses.schema.ts (lines 5-21)
export const TenantClasses = drizzle.pgTable('TenantClasses', {
    id: drizzle.serial('id').primaryKey(),
    name: drizzle.varchar('name', { length: 128 }).unique().notNull(),
    // Examples: 'enterprise', 'professional', 'standard', 'free'
    createdAt: drizzle.timestamp('created_at').notNull().defaultNow(),
});

// Usage strategy
const tenantClasses = {
    enterprise: {
        isolationPattern: 'dedicated-database',
        maxUsers: Infinity,
        customization: 'full',
        cost: 'high'
    },
    professional: {
        isolationPattern: 'dedicated-schema',
        maxUsers: 1000,
        customization: 'limited',
        cost: 'medium'
    },
    standard: {
        isolationPattern: 'shared-tables',
        maxUsers: 100,
        customization: 'none',
        cost: 'low'
    }
};
```

**Pros:**
- ✅ **Flexible** - Match pattern to tenant needs
- ✅ **Cost-optimized** - Pay for what you need
- ✅ **Revenue-aligned** - Higher tier = better isolation
- ✅ **Scalable** - Can mix patterns

**Cons:**
- ❌ **Complex implementation** - Must support multiple patterns
- ❌ **Migration complexity** - Moving between tiers
- ❌ **Testing overhead** - Test all patterns

---

### BYO DPP Implementation Deep Dive

#### Current Architecture Components

##### 1. Tenant Data Model

**Tenant Entity** ([tenants.schema.ts:39-72](src/infra/schemas/drizzle-pg-orm/byo-dpp/tenants.schema.ts:39-72)):
```typescript
export const Tenants = drizzle.pgTable('Tenants', {
    id: drizzle.serial('id').primaryKey(),
    
    // Hierarchical Support - Parent-child tenant relationships
    tenantId: drizzle.integer('tenant_id')
        .references(() => Tenants.id, { 
            onDelete: 'no action', 
            onUpdate: 'cascade' 
        }),
    
    // Classification & Categorization
    industryId: drizzle.integer('industry_id').notNull()
        .references(() => Industries.id),
    tenantClassId: drizzle.integer('tenant_class_id').notNull()
        .references(() => TenantClasses.id),
    
    // Identity & Branding
    name: drizzle.varchar('name', { length: 128 }).notNull(),
    domain: drizzle.varchar('domain', { length: 128 }),
    ulid: drizzle.varchar('ulid', { length: 32 }).unique().notNull(),
    brandColor: drizzle.json('brand_color'),
    logo: drizzle.json('logo'),
    
    // Business Details
    address: drizzle.text('address').notNull(),
    type: TenantsTypeEnum('type').notNull(),  // 11 types
    tenantSize: TenantsTenantSizeEnum('tenant_size').notNull(),
    status: TenantsStatusEnum('status').notNull().default('invited'),
    
    // Extensibility
    metadata: drizzle.json('metadata'),
});
```

**Tenant Types** (Lines 10-22):
```typescript
export const TenantsTypeEnum = drizzle.pgEnum('tenants_type', [
    'kam',           // Key Account Manager
    'manufacturer',  // Product manufacturers
    'supplier',      // Raw material suppliers
    'importer',      // Import businesses
    'distributor',   // Distribution network
    'dealer',        // Retail dealers
    'retailer',      // End retailers
    'collector',     // Waste collectors
    'sorter',        // Recycling sorters
    'refurbisher',   // Product refurbishment
    'others'         // Catch-all
]);
```

**Tenant Lifecycle States** (Lines 31-37):
```typescript
export const TenantsStatusEnum = drizzle.pgEnum('tenants_status', [
    'invited',      // Initial invitation sent
    'accepted',     // Invitation accepted, pending setup
    'rejected',     // Declined invitation
    'activated',    // Active and operational
    'deactivated'   // Suspended or terminated
]);
```

##### 2. Tenant-User Mapping

**TenantUsers Junction** ([tenantUsers.schema.ts:6-36](src/infra/schemas/drizzle-pg-orm/byo-dpp/tenantUsers.schema.ts:6-36)):
```typescript
export const TenantUsers = drizzle.pgTable('TenantUsers', {
    id: drizzle.serial('id').primaryKey(),
    tenantId: drizzle.integer('tenant_id').notNull()
        .references(() => Tenants.id, { 
            onDelete: 'cascade',  // Delete users when tenant removed
            onUpdate: 'cascade' 
        }),
    userId: drizzle.integer('user_id').notNull()
        .references(() => Users.id, { 
            onDelete: 'cascade',  // Remove mapping when user deleted
            onUpdate: 'cascade' 
        }),
}, (t) => [
    drizzle.unique().on(t.tenantId, t.userId)  // One user per tenant
]);
```

**Multi-Tenant User Support:**
- ✅ User can belong to multiple tenants
- ✅ Each tenant-user relationship is tracked
- ✅ Cascade deletes maintain referential integrity

##### 3. Tenant-Specific Roles

**Roles Table** ([roles.schema.ts:6-30](src/infra/schemas/drizzle-pg-orm/byo-dpp/roles.schema.ts:6-30)):
```typescript
export const Roles = drizzle.pgTable('Roles', {
    id: drizzle.serial('id').primaryKey(),
    tenantId: drizzle.integer('tenant_id').notNull()
        .references(() => Tenants.id, { 
            onDelete: 'cascade',  // Delete roles when tenant removed
            onUpdate: 'cascade' 
        }),
    name: drizzle.varchar('name', { length: 32 }).notNull(),
}, (t) => [
    drizzle.index('Roles_name_index').on(t.name)
]);
```

**Role Strategy:**
- Roles are **tenant-specific** (not global)
- Same role name can exist across tenants with different permissions
- Example: "Admin" in Tenant A ≠ "Admin" in Tenant B

##### 4. Tenant-Scoped Policies

**Policies Table** ([policies.schema.ts:7-30](src/infra/schemas/drizzle-pg-orm/byo-dpp/policies.schema.ts:7-30)):
```typescript
export const Policies = drizzle.pgTable('Policies', {
    id: drizzle.serial('id').primaryKey(),
    ptype: drizzle.varchar('ptype', { length: 128 }).notNull(),
    subject: drizzle.varchar('subject', { length: 128 }).notNull(),
    object: drizzle.varchar('object', { length: 128 }).notNull(),
    action: drizzle.varchar('action', { length: 128 }).notNull(),
    tenantId: drizzle.integer('tenant_id')
        .references(() => Tenants.id, { 
            onDelete: 'cascade',  // Remove policies when tenant removed
            onUpdate: 'cascade' 
        }),
    effect: PoliciesEffectEnum('effect').notNull().default('allow'),
    condition: drizzle.varchar('condition', { length: 256 }).notNull(),
});
```

**Casbin Enforcement** ([casbin-service.ts:27-31](src/infra/lib/authorization/casbin/src/casbin-service.ts:27-31)):
```typescript
public async evaluate(subject: any, resource: any, action: string): Promise<boolean> {
    console.log('tenantId id:', subject.tenantId);
    // Pass tenantId as domain context for tenant-scoped policies
    const [allowed] = await this.fullEnforcer!.enforceEx(
        subject, 
        resource, 
        action, 
        String(subject.tenantId)  // ← Tenant context
    );
    return allowed;
}
```

##### 5. Identity Management (Keycloak Multi-Realm)

**Realm-per-Tenant Strategy** ([keycloak-service.adapter.ts:64-76](src/infra/lib/identity/keycloak/src/keycloak-service.adapter.ts:64-76)):
```typescript
async addTenant(
    tenantId: string,           // Unique tenant identifier (ULID)
    displayName: string,        // Human-readable name
    roles: string[],            // Initial roles for tenant
    frontendRedirectURI: string[],  // OAuth redirect URIs
    frontendOrigin: string[]    // CORS allowed origins
) {
    await this.authenticateAdminClient();
    const realmExists = await this.kcAdminClient.realms.findOne({ realm: tenantId });
    if (!realmExists) {
        await this.addRealm(tenantId, displayName, roles, frontendRedirectURI, frontendOrigin);
    }
}
```

**Realm Configuration** ([keycloak-service.adapter.ts:243-267](src/infra/lib/identity/keycloak/src/keycloak-service.adapter.ts:243-267)):
```typescript
async addRealm(tenantId: string, displayName: string, roles: string[], ...) {
    // 1. Create Keycloak realm for tenant
    await this.createTenant(tenantId, displayName);
    
    // 2. Apply custom theme (optional branding)
    await this.setTheme(tenantId, this.config.accountTheme);
    
    // 3. Create OAuth client for frontend
    await this.createClient(
        tenantId,
        this.config.frontendClientId,
        this.config.frontendClientSecret,
        frontendRedirectURI,
        frontendOrigin,
        true,
        { directAccessGrantsEnabled: true }
    );
    
    // 4. Configure email service for tenant
    await this.configureEmail(tenantId, this.config.emailConfig || {});
    
    // 5. Create initial roles
    for (const role of roles) {
        await this.createRole(tenantId, role.toString());
    }
}
```

**Realm Security Settings** ([keycloak-service.adapter.ts:211-241](src/infra/lib/identity/keycloak/src/keycloak-service.adapter.ts:211-241)):
```typescript
private async createTenant(tenant: string, displayName?: string) {
    const realmRepresentation: RealmRepresentation = {
        realm: tenant,
        enabled: true,
        displayName: displayName,
        
        // Password Security
        resetPasswordAllowed: true,
        passwordPolicy: 'notUsername and length(12) and passwordHistory(10)',
        
        // Brute Force Protection
        bruteForceProtected: true,
        maxFailureWaitSeconds: 900,        // 15 minutes lockout
        permanentLockout: true,
        failureFactor: 3,                  // Lock after 3 failures
        maxTemporaryLockouts: 3,
        
        // Session Management
        accessTokenLifespan: 3600,         // 1 hour
        ssoSessionIdleTimeout: 1209600,    // 14 days
        ssoSessionMaxLifespan: 1209600,    // 14 days
        offlineSessionIdleTimeout: 1209600,
        offlineSessionMaxLifespan: 1209600,
        
        // Login Configuration
        loginWithEmailAllowed: true,
        duplicateEmailsAllowed: false,
        loginTheme: this.config.accountTheme,
    };
    await this.kcAdminClient.realms.create(realmRepresentation);
}
```

**User Authentication Flow** ([keycloak-service.adapter.ts:25-62](src/infra/lib/identity/keycloak/src/keycloak-service.adapter.ts:25-62)):
```typescript
async getAuthenticatedUser(token: string): Promise<IRequestUser | null> {
    try {
        // 1. Decode JWT to get issuer
        const payload = decodeJwt(token);
        const issuer = payload.iss ?? '';
        
        // 2. Extract tenant (realm) from issuer URL
        // Format: https://keycloak.example.com/realms/{tenantId}
        const jwksUri = `${this.config.baseURL}/realms/${this.getRealm(issuer)}/protocol/openid-connect/certs`;
        const jwks = this.getJwks(jwksUri);
        
        // 3. Verify JWT signature
        const { payload: verifiedPayload } = await jwtVerify(token, jwks, {
            issuer,
            algorithms: ['RS256']
        });
        
        // 4. Extract user and tenant information
        return {
            id: verifiedPayload.sub ?? '',
            username: verifiedPayload.preferred_username as string,
            fullname: verifiedPayload.name as string,
            email: verifiedPayload.email as string,
            emailVerified: verifiedPayload.email_verified as boolean,
            issuer: verifiedPayload.iss ?? '',
            expiresAt: verifiedPayload.exp ?? 0,
            isAuthenticated: true,
            
            // ← Extract tenantId from issuer URL
            tenantId: issuer.split('/').pop() ?? '',
            
            // ← Extract roles from JWT claims
            roles: (verifiedPayload.realm_access as { roles: string[] })?.roles || []
        };
    } catch (err) {
        console.error('Failed to fetch user from token:', err);
        return null;
    }
}
```

##### 6. Tenant Onboarding Flow

**Complete Tenant Provisioning:**
```typescript
// Step 1: Create tenant record
const tenant = await tenantsRepository.create({
    name: "Acme Manufacturing",
    industryId: 5,  // Manufacturing
    tenantClassId: 2,  // Professional tier
    type: "manufacturer",
    tenantSize: "hundred_to_five_hundred",
    status: "invited",
    ulid: generateULID(),  // Unique identifier
    address: "123 Industrial Ave",
    metadata: {
        customFields: true,
        maxUsers: 500,
        features: ["dpp", "supply-chain", "analytics"]
    }
});

// Step 2: Create Keycloak realm
await identityService.addTenant(
    tenant.ulid,  // Use ULID as realm ID
    tenant.name,
    ["Admin", "Manager", "Viewer"],  // Initial roles
    [`https://${tenant.domain}/auth/callback`],  // Redirect URIs
    [`https://${tenant.domain}`]  // CORS origins
);

// Step 3: Create admin user
const adminUserId = await identityService.addUser(
    generateULID(),
    tenant.ulid,  // Realm/tenant
    "John",
    "Doe",
    "admin@acme.com",
    ["Admin"]  // Assign admin role
);

// Step 4: Send onboarding email
await identityService.sendOnBoardEmail(
    tenant.ulid,
    adminUserId,
    86400,  // 24 hours to complete onboarding
    `https://${tenant.domain}/complete-setup`
);

// Step 5: Create tenant-user mapping
await tenantUsersRepository.create({
    tenantId: tenant.id,
    userId: adminUserId
});

// Step 6: Assign default templates
await tenantTemplatesRepository.batchCreate([
    { tenantId: tenant.id, templateId: 1 },  // Product DPP
    { tenantId: tenant.id, templateId: 2 }   // Supply Chain DPP
]);

// Step 7: Initialize default policies
await policiesRepository.batchCreate([
    {
        tenantId: tenant.id,
        ptype: 'p',
        subject: 'Admin',
        object: '*',
        action: '*',
        effect: 'allow',
        condition: 'tenant_id == context.tenant_id'
    }
]);
```

---

### Data Isolation Strategy

#### Level 1: Application-Level Isolation (Current Implementation)

**Automatic Tenant Context Injection:**

1. **Authentication Middleware** extracts tenant from JWT:
```typescript
// From AuthenticatorMiddleware
const user = await identityService.getAuthenticatedUser(token);
// user.tenantId is extracted from JWT issuer
request.user = user;  // Attach to request context
```

2. **Repository Pattern** automatically adds tenant filter:
```typescript
// Pseudo-code for repository with tenant isolation
class BaseRepository {
    private getTenantFilter(user: IRequestUser): object {
        return { tenantId: user.tenantId };
    }
    
    async findAll(user: IRequestUser) {
        const tenantFilter = this.getTenantFilter(user);
        // Always add tenant filter
        return this.db.query(this.table)
            .where(tenantFilter)
            .execute();
    }
    
    async create(user: IRequestUser, data: any) {
        // Automatically inject tenantId
        return this.db.insert(this.table)
            .values({
                ...data,
                tenantId: user.tenantId  // ← Force tenant context
            })
            .execute();
    }
}
```

**Protection Mechanisms:**
- ✅ Tenant ID extracted from verified JWT (can't be forged)
- ✅ All queries automatically scoped to tenant
- ✅ Cross-tenant queries blocked at application layer

**Risks:**
- ❌ Developer error: Forgot to add tenant filter
- ❌ Raw SQL queries bypass application logic
- ❌ Batch operations might miss tenant check

---

#### Level 2: Database-Level Isolation (Recommended Addition)

**PostgreSQL Row-Level Security (RLS):**

```sql
-- Enable RLS on all multi-tenant tables
ALTER TABLE "Products" ENABLE ROW LEVEL SECURITY;
ALTER TABLE "Users" ENABLE ROW LEVEL SECURITY;
ALTER TABLE "Roles" ENABLE ROW LEVEL SECURITY;
ALTER TABLE "Policies" ENABLE ROW LEVEL SECURITY;

-- Create RLS policy: Users can only see their tenant's data
CREATE POLICY tenant_isolation_policy ON "Products"
    FOR ALL
    TO PUBLIC
    USING (tenant_id = current_setting('app.current_tenant_id')::INTEGER);

-- Apply to all tenant-scoped tables
CREATE POLICY tenant_isolation_policy ON "Users"
    FOR ALL TO PUBLIC
    USING (tenant_id = current_setting('app.current_tenant_id')::INTEGER);

-- Set tenant context per connection
SET app.current_tenant_id = '123';  -- From JWT
SELECT * FROM "Products";  -- Only sees tenant 123's products
```

**Implementation in BYO DPP:**
```typescript
// Add to database connection setup
class DrizzleORMRepository {
    async setTenantContext(tenantId: number) {
        await this.db.execute(
            sql`SET LOCAL app.current_tenant_id = ${tenantId}`
        );
    }
    
    async findAll() {
        // Set tenant context before query
        await this.setTenantContext(this.user.tenantId);
        
        // RLS automatically filters
        return this.db.select().from(this.table);
    }
}
```

**Benefits:**
- ✅ **Defense in depth** - Even if application logic fails, DB blocks cross-tenant access
- ✅ **SQL injection protection** - Can't bypass with raw queries
- ✅ **Audit compliance** - Database enforces isolation

---

#### Level 3: Network-Level Isolation (Future Enhancement)

**Private VPC per Tenant


**Tenant-Specific VPCs (Enterprise Tier):**
```
┌─────────────────────────────────────────────────────────────┐
│                     Cloud Infrastructure                     │
├─────────────────────────────────────────────────────────────┤
│  VPC A (Tenant A - Enterprise)                              │
│  ├── Private Subnet (Database, App)                         │
│  ├── Public Subnet (Load Balancer)                          │
│  └── VPN Gateway (Corporate network access)                 │
├─────────────────────────────────────────────────────────────┤
│  Shared VPC (Professional & Standard Tenants)               │
│  ├── Shared Database (with RLS)                             │
│  ├── Shared Application Cluster                             │
│  └── Security Groups per tenant                             │
└─────────────────────────────────────────────────────────────┘
```

**Benefits:**
- ✅ **Network-level isolation** for sensitive data
- ✅ **Custom firewall rules** per tenant
- ✅ **Private connectivity** to tenant's infrastructure
- ✅ **Compliance support** for regulated industries

---

### Security & Authorization Architecture

#### Identity & Access Management (IAM) Strategy

**1. Multi-Realm Architecture (Keycloak)**

```
┌─────────────────────────────────────────────────────────────┐
│              Keycloak Multi-Realm Architecture               │
├─────────────────────────────────────────────────────────────┤
│  Master Realm (Admin Management)                            │
│  ├── Super Admin Users                                      │
│  ├── Tenant Management Roles                                │
│  └── Cross-Realm Administration                             │
├─────────────────────────────────────────────────────────────┤
│  Tenant A Realm (tenant-a)                                  │
│  ├── Users: admin@tenant-a.com, user1@tenant-a.com          │
│  ├── Roles: Admin, Manager, Viewer                          │
│  ├── Groups: Engineering, Sales, Support                    │
│  ├── Clients: frontend-app, mobile-app, api-gateway         │
│  └── Policies: Password, MFA, Session timeout               │
├─────────────────────────────────────────────────────────────┤
│  Tenant B Realm (tenant-b)                                  │
│  ├── Users: admin@tenant-b.com, user1@tenant-b.com          │
│  ├── Roles: Owner, Employee, Partner                        │
│  └── Custom configurations                                  │
└─────────────────────────────────────────────────────────────┘
```

**Realm Isolation Benefits:**
- ✅ **User namespace isolation** - Same email in different realms
- ✅ **Custom authentication** - MFA, SSO, SAML per tenant
- ✅ **Branding** - Custom login themes per tenant
- ✅ **Password policies** - Different requirements per tenant
- ✅ **Session management** - Tenant-specific timeout rules

**JWT Token Structure:**
```json
{
  "iss": "https://keycloak.example.com/realms/tenant-abc123",
  "sub": "user-uuid-123",
  "preferred_username": "john.doe@acme.com",
  "email": "john.doe@acme.com",
  "realm_access": {
    "roles": ["Admin", "Manager"]
  },
  "tenant_id": "tenant-abc123",
  "exp": 1735262400
}
```

**Token Validation Flow:**
```typescript
// From keycloak-service.adapter.ts
async getAuthenticatedUser(token: string): Promise<IRequestUser | null> {
    // 1. Extract issuer (contains tenant/realm)
    const payload = decodeJwt(token);
    const issuer = payload.iss ?? '';
    
    // 2. Get tenant-specific JWKS for verification
    const realm = this.getRealm(issuer);  // Extract from URL path
    const jwksUri = `${baseURL}/realms/${realm}/protocol/openid-connect/certs`;
    
    // 3. Verify signature with tenant's public key
    const { payload: verified } = await jwtVerify(token, jwks, { issuer });
    
    // 4. Return user with tenant context
    return {
        id: verified.sub,
        tenantId: realm,  // ← Extracted from verified JWT
        roles: verified.realm_access.roles,
        // ... other fields
    };
}
```

---

#### Authorization Architecture (Casbin ABAC)

**Attribute-Based Access Control (ABAC) with Tenant Context:**

```
┌─────────────────────────────────────────────────────────────┐
│                 Casbin ABAC Authorization                    │
├─────────────────────────────────────────────────────────────┤
│  Policy Format:                                             │
│  p, {subject}, {object}, {action}, {tenant}, {effect}       │
├─────────────────────────────────────────────────────────────┤
│  Examples:                                                  │
│  p, Admin, Products, read, tenant-123, allow               │
│  p, Admin, Products, write, tenant-123, allow              │
│  p, Manager, Products, read, tenant-123, allow             │
│  p, Viewer, Products, read, tenant-123, allow              │
│  p, Viewer, Products, write, tenant-123, deny              │
└─────────────────────────────────────────────────────────────┘
```

**ABAC Model Configuration** ([abac_model.conf](src/infra/lib/authorization/casbin/abac_model.conf)):
```ini
[request_definition]
r = sub, obj, act, dom

[policy_definition]
p = sub, obj, act, dom, eft

[policy_effect]
e = some(where (p.eft == allow)) && !some(where (p.eft == deny))

[matchers]
m = r.sub == p.sub && r.obj == p.obj && r.act == p.act && r.dom == p.dom
```

**Enforcement with Tenant Context** ([casbin-service.ts:27-31](src/infra/lib/authorization/casbin/src/casbin-service.ts:27-31)):
```typescript
public async evaluate(
    subject: any,      // User with roles
    resource: any,     // Resource being accessed
    action: string,    // Operation: read, write, delete
): Promise<boolean> {
    console.log('tenantId id:', subject.tenantId);
    
    // Pass tenantId as domain parameter for tenant-scoped evaluation
    const [allowed] = await this.fullEnforcer!.enforceEx(
        subject,           // Role or user
        resource,          // Resource name
        action,            // Action to perform
        String(subject.tenantId)  // ← Tenant domain context
    );
    
    return allowed;
}
```

**Policy Storage** ([policies.schema.ts:7-30](src/infra/schemas/drizzle-pg-orm/byo-dpp/policies.schema.ts:7-30)):
```typescript
export const Policies = drizzle.pgTable('Policies', {
    id: drizzle.serial('id').primaryKey(),
    ptype: drizzle.varchar('ptype', { length: 128 }).notNull(),     // 'p' for policy
    subject: drizzle.varchar('subject', { length: 128 }).notNull(), // Role/User
    object: drizzle.varchar('object', { length: 128 }).notNull(),   // Resource
    action: drizzle.varchar('action', { length: 128 }).notNull(),   // Operation
    tenantId: drizzle.integer('tenant_id')
        .references(() => Tenants.id, { onDelete: 'cascade' }),
    effect: PoliciesEffectEnum('effect').notNull().default('allow'), // allow/deny
    condition: drizzle.varchar('condition', { length: 256 }).notNull(),
});
```

**Authorization Flow in Request:**
```
HTTP Request → AuthenticatorMiddleware → Controller → Service
                      ↓
              Extract tenantId from JWT
                      ↓
              Set request.user.tenantId
                      ↓
              CasbinService.evaluate(user, resource, action)
                      ↓
              Check policies WHERE tenant_id = user.tenantId
                      ↓
              Allow/Deny based on ABAC rules
```

---

#### Security Best Practices

**1. Token Security:**
```typescript
// Implement token revocation
class TokenRevocationService {
    private revokedTokens: Set<string> = new Set();
    
    async revokeToken(jti: string) {
        this.revokedTokens.add(jti);
        // Also store in Redis with TTL
        await redis.setex(`revoked:${jti}`, 3600, 'true');
    }
    
    async isTokenRevoked(jti: string): Promise<boolean> {
        return this.revokedTokens.has(jti) || 
               await redis.exists(`revoked:${jti}`);
    }
}
```

**2. Tenant Context Validation:**
```typescript
// Always validate tenant context in middleware
class TenantValidationMiddleware implements IMiddleware {
    async execute(request: IRequest, response: IResponse): Promise<void> {
        const user = request.user;
        const tenantInPath = request.params.tenantId;
        
        // Ensure user can only access their tenant
        if (user.tenantId !== tenantInPath) {
            throw new ForbiddenError('Access denied to different tenant');
        }
    }
}
```

**3. Audit Logging:**
```typescript
// Log all tenant-scoped operations
interface AuditLog {
    tenantId: string;
    userId: string;
    action: string;
    resource: string;
    timestamp: Date;
    ipAddress: string;
    userAgent: string;
    result: 'success' | 'failure';
    details: object;
}

async function auditLog(log: AuditLog) {
    await auditLogsRepository.create({
        ...log,
        // Store in tenant-scoped table
    });
}
```

**4. Rate Limiting per Tenant:**
```typescript
class TenantRateLimiter {
    async checkLimit(tenantId: string, endpoint: string): Promise<boolean> {
        const key = `ratelimit:${tenantId}:${endpoint}`;
        const count = await redis.incr(key);
        
        if (count === 1) {
            await redis.expire(key, 60); // 1 minute window
        }
        
        const limit = await this.getTenantLimit(tenantId);
        return count <= limit;
    }
    
    async getTenantLimit(tenantId: string): Promise<number> {
        const tenant = await tenantsRepository.findById(tenantId);
        // Different limits per tenant class
        return tenant.tenantClass.rateLimit || 100;
    }
}
```

---

### Tenant Lifecycle Management
```mermaid
stateDiagram-v2
    [*] --> Invited: Create Tenant Record
    
    Invited --> Accepted: User Accepts Invitation
    Invited --> Rejected: User Declines
    
    Accepted --> Activated: Complete Onboarding
    note right of Activated
        - Keycloak realm created
        - Admin user provisioned
        - Default templates assigned
        - Policies initialized
    end note
    
    Activated --> Deactivated: Suspend Account
    note left of Deactivated
        Reasons:
        - Payment failure
        - Policy violation
        - Admin suspension
        - Security breach
    end note
    
    Deactivated --> Activated: Reactivate
    note right of Activated
        - Issue resolved
        - Payment received
        - Security verified
    end note
    
    Activated --> Offboarding: Initiate Deletion
    Deactivated --> Offboarding: Permanent Remove
    
    Offboarding --> [*]: Deletion Complete
    note left of Offboarding
        Process:
        1. Export all data
        2. Delete Keycloak realm
        3. Remove database records
        4. Clean file storage
        5. Archive logs
    end note
    
    Rejected --> [*]: End
```


#### Phase 1: Tenant Onboarding

**Step-by-Step Provisioning:**

```typescript
interface TenantOnboardingConfig {
    companyInfo: {
        name: string;
        industry: string;
        size: string;
        address: string;
    };
    adminUser: {
        firstName: string;
        lastName: string;
        email: string;
        phone: string;
    };
    subscription: {
        tier: 'enterprise' | 'professional' | 'standard';
        features: string[];
    };
    customization: {
        logo?: string;
        brandColor?: string;
        domain?: string;
    };
}

async function onboardTenant(config: TenantOnboardingConfig) {
    // 1. Create tenant record
    const tenant = await createTenantRecord(config);
    
    // 2. Provision identity realm
    await provisionKeycloakRealm(tenant);
    
    // 3. Create admin user
    const admin = await createAdminUser(tenant, config.adminUser);
    
    // 4. Initialize default data
    await initializeTenantData(tenant);
    
    // 5. Send welcome email
    await sendOnboardingEmail(admin, tenant);
    
    // 6. Notify platform admin
    await notifyPlatformAdmin('New tenant onboarded', tenant);
    
    return tenant;
}
```

**Database Initialization:**
```typescript
async function initializeTenantData(tenant: Tenant) {
    // 1. Create default roles
    const defaultRoles = ['Admin', 'Manager', 'Viewer'];
    for (const roleName of defaultRoles) {
        await rolesRepository.create({
            tenantId: tenant.id,
            name: roleName
        });
    }
    
    // 2. Assign default templates
    const templates = await getDefaultTemplates(tenant.industryId);
    for (const template of templates) {
        await tenantTemplatesRepository.create({
            tenantId: tenant.id,
            templateId: template.id
        });
    }
    
    // 3. Create default policies
    await createDefaultPolicies(tenant.id);
    
    // 4. Initialize settings
    await settingsRepository.create({
        tenantId: tenant.id,
        emailNotifications: true,
        twoFactorAuth: false,
        sessionTimeout: 3600,
        // ... other settings
    });
}
```

---

#### Phase 2: Tenant Operations

**User Management:**
```typescript
// Add user to tenant
async function addUserToTenant(
    tenantId: string,
    userInfo: UserInfo,
    roles: string[]
) {
    // 1. Create user in Keycloak realm
    const userId = await identityService.addUser(
        generateULID(),
        tenantId,
        userInfo.firstName,
        userInfo.lastName,
        userInfo.email,
        roles
    );
    
    // 2. Create user record in database
    const user = await usersRepository.create({
        name: `${userInfo.firstName} ${userInfo.lastName}`,
        userName: userInfo.email,
        email: userInfo.email,
        phone: userInfo.phone,
        status: 'invited'
    });
    
    // 3. Link user to tenant
    await tenantUsersRepository.create({
        tenantId: parseInt(tenantId),
        userId: user.id
    });
    
    // 4. Send invitation email
    await identityService.sendOnBoardEmail(
        tenantId,
        userId,
        86400,  // 24 hours
        `https://${tenant.domain}/accept-invitation`
    );
    
    return user;
}

// Update user roles
async function updateUserRoles(
    tenantId: string,
    userName: string,
    newRoles: string[]
) {
    // Sync roles in Keycloak
    await identityService.syncUserRoles(tenantId, userName, newRoles);
    
    // Update roles in database
    const user = await usersRepository.findOne({ userName });
    await userRolesRepository.deleteWhere({ userId: user.id });
    
    for (const roleName of newRoles) {
        const role = await rolesRepository.findOne({ 
            tenantId, 
            name: roleName 
        });
        await userRolesRepository.create({
            userId: user.id,
            roleId: role.id
        });
    }
}
```

**Tenant Configuration Updates:**
```typescript
async function updateTenantConfig(
    tenantId: string,
    updates: Partial<TenantConfig>
) {
    // 1. Update tenant record
    await tenantsRepository.update(tenantId, updates);
    
    // 2. Update Keycloak realm settings
    if (updates.domain || updates.brandColor) {
        await identityService.updateRealmSettings(tenantId, {
            displayName: updates.name,
            loginTheme: updates.theme,
            // ... other settings
        });
    }
    
    // 3. Trigger cache invalidation
    await cacheService.invalidate(`tenant:${tenantId}`);
    
    // 4. Notify users of changes
    await notificationService.broadcastToTenant(
        tenantId,
        'Configuration updated',
        updates
    );
}
```

---

#### Phase 3: Tenant Suspension/Deactivation

**Graceful Suspension:**
```typescript
async function suspendTenant(
    tenantId: string,
    reason: string
) {
    // 1. Update tenant status
    await tenantsRepository.update(tenantId, {
        status: 'deactivated',
        metadata: {
            suspensionReason: reason,
            suspendedAt: new Date(),
            suspendedBy: currentUser.id
        }
    });
    
    // 2. Disable Keycloak realm
    await identityService.updateRealm(tenantId, {
        enabled: false
    });
    
    // 3. Logout all users
    await identityService.logoutAllUsers(tenantId);
    
    // 4. Stop background jobs
    await jobScheduler.pauseTenantJobs(tenantId);
    
    // 5. Notify tenant admin
    await emailService.send({
        to: tenantAdmin.email,
        subject: 'Account Suspended',
        body: `Your account has been suspended. Reason: ${reason}`
    });
}
```

**Reactivation:**
```typescript
async function reactivateTenant(tenantId: string) {
    // 1. Update tenant status
    await tenantsRepository.update(tenantId, {
        status: 'activated',
        metadata: {
            reactivatedAt: new Date()
        }
    });
    
    // 2. Enable Keycloak realm
    await identityService.updateRealm(tenantId, {
        enabled: true
    });
    
    // 3. Resume background jobs
    await jobScheduler.resumeTenantJobs(tenantId);
    
    // 4. Notify tenant admin
    await emailService.send({
        to: tenantAdmin.email,
        subject: 'Account Reactivated',
        body: 'Your account has been reactivated.'
    });
}
```

---

#### Phase 4: Tenant Offboarding

**Complete Data Export:**
```typescript
async function exportTenantData(tenantId: string): Promise<string> {
    const exportPath = `/exports/tenant-${tenantId}-${Date.now()}`;
    
    // 1. Export all tenant data
    const data = {
        tenant: await tenantsRepository.findById(tenantId),
        users: await usersRepository.findAllByTenant(tenantId),
        products: await productsRepository.findAllByTenant(tenantId),
        dpps: await dppsRepository.findAllByTenant(tenantId),
        // ... all other resources
    };
    
    // 2. Create backup
    await fs.writeFile(
        `${exportPath}/data.json`,
        JSON.stringify(data, null, 2)
    );
    
    // 3. Export files/attachments
    await exportTenantFiles(tenantId, exportPath);
    
    // 4. Create archive
    await createZipArchive(exportPath);
    
    return `${exportPath}.zip`;
}
```

**Permanent Deletion:**
```typescript
async function deleteTenant(
    tenantId: string,
    confirmed: boolean = false
) {
    if (!confirmed) {
        throw new Error('Deletion must be explicitly confirmed');
    }
    
    // 1. Export data for backup
    const backupPath = await exportTenantData(tenantId);
    await uploadToArchiveStorage(backupPath);
    
    // 2. Delete Keycloak realm (cascades to users)
    await identityService.deleteRealm(tenantId);
    
    // 3. Delete tenant data (cascades to related records)
    await tenantsRepository.delete(tenantId);
    
    // 4. Clean up files/storage
    await fileStorage.deleteTenantFolder(tenantId);
    
    // 5. Clear caches
    await cacheService.invalidateAll(`tenant:${tenantId}:*`);
    
    // 6. Audit log
    await auditLog({
        action: 'TENANT_DELETED',
        tenantId,
        details: { backupPath },
        timestamp: new Date()
    });
}
```

---

### Scaling Strategy

#### Horizontal Scaling Architecture

```
                    ┌─────────────────┐
                    │  Load Balancer  │
                    └────────┬────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
   ┌────▼────┐          ┌────▼────┐         ┌────▼────┐
   │ App     │          │ App     │         │ App     │
   │ Node 1  │          │ Node 2  │         │ Node 3  │
   └────┬────┘          └────┬────┘         └────┬────┘
        │                    │                    │
        └────────────────────┼────────────────────┘
                             │
                    ┌────────▼────────┐
                    │   PostgreSQL    │
                    │  (with replicas)│
                    └─────────────────┘
```

**Session Management:**
```typescript
// Use Redis for shared session storage
class SessionManager {
    constructor(private redis: RedisClient) {}
    
    async createSession(user: IRequestUser): Promise<string> {
        const sessionId = generateULID();
        await this.redis.setex(
            `session:${sessionId}`,
            3600,  // 1 hour TTL
            JSON.stringify({
                userId: user.id,
                tenantId: user.tenantId,
                roles: user.roles
            })
        );
        return sessionId;
    }
    
    async getSession(sessionId: string): Promise<IRequestUser | null> {
        const data = await this.redis.get(`session:${sessionId}`);
        return data ? JSON.parse(data) : null;
    }
}
```

---

#### Database Scaling Strategies

**Read Replicas:**
```typescript
class DatabaseRouter {
    constructor(
        private primary: DatabaseConnection,
        private replicas: DatabaseConnection[]
    ) {}
    
    getConnection(operation: 'read' | 'write'): DatabaseConnection {
        if (operation === 'write') {
            return this.primary;
        }
        
        // Round-robin read distribution
        const index = Math.floor(Math.random() * this.replicas.length);
        return this.replicas[index];
    }
}
```

**Connection Pooling:**
```typescript
import { Pool } from 'pg';

const tenantPools = new Map<string, Pool>();

function getTenantPool(tenantId: string): Pool {
    if (!tenantPools.has(tenantId)) {
        tenantPools.set(tenantId, new Pool({
            host: process.env.DB_HOST,
            database: process.env.DB_NAME,
            max: 20,  // Max connections per tenant
            idleTimeoutMillis: 30000,
            connectionTimeoutMillis: 2000,
        }));
    }
    return tenantPools.get(tenantId)!;
}
```

**Query Optimization:**
```typescript
// Add tenant_id to all indexes
CREATE INDEX idx_products_tenant_id ON "Products"(tenant_id);
CREATE INDEX idx_products_tenant_status ON "Products"(tenant_id, status);

// Composite indexes for common queries
CREATE INDEX idx_dpps_tenant_product ON "Dpps"(tenant_id, product_id);
```

---

#### Caching Strategy

**Multi-Level Cache:**
```typescript
class TenantCache {
    constructor(
        private redis: RedisClient,
        private memoryCache: Map<string, any>
    ) {}
    
    async get(tenantId: string, key: string): Promise<any> {
        // L1: Memory cache
        const memKey = `${tenantId}:${key}`;
        if (this.memoryCache.has(memKey)) {
            return this.memoryCache.get(memKey);
        }
        
        // L2: Redis cache
        const redisKey = `tenant:${tenantId}:${key}`;
        const data = await this.redis.get(redisKey);
        if (data) {
            const parsed = JSON.parse(data);
            this.memoryCache.set(memKey, parsed);
            return parsed;
        }
        
        return null;
    }
    
    async set(tenantId: string, key: string, value: any, ttl: number = 3600) {
        const memKey = `${tenantId}:${key}`;
        const redisKey = `tenant:${tenantId}:${key}`;
        
        // Store in both caches
        this.memoryCache.set(memKey, value);
        await this.redis.setex(redisKey, ttl, JSON.stringify(value));
    }
    
    async invalidate(tenantId: string, pattern: string = '*') {
        // Clear memory cache
        for (const key of this.memoryCache.keys()) {
            if (key.startsWith(`${tenantId}:`)) {
                this.memoryCache.delete(key);
            }
        }
        
        // Clear Redis cache
        const keys = await this.redis.keys(`tenant:${tenantId}:${pattern}`);
        if (keys.length > 0) {
            await this.redis.del(...keys);
        }
    }
}
```

---

### Migration & Evolution Roadmap

#### Phase 1: Current State Assessment (Weeks 1-2)

**Tasks:**
- ✅ Document current tenant data model
- ✅ Analyze tenant distribution and growth rate
- ✅ Identify performance bottlenecks
- ✅ Review security vulnerabilities
- ✅ Audit compliance requirements

**Deliverables:**
- Current architecture documentation
- Performance baseline metrics
- Security audit report
- Tenant growth projections

---

#### Phase 2: Quick Wins & Improvements (Weeks 3-6)

**2.1 Add Row-Level Security (RLS):**
```sql
-- Week 3: Implement RLS on critical tables
ALTER TABLE "Products" ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON "Products"
    USING (tenant_id = current_setting('app.current_tenant_id')::INTEGER);

-- Week 4: Roll out to all tables
-- Scripts to automate RLS policy creation
```

**2.2 Implement Connection Pooling:**
```typescript
// Week 5: Add tenant-aware connection pooling
class TenantConnectionPool {
    private pools: Map<string, Pool> = new Map();
    
    getPool(tenantId: string): Pool {
        // Create pool per tenant with limits
    }
}
```

**2.3 Add Monitoring:**
```typescript
// Week 6: Tenant-specific metrics
metrics.recordTenantQuery(tenantId, duration, queryType);
metrics.recordTenantStorage(tenantId, sizeInBytes);
metrics.recordTenantUsers(tenantId, activeUsers);
```

---

#### Phase 3: Database Optimization (Weeks 7-10)

**3.1 Partitioning Strategy:**
```sql
-- Week 7-8: Implement table partitioning by tenant_id range
CREATE TABLE "Products" (...) PARTITION BY RANGE (tenant_id);

CREATE TABLE "Products_p1" PARTITION OF "Products"
    FOR VALUES FROM (1) TO (1000);
    
CREATE TABLE "Products_p2" PARTITION OF "Products"
    FOR VALUES FROM (1000) TO (2000);
```

**3.2 Index Optimization:**
```sql
-- Week 9: Add covering indexes
CREATE INDEX idx_products_tenant_covering 
    ON "Products"(tenant_id) 
    INCLUDE (name, status, created_at);
```

**3.3 Query Performance:**
```typescript
// Week 10: Add query optimization
- Implement query result caching
- Add slow query logging
- Optimize N+1 queries
```

---

#### Phase 4: Hybrid Multi-Tenancy (Weeks 11-16)

**4.1 Implement Tenant Classes:**
```typescript
// Week 11-12: Tier-based isolation
enum TenantIsolationStrategy {
    DEDICATED_DB = 'dedicated-database',
    DEDICATED_SCHEMA = 'dedicated-schema',
    SHARED_TABLE = 'shared-table'
}

const strategyMap = {
    enterprise: TenantIsolationStrategy.DEDICATED_DB,
    professional: TenantIsolationStrategy.DEDICATED_SCHEMA,
    standard: TenantIsolationStrategy.SHARED_TABLE
};
```

**4.2 Database Provisioning Automation:**
```typescript
// Week 13-14: Auto-provision based on tier
async function provisionTenantDatabase(tenant: Tenant) {
    const strategy = getIsolationStrategy(tenant.tenantClassId);
    
    switch (strategy) {
        case 'dedicated-database':
            await createDedicatedDatabase(tenant);
            break;
        case 'dedicated-schema':
            await createDedicatedSchema(tenant);
            break;
        case 'shared-table':
            // Already supported
            break;
    }
}
```

**4.3 Migration Tools:**
```typescript
// Week 15-16: Tenant migration between tiers
async function migrateTenant(
    tenantId: string,
    fromStrategy: string,
    toStrategy: string
) {
    // 1. Export data
    const data = await exportTenantData(tenantId);
    
    // 2. Provision new storage
    await provisionStorage(tenantId, toStrategy);
    
    // 3. Import data
    await importTenantData(tenantId, data, toStrategy);
    
    // 4. Validate
    await validateMigration(tenantId);
    
    // 5. Switch over
    await switchTenantStrategy(tenantId, toStrategy);
    
    // 6. Cleanup old storage
    await cleanupOldStorage(tenantId, fromStrategy);
}
```

---

#### Phase 5: Advanced Features (Weeks 17-24)

**5.1 Tenant Data Residency:**
```typescript
// Support data sovereignty requirements
interface TenantRegion {
    tenantId: string;
    region: 'us-east' | 'eu-west' | 'ap-south';
    dataResidency: boolean;
}

async function getTenantDatabase(tenantId: string) {
    const region = await getTenantRegion(tenantId);
    return getDatabaseForRegion(region);
}
```

**5.2 Cross-Tenant Analytics:**
```typescript
// Platform-wide analytics (anonymized)
async function getPlatformMetrics() {
    return db.query(`
        SELECT 
            tenant_class_id,
            COUNT(*) as tenant_count,
            AVG(user_count) as avg_users,
            SUM(storage_used) as total_storage
        FROM tenant_metrics
        GROUP BY tenant_class_id
    `);
}
```

**5.3 Tenant Self-Service Portal:**
```typescript
// Allow tenants to manage their own settings
class TenantPortalController
 {
    async updateSettings(tenantId: string, settings: TenantSettings) {
        // Validate user is admin
        // Update tenant settings
        // Trigger configuration updates
    }
    
    async viewUsageMetrics(tenantId: string) {
        return {
            users: await getUserCount(tenantId),
            storage: await getStorageUsage(tenantId),
            apiCalls: await getAPIUsage(tenantId),
            costs: await calculateCosts(tenantId)
        };
    }
}
```

---

### Performance Optimization

#### Query Performance Strategies

**1. Tenant-Scoped Query Optimization:**

```typescript
// BAD: Missing tenant filter leads to full table scan
const products = await db.select().from(Products).where(eq(Products.status, 'active'));

// GOOD: Always include tenant_id for index usage
const products = await db.select()
    .from(Products)
    .where(and(
        eq(Products.tenantId, user.tenantId),
        eq(Products.status, 'active')
    ));
```

**2. Index Strategy:**
```sql
-- Tenant-first composite indexes
CREATE INDEX idx_products_tenant_status 
    ON "Products"(tenant_id, status);

CREATE INDEX idx_products_tenant_created 
    ON "Products"(tenant_id, created_at DESC);

CREATE INDEX idx_dpps_tenant_product 
    ON "Dpps"(tenant_id, product_id);

-- Covering indexes for common queries
CREATE INDEX idx_products_tenant_list 
    ON "Products"(tenant_id) 
    INCLUDE (name, status, created_at);
```

**3. Connection Pool Tuning:**
```typescript
const poolConfig = {
    // Base configuration
    max: 20,  // Maximum connections per pool
    min: 2,   // Minimum idle connections
    
    // Timeouts
    idleTimeoutMillis: 30000,      // Close idle connections after 30s
    connectionTimeoutMillis: 2000,  // Fail fast on connection errors
    
    // Statement timeout
    statement_timeout: 10000,  // 10 second query timeout
    
    // Per-tenant limits
    maxPerTenant: 5,  // Prevent one tenant from hogging connections
};
```

**4. Query Result Caching:**
```typescript
class TenantQueryCache {
    async getCachedQuery(
        tenantId: string,
        queryKey: string,
        queryFn: () => Promise<any>,
        ttl: number = 300  // 5 minutes
    ): Promise<any> {
        const cacheKey = `query:${tenantId}:${queryKey}`;
        
        // Check cache
        let result = await this.cache.get(cacheKey);
        if (result) {
            metrics.incrementCounter('cache.hit', { tenant: tenantId });
            return result;
        }
        
        // Execute query
        metrics.incrementCounter('cache.miss', { tenant: tenantId });
        result = await queryFn();
        
        // Store in cache
        await this.cache.set(cacheKey, result, ttl);
        
        return result;
    }
}

// Usage
const products = await queryCache.getCachedQuery(
    user.tenantId,
    'products:list:active',
    () => productsRepository.findAll({ status: 'active' }),
    300
);
```

---

#### Resource Quotas & Throttling

**1. Tenant Resource Limits:**
```typescript
interface TenantQuotas {
    maxUsers: number;
    maxStorage: number;  // bytes
    maxAPICallsPerHour: number;
    maxConcurrentRequests: number;
    maxDatabaseConnections: number;
}

const quotasByTier: Record<string, TenantQuotas> = {
    enterprise: {
        maxUsers: Infinity,
        maxStorage: 1024 * 1024 * 1024 * 100,  // 100 GB
        maxAPICallsPerHour: 100000,
        maxConcurrentRequests: 100,
        maxDatabaseConnections: 20
    },
    professional: {
        maxUsers: 1000,
        maxStorage: 1024 * 1024 * 1024 * 10,  // 10 GB
        maxAPICallsPerHour: 10000,
        maxConcurrentRequests: 50,
        maxDatabaseConnections: 10
    },
    standard: {
        maxUsers: 100,
        maxStorage: 1024 * 1024 * 1024,  // 1 GB
        maxAPICallsPerHour: 1000,
        maxConcurrentRequests: 10,
        maxDatabaseConnections: 5
    }
};
```

**2. Quota Enforcement:**
```typescript
class QuotaEnforcer {
    async checkQuota(tenantId: string, resource: string): Promise<boolean> {
        const tenant = await this.getTenant(tenantId);
        const quotas = quotasByTier[tenant.tier];
        
        switch (resource) {
            case 'users':
                const userCount = await this.getUserCount(tenantId);
                return userCount < quotas.maxUsers;
                
            case 'storage':
                const storageUsed = await this.getStorageUsage(tenantId);
                return storageUsed < quotas.maxStorage;
                
            case 'api_calls':
                const apiCalls = await this.getAPICallsThisHour(tenantId);
                return apiCalls < quotas.maxAPICallsPerHour;
                
            default:
                return true;
        }
    }
    
    async enforceQuota(tenantId: string, resource: string): Promise<void> {
        const allowed = await this.checkQuota(tenantId, resource);
        if (!allowed) {
            throw new QuotaExceededError(
                `Quota exceeded for resource: ${resource}`
            );
        }
    }
}
```

**3. Rate Limiting Middleware:**
```typescript
class TenantRateLimitMiddleware implements IMiddleware {
    async execute(request: IRequest, response: IResponse): Promise<void> {
        const tenantId = request.user.tenantId;
        const key = `ratelimit:${tenantId}:${request.path}`;
        
        // Get current count
        const count = await redis.incr(key);
        
        // Set expiry on first request
        if (count === 1) {
            await redis.expire(key, 3600);  // 1 hour window
        }
        
        // Check limit
        const limit = await this.getTenantLimit(tenantId);
        if (count > limit) {
            throw new RateLimitExceededError(
                `Rate limit exceeded: ${count}/${limit} requests per hour`
            );
        }
        
        // Add rate limit headers
        response.setHeader('X-RateLimit-Limit', limit);
        response.setHeader('X-RateLimit-Remaining', Math.max(0, limit - count));
        response.setHeader('X-RateLimit-Reset', await redis.ttl(key));
    }
}
```

---

#### Background Job Optimization

**1. Tenant-Scoped Job Queues:**
```typescript
class TenantJobQueue {
    private queues: Map<string, Queue> = new Map();
    
    getQueue(tenantId: string): Queue {
        if (!this.queues.has(tenantId)) {
            this.queues.set(tenantId, new Queue(`tenant:${tenantId}`, {
                redis: redisConfig,
                defaultJobOptions: {
                    attempts: 3,
                    backoff: {
                        type: 'exponential',
                        delay: 2000
                    },
                    removeOnComplete: true,
                    removeOnFail: false
                }
            }));
        }
        return this.queues.get(tenantId)!;
    }
    
    async addJob(tenantId: string, jobType: string, data: any) {
        const queue = this.getQueue(tenantId);
        await queue.add(jobType, {
            tenantId,
            ...data
        }, {
            priority: this.getJobPriority(tenantId, jobType),
            delay: this.getJobDelay(tenantId, jobType)
        });
    }
    
    private getJobPriority(tenantId: string, jobType: string): number {
        const tenant = this.getTenant(tenantId);
        // Higher tier = higher priority
        const tierPriority = {
            enterprise: 1,
            professional: 5,
            standard: 10
        };
        return tierPriority[tenant.tier] || 10;
    }
}
```

**2. Job Concurrency Limits:**
```typescript
class TenantJobProcessor {
    async processJobs() {
        const worker = new Worker('tenant-jobs', async (job) => {
            const { tenantId, jobType, data } = job.data;
            
            // Check if tenant has available job slots
            const activeJobs = await this.getActiveJobCount(tenantId);
            const maxJobs = await this.getMaxConcurrentJobs(tenantId);
            
            if (activeJobs >= maxJobs) {
                throw new Error('Tenant job limit reached');
            }
            
            // Process job
            return await this.executeJob(jobType, data);
        }, {
            concurrency: 10,
            limiter: {
                max: 100,      // Max 100 jobs
                duration: 60000  // per minute
            }
        });
    }
}
```

---

### Monitoring & Observability

#### Tenant-Specific Metrics

**1. Key Performance Indicators (KPIs):**
```typescript
interface TenantMetrics {
    // User Activity
    activeUsers: number;
    totalUsers: number;
    loginCount: number;
    lastLoginAt: Date;
    
    // Resource Usage
    storageUsed: number;
    apiCallsToday: number;
    databaseConnections: number;
    
    // Performance
    avgResponseTime: number;
    errorRate: number;
    slowQueries: number;
    
    // Business Metrics
    dppsCreated: number;
    productsManaged: number;
    templatesUsed: number;
    
    // Costs
    computeCost: number;
    storageCost: number;
    totalCost: number;
}
```

**2. Metrics Collection:**
```typescript
class TenantMetricsCollector {
    async collectMetrics(tenantId: string): Promise<TenantMetrics> {
        const [
            users,
            storage,
            apiCalls,
            performance,
            business
        ] = await Promise.all([
            this.getUserMetrics(tenantId),
            this.getStorageMetrics(tenantId),
            this.getAPIMetrics(tenantId),
            this.getPerformanceMetrics(tenantId),
            this.getBusinessMetrics(tenantId)
        ]);
        
        return {
            ...users,
            ...storage,
            ...apiCalls,
            ...performance,
            ...business,
            totalCost: this.calculateCost(tenantId, { 
                storage, 
                apiCalls, 
                users 
            })
        };
    }
    
    async recordAPICall(tenantId: string, endpoint: string, duration: number) {
        await metrics.recordTiming(
            'api.response_time',
            duration,
            { tenant: tenantId, endpoint }
        );
        
        await redis.incr(`api_calls:${tenantId}:${getToday()}`);
    }
    
    async recordError(tenantId: string, error: Error) {
        await metrics.incrementCounter(
            'api.errors',
            { tenant: tenantId, type: error.name }
        );
        
        await logger.error('Tenant error', {
            tenantId,
            error: error.message,
            stack: error.stack
        });
    }
}
```

**3. Real-Time Dashboards:**
```typescript
// Metrics endpoint for tenant dashboard
router.get('/api/metrics/tenant/:tenantId', async (req, res) => {
    const tenantId = req.params.tenantId;
    
    // Verify user can access tenant metrics
    if (req.user.tenantId !== tenantId) {
        throw new ForbiddenError('Access denied');
    }
    
    const metrics = await metricsCollector.collectMetrics(tenantId);
    const quotas = await quotaService.getQuotas(tenantId);
    
    res.json({
        metrics,
        quotas,
        usage: {
            users: (metrics.totalUsers / quotas.maxUsers) * 100,
            storage: (metrics.storageUsed / quotas.maxStorage) * 100,
            apiCalls: (metrics.apiCallsToday / quotas.maxAPICallsPerHour) * 100
        }
    });
});
```

---

#### Alerting & Notifications

**1. Threshold-Based Alerts:**
```typescript
interface AlertRule {
    metric: string;
    condition: 'gt' | 'lt' | 'eq';
    threshold: number;
    severity: 'info' | 'warning' | 'critical';
    actions: string[];
}

const tenantAlertRules: AlertRule[] = [
    {
        metric: 'storage_usage_percent',
        condition: 'gt',
        threshold: 90,
        severity: 'warning',
        actions: ['email_admin', 'slack_notification']
    },
    {
        metric: 'error_rate',
        condition: 'gt',
        threshold: 0.05,  // 5%
        severity: 'critical',
        actions: ['email_admin', 'pager_duty', 'slack_notification']
    },
    {
        metric: 'api_calls_remaining',
        condition: 'lt',
        threshold: 100,
        severity: 'warning',
        actions: ['email_admin']
    }
];
```

**2. Alert Processing:**
```typescript
class TenantAlertManager {
    async checkAlerts(tenantId: string) {
        const metrics = await metricsCollector.collectMetrics(tenantId);
        
        for (const rule of tenantAlertRules) {
            const metricValue = metrics[rule.metric];
            const shouldAlert = this.evaluateCondition(
                metricValue,
                rule.condition,
                rule.threshold
            );
            
            if (shouldAlert) {
                await this.triggerAlert(tenantId, rule, metricValue);
            }
        }
    }
    
    async triggerAlert(
        tenantId: string,
        rule: AlertRule,
        value: number
    ) {
        const alert = {
            tenantId,
            metric: rule.metric,
            value,
            threshold: rule.threshold,
            severity: rule.severity,
            timestamp: new Date()
        };
        
        // Execute actions
        for (const action of rule.actions) {
            await this.executeAction(action, alert);
        }
        
        // Store alert history
        await alertsRepository.create(alert);
    }
    
    private async executeAction(action: string, alert: any) {
        switch (action) {
            case 'email_admin':
                await this.emailTenantAdmin(alert);
                break;
            case 'slack_notification':
                await this.sendSlackNotification(alert);
                break;
            case 'pager_duty':
                await this.triggerPagerDuty(alert);
                break;
        }
    }
}
```

---

#### Logging Strategy

**1. Structured Logging:**
```typescript
interface LogEntry {
    timestamp: Date;
    level: 'debug' | 'info' | 'warn' | 'error';
    message: string;
    tenantId?: string;
    userId?: string;
    requestId?: string;
    context: Record<string, any>;
}

class TenantLogger {
    log(entry: LogEntry) {
        const enriched = {
            ...entry,
            environment: process.env.NODE_ENV,
            service: 'byo-dpp-backend',
            version: process.env.APP_VERSION
        };
        
        // Send to logging service (e.g., CloudWatch, ELK)
        loggingService.send(enriched);
        
        // Also log to console in development
        if (process.env.NODE_ENV === 'development') {
            console.log(JSON.stringify(enriched, null, 2));
        }
    }
    
    logRequest(req: Request) {
        this.log({
            timestamp: new Date(),
            level: 'info',
            message: 'HTTP Request',
            tenantId: req.user?.tenantId,
            userId: req.user?.id,
            requestId: req.id,
            context: {
                method: req.method,
                path: req.path,
                query: req.query,
                ip: req.ip,
                userAgent: req.headers['user-agent']
            }
        });
    }
}
```

**2. Audit Logging:**
```typescript
interface AuditLog {
    tenantId: string;
    userId: string;
    action: string;
    resource: string;
    resourceId?: string;
    changes?: any;
    ipAddress: string;
    userAgent: string;
    timestamp: Date;
    result: 'success' | 'failure';
    reason?: string;
}

async function auditLog(log: AuditLog) {
    await auditLogsRepository.create(log);
    
    // Also send to external audit system if required
    if (tenant.requiresCompliance) {
        await externalAuditService.send(log);
    }
}

// Usage in controllers
async function updateProduct(req: Request, res: Response) {
    const oldProduct = await productsRepository.findById(req.params.id);
    const newProduct = await productsRepository.update(req.params.id, req.body);
    
    await auditLog({
        tenantId: req.user.tenantId,
        userId: req.user.id,
        action: 'UPDATE',
        resource: 'Product',
        resourceId: req.params.id,
        changes: {
            before: oldProduct,
            after: newProduct
        },
        ipAddress: req.ip,
        userAgent: req.headers['user-agent'],
        timestamp: new Date(),
        result: 'success'
    });
    
    res.json(newProduct);
}
```

---

### Implementation Checklist

#### Development Phase

**Week 1-2: Foundation**
- [ ] Review current multi-tenancy implementation
- [ ] Document tenant data model
- [ ] Identify all tenant-scoped tables
- [ ] Create tenant isolation test suite
- [ ] Set up development environment with multiple test tenants

**Week 3-4: Security Hardening**
- [ ] Implement Row-Level Security (RLS) policies
- [ ] Add tenant context validation middleware
- [ ] Audit all database queries for tenant_id inclusion
- [ ] Implement token revocation mechanism
- [ ] Add audit logging for sensitive operations

**Week 5-6: Performance Optimization**
- [ ] Create tenant-first composite indexes
- [ ] Implement connection pooling with tenant limits
- [ ] Add query result caching
- [ ] Set up slow query monitoring
- [ ] Optimize N+1 queries

**Week 7-8: Monitoring Setup**
- [ ] Implement metrics collection per tenant
- [ ] Create tenant dashboard endpoints
- [ ] Set up alerting rules
- [ ] Configure structured logging
- [ ] Deploy monitoring infrastructure

---

#### Testing Phase

**Unit Tests:**
```typescript
describe('Tenant Isolation', () => {
    it('should only return data for user tenant', async () => {
        const userA = { tenantId: 'tenant-a' };
        const userB = { tenantId: 'tenant-b' };
        
        const productsA = await productsRepository.findAll(userA);
        const productsB = await productsRepository.findAll(userB);
        
        // Verify no overlap
        expect(productsA).not.toEqual(productsB);
        expect(productsA.every(p => p.tenantId === 'tenant-a')).toBe(true);
        expect(productsB.every(p => p.tenantId === 'tenant-b')).toBe(true);
    });
    
    it('should prevent cross-tenant access', async () => {
        const userA = { tenantId: 'tenant-a' };
        const productB = await createProduct({ tenantId: 'tenant-b' });
        
        await expect(
            productsRepository.findById(productB.id, userA)
        ).rejects.toThrow('Not found');
    });
});
```

**Integration Tests:**
```typescript
describe('End-to-End Tenant Flow', () => {
    it('should complete full tenant onboarding', async () => {
        // 1. Create tenant
        const tenant = await onboardTenant({
            companyInfo: { name: 'Test Corp' },
            adminUser: { email: 'admin@test.com' }
        });
        
        // 2. Verify Keycloak realm created
        const realm = await keycloak.getRealm(tenant.ulid);
        expect(realm).toBeDefined();
        
        // 3. Verify admin user created
        const users = await keycloak.getUsers(tenant.ulid);
        expect(users).toHaveLength(1);
        
        // 4. Verify can authenticate
        const token = await authenticate(tenant.ulid, 'admin@test.com', 'password');
        expect(token).toBeDefined();
        
        // 5. Verify can access API
        const products = await api.get('/products', { 
            headers: { Authorization: `Bearer ${token}` }
        });
        expect(products.status).toBe(200);
    });
});
```

**Load Tests:**
```typescript
// k6 load test script
import http from 'k6/http';
import { check } from 'k6';

export const options = {
    stages: [
        { duration: '2m', target: 100 },  // Ramp up to 100 users
        { duration: '5m', target: 100 },  // Stay at 100 users
        { duration: '2m', target: 0 },    // Ramp down
    ],
    thresholds: {
        http_req_duration: ['p(95)<500'],  // 95% of requests under 500ms
        http_req_failed: ['rate<0.01'],    // Less than 1% errors
    },
};

export default function () {
    const tenants = ['tenant-a', 'tenant-b', 'tenant-c'];
    const tenant = tenants[Math.floor(Math.random() * tenants.length)];
    
    const token = getTenantToken(tenant);
    const res = http.get(`https://api.example.com/products`, {
        headers: { Authorization: `Bearer ${token}` },
    });
    
    check(res, {
        'status is 200': (r) => r.status === 200,
        'response time < 500ms': (r) => r.timings.duration < 500,
        'correct tenant data': (r) => {
            const products = JSON.parse(r.body);
            return products.every(p => p.tenantId === tenant);
        },
    });
}
```

---

#### Deployment Phase

**Pre-Deployment Checklist:**
- [ ] Database migrations tested
- [ ] RLS policies verified
- [ ] Backup strategy in place
- [ ] Rollback plan documented
- [ ] Monitoring dashboards ready
- [ ] Alert rules configured
- [ ] Load tests passed
- [ ] Security audit completed

**Deployment Steps:**
1. **Database Migration:**
   ```bash
   # 1. Backup database
   pg_dump byo_dpp > backup_$(date +%Y%m%d).sql
   
   # 2. Apply migrations
   npm run migrate
   
   # 3. Enable RLS
   psql -f scripts/enable-rls.sql
   
   # 4. Verify
   npm run verify:tenant-isolation
   ```

2. **Application Deployment:**
   ```bash
   # 1. Deploy new version (blue-green)
   kubectl apply -f k8s/deployment-v2.yaml
   
   # 2. Wait for health checks
   kubectl rollout status deployment/byo-dpp-backend
   
   # 3. Switch traffic
   kubectl patch service byo-dpp-backend -p '{"spec":{"selector":{"version":"v2"}}}'
   
   # 4. Monitor metrics
   watch -n 1 'kubectl top pods'
   ```

3. **Post-Deployment Verification:**
   ```bash
   # Run smoke tests
   npm run test:smoke
   
   # Verify tenant isolation
   npm run verify:tenant-isolation
   
   # Check error rates
   curl http://monitoring/api/error-rate
   ```

---

#### Production Monitoring

**Daily Tasks:**
- [ ] Review error logs for tenant-related issues
- [ ] Check quota usage and alerts
- [ ] Monitor performance metrics
- [ ] Review slow query logs

**Weekly Tasks:**
- [ ] Analyze tenant growth trends
- [ ] Review resource utilization
- [ ] Check backup integrity
- [ ] Update capacity planning

**Monthly Tasks:**
- [ ] Security audit review
- [ ] Compliance check
- [ ] Cost optimization analysis
- [ ] Tenant satisfaction survey

---

### Conclusion

This comprehensive multi-tenancy implementation guide provides:

✅ **Complete Architecture Analysis** - Understanding BYO DPP's hybrid approach
✅ **Pattern Comparison** - Trade-offs of different multi-tenancy strategies
✅ **Security Deep Dive** - Identity, authorization, and data isolation
✅ **Scaling Strategy** - Horizontal scaling, caching, and optimization
✅ **Migration Roadmap** - 24-week phased implementation plan
✅ **Performance Optimization** - Query optimization and resource management
✅ **Monitoring & Observability** - Metrics, alerts, and logging
✅ **Implementation Checklist** - Step-by-step deployment guide

**Key Takeaways:**

1. **Current Strengths:**
   - Well-structured tenant data model with hierarchical support
   - Keycloak multi-realm provides strong identity isolation
   - Casbin ABAC enables flexible, tenant-scoped authorization
   - Cascading deletes maintain referential integrity

2. **Recommended Improvements:**
   - Add PostgreSQL Row-Level Security (RLS) for defense in depth
   - Implement connection pooling with per-tenant limits
   - Add comprehensive metrics and monitoring
   - Create hybrid isolation strategy for different tenant tiers

3. **Future Enhancements:**
   - Support for data residency requirements
   - Tenant self-service portal
   - Advanced analytics and reporting
   - Multi-region deployment

**Next Steps:**
1. Review this guide with the development team
2. Prioritize improvements based on business needs
3. Start with quick wins (RLS, monitoring, caching)
4. Plan long-term migration to hybrid model
5. Continuously monitor and optimize

---



