# Architecture Diagrams — Tattoo Release Online (TRO)

## 1. High-Level System Architecture

```mermaid
graph TB
    subgraph Web["React Admin Console"]
        ADMIN["Shop Management"]
        FORMS["Form Builder"]
        SUBS["Submission Viewer"]
        ARTISTS["Artist Manager"]
    end

    subgraph Mobile["React Native Kiosk App"]
        KIOSK["Client Intake"]
        SIG["Signature Capture"]
        CAM["Photo Capture"]
        WIZARD["Consent Wizard"]
    end

    subgraph API["Node.js / Express API"]
        AUTH["Auth Middleware\n(API Key + JWT)"]
        ROUTES["REST Routes\n(/api/v1/Consent/*)"]
        MAILER["Email Service\n(Gmail OAuth2)"]
        SWAGGER["Swagger Docs"]
    end

    subgraph Data["Data Layer"]
        MONGO["MongoDB Atlas\n(Multi-tenant)"]
        CLOUD["Cloudinary\n(Photos + Signatures)"]
    end

    Web --> AUTH
    Mobile --> AUTH
    AUTH --> ROUTES
    ROUTES --> MONGO
    ROUTES --> CLOUD
    ROUTES --> MAILER
```

## 2. Authentication Flow

Dual authentication model — admin accounts and kiosk accounts use the same PBKDF2 algorithm but separate credential stores and login endpoints.

```mermaid
sequenceDiagram
    participant Client as Web / Mobile Client
    participant API as Express API
    participant DB as MongoDB Atlas

    Note over Client,DB: Admin Login Flow
    Client->>API: POST /getUserSalt {username}
    API->>DB: Find user by Userid
    DB-->>API: {Salt}
    API-->>Client: {Salt}
    Note over Client: PBKDF2-SHA512(password, salt, 1000 iter, 64 bytes)
    Client->>API: POST /login {username, hash, x-api-key}
    API->>DB: Find user, compare Hash
    DB-->>API: User document (with nested artists, clauses, forms)
    API-->>Client: {user, apiKey, JWT (7-day)}

    Note over Client,DB: Kiosk Login Flow
    Client->>API: POST /getkioskusersalt {username}
    API->>DB: Find kiosk user within tenant
    DB-->>API: {KioskSalt}
    API-->>Client: {Salt}
    Note over Client: PBKDF2-SHA512(password, salt, 1000 iter, 64 bytes)
    Client->>API: POST /loginkioskusermobile {username, hash}
    API->>DB: Validate kiosk credentials
    DB-->>API: Kiosk user + tenant context
    API->>DB: Check subscription status
    DB-->>API: {SubscriptionDate, SubscriptionStatus}
    API-->>Client: {user, role:"Kiosk", cloudinary creds, apiKey}
```

## 3. Client Intake Workflow (Kiosk)

The complete flow from client arrival to stored consent submission.

```mermaid
flowchart TD
    A["Client Arrives at Shop"] --> B["Kiosk Tablet\n(Already Logged In)"]
    B --> C["Select Consent Form"]
    C --> D["Step 1: Select Artist"]
    D --> E["Step 2: Review Legal Clauses"]
    E --> F{"All Required\nClauses Acknowledged?"}
    F -->|No| E
    F -->|Yes| G["Step 3: Personal Information"]
    G --> H["Enter: Name, Address,\nPhone, Email, DOB"]
    H --> I{"Age ≥ Limit?"}
    I -->|No| J["Guardian Info +\nGuardian Signature"]
    I -->|Yes| K["Capture Photo\n(expo-camera)"]
    J --> K
    K --> L["Digital Signature\n(react-native-signature-canvas)"]
    L --> M["Upload Photo → Cloudinary"]
    M --> N["Upload Signature → Cloudinary"]
    N --> O["POST Submission → API"]
    O --> P["Save to MongoDB\n(ConsentSubmission)"]
    P --> Q["Send Email Notification\n(Gmail OAuth2)"]
    Q --> R["Confirmation Screen"]

    style A fill:#e8f5e9
    style R fill:#e8f5e9
    style M fill:#fff3e0
    style N fill:#fff3e0
    style P fill:#e3f2fd
    style Q fill:#fce4ec
```

## 4. Admin Workflow

Shop owner management flows through the admin console.

```mermaid
flowchart TD
    LOGIN["Admin Login\n(Web Console)"] --> DASH["Dashboard"]

    DASH --> ART["Artist Management"]
    ART --> ART_ADD["Add Artist\n(Name, License #)"]
    ART --> ART_EDIT["Edit / Remove Artist"]

    DASH --> CLS["Clause Management"]
    CLS --> CLS_ADD["Add Clause\n(Title, Text, Type)"]
    CLS --> CLS_EDIT["Edit / Remove Clause"]

    DASH --> FORM["Consent Form Builder"]
    FORM --> F1["Step 1: Form Settings\n(Name, Header, Camera,\nAge Limit, Email Config)"]
    F1 --> F2["Step 2: Assign Artists\n(Include/Exclude per form)"]
    F2 --> F3["Step 3: Assign Clauses\n(Required/Optional per form)"]
    F3 --> SAVE["Save Form Configuration"]

    DASH --> VIEW["Submission Viewer"]
    VIEW --> SEARCH["Search Submissions\n(Name, Artist, Date Range)"]
    SEARCH --> DETAIL["View Detail\n(Signature, Photo, Clauses)"]

    DASH --> KU["Kiosk User Admin"]
    KU --> KU_ADD["Create Kiosk Account"]
    KU --> KU_EDIT["Edit / Deactivate"]

    DASH --> SUB["Subscription Status"]
    SUB --> TIER["View Tier & Expiry"]
```

## 5. CI/CD Pipeline

```mermaid
flowchart LR
    DEV["Developer\nPush"] --> GIT["GitHub\nRepository"]
    GIT --> BUILD["Build Step\n(npm run build)"]
    BUILD --> TEST["Test Step\n(Jest)"]
    TEST --> DEPLOY_API["Deploy API\n(Node.js)"]
    TEST --> DEPLOY_WEB["Deploy Web\n(Static Assets)"]
    TEST --> DEPLOY_MOBILE["Expo Build\n(iOS / Android)"]
    DEPLOY_API --> PROD_API["Production API\nconsent.tattooreleaseonline.com"]
    DEPLOY_WEB --> PROD_WEB["Production Web\n(Admin Console)"]
    DEPLOY_MOBILE --> STORES["App Distribution"]
```

## 6. Database Entity Relationship

```mermaid
erDiagram
    TENANT ||--o{ ARTIST : "embeds"
    TENANT ||--o{ CLAUSE : "embeds"
    TENANT ||--o{ CONSENT_FORM : "embeds"
    TENANT ||--o{ KIOSK_USER : "embeds"
    TENANT ||--o{ SUBMISSION : "references"
    CONSENT_FORM ||--o{ INCLUDED_ARTIST : "includes"
    CONSENT_FORM ||--o{ INCLUDED_CLAUSE : "includes"
    SUBMISSION ||--|| CONSENT_FORM : "based on"
    SUBMISSION }|--|| CLOUDINARY_ASSET : "stores media"

    TENANT {
        string Userid PK
        string Email
        string Company
        string Fname
        string Lname
        string Role
        string Hash
        string Salt
        boolean SubscriptionStatus
        string SubscriptionType
        number SubscriptionPrice
        string CustomerXRef
    }

    ARTIST {
        ObjectId _id PK
        string Fname
        string Lname
        string LicenseNbr
    }

    CLAUSE {
        ObjectId _id PK
        string Title
        string Clause
        string Type
        boolean Required
    }

    CONSENT_FORM {
        ObjectId _id PK
        string Name
        boolean Enabled
        string Header
        string TitleImage
        boolean EnableCamera
        boolean PhotoRequired
        number AgeLimit
        boolean AskRelation
        string EmailTo
        string EmailSubject
    }

    KIOSK_USER {
        ObjectId _id PK
        string Userid
        string Hash
        string Salt
        string Role
    }

    SUBMISSION {
        ObjectId _id PK
        string TenantId FK
        string ConsentId FK
        string ConsentFormName
        string Date
        string Artist
        string ConsentName
        string ConsentSignature
        string ConsentSignaturePublicId
        string ConsentPhotoId
        string ConsentPhotoIdPublicId
        string ConsentGuardianName
        string ConsentGuardianSignature
    }

    CLOUDINARY_ASSET {
        string public_id PK
        string secure_url
        string resource_type
    }
```
