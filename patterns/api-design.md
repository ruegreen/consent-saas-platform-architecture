# Pattern: REST API Design

## Brand-Aware API Architecture

The consent platform API follows a hierarchical route structure where all operations are automatically scoped to the requesting brand context. The API gateway provides brand resolution, authentication, and routing to ensure complete data isolation between brands while maintaining a clean RESTful interface.

### Route Organization

```
/api/v1/
│
├── Brand Resolution & Authentication
│   ├── POST /auth/login                     → Brand admin authentication
│   ├── POST /auth/logout                    → Session termination
│   ├── GET  /auth/session                   → Current session info
│   ├── POST /auth/password-reset            → Request password reset
│   └── POST /auth/password-reset/confirm    → Complete password reset
│
├── Brand-Scoped Admin APIs
│   ├── GET    /admin/brands/current         → Current brand configuration
│   ├── PATCH  /admin/brands/current         → Update brand settings
│   │
│   ├── GET/POST    /admin/tenants           → List/create locations
│   ├── GET/PATCH/DELETE /admin/tenants/:id  → Tenant CRUD operations
│   │
│   ├── GET/POST    /admin/providers         → List/create staff/artists
│   ├── GET/PATCH/DELETE /admin/providers/:id → Provider CRUD operations
│   │
│   ├── GET/POST    /admin/consent-forms     → List/create consent forms
│   ├── GET/PATCH/DELETE /admin/consent-forms/:id → Form CRUD operations
│   ├── POST /admin/consent-forms/:id/publish    → Publish form (make public)
│   ├── POST /admin/consent-forms/:id/unpublish  → Unpublish form
│   │
│   ├── GET/POST    /admin/clauses           → List/create legal clauses
│   ├── GET/PATCH/DELETE /admin/clauses/:id  → Clause CRUD operations
│   │
│   ├── GET  /admin/submissions              → List consent submissions (paginated)
│   ├── GET  /admin/submissions/:id          → Get submission details
│   ├── PATCH /admin/submissions/:id         → Update submission notes
│   ├── DELETE /admin/submissions/:id        → Archive submission
│   │
│   └── GET/POST    /admin/admin-users       → List/invite admin users
│       GET/PATCH/DELETE /admin/admin-users/:id → User management
│
├── Public APIs (Brand-Scoped)
│   ├── GET  /public/forms/:slug             → Get published form configuration
│   ├── POST /public/forms/:slug/submit      → Submit completed consent form
│   ├── POST /public/images/upload           → Upload photos/signatures
│   └── GET  /public/submissions/:id/receipt → Get submission receipt
│
└── File Management
    ├── POST /files/upload                   → Upload file to MongoDB GridFS
    ├── GET  /files/:id                      → Retrieve file (brand-scoped)
    └── DELETE /files/:id                    → Delete file (brand-scoped)
```

## Middleware Stack

```typescript
// Express middleware pipeline with brand awareness
import express from 'express';
import cors from 'cors';
import helmet from 'helmet';
import rateLimit from 'express-rate-limit';

const app = express();

// Security and parsing
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      scriptSrc: ["'self'"],
      imgSrc: ["'self'", "data:", "https:"],
    }
  },
  crossOriginEmbedderPolicy: false
}));

app.use(cors({
  origin: process.env.ALLOWED_ORIGINS?.split(',') || ['http://localhost:3000'],
  credentials: true,
  optionsSuccessStatus: 200
}));

app.use(express.json({ limit: '10mb' }));
app.use(express.urlencoded({ extended: true, limit: '10mb' }));

// Rate limiting by IP and brand
const createRateLimiter = (windowMs: number, max: number) => 
  rateLimit({
    windowMs,
    max,
    message: { error: 'Too many requests, please try again later' },
    standardHeaders: true,
    legacyHeaders: false,
    keyGenerator: (req) => `${req.ip}:${req.get('host')}` // IP + brand host
  });

// Different limits for different endpoints
app.use('/api/v1/auth', createRateLimiter(15 * 60 * 1000, 5)); // 5 per 15min
app.use('/api/v1/admin', createRateLimiter(60 * 1000, 100));    // 100 per min
app.use('/api/v1/public', createRateLimiter(60 * 1000, 20));    // 20 per min

// Brand resolution middleware
app.use('/api/v1', brandResolutionMiddleware);

// Session management
app.use(session(sessionConfig));
app.use(csrfProtection);

// Request logging
app.use((req, res, next) => {
  console.log(`${req.method} ${req.path} - Brand: ${req.brandId} - IP: ${req.ip}`);
  next();
});
```

### Brand Resolution Middleware

```typescript
// Automatic brand detection and context injection
import { Brand } from '../models/Brand';

interface BrandRequest extends Request {
  brandId: string;
  brand: Brand;
}

// In-memory brand cache for performance
const brandCache = new Map<string, { brand: Brand; cachedAt: number }>();
const CACHE_TTL = 5 * 60 * 1000; // 5 minutes

export async function brandResolutionMiddleware(
  req: BrandRequest, 
  res: Response, 
  next: NextFunction
) {
  try {
    const hostname = req.get('host') || '';
    
    // Check cache first
    const cached = brandCache.get(hostname);
    if (cached && (Date.now() - cached.cachedAt) < CACHE_TTL) {
      req.brand = cached.brand;
      req.brandId = cached.brand._id.toString();
      return next();
    }

    // Resolve brand from hostname
    let brand: Brand | null = null;

    // Check platform subdomain (brand.app.platform.com)
    const subdomainMatch = hostname.match(/^([^.]+)\.app\.gurueconsulting\.com$/);
    if (subdomainMatch) {
      const brandSlug = subdomainMatch[1];
      brand = await Brand.findOne({ 
        slug: brandSlug,
        status: 'active' 
      });
    } else {
      // Check custom domains
      brand = await Brand.findOne({
        'domains.customDomains': hostname,
        status: 'active'
      });
    }

    if (!brand) {
      return res.status(404).json({
        error: 'Brand not found',
        hostname,
        message: 'No active brand found for this domain'
      });
    }

    // Cache the result
    brandCache.set(hostname, { 
      brand, 
      cachedAt: Date.now() 
    });

    req.brand = brand;
    req.brandId = brand._id.toString();
    
    next();
    
  } catch (error) {
    console.error('Brand resolution error:', error);
    res.status(500).json({ error: 'Brand resolution failed' });
  }
}

// Clear cache when brands are updated
export function clearBrandCache(brandSlug?: string) {
  if (brandSlug) {
    // Clear specific brand
    for (const [hostname, cached] of brandCache.entries()) {
      if (cached.brand.slug === brandSlug) {
        brandCache.delete(hostname);
      }
    }
  } else {
    // Clear all
    brandCache.clear();
  }
}
```

## Standard CRUD Pattern

```typescript
// Generic CRUD controller with brand scoping
export class BrandScopedController<T> {
  constructor(
    private model: mongoose.Model<T & { brandId: string }>,
    private entityName: string
  ) {}

  // GET /admin/{entity} - List entities for current brand
  list = async (req: BrandRequest, res: Response) => {
    try {
      const { page = 1, limit = 50, search, ...filters } = req.query;
      
      const query: any = { 
        brandId: req.brandId,
        ...filters
      };

      // Add text search if provided
      if (search && typeof search === 'string') {
        query.$text = { $search: search };
      }

      const skip = (parseInt(page as string) - 1) * parseInt(limit as string);
      
      const [items, total] = await Promise.all([
        this.model.find(query)
          .sort({ createdAt: -1 })
          .skip(skip)
          .limit(parseInt(limit as string)),
        this.model.countDocuments(query)
      ]);

      res.json({
        items,
        pagination: {
          page: parseInt(page as string),
          limit: parseInt(limit as string),
          total,
          pages: Math.ceil(total / parseInt(limit as string))
        }
      });

    } catch (error) {
      console.error(`List ${this.entityName} error:`, error);
      res.status(500).json({ error: `Failed to list ${this.entityName}` });
    }
  };

  // POST /admin/{entity} - Create new entity
  create = async (req: BrandRequest, res: Response) => {
    try {
      const entityData = {
        ...req.body,
        brandId: req.brandId,
        createdBy: req.userId
      };

      const entity = await this.model.create(entityData);

      // Log creation for audit
      await AuditEvent.create({
        action: `${this.entityName}.created`,
        brandId: req.brandId,
        userId: req.userId,
        entityId: entity._id,
        details: { name: entity.name },
        ip: req.ip,
        userAgent: req.get('User-Agent')
      });

      res.status(201).json(entity);

    } catch (error) {
      console.error(`Create ${this.entityName} error:`, error);
      
      if (error.code === 11000) { // Duplicate key
        return res.status(409).json({ 
          error: `${this.entityName} already exists` 
        });
      }

      res.status(500).json({ error: `Failed to create ${this.entityName}` });
    }
  };

  // GET /admin/{entity}/:id - Get single entity
  get = async (req: BrandRequest, res: Response) => {
    try {
      const entity = await this.model.findOne({
        _id: req.params.id,
        brandId: req.brandId
      });

      if (!entity) {
        return res.status(404).json({ 
          error: `${this.entityName} not found` 
        });
      }

      res.json(entity);

    } catch (error) {
      console.error(`Get ${this.entityName} error:`, error);
      res.status(500).json({ error: `Failed to get ${this.entityName}` });
    }
  };

  // PATCH /admin/{entity}/:id - Update entity
  update = async (req: BrandRequest, res: Response) => {
    try {
      const entity = await this.model.findOneAndUpdate(
        { 
          _id: req.params.id, 
          brandId: req.brandId 
        },
        { 
          ...req.body, 
          updatedBy: req.userId,
          updatedAt: new Date()
        },
        { new: true }
      );

      if (!entity) {
        return res.status(404).json({ 
          error: `${this.entityName} not found` 
        });
      }

      // Log update for audit
      await AuditEvent.create({
        action: `${this.entityName}.updated`,
        brandId: req.brandId,
        userId: req.userId,
        entityId: entity._id,
        details: { 
          changes: Object.keys(req.body),
          name: entity.name 
        },
        ip: req.ip,
        userAgent: req.get('User-Agent')
      });

      res.json(entity);

    } catch (error) {
      console.error(`Update ${this.entityName} error:`, error);
      res.status(500).json({ error: `Failed to update ${this.entityName}` });
    }
  };

  // DELETE /admin/{entity}/:id - Soft delete entity
  delete = async (req: BrandRequest, res: Response) => {
    try {
      const entity = await this.model.findOneAndUpdate(
        { 
          _id: req.params.id, 
          brandId: req.brandId 
        },
        { 
          status: 'archived',
          archivedBy: req.userId,
          archivedAt: new Date()
        },
        { new: true }
      );

      if (!entity) {
        return res.status(404).json({ 
          error: `${this.entityName} not found` 
        });
      }

      // Log deletion for audit
      await AuditEvent.create({
        action: `${this.entityName}.archived`,
        brandId: req.brandId,
        userId: req.userId,
        entityId: entity._id,
        details: { name: entity.name },
        ip: req.ip,
        userAgent: req.get('User-Agent')
      });

      res.json({ success: true });

    } catch (error) {
      console.error(`Delete ${this.entityName} error:`, error);
      res.status(500).json({ error: `Failed to delete ${this.entityName}` });
    }
  };
}

// Usage for specific controllers
export class TenantsController extends BrandScopedController {
  constructor() {
    super(Tenant, 'tenant');
  }

  // Additional tenant-specific methods
  async getStats(req: BrandRequest, res: Response) {
    const stats = await Tenant.aggregate([
      { $match: { brandId: new mongoose.Types.ObjectId(req.brandId) } },
      { $group: {
          _id: null,
          totalTenants: { $sum: 1 },
          activeTenants: {
            $sum: { $cond: [{ $eq: ['$status', 'active'] }, 1, 0] }
          }
        }
      }
    ]);

    res.json(stats[0] || { totalTenants: 0, activeTenants: 0 });
  }
}

// Mount controllers with authentication
const tenantsController = new TenantsController();
const providersController = new BrandScopedController(Provider, 'provider');

router.get('/admin/tenants', requireAuth, requireBrandAdmin, tenantsController.list);
router.post('/admin/tenants', requireAuth, requireBrandAdmin, tenantsController.create);
router.get('/admin/tenants/:id', requireAuth, requireBrandAdmin, tenantsController.get);
router.patch('/admin/tenants/:id', requireAuth, requireBrandAdmin, tenantsController.update);
router.delete('/admin/tenants/:id', requireAuth, requireBrandOwner, tenantsController.delete);
```

### Submission Query API

Advanced submission querying with brand-scoped filtering and analytics:

```typescript
// Advanced submission queries with filtering and analytics
export class SubmissionsController {
  // GET /admin/submissions - List submissions with filters
  async list(req: BrandRequest, res: Response) {
    try {
      const {
        page = 1,
        limit = 50,
        tenantId,
        providerId,
        formId,
        startDate,
        endDate,
        search,
        status = 'completed'
      } = req.query;

      // Build brand-scoped query
      const query: any = { 
        brandId: req.brandId,
        status
      };

      // Apply tenant scope for non-owners
      if (!PermissionChecker.canManageBrand(req.userRoles)) {
        if (req.tenantScope?.length) {
          query.tenantId = { $in: req.tenantScope };
        } else {
          return res.status(403).json({ error: 'No tenant access' });
        }
      }

      // Additional filters
      if (tenantId) query.tenantId = tenantId;
      if (providerId) query.providerId = providerId;
      if (formId) query.formId = formId;

      // Date range filter
      if (startDate || endDate) {
        query.submittedAt = {};
        if (startDate) query.submittedAt.$gte = new Date(startDate as string);
        if (endDate) query.submittedAt.$lte = new Date(endDate as string);
      }

      // Text search on subject name/email
      if (search) {
        query.$or = [
          { 'subject.name': { $regex: search, $options: 'i' } },
          { 'subject.email': { $regex: search, $options: 'i' } }
        ];
      }

      const skip = (parseInt(page as string) - 1) * parseInt(limit as string);

      // Execute query with population
      const [submissions, total] = await Promise.all([
        ConsentSubmission.find(query)
          .populate('tenantId', 'name')
          .populate('providerId', 'name')
          .populate('formId', 'name')
          .sort({ submittedAt: -1 })
          .skip(skip)
          .limit(parseInt(limit as string)),
        ConsentSubmission.countDocuments(query)
      ]);

      res.json({
        submissions,
        pagination: {
          page: parseInt(page as string),
          limit: parseInt(limit as string),
          total,
          pages: Math.ceil(total / parseInt(limit as string))
        }
      });

    } catch (error) {
      console.error('List submissions error:', error);
      res.status(500).json({ error: 'Failed to list submissions' });
    }
  }

  // GET /admin/submissions/analytics - Submission analytics
  async getAnalytics(req: BrandRequest, res: Response) {
    try {
      const { timeframe = 'month' } = req.query;
      const startDate = this.getStartDate(timeframe as string);

      const pipeline = [
        // Brand and date filtering
        { $match: {
            brandId: new mongoose.Types.ObjectId(req.brandId),
            submittedAt: { $gte: startDate },
            status: 'completed'
          }
        },

        // Group by time period
        { $group: {
            _id: {
              year: { $year: '$submittedAt' },
              month: { $month: '$submittedAt' },
              day: timeframe === 'day' ? { $dayOfMonth: '$submittedAt' } : null
            },
            count: { $sum: 1 },
            avgCompletionTime: { $avg: '$sessionInfo.completionTime' }
          }
        },

        // Sort by date
        { $sort: { 
            '_id.year': 1, 
            '_id.month': 1, 
            '_id.day': 1 
          }
        }
      ];

      const timeSeriesData = await ConsentSubmission.aggregate(pipeline);

      // Get top performers
      const topProviders = await ConsentSubmission.aggregate([
        { $match: {
            brandId: new mongoose.Types.ObjectId(req.brandId),
            submittedAt: { $gte: startDate },
            status: 'completed',
            providerId: { $exists: true }
          }
        },
        { $group: {
            _id: '$providerId',
            count: { $sum: 1 },
            avgCompletionTime: { $avg: '$sessionInfo.completionTime' }
          }
        },
        { $lookup: {
            from: 'providers',
            localField: '_id',
            foreignField: '_id',
            as: 'provider'
          }
        },
        { $unwind: '$provider' },
        { $match: { 
            'provider.brandId': new mongoose.Types.ObjectId(req.brandId)
          }
        },
        { $sort: { count: -1 } },
        { $limit: 10 }
      ]);

      res.json({
        timeSeries: timeSeriesData,
        topProviders,
        summary: {
          totalSubmissions: timeSeriesData.reduce((sum, period) => sum + period.count, 0),
          avgCompletionTime: timeSeriesData.length > 0 
            ? timeSeriesData.reduce((sum, period) => sum + period.avgCompletionTime, 0) / timeSeriesData.length
            : 0
        }
      });

    } catch (error) {
      console.error('Submission analytics error:', error);
      res.status(500).json({ error: 'Failed to generate analytics' });
    }
  }

  private getStartDate(timeframe: string): Date {
    const now = new Date();
    switch (timeframe) {
      case 'day': return new Date(now.getTime() - 24 * 60 * 60 * 1000);
      case 'week': return new Date(now.getTime() - 7 * 24 * 60 * 60 * 1000);
      case 'month': return new Date(now.getTime() - 30 * 24 * 60 * 60 * 1000);
      case 'year': return new Date(now.getTime() - 365 * 24 * 60 * 60 * 1000);
      default: return new Date(now.getTime() - 30 * 24 * 60 * 60 * 1000);
    }
  }
}
```

### Public API (Brand-Scoped)

```typescript
// Public APIs for consent form submission
export class PublicController {
  // GET /public/forms/:slug - Get published form configuration
  async getForm(req: BrandRequest, res: Response) {
    try {
      const { slug } = req.params;

      const form = await ConsentForm.findOne({
        slug,
        brandId: req.brandId,
        status: 'published'
      })
      .populate('assignedProviders', 'name specialties')
      .populate('assignedClauses', 'title content type required');

      if (!form) {
        return res.status(404).json({ error: 'Form not found' });
      }

      // Include brand theming for dynamic styling
      res.json({
        form: {
          id: form._id,
          name: form.name,
          description: form.description,
          capture: form.capture,
          presentation: form.presentation,
          providers: form.assignedProviders,
          clauses: form.assignedClauses
        },
        brand: {
          name: req.brand.name,
          theme: req.brand.theme,
          terminology: req.brand.terminology
        }
      });

    } catch (error) {
      console.error('Get form error:', error);
      res.status(500).json({ error: 'Failed to load form' });
    }
  }

  // POST /public/forms/:slug/submit - Submit consent form
  async submitForm(req: BrandRequest, res: Response) {
    try {
      const { slug } = req.params;
      const submissionData = req.body;

      // Validate form exists and is published
      const form = await ConsentForm.findOne({
        slug,
        brandId: req.brandId,
        status: 'published'
      });

      if (!form) {
        return res.status(404).json({ error: 'Form not found or not available' });
      }

      // Create submission record
      const submission = await ConsentSubmission.create({
        brandId: req.brandId,
        tenantId: submissionData.tenantId,
        formId: form._id,
        providerId: submissionData.providerId,
        subject: submissionData.subject,
        guardian: submissionData.guardian,
        acknowledgedClauses: submissionData.acknowledgedClauses,
        photos: submissionData.photos,
        signatures: submissionData.signatures,
        sessionInfo: {
          ip: req.ip,
          userAgent: req.get('User-Agent'),
          submissionSource: 'web',
          completionTime: submissionData.completionTime
        }
      });

      // Update form usage statistics
      await ConsentForm.findByIdAndUpdate(form._id, {
        $inc: { 'usage.totalSubmissions': 1 },
        'usage.lastSubmissionAt': new Date()
      });

      // Send notification email if configured
      if (form.notifications.sendToTenant || form.notifications.sendToProvider) {
        await EmailService.sendSubmissionNotification({
          submission,
          form,
          brand: req.brand
        });
      }

      // Generate receipt
      const receipt = {
        id: submission._id,
        submittedAt: submission.submittedAt,
        formName: form.name,
        subjectName: submission.subject.name,
        confirmationCode: submission._id.toString().slice(-8).toUpperCase()
      };

      res.status(201).json(receipt);

    } catch (error) {
      console.error('Submit form error:', error);
      res.status(500).json({ error: 'Submission failed' });
    }
  }
}
```

## Error Handling

```typescript
// Global error handler with brand context
export function errorHandler(
  err: Error, 
  req: BrandRequest, 
  res: Response, 
  next: NextFunction
) {
  console.error('API Error:', {
    error: err.message,
    stack: err.stack,
    brandId: req.brandId,
    path: req.path,
    method: req.method,
    ip: req.ip,
    userAgent: req.get('User-Agent')
  });

  // Log error for audit if brand context available
  if (req.brandId && req.userId) {
    AuditEvent.create({
      action: 'api.error',
      brandId: req.brandId,
      userId: req.userId,
      details: { 
        error: err.message,
        path: req.path,
        method: req.method
      },
      ip: req.ip,
      userAgent: req.get('User-Agent')
    }).catch(console.error);
  }

  // Don't expose internal errors in production
  if (process.env.NODE_ENV === 'production') {
    res.status(500).json({ error: 'Internal server error' });
  } else {
    res.status(500).json({ 
      error: err.message,
      stack: err.stack 
    });
  }
}

// Async error wrapper
export function asyncHandler(fn: Function) {
  return (req: Request, res: Response, next: NextFunction) => {
    Promise.resolve(fn(req, res, next)).catch(next);
  };
}
```

This API architecture provides enterprise-grade multi-brand isolation while maintaining clean RESTful design principles. Every endpoint automatically operates within brand context, ensuring complete data separation and security across the platform.
