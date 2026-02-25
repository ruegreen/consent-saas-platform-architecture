# Pattern: Database Schema Design

## Schema Strategy: Embedding vs Referencing

TRO uses a hybrid approach — tenant-owned entities are embedded as sub-documents, while high-volume entities (submissions) are stored in a separate collection with tenant ID references.

### Why This Split

| Entity | Strategy | Reason |
|---|---|---|
| Artists | Embedded in Tenant | Bounded cardinality (tens per shop), always accessed with tenant context |
| Clauses | Embedded in Tenant | Same as artists — bounded, tenant-scoped |
| Consent Forms | Embedded in Tenant | Small count per tenant, includes form config |
| Kiosk Users | Embedded in Tenant | Few per shop (1 per tablet) |
| Submissions | Separate Collection | Unbounded growth, needs pagination/filtering/date-range queries |
| Subscriptions | Separate Collection | Event log pattern, queried independently |

### Tenant Document (User Schema)

```javascript
const UserSchema = new Schema({
    // Identity
    Userid: { type: String, unique: true, required: true },
    Email: String,
    Fname: String,
    Lname: String,
    Company: String,

    // Address
    Address: String,
    City: String,
    State: String,
    Country: String,
    Postal: String,
    Phone: String,

    // Role & Access
    Role: String,  // "Admin" or "User"

    // Security (PBKDF2)
    Hash: String,
    Salt: String,

    // Subscription
    SubscriptionDate: String,
    SubscriptionStatus: Boolean,
    SubscriptionType: String,     // "Inactive", "Monthly", "Yearly"
    SubscriptionMultiShopCount: Number,
    SubscriptionPrice: Number,    // Cents
    CustomerXRef: String,         // Payment processor reference

    // Embedded sub-documents
    Artists: [ArtistSchema],
    Clauses: [ClauseSchema],
    ConsentForms: [ConsentSchema],
    Kioskusers: [KioskuserSchema]
}, { timestamps: true });
```

### Artist Sub-Document

```javascript
const ArtistSchema = new Schema({
    Fname: String,
    Lname: String,
    LicenseNbr: String  // State license number
});
```

### Clause Sub-Document

```javascript
const ClauseSchema = new Schema({
    Title: String,
    Clause: String,      // Full legal text
    Type: String,        // Classification (liability, medical, etc.)
    Required: Boolean    // Must acknowledge to proceed
});
```

### Consent Form Sub-Document

```javascript
const ConsentSchema = new Schema({
    Name: String,
    Enabled: Boolean,
    Header: String,
    TitleImage: String,           // Cloudinary URL

    // Shop info
    Company: String,

    // Email configuration
    EmailTo: String,
    EmailFrom: String,
    EmailSubject: String,
    EmailText: String,

    // Form behavior
    EnableCamera: Boolean,
    PhotoRequired: Boolean,
    AgeLimit: Number,             // Triggers guardian consent
    AskRelation: Boolean,         // Ask guardian relationship
    EnableNotary: Boolean,

    // Headers/Footers
    ParentHeader: String,
    ConsentFooter: String,

    // Required fields (toggles)
    RequireName: Boolean,
    RequireEmail: Boolean,
    RequireAddress: Boolean,
    RequirePhone: Boolean,
    RequireNationality: Boolean,
    RequireGender: Boolean,

    // Entity assignments (included in this form)
    IncludedArtists: [{
        _id: Schema.Types.ObjectId,
        Fname: String,
        Lname: String,
        Included: Boolean          // Artist available on this form
    }],
    IncludedClauses: [{
        _id: Schema.Types.ObjectId,
        Title: String,
        Clause: String,
        Type: String,
        Required: Boolean          // Must acknowledge on this form
    }]
});
```

### Consent Submission (Separate Collection)

```javascript
const ConsentSubmissionSchema = new Schema({
    // References
    TenantId: String,
    ConsentId: String,
    ConsentFormName: String,

    // Session
    Date: String,
    Artist: String,

    // Clause acknowledgments
    ConsentClauses: [{
        Title: String,
        Clause: String,
        Type: String,
        Required: String,
        Acknowledged: Boolean      // Client checked the box
    }],

    // Personal information
    ConsentName: String,
    ConsentAddress: String,
    ConsentPhone: String,
    ConsentEmail: String,
    ConsentNationality: String,
    ConsentGender: String,
    ConsentDOB: String,
    ConsentAge: String,

    // Media (Cloudinary references)
    ConsentPhotoId: String,              // Secure URL
    ConsentPhotoIdPublicId: String,      // Cloudinary asset ID
    ConsentSignature: String,            // Secure URL
    ConsentSignaturePublicId: String,    // Cloudinary asset ID

    // Guardian (when under age limit)
    ConsentGuardianName: String,
    ConsentGuardianRelation: String,
    ConsentGuardianSignature: String,
    ConsentGuardianSignaturePublicId: String,

    // Notes & Footer
    ConsentNotes: String,
    ConsentFooter: String,

    // Auth context (who submitted)
    UserHash: String,
    UserSalt: String,
    KioskHash: String,
    KioskSalt: String
}, { timestamps: true });
```

## Query Patterns

### Tenant Context (Single Query Gets Everything)

```javascript
// One query returns the entire tenant context — artists, clauses,
// forms, kiosk users — because they're embedded sub-documents
const tenant = await User.findById(tenantId);
// tenant.Artists, tenant.Clauses, tenant.ConsentForms all available
```

### Sub-Document Update (Positional Operator)

```javascript
// Update a specific nested artist using MongoDB positional operator
await User.findOneAndUpdate(
    { "Artists._id": artistId },
    { "$set": { "Artists.$": updatedArtist } },
    { new: true }
);

// Remove a nested clause using $pull
await User.findOneAndUpdate(
    { "Clauses._id": clauseId },
    { "$pull": { "Clauses": { "_id": clauseId } } },
    { new: true }
);
```

### Submission Queries (Separate Collection with Filters)

```javascript
// Recent submissions (default view)
ConsentSubmission.find({ TenantId: tenantId })
    .sort({ $natural: -1 })
    .limit(50);

// Search by client name (case-insensitive regex)
ConsentSubmission.find({
    TenantId: tenantId,
    ConsentName: { $regex: searchTerm, $options: 'i' }
}).limit(50);

// Date range query (timezone-aware)
ConsentSubmission.find({
    TenantId: tenantId,
    createdAt: {
        $gte: new Date(startDate),
        $lte: new Date(endDate)
    }
}).limit(50);
```

## Multi-Tenant Isolation

Tenant isolation is enforced through document structure:

1. **Artists, Clauses, Forms, Kiosk Users** — embedded within the tenant document. No query can accidentally cross tenant boundaries because there's no shared collection.
2. **Submissions** — stored in a shared collection but always filtered by `TenantId`. Every submission query includes the tenant context.
3. **API layer** — the tenant ID comes from the authenticated session, not from user input, preventing parameter tampering.
