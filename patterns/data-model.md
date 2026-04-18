# Pattern: Database Schema Design

## Multi-Brand Data Architecture

The consent platform uses a brand-first data model where every entity is scoped to a specific brand, providing complete data isolation and enabling unlimited brand scaling. The architecture balances performance, security, and maintainability through strategic use of references and brand-scoped collections.

### Brand Isolation Strategy

| Entity | Strategy | Reason |
|---|---|---|
| Brands | Root collection | Platform-level entities, minimal count |
| Tenants | Separate collection, brand-scoped | Bounded per brand, needs independent queries |
| Providers | Separate collection, brand-scoped | Bounded cardinality, tenant assignment flexibility |
| Admin Users | Separate collection, brand-scoped | Security isolation, role-based access |
| Consent Forms | Separate collection, brand-scoped | Independent lifecycle, publication workflow |
| Clauses | Separate collection, brand-scoped | Reusable across forms, versioning support |
| Submissions | Separate collection, brand-scoped | Unbounded growth, requires pagination and analytics |
| Files | GridFS with brand buckets | Complete file isolation, security boundary |

### Brand Root Schema

```typescript
const BrandSchema = new Schema({
  // Identity
  name: { type: String, required: true },
  slug: { type: String, unique: true, required: true },
  displayName: String,
  
  // Industry and configuration
  industry: {
    type: String,
    enum: ['tattoo', 'medical', 'salon', 'fitness', 'spa', 'beauty', 'other'],
    required: true
  },
  
  // Domain configuration
  domains: {
    platformDomain: String,      // brand.app.platform.com
    customDomains: [String],     // custom.domain.com
    primaryDomain: String        // Which domain is primary
  },
  
  // Visual theming
  theme: {
    primaryColor: { type: String, default: '#007bff' },
    secondaryColor: { type: String, default: '#6c757d' },
    textColor: { type: String, default: '#212529' },
    logoUrl: String,
    faviconUrl: String,
    fontFamily: String
  },
  
  // Industry-specific terminology
  terminology: {
    provider: { type: String, default: 'Provider' },    // Artist, Doctor, Stylist
    tenant: { type: String, default: 'Location' },      // Studio, Clinic, Salon
    service: { type: String, default: 'Service' },      // Tattoo, Treatment, Session
    client: { type: String, default: 'Client' }         // Patient, Customer, Guest
  },
  
  // Email configuration
  emailConfig: {
    fromAddress: String,         // noreply@brand.com
    fromName: String,           // "Brand Name Team"
    replyTo: String,            // support@brand.com
    supportEmail: String,
    customTemplates: Boolean    // Use custom email templates
  },
  
  // Platform settings
  settings: {
    allowSelfSignup: { type: Boolean, default: false },
    requireEmailVerification: { type: Boolean, default: true },
    dataRetentionDays: { type: Number, default: 2555 }, // 7 years
    supportPhone: String,
    privacyPolicyUrl: String,
    termsOfServiceUrl: String,
    complianceLevel: {
      type: String,
      enum: ['basic', 'healthcare', 'financial'],
      default: 'basic'
    }
  },
  
  // Subscription and billing
  subscription: {
    tier: {
      type: String,
      enum: ['trial', 'starter', 'professional', 'enterprise'],
      default: 'trial'
    },
    status: {
      type: String,
      enum: ['active', 'suspended', 'cancelled'],
      default: 'active'
    },
    expiresAt: Date,
    billingEmail: String,
    maxTenants: Number,
    maxUsers: Number,
    customBranding: Boolean
  },
  
  // Platform metadata
  status: {
    type: String,
    enum: ['active', 'suspended', 'archived'],
    default: 'active'
  },
  createdBy: { type: Schema.Types.ObjectId, ref: 'AdminUser' },
  
}, { 
  timestamps: true,
  collection: 'brands' 
});

// Indexes for performance
BrandSchema.index({ slug: 1 }, { unique: true });
BrandSchema.index({ 'domains.platformDomain': 1 });
BrandSchema.index({ 'domains.customDomains': 1 });
BrandSchema.index({ status: 1, 'subscription.status': 1 });
```

### Tenant Schema (Locations within Brands)

```typescript
const TenantSchema = new Schema({
  // Brand relationship
  brandId: { type: Schema.Types.ObjectId, ref: 'Brand', required: true },
  
  // Identity
  name: { type: String, required: true },
  slug: { type: String, required: true },
  displayName: String,
  description: String,
  
  // Address information
  address: {
    street: String,
    street2: String,
    city: String,
    state: String,
    postalCode: String,
    country: { type: String, default: 'US' },
    timezone: String
  },
  
  // Contact information
  contactInfo: {
    phone: String,
    email: String,
    website: String,
    socialMedia: {
      instagram: String,
      facebook: String,
      twitter: String
    }
  },
  
  // Business hours
  hours: {
    monday: { open: String, close: String, closed: Boolean },
    tuesday: { open: String, close: String, closed: Boolean },
    wednesday: { open: String, close: String, closed: Boolean },
    thursday: { open: String, close: String, closed: Boolean },
    friday: { open: String, close: String, closed: Boolean },
    saturday: { open: String, close: String, closed: Boolean },
    sunday: { open: String, close: String, closed: Boolean }
  },
  
  // Tenant-specific settings
  settings: {
    requirePhoneNumber: { type: Boolean, default: false },
    requireAddress: { type: Boolean, default: false },
    defaultAgeLimit: { type: Number, default: 18 },
    allowWalkIns: { type: Boolean, default: true },
    sendConfirmationEmails: { type: Boolean, default: true },
    customInstructions: String
  },
  
  // Status and metadata
  status: {
    type: String,
    enum: ['active', 'inactive', 'archived'],
    default: 'active'
  }
  
}, { 
  timestamps: true,
  collection: 'tenants'
});

// Brand-scoped indexes
TenantSchema.index({ brandId: 1, slug: 1 }, { unique: true });
TenantSchema.index({ brandId: 1, status: 1 });
```

### Provider Schema (Staff/Artists/Doctors within Brands)

```typescript
const ProviderSchema = new Schema({
  // Brand and tenant relationship
  brandId: { type: Schema.Types.ObjectId, ref: 'Brand', required: true },
  tenantId: { type: Schema.Types.ObjectId, ref: 'Tenant', required: true },
  
  // Identity
  name: { type: String, required: true },
  email: String,
  phone: String,
  
  // Professional information
  licenseNumber: String,
  licenseState: String,
  licenseExpiration: Date,
  certifications: [String],
  specialties: [String],        // Industry-specific (styles, procedures, etc.)
  
  // Availability
  schedule: {
    monday: { start: String, end: String, available: Boolean },
    tuesday: { start: String, end: String, available: Boolean },
    wednesday: { start: String, end: String, available: Boolean },
    thursday: { start: String, end: String, available: Boolean },
    friday: { start: String, end: String, available: Boolean },
    saturday: { start: String, end: String, available: Boolean },
    sunday: { start: String, end: String, available: Boolean }
  },
  
  // Bio and portfolio
  bio: String,
  portfolioImages: [String],    // File URLs from MongoDB GridFS
  
  // Settings
  settings: {
    allowOnlineBooking: { type: Boolean, default: true },
    requireConsultation: { type: Boolean, default: false },
    customInstructions: String
  },
  
  status: {
    type: String,
    enum: ['active', 'inactive', 'archived'],
    default: 'active'
  }
  
}, { 
  timestamps: true,
  collection: 'providers'
});

// Performance indexes
ProviderSchema.index({ brandId: 1, tenantId: 1, status: 1 });
ProviderSchema.index({ brandId: 1, licenseNumber: 1 });
```

### Consent Form Schema (Brand-Specific Templates)

```typescript
const ConsentFormSchema = new Schema({
  // Brand relationship
  brandId: { type: Schema.Types.ObjectId, ref: 'Brand', required: true },
  
  // Form identity
  name: { type: String, required: true },
  slug: { type: String, required: true },
  description: String,
  version: { type: Number, default: 1 },
  
  // Form status and lifecycle
  status: {
    type: String,
    enum: ['draft', 'published', 'archived'],
    default: 'draft'
  },
  publishedAt: Date,
  publishedBy: { type: Schema.Types.ObjectId, ref: 'AdminUser' },
  
  // Data capture configuration
  capture: {
    requireName: { type: Boolean, default: true },
    requireEmail: { type: Boolean, default: false },
    requirePhone: { type: Boolean, default: false },
    requireAddress: { type: Boolean, default: false },
    requireDateOfBirth: { type: Boolean, default: true },
    requirePhoto: { type: Boolean, default: false },
    requireSignature: { type: Boolean, default: true },
    ageLimit: { type: Number, default: 18 },
    allowGuardianConsent: { type: Boolean, default: true },
    
    // Dynamic fields for industry-specific data
    customFields: [{
      name: String,
      label: String,
      type: {
        type: String,
        enum: ['text', 'email', 'phone', 'date', 'select', 'checkbox', 'textarea']
      },
      required: Boolean,
      options: [String],        // For select fields
      validation: String,       // Regex pattern
      helpText: String
    }]
  },
  
  // Form presentation
  presentation: {
    title: String,
    description: String,
    instructions: String,
    titleImage: String,         // GridFS file URL
    headerColor: String,
    showBrandLogo: { type: Boolean, default: true },
    customCss: String,
    afterSubmitMessage: String,
    thankYouPage: String
  },
  
  // Email notifications
  notifications: {
    sendToProvider: { type: Boolean, default: false },
    sendToTenant: { type: Boolean, default: true },
    sendToClient: { type: Boolean, default: false },
    customSubject: String,
    customMessage: String,
    recipients: [String]        // Additional email addresses
  },
  
  // Provider and clause assignments
  assignedProviders: [{ type: Schema.Types.ObjectId, ref: 'Provider' }],
  assignedClauses: [{ type: Schema.Types.ObjectId, ref: 'Clause' }],
  
  // Analytics and usage
  usage: {
    totalSubmissions: { type: Number, default: 0 },
    lastSubmissionAt: Date,
    averageCompletionTime: Number // seconds
  }
  
}, { 
  timestamps: true,
  collection: 'consentForms'
});

// Brand and slug uniqueness
ConsentFormSchema.index({ brandId: 1, slug: 1 }, { unique: true });
ConsentFormSchema.index({ brandId: 1, status: 1 });
ConsentFormSchema.index({ assignedProviders: 1 });
```

### Consent Submission Schema (Client Data)

```typescript
const ConsentSubmissionSchema = new Schema({
  // Brand and relationship context
  brandId: { type: Schema.Types.ObjectId, ref: 'Brand', required: true },
  tenantId: { type: Schema.Types.ObjectId, ref: 'Tenant', required: true },
  formId: { type: Schema.Types.ObjectId, ref: 'ConsentForm', required: true },
  providerId: { type: Schema.Types.ObjectId, ref: 'Provider' },
  
  // Subject information
  subject: {
    name: String,
    email: String,
    phone: String,
    dateOfBirth: Date,
    address: {
      street: String,
      city: String,
      state: String,
      postalCode: String
    },
    customFields: Map         // Dynamic field values
  },
  
  // Guardian information (when subject is under age limit)
  guardian: {
    name: String,
    relationship: String,
    phone: String,
    email: String
  },
  
  // Legal clause acknowledgments
  acknowledgedClauses: [{
    clauseId: { type: Schema.Types.ObjectId, ref: 'Clause' },
    title: String,
    content: String,          // Snapshot of clause at submission time
    acknowledged: Boolean,
    acknowledgedAt: Date
  }],
  
  // Media captures
  photos: {
    subject: String,          // GridFS file URL
    identification: String,   // ID verification photo
    additional: [String]      // Additional photos if required
  },
  
  signatures: {
    subject: String,          // GridFS file URL
    guardian: String          // Guardian signature if applicable
  },
  
  // Session and metadata
  sessionInfo: {
    ip: String,
    userAgent: String,
    submissionSource: {
      type: String,
      enum: ['web', 'mobile', 'kiosk', 'api'],
      default: 'web'
    },
    completionTime: Number,   // Time to complete form (seconds)
    referrer: String
  },
  
  // Processing status
  status: {
    type: String,
    enum: ['completed', 'pending', 'archived', 'deleted'],
    default: 'completed'
  },
  
  // Notes and follow-up
  notes: String,
  followUpRequired: Boolean,
  followUpDate: Date,
  
  // Compliance and audit
  complianceFlags: [String],
  auditTrail: [{
    action: String,
    performedBy: { type: Schema.Types.ObjectId, ref: 'AdminUser' },
    performedAt: { type: Date, default: Date.now },
    details: String
  }]
  
}, { 
  timestamps: true,
  collection: 'consentSubmissions'
});

// Performance indexes for common queries
ConsentSubmissionSchema.index({ brandId: 1, submittedAt: -1 });
ConsentSubmissionSchema.index({ brandId: 1, tenantId: 1, submittedAt: -1 });
ConsentSubmissionSchema.index({ brandId: 1, formId: 1, submittedAt: -1 });
ConsentSubmissionSchema.index({ brandId: 1, providerId: 1, submittedAt: -1 });
ConsentSubmissionSchema.index({ 'subject.email': 1 });
ConsentSubmissionSchema.index({ 'subject.phone': 1 });
```

## Query Patterns

### Brand-Scoped Operations

```typescript
// All database operations automatically include brand context
class BrandScopedRepository<T> {
  constructor(
    private model: mongoose.Model<T>,
    private brandId: string
  ) {}

  async find(filter: any = {}): Promise<T[]> {
    return this.model.find({
      brandId: this.brandId,
      ...filter
    });
  }

  async findById(id: string): Promise<T | null> {
    return this.model.findOne({
      _id: id,
      brandId: this.brandId
    });
  }

  async create(data: Partial<T>): Promise<T> {
    return this.model.create({
      ...data,
      brandId: this.brandId
    });
  }

  async update(id: string, update: Partial<T>): Promise<T | null> {
    return this.model.findOneAndUpdate(
      { _id: id, brandId: this.brandId },
      update,
      { new: true }
    );
  }

  async delete(id: string): Promise<boolean> {
    const result = await this.model.deleteOne({
      _id: id,
      brandId: this.brandId
    });
    return result.deletedCount > 0;
  }
}

// Usage in API controllers
export class TenantsController {
  async list(req: BrandRequest, res: Response) {
    const repository = new BrandScopedRepository(Tenant, req.brandId);
    const tenants = await repository.find({ status: 'active' });
    res.json(tenants);
  }

  async create(req: BrandRequest, res: Response) {
    const repository = new BrandScopedRepository(Tenant, req.brandId);
    const tenant = await repository.create(req.body);
    res.json(tenant);
  }
}
```

### Submission Analytics (Brand-Scoped)

```typescript
// Analytics queries with brand isolation
export class SubmissionAnalytics {
  constructor(private brandId: string) {}

  async getSubmissionStats(timeframe: 'day' | 'week' | 'month' | 'year') {
    const startDate = this.getStartDate(timeframe);
    
    const pipeline = [
      // Brand isolation first
      { $match: { 
          brandId: new mongoose.Types.ObjectId(this.brandId),
          submittedAt: { $gte: startDate }
        }
      },
      
      // Group by time period
      { $group: {
          _id: {
            year: { $year: '$submittedAt' },
            month: { $month: '$submittedAt' },
            day: { $dayOfMonth: '$submittedAt' }
          },
          count: { $sum: 1 },
          avgCompletionTime: { $avg: '$sessionInfo.completionTime' }
        }
      },
      
      { $sort: { '_id.year': 1, '_id.month': 1, '_id.day': 1 } }
    ];

    return ConsentSubmission.aggregate(pipeline);
  }

  async getProviderStats() {
    const pipeline = [
      { $match: { brandId: new mongoose.Types.ObjectId(this.brandId) } },
      
      { $group: {
          _id: '$providerId',
          totalSubmissions: { $sum: 1 },
          avgCompletionTime: { $avg: '$sessionInfo.completionTime' },
          lastSubmission: { $max: '$submittedAt' }
        }
      },
      
      // Lookup provider details
      { $lookup: {
          from: 'providers',
          localField: '_id',
          foreignField: '_id',
          as: 'provider'
        }
      },
      
      { $unwind: '$provider' },
      { $match: { 'provider.brandId': new mongoose.Types.ObjectId(this.brandId) } },
      
      { $sort: { totalSubmissions: -1 } }
    ];

    return ConsentSubmission.aggregate(pipeline);
  }
}
```

### Data Cleanup and Cascade Deletion

```typescript
// Cascading deletion with brand isolation
export class BrandDataManager {
  async deleteTenant(tenantId: string, brandId: string) {
    const session = await mongoose.startSession();
    
    try {
      await session.withTransaction(async () => {
        // Verify tenant belongs to brand
        const tenant = await Tenant.findOne({ 
          _id: tenantId, 
          brandId 
        }).session(session);
        
        if (!tenant) {
          throw new Error('Tenant not found or access denied');
        }

        // Delete related data in correct order
        await ConsentSubmission.deleteMany({ 
          tenantId, 
          brandId 
        }).session(session);
        
        await Provider.deleteMany({ 
          tenantId, 
          brandId 
        }).session(session);
        
        // Remove tenant assignments from admin users
        await AdminUser.updateMany(
          { brandId },
          { $pull: { tenantScope: tenantId } }
        ).session(session);
        
        // Delete the tenant
        await Tenant.deleteOne({ 
          _id: tenantId, 
          brandId 
        }).session(session);
        
        // Create audit log
        await AuditEvent.create([{
          brandId,
          action: 'tenant.deleted',
          entityId: tenantId,
          details: `Tenant ${tenant.name} and all related data deleted`,
          performedAt: new Date()
        }], { session });
      });
      
    } finally {
      await session.endSession();
    }
  }

  async deleteEntireBrand(brandId: string) {
    // WARNING: This deletes all brand data - use carefully!
    const session = await mongoose.startSession();
    
    try {
      await session.withTransaction(async () => {
        // Delete all brand-related data
        await ConsentSubmission.deleteMany({ brandId }).session(session);
        await Provider.deleteMany({ brandId }).session(session);
        await ConsentForm.deleteMany({ brandId }).session(session);
        await Clause.deleteMany({ brandId }).session(session);
        await AdminUser.deleteMany({ brandId }).session(session);
        await Tenant.deleteMany({ brandId }).session(session);
        
        // Delete GridFS files
        const bucket = new GridFSBucket(mongoose.connection.db, {
          bucketName: `files_${brandId}`
        });
        await bucket.drop();
        
        // Finally delete the brand
        await Brand.deleteOne({ _id: brandId }).session(session);
      });
      
    } finally {
      await session.endSession();
    }
  }
}
```

## Multi-Brand Isolation Guarantees

The data model ensures complete brand isolation through:

1. **Schema-Level Isolation**: Every entity includes `brandId` as a required field
2. **Index-Level Enforcement**: All indexes include `brandId` for performance and isolation
3. **Repository Pattern**: All database operations automatically scope by brand
4. **Cascade Deletion**: Related data cleanup maintains referential integrity
5. **File Isolation**: GridFS uses brand-specific buckets for complete file separation
6. **Audit Trail**: All operations logged with brand context for compliance and debugging

This architecture supports unlimited brand scaling while maintaining enterprise-grade security and data isolation.
