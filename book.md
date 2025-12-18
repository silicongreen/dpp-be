# BYO DPP Developer Onboarding Handbook

## Table of Contents

1. [Introduction](#introduction)
2. [Architecture Overview](#architecture-overview)
3. [Domain Driven Design Alignment](#domain-driven-design-alignment)
4. [Module Lifecycle Management](#module-lifecycle-management)
5. [Event Orchestration and Pipeline Patterns](#event-orchestration-and-pipeline-patterns)
6. [Error Handling and Retry Architecture](#error-handling-and-retry-architecture)
7. [Deployment Architecture and Runtime Concerns](#deployment-architecture-and-runtime-concerns)
8. [API → Service → Domain → Infra Implementation Walk-through](#api--service--domain--infra-implementation-walk-through)
9. [Code Samples](#code-samples)
10. [Diagrams](#diagrams)
11. [Tables and References](#tables-and-references)
12. [Glossary](#glossary)
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

## Code Samples

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