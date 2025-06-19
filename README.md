# PaperLite Communication Service 2.0

A robust, scalable microservice for handling multi-channel communications (SMS, WhatsApp, Email) with queue-based processing, Firebase integration, and comprehensive monitoring.

## 🏗️ Architecture Overview

### System Architecture

```mermaid
graph TB
    subgraph "Client Layer"
        WEB[Web App]
        MOBILE[Mobile App]
        ADMIN[Admin Panel]
        API_CLIENT[External API Clients]
    end

    subgraph "API Gateway Layer"
        LB[Load Balancer]
        CORS[CORS Middleware]
        AUTH[API Key Validation]
        VALIDATOR[Request Validator]
    end

    subgraph "Communication Service Core"
        EXPRESS[Express.js Server]
        ROUTES[Communication Routes]
        CONTROLLER[Communication Controller]
        MIDDLEWARE[Security Middleware]
    end

    subgraph "Queue Processing Layer"
        REDIS[(Redis)]
        NOTIFICATION_QUEUE[Notification Queue]
        INTERVAL_QUEUE[Interval Queue]
        WORKERS[BullMQ Workers]
    end

    subgraph "External Services"
        MSG91_SMS[MSG91 SMS]
        MSG91_WA[MSG91 WhatsApp]
        GUPSHUP[Gupshup Email]
        GMAIL[Gmail API]
    end

    subgraph "Data Layer"
        FIRESTORE[(Firebase Firestore)]
        LOGS[(Winston Logs)]
        CONFIG[Environment Config]
    end

    subgraph "Monitoring & Logging"
        WINSTON[Winston Logger]
        ERROR_HANDLER[Error Handler]
        HEALTH_CHECK[Health Monitor]
    end

    %% Client to API Gateway
    WEB --> LB
    MOBILE --> LB
    ADMIN --> LB
    API_CLIENT --> LB

    %% API Gateway Flow
    LB --> CORS
    CORS --> AUTH
    AUTH --> VALIDATOR
    VALIDATOR --> EXPRESS

    %% Core Service Flow
    EXPRESS --> ROUTES
    ROUTES --> CONTROLLER
    CONTROLLER --> MIDDLEWARE

    %% Queue Integration
    CONTROLLER --> NOTIFICATION_QUEUE
    CONTROLLER --> INTERVAL_QUEUE
    NOTIFICATION_QUEUE --> REDIS
    INTERVAL_QUEUE --> REDIS
    REDIS --> WORKERS

    %% External Service Integration
    WORKERS --> MSG91_SMS
    WORKERS --> MSG91_WA
    WORKERS --> GUPSHUP
    WORKERS --> GMAIL

    %% Data Storage
    CONTROLLER --> FIRESTORE
    WORKERS --> FIRESTORE
    EXPRESS --> LOGS

    %% Monitoring
    EXPRESS --> WINSTON
    WORKERS --> WINSTON
    WINSTON --> ERROR_HANDLER
    REDIS --> HEALTH_CHECK

    style EXPRESS fill:#e1f5fe
    style REDIS fill:#fff3e0
    style FIRESTORE fill:#f3e5f5
    style WORKERS fill:#e8f5e8
```

### Communication Flow Architecture

```mermaid
sequenceDiagram
    participant Client
    participant API as API Gateway
    participant Service as Communication Service
    participant Queue as Redis Queue
    participant Worker as BullMQ Worker
    participant Provider as External Provider
    participant DB as Firestore

    Note over Client,DB: Real-time Notification Flow

    Client->>API: POST /api/communication/send-{type}
    API->>API: Validate API Key
    API->>API: Validate Request Data
    
    API->>Service: Forward Request
    Service->>DB: Store Notification (PENDING)
    DB-->>Service: Return Document ID
    
    Service->>Queue: Add Job to Queue
    Queue-->>Service: Job Queued
    Service->>DB: Update Status (IN_QUEUE)
    Service-->>Client: Response: Job Queued
    
    Note over Queue,Provider: Asynchronous Processing
    
    Queue->>Worker: Pick Up Job
    Worker->>DB: Update Status (PROCESSING)
    
    alt SMS Notification
        Worker->>Provider: MSG91 SMS API
    else WhatsApp Notification
        Worker->>Provider: MSG91 WhatsApp API
    else Email Notification
        Worker->>Provider: Gupshup/Gmail API
    end
    
    Provider-->>Worker: Response
    
    alt Success
        Worker->>DB: Update Status (SENT)
        Worker->>DB: Store Provider Response
    else Failure
        Worker->>DB: Update Status (FAILED)
        Worker->>DB: Store Error Details
        Worker->>Queue: Retry Job (if attempts < 3)
    end
```

### Redis Queue Scheduling Architecture

```mermaid
graph TB
    subgraph "Firebase Application Startup"
        APP_START[Application Start]
        REDIS_CONNECT[Connect to Redis]
        SCHEDULE_JOBS[Schedule Interval Jobs]
    end

    subgraph "Redis Interval Queue System"
        subgraph "Queue Definitions"
            INTERVAL_QUEUE[interval-queue]
            NOTIFICATION_QUEUE[notification-queue]
        end
        
        subgraph "4 Scheduled Interval Jobs"
            HOURLY_JOB[HOURLY_QUEUE<br/>every: 3,600,000ms<br/>1 hour]
            THREE_HOURLY_JOB[THREE_HOURLY_QUEUE<br/>every: 10,800,000ms<br/>3 hours]
            SIX_HOURLY_JOB[SIX_HOURLY_QUEUE<br/>every: 21,600,000ms<br/>6 hours]
            TWELVE_HOURLY_JOB[TWELVE_HOURLY_QUEUE<br/>every: 43,200,000ms<br/>12 hours]
        end
        
        subgraph "Queue Workers"
            INTERVAL_WORKER[Interval Worker<br/>Concurrency: 5]
            NOTIFICATION_WORKER[Notification Worker<br/>Concurrency: 5]
        end
    end

    subgraph "Firebase Integration"
        subgraph "Firebase Functions"
            GET_NOTIFICATIONS[getHourlyNotifications]
            GROUP_NOTIFICATIONS[groupNotificationsByUserAndSubgroup]
            SEND_GROUPED[sendGroupedNotifications]
        end
        
        subgraph "Firebase Firestore"
            FIRESTORE_DB[(Notification Documents)]
            STATUS_UPDATE[Status Updates]
            METADATA_STORE[Metadata Storage]
        end
    end

    subgraph "External Communication Services"
        MSG91_SMS[MSG91 SMS Service]
        MSG91_WA[MSG91 WhatsApp Service]
        GUPSHUP_EMAIL[Gupshup Email Service]
        GMAIL_API[Gmail API Service]
    end

    %% Application Flow
    APP_START --> REDIS_CONNECT
    REDIS_CONNECT --> SCHEDULE_JOBS
    
    %% Job Scheduling
    SCHEDULE_JOBS --> HOURLY_JOB
    SCHEDULE_JOBS --> THREE_HOURLY_JOB
    SCHEDULE_JOBS --> SIX_HOURLY_JOB
    SCHEDULE_JOBS --> TWELVE_HOURLY_JOB

    %% Queue Processing
    HOURLY_JOB --> INTERVAL_WORKER
    THREE_HOURLY_JOB --> INTERVAL_WORKER
    SIX_HOURLY_JOB --> INTERVAL_WORKER
    TWELVE_HOURLY_JOB --> INTERVAL_WORKER

    %% Firebase Integration Flow
    INTERVAL_WORKER --> GET_NOTIFICATIONS
    GET_NOTIFICATIONS --> FIRESTORE_DB
    FIRESTORE_DB --> GROUP_NOTIFICATIONS
    GROUP_NOTIFICATIONS --> SEND_GROUPED
    
    %% Individual Notification Processing
    SEND_GROUPED --> NOTIFICATION_QUEUE
    NOTIFICATION_QUEUE --> NOTIFICATION_WORKER
    
    %% External Service Calls
    NOTIFICATION_WORKER --> MSG91_SMS
    NOTIFICATION_WORKER --> MSG91_WA
    NOTIFICATION_WORKER --> GUPSHUP_EMAIL
    NOTIFICATION_WORKER --> GMAIL_API
    
    %% Status Updates
    NOTIFICATION_WORKER --> STATUS_UPDATE
    STATUS_UPDATE --> FIRESTORE_DB

    style HOURLY_JOB fill:#e8f5e8
    style THREE_HOURLY_JOB fill:#e3f2fd
    style SIX_HOURLY_JOB fill:#fff3e0
    style TWELVE_HOURLY_JOB fill:#fce4ec
    style FIRESTORE_DB fill:#f3e5f5
    style INTERVAL_WORKER fill:#ffecb3
```

### Detailed Queue Processing Flow

```mermaid
sequenceDiagram
    participant Startup as Firebase App Startup
    participant Redis as Redis Server
    participant Scheduler as Job Scheduler
    participant Worker as Interval Worker
    participant Firebase as Firebase Functions
    participant Firestore as Firestore DB
    participant NotifQueue as Notification Queue
    participant ExtAPI as External APIs

    Note over Startup,ExtAPI: Application Initialization & Queue Setup

    Startup->>Redis: Connect to Redis
    Redis-->>Startup: Connection Established
    
    Startup->>Scheduler: Schedule 4 Interval Jobs
    
    Note over Scheduler: Setting up recurring jobs
    Scheduler->>Redis: Add HOURLY_QUEUE (every 1hr)
    Scheduler->>Redis: Add THREE_HOURLY_QUEUE (every 3hr)
    Scheduler->>Redis: Add SIX_HOURLY_QUEUE (every 6hr)
    Scheduler->>Redis: Add TWELVE_HOURLY_QUEUE (every 12hr)
    
    Redis-->>Scheduler: Jobs Scheduled Successfully
    
    Note over Redis,ExtAPI: Queue Processing Cycle (Repeats every interval)
    
    Redis->>Worker: Trigger Interval Job
    Worker->>Firebase: getHourlyNotifications interval
    Firebase->>Firestore: Query notifications by interval
    Firestore-->>Firebase: Return matching notifications
    
    Firebase->>Firebase: groupNotificationsByUserAndSubgroup
    Note over Firebase: Group notifications to reduce spam
    
    Firebase->>Firebase: sendGroupedNotifications
    
    loop For each grouped notification
        Firebase->>NotifQueue: Add individual notification job
        NotifQueue->>ExtAPI: Send via appropriate service
        ExtAPI-->>NotifQueue: Response
        NotifQueue->>Firestore: Update notification status
    end
    
    Worker-->>Redis: Job Completed
    
    Note over Redis: Wait for next interval trigger
```

### Firebase Notification Lifecycle

```mermaid
stateDiagram-v2
    [*] --> ScheduledInFirestore: Notification Created
    
    ScheduledInFirestore --> PendingInQueue: Interval Queue Triggered
    note right of PendingInQueue
        1hr, 3hr, 6hr, or 12hr
        queue picks up notification
    end note
    
    PendingInQueue --> GroupingPhase: Firebase Function Processing
    
    GroupingPhase --> UserGrouping: Group by User & Subgroup
    GroupingPhase --> TemplateConsolidation: Consolidate Templates
    GroupingPhase --> IntervalFiltering: Filter by Interval Type
    
    UserGrouping --> ReadyToSend: Grouped Notifications Ready
    TemplateConsolidation --> ReadyToSend
    IntervalFiltering --> ReadyToSend
    
    ReadyToSend --> NotificationQueue: Add to Notification Queue
    
    NotificationQueue --> ProcessingSMS: SMS Job
    NotificationQueue --> ProcessingWhatsApp: WhatsApp Job
    NotificationQueue --> ProcessingEmail: Email Job
    
    ProcessingSMS --> SentViaSMS: MSG91 SMS Success
    ProcessingSMS --> FailedSMS: MSG91 SMS Failed
    
    ProcessingWhatsApp --> SentViaWhatsApp: MSG91 WhatsApp Success
    ProcessingWhatsApp --> FailedWhatsApp: MSG91 WhatsApp Failed
    
    ProcessingEmail --> SentViaEmail: Gupshup/Gmail Success
    ProcessingEmail --> FailedEmail: Email Service Failed
    
    SentViaSMS --> StatusUpdated: Update Firestore
    SentViaWhatsApp --> StatusUpdated
    SentViaEmail --> StatusUpdated
    FailedSMS --> StatusUpdated
    FailedWhatsApp --> StatusUpdated
    FailedEmail --> StatusUpdated
    
    StatusUpdated --> [*]: Complete
    
    note right of StatusUpdated
        Firebase updates notification
        status in Firestore
    end note
```

### Redis Queue Configuration Details

```mermaid
graph TB
    subgraph "Redis Queue Configuration"
        subgraph "Connection Config"
            REDIS_URL[Redis URL]
            REDIS_HOST[Host parsedUrl.hostname]
            REDIS_PORT[Port parsedUrl.port]
            REDIS_PASSWORD[Password parsedUrl.password]
            CONNECT_TIMEOUT[connectTimeout 10000ms]
        end
        
        subgraph "Queue Settings"
            MAX_RETRIES[maxRetriesPerRequest null]
            KEEP_ALIVE[keepAlive 30000ms]
            RETRY_STRATEGY[Retry Strategy exponential]
            RECONNECT_ERROR[reconnectOnError ECONNRESET]
        end
    end

    subgraph "4 Interval Queue Jobs"
        subgraph "Job 1 Hourly"
            H1[Name HOURLY_QUEUE]
            H2[Interval 3,600,000ms]
            H3[Target NotificationInterval.ONE]
            H4[Cron Every 1 hour]
        end
        
        subgraph "Job 2 Three Hourly"
            T1[Name THREE_HOURLY_QUEUE]
            T2[Interval 10,800,000ms]
            T3[Target NotificationInterval.THREE]
            T4[Cron Every 3 hours]
        end
        
        subgraph "Job 3 Six Hourly"
            S1[Name SIX_HOURLY_QUEUE]
            S2[Interval 21,600,000ms]
            S3[Target NotificationInterval.SIX]
            S4[Cron Every 6 hours]
        end
        
        subgraph "Job 4 Twelve Hourly"
            TW1[Name TWELVE_HOURLY_QUEUE]
            TW2[Interval 43,200,000ms]
            TW3[Target NotificationInterval.TWELVE]
            TW4[Cron Every 12 hours]
        end
    end

    subgraph "Worker Configuration"
        WORKER_CONCURRENCY[Concurrency 5]
        WORKER_ATTEMPTS[Attempts 3]
        WORKER_BACKOFF[Backoff exponential 1000ms]
        WORKER_CONNECTION[Connection REDIS_CONFIG]
    end

    subgraph "Firebase Processing"
        FB_GET[getHourlyNotifications interval]
        FB_GROUP[groupNotificationsByUserAndSubgroup]
        FB_SEND[sendGroupedNotifications]
        FB_UPDATE[updateNotificationStatus]
    end

    %% Connections
    REDIS_URL --> REDIS_HOST
    REDIS_URL --> REDIS_PORT
    REDIS_URL --> REDIS_PASSWORD

    H1 --> FB_GET
    T1 --> FB_GET
    S1 --> FB_GET
    TW1 --> FB_GET

    FB_GET --> FB_GROUP
    FB_GROUP --> FB_SEND
    FB_SEND --> FB_UPDATE

    WORKER_CONCURRENCY --> H1
    WORKER_CONCURRENCY --> T1
    WORKER_CONCURRENCY --> S1
    WORKER_CONCURRENCY --> TW1

    style H1 fill:#e8f5e8
    style T1 fill:#e3f2fd
    style S1 fill:#fff3e0
    style TW1 fill:#fce4ec
    style FB_GET fill:#f3e5f5
```

### Queue Timing & Scheduling Matrix

```mermaid
gantt
    title Redis Queue Execution Schedule (24 Hour View)
    dateFormat HH:mm
    axisFormat %H:%M
    
    section Hourly Queue
    1hr Execution    :milestone, m1, 00:00, 0min
    1hr Execution    :milestone, m2, 01:00, 0min
    1hr Execution    :milestone, m3, 02:00, 0min
    1hr Execution    :milestone, m4, 03:00, 0min
    1hr Execution    :milestone, m5, 04:00, 0min
    1hr Execution    :milestone, m6, 05:00, 0min
    1hr Execution    :milestone, m7, 06:00, 0min
    1hr Execution    :milestone, m8, 07:00, 0min
    1hr Execution    :milestone, m9, 08:00, 0min
    1hr Execution    :milestone, m10, 09:00, 0min
    1hr Execution    :milestone, m11, 10:00, 0min
    1hr Execution    :milestone, m12, 11:00, 0min
    1hr Execution    :milestone, m13, 12:00, 0min
    1hr Execution    :milestone, m14, 13:00, 0min
    1hr Execution    :milestone, m15, 14:00, 0min
    1hr Execution    :milestone, m16, 15:00, 0min
    1hr Execution    :milestone, m17, 16:00, 0min
    1hr Execution    :milestone, m18, 17:00, 0min
    1hr Execution    :milestone, m19, 18:00, 0min
    1hr Execution    :milestone, m20, 19:00, 0min
    1hr Execution    :milestone, m21, 20:00, 0min
    1hr Execution    :milestone, m22, 21:00, 0min
    1hr Execution    :milestone, m23, 22:00, 0min
    1hr Execution    :milestone, m24, 23:00, 0min
    
    section 3-Hour Queue
    3hr Execution    :crit, milestone, t1, 00:00, 0min
    3hr Execution    :crit, milestone, t2, 03:00, 0min
    3hr Execution    :crit, milestone, t3, 06:00, 0min
    3hr Execution    :crit, milestone, t4, 09:00, 0min
    3hr Execution    :crit, milestone, t5, 12:00, 0min
    3hr Execution    :crit, milestone, t6, 15:00, 0min
    3hr Execution    :crit, milestone, t7, 18:00, 0min
    3hr Execution    :crit, milestone, t8, 21:00, 0min
    
    section 6-Hour Queue
    6hr Execution    :active, milestone, s1, 00:00, 0min
    6hr Execution    :active, milestone, s2, 06:00, 0min
    6hr Execution    :active, milestone, s3, 12:00, 0min
    6hr Execution    :active, milestone, s4, 18:00, 0min
    
    section 12-Hour Queue
    12hr Execution   :done, milestone, tw1, 00:00, 0min
    12hr Execution   :done, milestone, tw2, 12:00, 0min
```

### Firebase Queue Integration Architecture

```mermaid
graph TB
    subgraph "Firebase Application Layer"
        subgraph "Express Server"
            EXPRESS_APP[Express App]
            MIDDLEWARE_STACK[Middleware Stack]
            ROUTES[Communication Routes]
        end
        
        subgraph "Firebase Functions Environment"
            FIREBASE_RUNTIME[Firebase Functions Runtime]
            NODE_PROCESS[Node.js Process]
            ENV_CONFIG[Environment Configuration]
        end
    end

    subgraph "Redis Queue Management"
        subgraph "Queue Initialization"
            REDIS_CONNECTION[Redis Connection Pool]
            QUEUE_SETUP[Queue Setup on Startup]
            HEALTH_MONITOR[Health Check Every 25s]
        end
        
        subgraph "Interval Queues - 4 Types"
            QUEUE_1HR[1-Hour Queue<br/>Schedule: Every 3,600,000ms<br/>Target: NotificationInterval.ONE<br/>Repeating Job]
            QUEUE_3HR[3-Hour Queue<br/>Schedule: Every 10,800,000ms<br/>Target: NotificationInterval.THREE<br/>Repeating Job]
            QUEUE_6HR[6-Hour Queue<br/>Schedule: Every 21,600,000ms<br/>Target: NotificationInterval.SIX<br/>Repeating Job]
            QUEUE_12HR[12-Hour Queue<br/>Schedule: Every 43,200,000ms<br/>Target: NotificationInterval.TWELVE<br/>Repeating Job]
        end
        
        subgraph "Notification Processing Queue"
            NOTIF_QUEUE[Notification Queue]
            NOTIF_WORKERS[5 Concurrent Workers]
        end
    end

    subgraph "Firebase Firestore Integration"
        subgraph "Data Operations"
            QUERY_NOTIFICATIONS[getHourlyNotifications interval]
            GROUP_BY_USER[groupNotificationsByUserAndSubgroup]
            BATCH_PROCESSING[Batch Insert and Update]
            STATUS_TRACKING[Status Updates]
        end
        
        subgraph "Firestore Collections"
            PAPERLITE_NOTIF[(PAPERLITE_NOTIFICATION)]
            METADATA_DOCS[Notification Metadata]
            USER_PREFERENCES[User Notification Preferences]
        end
    end

    subgraph "External Service Integration"
        MSG91_SMS[MSG91 SMS API]
        MSG91_WHATSAPP[MSG91 WhatsApp API]
        GUPSHUP_EMAIL[Gupshup Email API]
        GMAIL_API[Gmail API]
    end

    %% Firebase to Redis Connection
    EXPRESS_APP --> REDIS_CONNECTION
    FIREBASE_RUNTIME --> QUEUE_SETUP
    
    %% Queue Scheduling
    QUEUE_SETUP --> QUEUE_1HR
    QUEUE_SETUP --> QUEUE_3HR
    QUEUE_SETUP --> QUEUE_6HR
    QUEUE_SETUP --> QUEUE_12HR
    
    %% Queue Processing Flow
    QUEUE_1HR --> QUERY_NOTIFICATIONS
    QUEUE_3HR --> QUERY_NOTIFICATIONS
    QUEUE_6HR --> QUERY_NOTIFICATIONS
    QUEUE_12HR --> QUERY_NOTIFICATIONS
    
    %% Firebase Operations
    QUERY_NOTIFICATIONS --> PAPERLITE_NOTIF
    PAPERLITE_NOTIF --> GROUP_BY_USER
    GROUP_BY_USER --> BATCH_PROCESSING
    BATCH_PROCESSING --> NOTIF_QUEUE
    
    %% Notification Processing
    NOTIF_QUEUE --> NOTIF_WORKERS
    NOTIF_WORKERS --> MSG91_SMS
    NOTIF_WORKERS --> MSG91_WHATSAPP
    NOTIF_WORKERS --> GUPSHUP_EMAIL
    NOTIF_WORKERS --> GMAIL_API
    
    %% Status Updates
    NOTIF_WORKERS --> STATUS_TRACKING
    STATUS_TRACKING --> PAPERLITE_NOTIF
    
    %% Health Monitoring
    REDIS_CONNECTION --> HEALTH_MONITOR

    style QUEUE_1HR fill:#e8f5e8
    style QUEUE_3HR fill:#e3f2fd
    style QUEUE_6HR fill:#fff3e0
    style QUEUE_12HR fill:#fce4ec
    style PAPERLITE_NOTIF fill:#f3e5f5
    style NOTIF_WORKERS fill:#ffecb3
```

### Queue Processing Implementation Details

```mermaid
flowchart TD
    subgraph "Application Startup Sequence"
        A1[Firebase Function Cold Start] --> A2[Load Environment Config]
        A2 --> A3[Connect to Redis]
        A3 --> A4[Initialize Express App]
        A4 --> A5[Setup Middleware Stack]
        A5 --> A6[Schedule 4 Interval Jobs]
    end

    subgraph "Queue Job Scheduling"
        A6 --> B1[intervalQueue.add HOURLY_QUEUE<br/>repeat: every 3600000ms]
        A6 --> B2[intervalQueue.add THREE_HOURLY_QUEUE<br/>repeat: every 10800000ms]
        A6 --> B3[intervalQueue.add SIX_HOURLY_QUEUE<br/>repeat: every 21600000ms]
        A6 --> B4[intervalQueue.add TWELVE_HOURLY_QUEUE<br/>repeat: every 43200000ms]
    end

    subgraph "Worker Processing Logic"
        B1 --> C1[intervalWorker receives job]
        B2 --> C1
        B3 --> C1
        B4 --> C1
        
        C1 --> C2{Switch on job.data.name}
        
        C2 -->|HOURLY_QUEUE| D1[getHourlyNotifications ONE]
        C2 -->|THREE_HOURLY_QUEUE| D2[getHourlyNotifications THREE]
        C2 -->|SIX_HOURLY_QUEUE| D3[getHourlyNotifications SIX]
        C2 -->|TWELVE_HOURLY_QUEUE| D4[getHourlyNotifications TWELVE]
    end

    subgraph "Firebase Processing Pipeline"
        D1 --> E1[Query Firestore by Interval]
        D2 --> E1
        D3 --> E1
        D4 --> E1
        
        E1 --> E2[groupNotificationsByUserAndSubgroup]
        E2 --> E3[sendGroupedNotifications]
        E3 --> E4[Add individual jobs to notification-queue]
    end

    subgraph "Notification Queue Processing"
        E4 --> F1[notification-queue workers]
        F1 --> F2{Notification Type}
        
        F2 -->|SMS| G1[SmsService.sendSms]
        F2 -->|WhatsApp| G2[WhatsappService.sendWhatsapp]
        F2 -->|Email| G3[EmailService.sendEmail]
        
        G1 --> H1[MSG91 SMS API Call]
        G2 --> H2[MSG91 WhatsApp API Call]
        G3 --> H3[Gupshup Gmail API Call]
        
        H1 --> I1[updateNotificationStatus]
        H2 --> I1
        H3 --> I1
        
        I1 --> I2[Update Firestore Document]
    end

    style B1 fill:#e8f5e8
    style B2 fill:#e3f2fd
    style B3 fill:#fff3e0
    style B4 fill:#fce4ec
    style E1 fill:#f3e5f5
    style F1 fill:#ffecb3
```

### Service Integration Architecture

```mermaid
graph TB
    subgraph "Communication Service Components"
        subgraph "Core Services"
            SMS_SERVICE[SMS Service]
            WA_SERVICE[WhatsApp Service]
            EMAIL_SERVICE[Email Service]
        end
        
        subgraph "Utility Services"
            API_CLIENT[API Client]
            LOGGER[Logger Service]
            HELPER[Helper Utils]
            API_KEY_MGR[API Key Manager]
        end
        
        subgraph "Configuration"
            ENV_CONFIG[Environment Config]
            REDIS_CONFIG[Redis Config]
            FIREBASE_CONFIG[Firebase Config]
            ERROR_CONFIG[Error Config]
        end
    end

    subgraph "External Provider APIs"
        subgraph "MSG91 Services"
            MSG91_SMS_ENDPOINT[SMS Endpoint]
            MSG91_WA_ENDPOINT[WhatsApp Endpoint]
        end
        
        subgraph "Email Providers"
            GUPSHUP_ENDPOINT[Gupshup Endpoint]
            GMAIL_ENDPOINT[Gmail API Endpoint]
        end
    end

    subgraph "Data Persistence"
        FIRESTORE_DB[(Firestore Database)]
        REDIS_CACHE[(Redis Cache)]
        LOG_FILES[(Log Files)]
    end

    %% Service Integrations
    SMS_SERVICE --> MSG91_SMS_ENDPOINT
    WA_SERVICE --> MSG91_WA_ENDPOINT
    EMAIL_SERVICE --> GUPSHUP_ENDPOINT
    EMAIL_SERVICE --> GMAIL_ENDPOINT

    %% Utility Integrations
    SMS_SERVICE --> API_CLIENT
    WA_SERVICE --> API_CLIENT
    EMAIL_SERVICE --> API_CLIENT
    
    API_CLIENT --> LOGGER
    SMS_SERVICE --> LOGGER
    WA_SERVICE --> LOGGER
    EMAIL_SERVICE --> LOGGER

    %% Configuration Dependencies
    SMS_SERVICE --> ENV_CONFIG
    WA_SERVICE --> ENV_CONFIG
    EMAIL_SERVICE --> ENV_CONFIG
    
    REDIS_CONFIG --> REDIS_CACHE
    FIREBASE_CONFIG --> FIRESTORE_DB
    LOGGER --> LOG_FILES

    %% Error Handling
    SMS_SERVICE --> ERROR_CONFIG
    WA_SERVICE --> ERROR_CONFIG
    EMAIL_SERVICE --> ERROR_CONFIG

    style SMS_SERVICE fill:#e3f2fd
    style WA_SERVICE fill:#e8f5e8
    style EMAIL_SERVICE fill:#fff3e0
    style FIRESTORE_DB fill:#f3e5f5
    style REDIS_CACHE fill:#ffecb3
```

### Security & Authentication Flow

```mermaid
graph TB
    subgraph "Security Layer"
        subgraph "Authentication"
            API_KEY_CHECK[API Key Validation]
            KEY_FORMAT[Key Format Check]
            ENV_CHECK[Environment Check]
            PROJECT_CHECK[Project Validation]
        end
        
        subgraph "Request Validation"
            CORS_CHECK[CORS Validation]
            INPUT_VALIDATION[Input Sanitization]
            RATE_LIMITING[Rate Limiting]
            IP_FILTERING[IP Whitelist Check]
        end
        
        subgraph "Error Handling"
            CUSTOM_ERRORS[Custom Error Classes]
            ERROR_HANDLER[Global Error Handler]
            LOGGING[Security Logging]
        end
    end

    subgraph "API Key Structure"
        STRUCTURE["PROJECT.ENVIRONMENT.KEY"]
        VALID_PROJECTS["PAPERLITE"]
        VALID_ENVS["DEV, QA, RELEASE, PREPROD, PROD"]
    end

    %% Authentication Flow
    REQUEST[Incoming Request] --> CORS_CHECK
    CORS_CHECK --> API_KEY_CHECK
    API_KEY_CHECK --> KEY_FORMAT
    KEY_FORMAT --> PROJECT_CHECK
    PROJECT_CHECK --> ENV_CHECK
    ENV_CHECK --> INPUT_VALIDATION
    INPUT_VALIDATION --> RATE_LIMITING
    RATE_LIMITING --> IP_FILTERING
    IP_FILTERING --> ALLOWED[Request Allowed]

    %% Error Paths
    API_KEY_CHECK -->|Invalid| CUSTOM_ERRORS
    KEY_FORMAT -->|Malformed| CUSTOM_ERRORS
    PROJECT_CHECK -->|Wrong Project| CUSTOM_ERRORS
    ENV_CHECK -->|Invalid Environment| CUSTOM_ERRORS
    INPUT_VALIDATION -->|Invalid Data| CUSTOM_ERRORS
    RATE_LIMITING -->|Too Many Requests| CUSTOM_ERRORS
    IP_FILTERING -->|Blocked IP| CUSTOM_ERRORS

    %% Error Handling
    CUSTOM_ERRORS --> ERROR_HANDLER
    ERROR_HANDLER --> LOGGING
    LOGGING --> REJECTED[Request Rejected]

    %% Key Structure
    STRUCTURE --> VALID_PROJECTS
    STRUCTURE --> VALID_ENVS

    style API_KEY_CHECK fill:#ffcdd2
    style CUSTOM_ERRORS fill:#ffecb3
    style ALLOWED fill:#c8e6c9
    style REJECTED fill:#ffcdd2
```

### Deployment & Infrastructure

```mermaid
graph TB
    subgraph "Development Environment"
        DEV_CODE[Source Code]
        DEV_ENV[Local Environment]
        NODEMON[Nodemon Hot Reload]
        TS_NODE[TypeScript Node]
    end

    subgraph "Build Pipeline"
        ESLINT[ESLint Validation]
        TS_COMPILE[TypeScript Compilation]
        BUILD_PROCESS[Build Process]
        ARTIFACT[Build Artifact]
    end

    subgraph "Firebase Infrastructure"
        FIREBASE_FUNCTIONS[Firebase Functions]
        FIRESTORE_PROD[Firestore Database]
        FIREBASE_HOSTING[Firebase Hosting]
        FIREBASE_AUTH[Firebase Authentication]
    end

    subgraph "External Infrastructure"
        REDIS_CLOUD[Redis Cloud Instance]
        MSG91_INFRA[MSG91 Infrastructure]
        GUPSHUP_INFRA[Gupshup Infrastructure]
        GMAIL_INFRA[Gmail API Infrastructure]
    end

    subgraph "Monitoring & Logging"
        WINSTON_LOGS[Winston Log Files]
        ERROR_TRACKING[Error Tracking]
        PERFORMANCE_MON[Performance Monitoring]
        HEALTH_CHECKS[Health Check Endpoints]
    end

    %% Development Flow
    DEV_CODE --> DEV_ENV
    DEV_ENV --> NODEMON
    NODEMON --> TS_NODE

    %% Build Flow
    DEV_CODE --> ESLINT
    ESLINT --> TS_COMPILE
    TS_COMPILE --> BUILD_PROCESS
    BUILD_PROCESS --> ARTIFACT

    %% Deployment Flow
    ARTIFACT --> FIREBASE_FUNCTIONS
    FIREBASE_FUNCTIONS --> FIRESTORE_PROD
    FIREBASE_FUNCTIONS --> FIREBASE_HOSTING

    %% External Connections
    FIREBASE_FUNCTIONS --> REDIS_CLOUD
    FIREBASE_FUNCTIONS --> MSG91_INFRA
    FIREBASE_FUNCTIONS --> GUPSHUP_INFRA
    FIREBASE_FUNCTIONS --> GMAIL_INFRA

    %% Monitoring
    FIREBASE_FUNCTIONS --> WINSTON_LOGS
    FIREBASE_FUNCTIONS --> ERROR_TRACKING
    FIREBASE_FUNCTIONS --> PERFORMANCE_MON
    FIREBASE_FUNCTIONS --> HEALTH_CHECKS

    style FIREBASE_FUNCTIONS fill:#e3f2fd
    style REDIS_CLOUD fill:#fff3e0
    style WINSTON_LOGS fill:#f3e5f5
    style DEV_ENV fill:#e8f5e8
```

## 🚀 Quick Start

### Prerequisites
- Node.js 20+
- Firebase CLI
- Redis instance
- Environment variables configured

### Installation
```bash
# Clone repository
git clone <repository-url>
cd paperlite_communication_service_2.0

# Install dependencies
cd functions
npm install

# Setup environment
cp .env.example .env.dev
# Configure your environment variables

# Start development server
npm run dev
```

### Environment Variables
```bash
# API Configuration
API_KEYS=PAPERLITE.DEV.your-dev-key,PAPERLITE.PROD.your-prod-key
PORT=3000

# Redis Configuration
REDIS_URL=redis://localhost:6379
REDIS_URL_STRING=false

# MSG91 Configuration
MSG91_AUTH_KEY=your-msg91-auth-key
MSG91_SENDER_ID=your-sender-id
MSG91_COUNTRY_CODE=91

# Email Configuration
GUPSHUP_USERID=your-gupshup-userid
GUPSHUP_PASSWORD=your-gupshup-password
GOOGLE_CLIENT_ID=your-google-client-id
GOOGLE_CLIENT_SECRET=your-google-client-secret
```

## 📊 API Endpoints

### Communication APIs
- `POST /api/communication/send-sms` - Send SMS notification
- `POST /api/communication/send-whatsapp` - Send WhatsApp message
- `POST /api/communication/send-email` - Send email notification
- `POST /api/communication/schedule-notification` - Schedule future notification
- `POST /api/communication/group-notification` - Send grouped notifications
- `POST /api/communication/paperlite-notification` - PaperLite specific notifications

### Health Check
- `GET /` - Service health check
- `GET /redis/:value` - Redis connectivity test

## 🛠️ Development Commands

```bash
# Development
npm run dev          # Start with hot reload
npm run start:dev    # Start with ts-node

# Production
npm run build        # Compile TypeScript
npm run start        # Build and start production

# Code Quality
npm run lint         # Run ESLint
```

## 📈 Monitoring & Logging

The service includes comprehensive logging with Winston:
- **Daily log rotation** with compression
- **Separate error logs** for debugging
- **Performance monitoring** with health checks
- **Queue monitoring** with job status tracking

## 🔧 Configuration

### Queue Configuration
- **Concurrency**: 5 workers per queue
- **Retry Strategy**: 3 attempts with exponential backoff
- **Health Checks**: Every 25 seconds

### Security Features
- **API Key validation** with environment-specific keys
- **Rate limiting** and IP whitelisting
- **Input validation** with express-validator
- **Custom error handling** with proper HTTP status codes
