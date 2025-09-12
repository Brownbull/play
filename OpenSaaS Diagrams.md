# OpenSaaS/Wasp Architecture & Workflow Diagrams

## Table of Contents
1. [Framework Architecture Overview](#1-framework-architecture-overview)
2. [Request/Response Flow](#2-requestresponse-flow)
3. [Database Operations Flow](#3-database-operations-flow)
4. [Authentication Workflow](#4-authentication-workflow)
5. [Payment & Subscription Flow](#5-payment--subscription-flow)
6. [Development Workflow](#6-development-workflow)
7. [Testing Strategy Flow](#7-testing-strategy-flow)
8. [Deployment Pipeline](#8-deployment-pipeline)

---

## 1. Framework Architecture Overview

```mermaid
graph TB
    subgraph "Client Side"
        React[React App]
        Router[React Router]
        Auth[Auth Hook]
        Operations[Operations/Hooks]
        UI[UI Components]
    end
    
    subgraph "Wasp Layer"
        WaspConfig[main.wasp]
        WaspCompiler[Wasp Compiler]
        Generated[Generated Code]
    end
    
    subgraph "Server Side"
        NodeJS[Node.js Server]
        API[API Endpoints]
        Queries[Queries]
        Actions[Actions]
        Jobs[Background Jobs]
        WebSockets[WebSockets]
    end
    
    subgraph "Data Layer"
        Prisma[Prisma ORM]
        PostgreSQL[(PostgreSQL)]
        Redis[(Redis Cache)]
    end
    
    subgraph "External Services"
        Stripe[Stripe API]
        SendGrid[SendGrid Email]
        S3[AWS S3]
        OAuth[OAuth Providers]
        Analytics[Analytics]
    end
    
    React --> Router
    Router --> Auth
    Router --> UI
    UI --> Operations
    Operations --> API
    
    WaspConfig --> WaspCompiler
    WaspCompiler --> Generated
    Generated --> React
    Generated --> NodeJS
    
    API --> Queries
    API --> Actions
    NodeJS --> Jobs
    NodeJS --> WebSockets
    
    Queries --> Prisma
    Actions --> Prisma
    Jobs --> Prisma
    Prisma --> PostgreSQL
    
    NodeJS --> Redis
    NodeJS --> Stripe
    NodeJS --> SendGrid
    NodeJS --> S3
    Auth --> OAuth
    React --> Analytics
    
    style WaspConfig fill:#f9f,stroke:#333,stroke-width:4px
    style PostgreSQL fill:#326ce5,stroke:#333,stroke-width:2px
    style React fill:#61dafb,stroke:#333,stroke-width:2px
    style NodeJS fill:#68a063,stroke:#333,stroke-width:2px
```

---

## 2. Request/Response Flow

```mermaid
sequenceDiagram
    participant User
    participant React as React Component
    participant Hook as useQuery/useAction
    participant API as API Layer
    participant Wasp as Wasp Runtime
    participant Operation as Query/Action
    participant Prisma
    participant DB as PostgreSQL
    
    User->>React: Interact with UI
    React->>Hook: Call useQuery/useAction
    Hook->>API: HTTP Request
    API->>Wasp: Route to Operation
    Wasp->>Wasp: Check Auth & Permissions
    Wasp->>Operation: Execute with Context
    Operation->>Prisma: Database Query
    Prisma->>DB: SQL Query
    DB-->>Prisma: Result Set
    Prisma-->>Operation: Entities
    Operation-->>Wasp: Processed Data
    Wasp-->>API: Response
    API-->>Hook: JSON Response
    Hook-->>React: Update State
    React-->>User: Update UI
    
    Note over Hook,API: Automatic retry & caching
    Note over Wasp: Type safety enforced
    Note over Operation,Prisma: Transaction support
```

---

## 3. Database Operations Flow

```mermaid
graph LR
    subgraph "Entity Definition"
        Schema[schema.prisma]
        Migration[Migration Files]
    end
    
    subgraph "Wasp Configuration"
        MainWasp[main.wasp]
        Queries[Query Definitions]
        Actions[Action Definitions]
    end
    
    subgraph "Implementation"
        QueryImpl[Query Implementation]
        ActionImpl[Action Implementation]
        Context[Context Object]
    end
    
    subgraph "Client Usage"
        UseQuery[useQuery Hook]
        UseAction[useAction Hook]
        Optimistic[Optimistic Updates]
    end
    
    subgraph "Database"
        Prisma[Prisma Client]
        Postgres[(PostgreSQL)]
        Transactions[Transactions]
    end
    
    Schema -->|wasp db migrate-dev| Migration
    Migration --> Postgres
    
    MainWasp --> Queries
    MainWasp --> Actions
    
    Queries --> QueryImpl
    Actions --> ActionImpl
    
    QueryImpl --> Context
    ActionImpl --> Context
    Context --> Prisma
    
    UseQuery -->|Read| QueryImpl
    UseAction -->|Write| ActionImpl
    UseAction --> Optimistic
    
    Prisma --> Postgres
    Prisma --> Transactions
    
    style Schema fill:#f96,stroke:#333,stroke-width:2px
    style MainWasp fill:#f9f,stroke:#333,stroke-width:2px
    style Postgres fill:#326ce5,stroke:#333,stroke-width:2px
```

### Database Operation Example Flow

```mermaid
flowchart TD
    Start([User Action]) --> Check{Auth Check}
    Check -->|Authenticated| Validate[Validate Input]
    Check -->|Not Authenticated| Error1[401 Unauthorized]
    
    Validate -->|Valid| Execute[Execute Operation]
    Validate -->|Invalid| Error2[400 Bad Request]
    
    Execute --> Transaction[Begin Transaction]
    Transaction --> Query1[Query User Data]
    Query1 --> Query2[Query Related Data]
    Query2 --> Modify[Modify Data]
    Modify --> Commit{Commit?}
    
    Commit -->|Success| Cache[Update Cache]
    Commit -->|Failure| Rollback[Rollback Transaction]
    
    Cache --> Response[Return Response]
    Rollback --> Error3[500 Server Error]
    
    Response --> End([Update UI])
    Error1 --> End
    Error2 --> End
    Error3 --> End
    
    style Start fill:#90EE90
    style End fill:#FFB6C1
    style Error1 fill:#FF6B6B
    style Error2 fill:#FF6B6B
    style Error3 fill:#FF6B6B
```

---

## 4. Authentication Workflow

```mermaid
stateDiagram-v2
    [*] --> Anonymous
    
    Anonymous --> SignUp: User Registration
    Anonymous --> Login: User Login
    Anonymous --> OAuth: Social Login
    
    SignUp --> EmailVerification: Send Verification Email
    EmailVerification --> Verified: Click Verify Link
    
    Login --> CheckCredentials: Validate
    CheckCredentials --> Authenticated: Valid
    CheckCredentials --> Anonymous: Invalid
    
    OAuth --> OAuthProvider: Redirect
    OAuthProvider --> OAuthCallback: Return with Token
    OAuthCallback --> Authenticated: Create/Update User
    
    Verified --> Authenticated: Auto Login
    
    Authenticated --> Session: Create Session
    Session --> Authorized: Has Permission
    Session --> Unauthorized: No Permission
    
    Authorized --> [*]: Access Granted
    Unauthorized --> Anonymous: Redirect to Login
    
    Authenticated --> Logout: User Logout
    Logout --> Anonymous: Clear Session
```

### Detailed Auth Flow

```mermaid
sequenceDiagram
    participant Client
    participant WaspAuth as Wasp Auth
    participant Server
    participant Database
    participant Email as Email Service
    participant Stripe
    
    alt Email Signup
        Client->>Server: POST /auth/email/signup
        Server->>Database: Check existing user
        Database-->>Server: User not found
        Server->>Database: Create user (unverified)
        Server->>Email: Send verification email
        Email-->>Client: Verification link
        Client->>Server: GET /auth/verify?token=xxx
        Server->>Database: Update user.verified = true
        Server->>Stripe: Create Stripe customer
        Server-->>Client: Redirect to login
    else OAuth Login
        Client->>Server: GET /auth/google
        Server-->>Client: Redirect to Google
        Client->>Google: Authorize
        Google-->>Server: Callback with token
        Server->>Database: Find or create user
        Server->>Stripe: Get or create customer
        Server->>WaspAuth: Generate JWT
        WaspAuth-->>Client: Set auth cookie
    else Email Login
        Client->>Server: POST /auth/email/login
        Server->>Database: Verify credentials
        Database-->>Server: User verified
        Server->>WaspAuth: Generate JWT
        WaspAuth-->>Client: Set auth cookie
    end
    
    Client->>Server: GET /api/protected
    Server->>WaspAuth: Verify JWT
    WaspAuth-->>Server: User context
    Server-->>Client: Protected data
```

---

## 5. Payment & Subscription Flow

```mermaid
graph TB
    subgraph "Subscription Lifecycle"
        Browse[Browse Plans]
        Select[Select Plan]
        Checkout[Stripe Checkout]
        Payment[Process Payment]
        Active[Active Subscription]
        Usage[Track Usage]
        Renew[Auto Renewal]
        Cancel[Cancellation]
    end
    
    subgraph "Stripe Integration"
        Products[Product Catalog]
        Prices[Price Tiers]
        Sessions[Checkout Sessions]
        Webhooks[Webhook Events]
        Portal[Customer Portal]
    end
    
    subgraph "Database Updates"
        UserSub[User Subscription]
        PaymentHistory[Payment History]
        UsageData[Usage Metrics]
        InvoiceRecords[Invoice Records]
    end
    
    Browse --> Select
    Select --> Checkout
    Checkout --> Sessions
    Sessions --> Payment
    Payment --> Webhooks
    Webhooks --> Active
    Webhooks --> UserSub
    
    Active --> Usage
    Usage --> UsageData
    Active --> Renew
    Renew --> Webhooks
    Webhooks --> PaymentHistory
    Webhooks --> InvoiceRecords
    
    Active --> Portal
    Portal --> Cancel
    Cancel --> Webhooks
    
    Products --> Prices
    Prices --> Sessions
    
    style Stripe fill:#635bff,stroke:#333,stroke-width:2px
    style Active fill:#90EE90,stroke:#333,stroke-width:2px
    style Cancel fill:#FF6B6B,stroke:#333,stroke-width:2px
```

### Detailed Payment Flow

```mermaid
sequenceDiagram
    participant User
    participant App
    participant Server
    participant Stripe
    participant Database
    participant Webhook
    
    User->>App: Click "Subscribe"
    App->>Server: POST /create-checkout-session
    Server->>Stripe: Create checkout session
    Stripe-->>Server: Session URL
    Server-->>App: Redirect URL
    App->>Stripe: Redirect to Checkout
    
    User->>Stripe: Enter payment info
    Stripe->>Stripe: Process payment
    
    alt Payment Success
        Stripe-->>User: Redirect to success_url
        Stripe->>Webhook: payment_intent.succeeded
        Webhook->>Database: Update subscription status
        Webhook->>Database: Create invoice record
        Webhook-->>Stripe: 200 OK
        User->>App: Land on success page
        App->>Server: GET /subscription-status
        Server->>Database: Query subscription
        Database-->>Server: Active subscription
        Server-->>App: Subscription details
        App-->>User: Show premium features
    else Payment Failed
        Stripe-->>User: Show error
        Stripe->>Webhook: payment_intent.failed
        Webhook->>Database: Log failed attempt
        Webhook-->>Stripe: 200 OK
        User->>App: Return to plans
    end
```

---

## 6. Development Workflow

```mermaid
flowchart LR
    subgraph "Development Process"
        direction TB
        Requirement[Requirements]
        Design[Design Entity]
        Schema[Update schema.prisma]
        Migration[Run Migration]
        WaspDef[Define in main.wasp]
        Implement[Implement Operations]
        Component[Create React Component]
        Test[Write Tests]
        Review[Code Review]
    end
    
    subgraph "File Structure"
        direction TB
        Root[Project Root]
        MainWasp[main.wasp]
        Prisma[schema.prisma]
        Src[src/]
        Pages[pages/]
        Operations[operations/]
        Components[components/]
        Tests[__tests__/]
    end
    
    subgraph "Commands"
        direction TB
        Start[wasp start]
        DBStart[wasp db start]
        Migrate[wasp db migrate-dev]
        Studio[wasp db studio]
        Build[wasp build]
        TestCmd[npm test]
    end
    
    Requirement --> Design
    Design --> Schema
    Schema --> Migration
    Migration --> WaspDef
    WaspDef --> Implement
    Implement --> Component
    Component --> Test
    Test --> Review
    
    Root --> MainWasp
    Root --> Prisma
    Root --> Src
    Src --> Pages
    Src --> Operations
    Src --> Components
    Src --> Tests
    
    Schema -.-> Migrate
    Migration -.-> DBStart
    WaspDef -.-> Start
    Test -.-> TestCmd
    Review -.-> Build
    
    style Requirement fill:#FFE4B5
    style Review fill:#90EE90
    style Test fill:#87CEEB
```

---

## 7. Testing Strategy Flow

```mermaid
graph TD
    subgraph "Testing Pyramid"
        Unit[Unit Tests<br/>70%]
        Integration[Integration Tests<br/>20%]
        E2E[E2E Tests<br/>10%]
    end
    
    subgraph "TDD Cycle"
        Red[Write Failing Test]
        Green[Write Minimal Code]
        Refactor[Refactor Code]
    end
    
    subgraph "Test Types"
        Component[Component Tests<br/>React Testing Library]
        Operation[Operation Tests<br/>Queries/Actions]
        API[API Tests<br/>Endpoints]
        Browser[Browser Tests<br/>Playwright]
    end
    
    subgraph "CI/CD Pipeline"
        Commit[Git Commit]
        Lint[ESLint]
        TypeCheck[TypeScript Check]
        UnitTest[Run Unit Tests]
        IntTest[Run Integration Tests]
        E2ETest[Run E2E Tests]
        Coverage[Coverage Report]
        Deploy[Deploy]
    end
    
    Unit --> Component
    Unit --> Operation
    Integration --> API
    E2E --> Browser
    
    Red --> Green
    Green --> Refactor
    Refactor --> Red
    
    Commit --> Lint
    Lint --> TypeCheck
    TypeCheck --> UnitTest
    UnitTest --> IntTest
    IntTest --> E2ETest
    E2ETest --> Coverage
    Coverage --> Deploy
    
    style Unit fill:#90EE90
    style Integration fill:#FFE4B5
    style E2E fill:#87CEEB
    style Red fill:#FF6B6B
    style Green fill:#90EE90
    style Refactor fill:#87CEEB
```

### Test Execution Flow

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant Git
    participant CI as CI/CD
    participant Vitest
    participant Playwright
    participant Coverage
    participant Deploy
    
    Dev->>Git: git push
    Git->>CI: Trigger pipeline
    
    CI->>CI: Install dependencies
    CI->>CI: Build application
    
    par Run Tests in Parallel
        CI->>Vitest: Run unit tests
        Vitest-->>CI: ✓ 150 passed
    and
        CI->>Playwright: Run E2E tests
        Playwright-->>CI: ✓ 25 passed
    end
    
    CI->>Coverage: Generate report
    Coverage-->>CI: 85% coverage
    
    alt All Tests Pass
        CI->>Deploy: Deploy to staging
        Deploy-->>CI: Deployed successfully
        CI-->>Dev: ✅ Pipeline passed
    else Tests Failed
        CI-->>Dev: ❌ Pipeline failed
        Dev->>Dev: Fix issues
        Dev->>Git: git push (retry)
    end
```

---

## 8. Deployment Pipeline

```mermaid
graph LR
    subgraph "Local Development"
        LocalCode[Local Code]
        LocalDB[(Local PostgreSQL)]
        LocalTest[Local Tests]
    end
    
    subgraph "Version Control"
        GitHub[GitHub]
        PR[Pull Request]
        Review[Code Review]
    end
    
    subgraph "CI/CD"
        Actions[GitHub Actions]
        BuildStep[Build]
        TestStep[Test]
        DockerBuild[Docker Build]
    end
    
    subgraph "Staging"
        StagingApp[Staging App]
        StagingDB[(Staging DB)]
        StagingTests[Smoke Tests]
    end
    
    subgraph "Production"
        Fly[Fly.io / Railway]
        ProdDB[(Production DB)]
        CDN[CDN]
        Monitoring[Monitoring]
    end
    
    subgraph "External Services"
        StripeAPI[Stripe API]
        EmailService[Email Service]
        Storage[S3 Storage]
    end
    
    LocalCode --> LocalTest
    LocalTest --> GitHub
    GitHub --> PR
    PR --> Review
    Review --> Actions
    
    Actions --> BuildStep
    BuildStep --> TestStep
    TestStep --> DockerBuild
    
    DockerBuild --> StagingApp
    StagingApp --> StagingDB
    StagingApp --> StagingTests
    
    StagingTests --> Fly
    Fly --> ProdDB
    Fly --> CDN
    Fly --> Monitoring
    
    StagingApp -.-> StripeAPI
    Fly -.-> StripeAPI
    Fly -.-> EmailService
    Fly -.-> Storage
    
    style GitHub fill:#333,color:#fff
    style Fly fill:#8B5CF6,color:#fff
    style ProdDB fill:#326ce5
    style Monitoring fill:#FF6B6B
```

### Deployment Process Detail

```mermaid
flowchart TD
    Start([Start Deployment]) --> CheckEnv{Check Environment}
    
    CheckEnv -->|Development| DevBuild[wasp build]
    CheckEnv -->|Production| ProdBuild[wasp build --production]
    
    DevBuild --> LocalDocker[Docker Compose]
    ProdBuild --> EnvVars[Set Production Env Vars]
    
    EnvVars --> Database{Database Setup}
    Database -->|New| CreateDB[Create Database]
    Database -->|Existing| BackupDB[Backup Database]
    
    CreateDB --> RunMigrations[Run Migrations]
    BackupDB --> RunMigrations
    
    RunMigrations --> BuildContainer[Build Docker Container]
    BuildContainer --> PushRegistry[Push to Registry]
    
    PushRegistry --> Deploy{Deployment Target}
    Deploy -->|Fly.io| FlyDeploy[fly deploy]
    Deploy -->|Railway| RailwayDeploy[railway up]
    Deploy -->|Custom| CustomDeploy[Custom Script]
    
    FlyDeploy --> HealthCheck[Health Check]
    RailwayDeploy --> HealthCheck
    CustomDeploy --> HealthCheck
    
    HealthCheck -->|Pass| ConfigureServices[Configure Services]
    HealthCheck -->|Fail| Rollback[Rollback]
    
    ConfigureServices --> Stripe[Setup Stripe Webhooks]
    Stripe --> DNS[Configure DNS]
    DNS --> SSL[SSL Certificate]
    SSL --> Monitor[Setup Monitoring]
    
    Monitor --> Success([Deployment Complete])
    Rollback --> Failed([Deployment Failed])
    
    style Start fill:#90EE90
    style Success fill:#90EE90
    style Failed fill:#FF6B6B
    style HealthCheck fill:#FFE4B5
```

---

## Data Flow Summary

```mermaid
graph TB
    subgraph "User Journey"
        Visit[User Visits App]
        Auth[Authentication]
        Browse[Browse Features]
        Subscribe[Subscribe to Plan]
        Use[Use Premium Features]
        Data[Generate Data]
    end
    
    subgraph "Application Layer"
        Frontend[React Frontend]
        API[API Routes]
        Operations[Operations Layer]
        Cache[Cache Layer]
    end
    
    subgraph "Data Persistence"
        Database[(PostgreSQL)]
        FileStorage[S3 Storage]
        Analytics[Analytics DB]
    end
    
    subgraph "Background Processes"
        Jobs[Cron Jobs]
        Webhooks[Webhook Handlers]
        EmailQueue[Email Queue]
    end
    
    Visit --> Frontend
    Frontend --> Auth
    Auth --> API
    API --> Operations
    Operations --> Database
    
    Browse --> Frontend
    Subscribe --> API
    API --> Stripe
    Stripe --> Webhooks
    Webhooks --> Database
    
    Use --> Frontend
    Frontend --> Cache
    Cache --> Operations
    Data --> FileStorage
    
    Jobs --> Database
    Jobs --> EmailQueue
    EmailQueue --> Users
    
    Frontend --> Analytics
    
    style Frontend fill:#61dafb
    style Database fill:#326ce5
    style Jobs fill:#68a063
    style Stripe fill:#635bff
```

---

## Key Takeaways

### 🏗️ Architecture Patterns
- **Separation of Concerns**: Clear boundaries between client, server, and data layers
- **Type Safety**: End-to-end type safety from database to UI
- **Code Generation**: Wasp generates boilerplate, reducing manual work
- **Convention over Configuration**: Standardized patterns for common tasks

### 🔄 Data Flow Patterns
- **Unidirectional Data Flow**: Client → API → Operations → Database
- **Optimistic Updates**: Immediate UI updates with background sync
- **Caching Strategy**: Multiple cache layers for performance
- **Event-Driven**: Webhooks and background jobs for async operations

### 🚀 Development Patterns
- **Database-First**: Start with schema, generate types
- **Operation-Centric**: Queries for reading, Actions for writing
- **Component Isolation**: Reusable, testable components
- **Progressive Enhancement**: Start simple, add complexity as needed

### 🔒 Security Patterns
- **Authentication Layers**: Multiple auth methods with verification
- **Authorization**: Role-based access control at operation level
- **Input Validation**: Server-side validation for all operations
- **Secure by Default**: HTTPS, CORS, CSP headers configured

### 📊 Performance Patterns
- **Lazy Loading**: Load data as needed
- **Query Optimization**: Prisma query optimization
- **Background Processing**: Offload heavy work to job queues
- **CDN Integration**: Static assets served from edge

These diagrams provide a comprehensive view of how OpenSaaS/Wasp orchestrates the various components to create a full-stack application with minimal boilerplate while maintaining flexibility and scalability.