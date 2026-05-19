# Thoth AI
## 1. Executive System Overview

The **Thoth AI Platform** is a multi-tenant, premium e-commerce and marketing optimization system. It connects directly with online stores (via Shopify) and advertising accounts (via Meta) to pull raw insights and ad telemetry. It analyzes this data with a custom analytics layer, processes high-dimensional metrics at the database level using PostgreSQL stored procedures, and triggers automated AI recommendations to dynamically adjust marketing spend and maximize return on ad spend (ROAS).

### Technology Stack & Services
* **Backend Framework:** NestJS (TypeScript) utilizing custom modules, guards, interceptors, and exception filters.
* **Database & Persistence:** Neon DB / PostgreSQL managed via TypeORM, implementing custom SQL procedures for fast multi-dimensional financial calculations.
* **Background Processing:** BullMQ (powered by Redis) for high-performance, asynchronous recommendation worker queues.
* **Task Scheduling:** NestJS `ScheduleModule` hosting hourly, bi-hourly, and daily cron jobs.
* **Authentication:** JWT (JSON Web Tokens), Passport.js strategies, and bcryptjs passwords.
* **External Integrations:**
  * **Shopify API:** Rest API for orders and GraphQL API for product variant inventory costs.
  * **Meta Graph API:** Ad account details, campaigns, adsets, ads, and ad-level performance metrics.
  * **OpenAI API:** GPT-based LLM recommendations utilizing structured JSON outputs.
  * **Twilio API:** Interactive SMS message processing and two-way webhook loops.
  * **Stripe API:** Checkout sessions and event-based payment status processing.

---

## 2. Core Architectural Flow

```mermaid
graph TD
    %% Styling Classes
    classDef client fill:#e0f7fa,stroke:#00acc1,stroke-width:2px,color:#01579b;
    classDef gate fill:#efebe9,stroke:#8d6e63,stroke-width:2px,color:#3e2723;
    classDef db fill:#e8f5e9,stroke:#4caf50,stroke-width:2px,color:#1b5e20;
    classDef ext fill:#fff3e0,stroke:#ff9800,stroke-width:2px,color:#e65100;
    classDef queue fill:#f3e5f5,stroke:#9c27b0,stroke-width:2px,color:#4a148c;

    %% Elements
    Client["User Dashboard / CLI"]:::client
    TwilioClient["User Mobile Phone"]:::client
    
    subgraph NestJsGateway ["NestJS API Gateway"]
        direction TB
        AuthCtrl["Auth Controller"]:::gate
        UserCtrl["Users Controller"]:::gate
        KeyCtrl["Keys Controller"]:::gate
        SyncCtrl["Sync Controller"]:::gate
        SubCtrl["Subscription Controller"]:::gate
        AutoCtrl["Automation Controller"]:::gate
        
        AuthServ["Auth Service"]:::gate
        UserServ["Users Service"]:::gate
        KeyServ["Keys Service"]:::gate
        SyncServ["Sync Service"]:::gate
        SubServ["Subscription Service"]:::gate
        AutoServ["Automation Service"]:::gate
    end
    
    subgraph JobsEngine ["Cron & Asynchronous Engine"]
        direction TB
        CronServ["Cron Service"]:::queue
        BullQueue["BullMQ: recommendation-queue"]:::queue
        Processor["Recommendation Processor"]:::queue
    end

    subgraph Storage ["Neon PostgreSQL & Cache"]
        direction TB
        Postgres["PostgreSQL DB"]:::db
        StoredProcs["PL/pgSQL Functions<br/>get_shopify_order_metrics<br/>get_shopify_order_metrics_daily<br/>get_shopify_order_metrics_last_24hrs"]:::db
        Redis["Redis Cache"]:::db
    end

    subgraph ExternalAPIs ["External Platforms & APIs"]
        direction TB
        ShopifyAPI["Shopify API<br/>REST & GraphQL"]:::ext
        MetaAPI["Meta Graph API<br/>v23.0"]:::ext
        OpenAI["OpenAI API<br/>gpt-4o-mini"]:::ext
        TwilioAPI["Twilio SMS Gateway"]:::ext
        StripeAPI["Stripe Gateway"]:::ext
    end

    %% Gateway Routing & Services
    Client -->|HTTPS or WSS| NestJsGateway
    AuthCtrl --> AuthServ
    UserCtrl --> UserServ
    KeyCtrl --> KeyServ
    SyncCtrl --> SyncServ
    SubCtrl --> SubServ
    AutoCtrl --> AutoServ

    %% Db interactions
    AuthServ -->|TypeORM| Postgres
    UserServ -->|TypeORM| Postgres
    KeyServ -->|TypeORM| Postgres
    SubServ -->|TypeORM| Postgres
    AutoServ -->|TypeORM| Postgres
    Redis -->|Cache Store| BullQueue
    BullQueue -->|Queue State| Redis
    
    %% Database aggregates
    SyncServ -->|Calls SQL functions| StoredProcs
    StoredProcs -->|Aggregates| Postgres

    %% Key Verification & Sync Lines
    KeyServ -->|Validate Tokens| ShopifyAPI
    KeyServ -->|Validate Tokens| MetaAPI
    SyncServ -->|Fetch Telemetry| ShopifyAPI
    SyncServ -->|Fetch Telemetry| MetaAPI
    ShopifyAPI -->|Save Orders and Variants| Postgres
    MetaAPI -->|Save Adset and Campaign metrics| Postgres
    
    %% Background Scheduler Lifecycle
    CronServ -->|Hourly Cron| SyncServ
    CronServ -->|Scheduled Queue Job| BullQueue
    BullQueue --> Processor
    Processor -->|Generates Context| CronServ
    CronServ -->|Builds Prompts| OpenAI
    OpenAI -->|Returns JSON Recommendation| CronServ
    
    %% Automation & SMS Webhook loop
    CronServ -->|Save Rec & Queue SMS| AutoServ
    AutoServ -->|Send SMS recommendation| TwilioAPI
    TwilioAPI -->|Delivers Text Message| TwilioClient
    TwilioClient -->|Replies YES or NO| TwilioAPI
    TwilioAPI -->|Webhook HTTPS POST| AutoCtrl
    AutoCtrl -->|Invokes action| AutoServ
    AutoServ -->|Pauses or Updates Budget| MetaAPI
    AutoServ -->|Confirms Action via SMS| TwilioAPI
    
    %% Payment integrations
    SubServ -->|Initializes Card Checkout| StripeAPI
    StripeAPI -->|Webhook checkout.completed| SubCtrl
```

---

## 3. Core Component Analysis

### A. Authentication & Onboarding
### B. Integration Keys Management
### C. The Asynchronous Data Sync Pipeline
### D. Advanced Metric Aggregations (PL/pgSQL Functions)
---

## 4. Sequence Diagram: SMS Budget Optimization Loop

The signature feature of Thoth AI is its interactive budget optimization loop. It provides an automated, secure path to optimization, allowing users to apply complex marketing adjustments with a simple text reply:

```mermaid
sequenceDiagram
    autonumber
    actor User as User Mobile Phone
    participant Twilio as Twilio Gateway
    participant Server as NestJS Backend
    participant Postgres as Neon PostgreSQL
    participant BullMQ as BullMQ & Redis
    participant OpenAI as OpenAI (GPT-4o-mini)
    participant Meta as Meta Graph API

    Note over Server, Meta: Phase 1: Scheduled Analysis & Recommendation Generation
    loop Every 8 Hours (Cron Triggered)
        Server->>Postgres: Fetch verified active user keys
        Postgres-->>Server: Return API Credentials
        Server->>BullMQ: Queue "generate-recommendation" job
    end

    BullMQ->>Server: Process Queue Job (Worker)
    activate Server
    Server->>Postgres: Query sales metrics (get_shopify_order_metrics)
    Postgres-->>Server: Return revenue, COGS, and Meta spend
    Server->>Postgres: Query recent Meta campaign/adset telemetry
    Postgres-->>Server: Return campaign lists and budget settings
    
    Server->>OpenAI: Request structured JSON optimization recommendation
    activate OpenAI
    OpenAI-->>Server: Return structured JSON recommendation
    deactivate OpenAI

    Server->>Postgres: Save recommendation to database (Status: PENDING)
    
    Note over Server, Twilio: Phase 2: Twilio SMS Notification
    Server->>Server: Format recommendation as text message + SMS YES/NO prompt
    Server->>Postgres: Save Automation Record (Status: pending)
    Server->>Twilio: Send SMS request via Twilio API
    Twilio->>User: Deliver SMS: "📊 Ad Recommendation... Reply YES to apply..."
    Server->>Postgres: Update Automation Record (Status: sent)
    deactivate Server

    Note over User, Meta: Phase 3: Interactive Action Execution
    User->>Twilio: Reply text: "YES"
    Twilio->>Server: HTTPS POST to webhook endpoint (/automation/webhook)
    activate Server
    Server->>Postgres: Find latest sent automation record for phone number
    Postgres-->>Server: Return Automation & associated Recommendation ID
    Server->>Postgres: Update Recommendation status to PROCESSING
    
    Server->>Meta: Fetch target campaign/adset current budget
    Meta-->>Server: Return current daily_budget = $100
    Server->>Server: Calculate new budget (Increase 20% = $120)
    
    Server->>Meta: Update Budget (POST /xyz with daily_budget = $120)
    activate Meta
    Meta-->>Server: Confirm Update Success (HTTP 200 OK)
    deactivate Meta
    
    Server->>Postgres: Update Recommendation status to COMPLETED & isApplied = true
    Server->>Twilio: Send confirmation SMS request
    Twilio->>User: Deliver SMS: "✅ Recommendation applied successfully..."
    deactivate Server
```

---

## 5. Sync State Machine

The following diagram tracks the lifecycle states of the data synchronization process:

```mermaid
stateDiagram-v2
    [*] --> Pending : Triggered by User Login or Cron
    
    state Pending {
        [*] --> InitSyncStatus
        InitSyncStatus --> SetPendingFlags : Set metaDataSynced/shopifyDataSynced to false
    }
    
    Pending --> InProgress : performSync() invoked asynchronously
    
    state InProgress {
        [*] --> FetchMetaCredentials
        FetchMetaCredentials --> SyncMetaTelemetry : Keys configured
        
        state SyncMetaTelemetry {
            [*] --> ParallelMetaAPICalls
            ParallelMetaAPICalls --> SaveCampaignsAdsetsAds
            ParallelMetaAPICalls --> SaveInsights
            SaveCampaignsAdsetsAds --> UpdateMetaCount
            SaveInsights --> UpdateMetaCount
        }
        
        UpdateMetaCount --> FetchShopifyCredentials
        FetchShopifyCredentials --> SyncShopifyTelemetry : Keys configured
        
        state SyncShopifyTelemetry {
            [*] --> PullOrders
            PullOrders --> FetchInventoryCosts : GraphQL query product variant costs
            FetchInventoryCosts --> ComputeLineMargins
            ComputeLineMargins --> SaveOrdersToDB
        }
        
        SaveOrdersToDB --> BroadcastSocketUpdate : notifyUserStatusChange(in_progress)
        BroadcastSocketUpdate --> GenerateReports : Calls DailyReportsService
    }
    
    InProgress --> Completed : Success
    InProgress --> Failed : Error Caught in Try/Catch
    
    state Completed {
        [*] --> MarkSyncComplete : Set completedAt timestamp
        MarkSyncComplete --> SocketEmitComplete : Emit "completed" via WebSocket
    }
    
    state Failed {
        [*] --> RecordErrorMessage : Set error message & completedAt
        RecordErrorMessage --> SocketEmitFail : Emit "failed" via WebSocket
    }

    Completed --> [*]
    Failed --> [*]
```

---

## 6. System Design Verification & Best Practices

1. **Security Isolation:** API keys and sensitive tokens (Shopify Token, Meta Token) are encrypted or stored under strict access guidelines in Neon DB. Custom authentication strategies utilize standard passport JWT guards, verifying signatures and checking JWT payloads (`sub`, `role`) dynamically.
2. **Resilience & Fault Tolerance:** Database synchronization functions utilize `try/catch` boundaries around each platform fetch call. If Shopify sync fails, Meta sync continues to run, updating flags gracefully to avoid breaking the background queue execution.
3. **Optimized DB Operations:** E-commerce telemetry represents highly fragmented transactional records. Instead of pulling millions of rows to NestJS to perform analysis, Neon DB's SQL analytical functions process high-volume sums, COGS margin aggregations, customer repeats, and channel splits in milliseconds inside PostgreSQL itself.
4. **BullMQ Backgrounding:** AI prompt assembly and LLM response parsing are slow, resource-heavy operations that can take several seconds. Offloading these tasks to BullMQ workers preserves NestJS API request handlers, guaranteeing that web controllers remain highly responsive to frontend calls.
5. **Interactive Edge Loops:** The integration of Twilio's incoming webhooks to dynamically update live production ad campaign budgets on Meta represents a highly decoupled, state-driven telemetry loop, offering friction-free operations to e-commerce administrators.
