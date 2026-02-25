# Pattern: REST API Design

## Route Architecture

The Express API follows a resource-nested route structure where tenant-owned entities (artists, clauses, consent forms, kiosk users) are accessed via tenant-scoped endpoints, while submissions and subscriptions have their own top-level routes.

### Route Organization

```
/api/v1/Consent/
│
├── Authentication
│   ├── POST /login                          → Admin auth (hash comparison)
│   ├── POST /getUserSalt                    → Retrieve salt for client-side hashing
│   ├── POST /loginkioskuser                 → Kiosk web auth
│   ├── POST /loginkioskusermobile           → Kiosk mobile auth
│   └── POST /getkioskusersalt              → Kiosk salt retrieval
│
├── Tenant-Scoped Resources
│   ├── GET/POST    /tenants                 → List/create tenants
│   ├── GET/PUT/DEL /tenants/:_id            → Single tenant CRUD
│   ├── GET         /tenants/subscriptionstatus/:_id → Check subscription
│   │
│   ├── GET/POST    /tenants/artists/:_id    → Artist CRUD (nested under tenant)
│   ├── GET/PUT/DEL /tenants/artist/:_id     → Single artist
│   │
│   ├── GET/POST    /tenants/clauses/:_id    → Clause CRUD (nested)
│   ├── GET/PUT/DEL /tenants/clause/:_id     → Single clause
│   │
│   ├── GET/POST    /tenants/consents/:_id   → Consent form CRUD (nested)
│   ├── GET/PUT/DEL /tenants/consent/:_id    → Single consent form
│   │
│   ├── Form-Entity Assignments
│   │   ├── GET/POST/DEL /tenants/consent/artists/:_id    → Assign artists to form
│   │   ├── GET          /tenants/consent/artists/combined/:_id
│   │   ├── GET/POST/DEL /tenants/consent/clauses/:_id    → Assign clauses to form
│   │   └── GET          /tenants/consent/clauses/combined/:_id
│   │
│   └── GET/POST    /tenants/kioskusers/:_id → Kiosk user CRUD
│       GET/PUT/DEL /tenants/kioskuser/:_id  → Single kiosk user
│
├── Submissions (Cross-tenant capable)
│   ├── PUT/POST    /submissions             → Query/create submissions
│   ├── PUT/POST/DEL /submissions/:_id       → Query by tenant/update/delete
│   └── GET         /submissions/submission/:_id → Single submission
│
├── Subscriptions
│   ├── GET/POST    /subscriptions           → List/create events
│   └── GET/PUT/DEL /subscriptions/subscription/:_id → Single event
│
└── Utility
    └── POST /sendmail                       → Email dispatch (Gmail OAuth2)
```

## Middleware Stack

```javascript
// Express middleware — order matters
const app = express();

// Request parsing
app.use(bodyParser.json({ limit: '50mb' }));        // Large payloads (images)
app.use(bodyParser.urlencoded({ extended: true }));
app.use(cookieParser('session-secret'));

// Session management
app.use(session({
    secret: 'session-secret',
    resave: false,
    saveUninitialized: true
}));

// Logging
app.use(morgan('dev'));

// CORS — allow cross-origin (admin console + kiosk on different origins)
app.use(cors());

// API documentation
app.use('/api-docs', swaggerUi.serve, swaggerUi.setup(swaggerDocument));

// API key validation applied per-route
function validateApiKey(req, res, next) {
    const apiKey = req.headers['x-api-key'];
    if (apiKey === VALID_API_KEY) {
        next();
    } else {
        res.status(401).json({ error: 'Invalid API key' });
    }
}
```

## Request/Response Patterns

### Standard CRUD Pattern

```javascript
// GET collection (with tenant context)
router.get('/tenants/artists/:_id', validateApiKey, async (req, res) => {
    const tenant = await User.findById(req.params._id);
    res.json(tenant.Artists);
});

// POST create (nested sub-document)
router.post('/tenants/artists/:_id', validateApiKey, async (req, res) => {
    const tenant = await User.findById(req.params._id);
    tenant.Artists.push(req.body);
    await tenant.save();
    res.json(tenant.Artists);
});

// PUT update (nested sub-document by ID)
router.put('/tenants/artist/:_id', validateApiKey, async (req, res) => {
    const result = await User.findOneAndUpdate(
        { "Artists._id": req.params._id },
        { "$set": { "Artists.$": req.body } },
        { new: true }
    );
    res.json(result);
});

// DELETE (pull from array)
router.delete('/tenants/artist/:_id', validateApiKey, async (req, res) => {
    const result = await User.findOneAndUpdate(
        { "Artists._id": req.params._id },
        { "$pull": { "Artists": { "_id": req.params._id } } },
        { new: true }
    );
    res.json(result);
});
```

### Submission Query Pattern

Submissions support multiple query modes — recent, by name, by artist, by date range:

```javascript
// POST query with filters (uses POST body for complex queries)
router.post('/submissions/:_id', validateApiKey, async (req, res) => {
    const { TenantId, queryType, searchValue, startDate, endDate } = req.body;

    let query = { TenantId };

    switch (queryType) {
        case 'recent':
            // Last 50 submissions, natural order descending
            return ConsentSubmission.find(query)
                .sort({ $natural: -1 })
                .limit(50);

        case 'byName':
            query.ConsentName = { $regex: searchValue, $options: 'i' };
            break;

        case 'byArtist':
            query.Artist = { $regex: searchValue, $options: 'i' };
            break;

        case 'byDateRange':
            query.createdAt = {
                $gte: new Date(startDate),
                $lte: new Date(endDate)
            };
            break;
    }

    const results = await ConsentSubmission.find(query).limit(50);
    res.json(results);
});
```

## Error Handling

```javascript
// Global error handler
app.use((err, req, res, next) => {
    console.error(err.stack);
    res.status(500).json({ error: 'Internal server error' });
});

// Mongoose connection error handling
mongoose.connection.on('error', (err) => {
    console.error('MongoDB connection error:', err);
});

mongoose.connection.on('disconnected', () => {
    console.log('MongoDB disconnected. Attempting reconnect...');
});
```
