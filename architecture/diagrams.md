# Architecture Diagrams — Multi-Brand Consent Platform

## 1. High-Level System Architecture

```mermaid
graph TB
    subgraph Web["React Admin Console (Multi-Brand)"]
        ADMIN["Brand Management"]
        FORMS["Form Builder"]
        SUBS["Submission Viewer"]
        PROVIDERS["Provider Manager"]
        TENANTS["Tenant Manager"]
        BRAND_CONFIG["Brand Configuration"]
    end

    subgraph Mobile["Public Forms (Mobile-Optimized)"]
        BRAND_PORTAL["Brand Portal"]
        SIG["Signature Capture"]
        CAM["Photo Capture"]
        WIZARD["Consent Wizard"]
        RECEIPT["Digital Receipt"]
    end

    subgraph API["Node.js / Express API Gateway"]
        BRAND_RESOLUTION["Brand Resolution\n(Hostname → Brand)"]
        ROUTES["REST Routes\n(/api/v1/admin/*, /api/v1/public/*)"]
        MAILER["Email Service\n(Brand-Aware Templates)"]
        AUTH["Auth Middleware\n(Cookie + CSRF)"]
    end

    subgraph Data["Data Layer (Enterprise)"]
        MONGO["MongoDB Atlas\n(Brand-Isolated Collections)"]
        GRIDFS["MongoDB GridFS\n(Native File Storage)"]
    end

    Web --> BRAND_RESOLUTION
    Mobile --> BRAND_RESOLUTION
    BRAND_RESOLUTION --> ROUTES
    ROUTES --> MONGO
    ROUTES --> GRIDFS
    ROUTES --> MAILER
```

## 2. Brand Resolution Flow

Multi-brand platform with hostname-based brand detection and complete data isolation.

```mermaid
sequenceDiagram
    participant Client as Client App
    participant Gateway as API Gateway
    participant BrandCache as Brand Cache
    participant DB as MongoDB Atlas

    Note over Client,DB: Brand Resolution Flow
    Client->>Gateway: Request with hostname
    Note over Gateway: Extract hostname from request
    Gateway->>BrandCache: Check cached brand config
    alt Brand in cache
        BrandCache-->>Gateway: Return brand config
    else Brand not cached
        Gateway->>DB: Query brand by hostname
        DB-->>Gateway: Return brand document
        Gateway->>BrandCache: Cache brand config
    end
    Gateway->>Gateway: Inject brand context into request
    Note over Gateway: All subsequent operations are brand-scoped
    Gateway-->>Client: Response with brand theming applied
```

## 3. Multi-Brand Client Intake Workflow

Generic consent capture flow that adapts to any industry with brand-specific terminology and requirements.

```mermaid
flowchart TD
    A["Client Arrives at Location"] --> B["Brand Portal\n(Custom Domain/Theme)"]
    B --> C["Select Consent Form\n(Industry Templates)"]
    C --> D["Step 1: Select Provider\n(Artist/Doctor/Stylist/Trainer)"]
    D --> E["Step 2: Review Legal Clauses\n(Industry-Specific)"]
    E --> F{"All Required\nClauses Acknowledged?"}
    F -->|No| E
    F -->|Yes| G["Step 3: Personal Information\n(Configurable Fields)"]
    G --> H["Enter Details: Name, Contact,\nDOB, Medical History (if req'd)"]
    H --> I{"Age ≥ Limit?"}
    I -->|No| J["Guardian Information +\nGuardian Signature"]
    I -->|Yes| K["Capture Photo\n(File Upload/Camera)"]
    J --> K
    K --> L["Digital Signature\n(Touch/Mouse Canvas)"]
    L --> M["Upload Photo → MongoDB"]
    M --> N["Upload Signature → MongoDB"]
    N --> O["POST Submission → API\n(Brand-Scoped)"]
    O --> P["Save to Brand Collection\n(Data Isolation)"]
    P --> Q["Send Email Notification\n(Brand Template)"]
    Q --> R["Confirmation Screen\n(Brand Themed)"]

    style A fill:#e8f5e9
    style R fill:#e8f5e9
    style M fill:#fff3e0
    style N fill:#fff3e0
    style P fill:#e3f2fd
    style Q fill:#fce4ec
```

## 4. Brand Admin Workflow

Platform administration enabling self-service brand management across industries.

```mermaid
flowchart TD
    LOGIN["Brand Admin Login\n(Brand-Scoped Session)"] --> DASH["Brand Dashboard"]

    DASH --> BRAND_SET["Brand Settings"]
    BRAND_SET --> THEME["Configure Theme\n(Colors, Logo, Domain)"]
    BRAND_SET --> TERMS["Industry Terminology\n(Artist vs Doctor vs Stylist)"]

    DASH --> TENANT["Tenant Management"]
    TENANT --> TENANT_ADD["Add Location\n(Clinic, Studio, Shop)"]
    TENANT --> TENANT_EDIT["Edit / Archive Location"]

    DASH --> PROV["Provider Management"]
    PROV --> PROV_ADD["Add Provider\n(License #, Specialties)"]
    PROV --> PROV_EDIT["Edit / Remove Provider"]

    DASH --> CLS["Clause Management"]
    CLS --> CLS_ADD["Add Industry Clause\n(Medical, Liability, Privacy)"]
    CLS --> CLS_EDIT["Edit / Remove Clause"]

    DASH --> FORM["Consent Form Builder"]
    FORM --> F1["Step 1: Form Settings\n(Name, Industry Template,\nAge Limit, Required Fields)"]
    F1 --> F2["Step 2: Assign Providers\n(Who can use this form)"]
    F2 --> F3["Step 3: Assign Clauses\n(Legal requirements)"]
    F3 --> SAVE["Publish Form\n(Make Available Publicly)"]

    DASH --> VIEW["Submission Management"]
    VIEW --> SEARCH["Filter Submissions\n(Provider, Date, Location)"]
    SEARCH --> DETAIL["View Details\n(Signature, Photo, Compliance)"]

    DASH --> USERS["User Management"]
    USERS --> USER_ADD["Invite Brand Admins"]
    USERS --> USER_ROLES["Manage Roles & Access"]
```

## 5. Platform CI/CD Pipeline

```mermaid
flowchart LR
    DEV["Developer\nPush"] --> GIT["GitHub\nRepository"]
    GIT --> BUILD["Build & Test\n(TypeScript, Jest)"]
    BUILD --> TEST["Integration Tests\n(Multi-Brand Isolation)"]
    TEST --> DEPLOY_API["Deploy API Gateway\n(Brand Resolution)"]
    TEST --> DEPLOY_ADMIN["Deploy Admin Interface\n(Dynamic Theming)"]
    TEST --> DEPLOY_PUBLIC["Deploy Public Forms\n(Mobile-Optimized)"]
    DEPLOY_API --> PROD_API["Production Platform\n(Multiple Brand Domains)"]
    DEPLOY_ADMIN --> PROD_ADMIN["Brand Admin Interfaces"]
    DEPLOY_PUBLIC --> PROD_PUBLIC["Public Consent Forms"]
```

## 6. Multi-Brand Database Architecture

```mermaid
erDiagram
    BRAND ||--o{ TENANT : "contains"
    BRAND ||--o{ ADMIN_USER : "manages"
    BRAND ||--o{ PROVIDER : "employs"
    BRAND ||--o{ CONSENT_FORM : "defines"
    BRAND ||--o{ CLAUSE : "owns"
    
    TENANT ||--o{ PROVIDER : "assigned to"
    TENANT ||--o{ SUBMISSION : "receives"
    
    CONSENT_FORM ||--o{ FORM_PROVIDER : "available to"
    CONSENT_FORM ||--o{ FORM_CLAUSE : "includes"
    CONSENT_FORM ||--o{ SUBMISSION : "generates"
    
    SUBMISSION }|--|| PROVIDER : "processed by"
    SUBMISSION ||--|| MONGODB_FILE : "contains media"

    BRAND {
        ObjectId _id PK
        string name
        string slug
        string industry
        object theme
        array customDomains
        object emailConfig
        boolean active
        DateTime createdAt
    }

    TENANT {
        ObjectId _id PK
        ObjectId brandId FK
        string name
        string slug
        object address
        object contactInfo
        object settings
    }

    ADMIN_USER {
        ObjectId _id PK
        ObjectId brandId FK
        string email
        string name
        array roles
        array tenantScope
        string hashedPassword
    }

    PROVIDER {
        ObjectId _id PK
        ObjectId brandId FK
        ObjectId tenantId FK
        string name
        string licenseNumber
        array specialties
        string status
    }

    CONSENT_FORM {
        ObjectId _id PK
        ObjectId brandId FK
        string name
        string slug
        string status
        object capture
        object presentation
        object notifications
    }

    CLAUSE {
        ObjectId _id PK
        ObjectId brandId FK
        string title
        string content
        string type
        boolean required
        string industry
    }

    SUBMISSION {
        ObjectId _id PK
        ObjectId brandId FK
        ObjectId tenantId FK
        ObjectId formId FK
        ObjectId providerId FK
        object subject
        object guardian
        object photos
        object signatures
        array acknowledgedClauses
        DateTime submittedAt
    }

    MONGODB_FILE {
        ObjectId _id PK
        ObjectId brandId FK
        string filename
        string contentType
        number length
        DateTime uploadDate
    }
```
