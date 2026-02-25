# Pattern: Authentication & Security

## Dual Authentication Model

TRO implements two separate authentication flows — admin accounts (shop owners/managers) and kiosk accounts (in-shop tablet devices) — using the same cryptographic algorithm but different credential stores and access levels.

### Password Hashing (PBKDF2)

```javascript
// Shared across Node.js API, React web, and React Native mobile
const crypto = require('crypto');

function generateSalt() {
    return crypto.randomBytes(16).toString('hex');
}

function hashPassword(password, salt) {
    return crypto.pbkdf2Sync(
        password,           // User's plaintext password
        salt,               // Per-user random salt
        1000,               // Iterations
        64,                 // Key length (bytes)
        'sha512'            // Hash algorithm
    ).toString('hex');
}

// Verification: compare computed hash against stored hash
function verifyPassword(password, storedHash, storedSalt) {
    const computedHash = hashPassword(password, storedSalt);
    return computedHash === storedHash;
}
```

### Admin Login Flow

```
Client                          API Server                     MongoDB
  │                                │                              │
  │  POST /getUserSalt             │                              │
  │  {username}                    │                              │
  │──────────────────────────────→ │                              │
  │                                │  Find user by Userid         │
  │                                │─────────────────────────────→│
  │                                │  Return {Salt}               │
  │                                │←─────────────────────────────│
  │  {salt}                        │                              │
  │←────────────────────────────── │                              │
  │                                │                              │
  │  PBKDF2(password, salt)        │                              │
  │  (computed client-side)        │                              │
  │                                │                              │
  │  POST /login                   │                              │
  │  {username, hash, x-api-key}   │                              │
  │──────────────────────────────→ │                              │
  │                                │  Validate API key            │
  │                                │  Find user, compare hash     │
  │                                │─────────────────────────────→│
  │                                │  User document               │
  │                                │←─────────────────────────────│
  │  {user, JWT, apiKey}           │                              │
  │←────────────────────────────── │                              │
```

**Key design choice**: Password hashing happens client-side. The server never receives the plaintext password — only the hash. This means even HTTPS-intercepted traffic doesn't expose the password. The trade-off is that the salt must be retrieved first (extra round trip).

### Kiosk Login Flow

```javascript
// Kiosk accounts have separate credentials stored as sub-documents
// within the tenant's Kioskusers array

// 1. Client retrieves kiosk salt
POST /getkioskusersalt
{
    "KioskUserid": "front-desk-tablet-1"
}
// Response: { "KioskSalt": "a1b2c3..." }

// 2. Client computes hash and authenticates
POST /loginkioskusermobile
{
    "KioskUserid": "front-desk-tablet-1",
    "KioskHash": "computed-hash...",
    "KioskSalt": "a1b2c3..."
}
// Response: {
//     "authenticated": true,
//     "role": "Kiosk",
//     "Userid": "shop-owner-id",
//     "SubscriptionStatus": true,
//     "cloudinaryConfig": { ... },
//     "apiKey": "..."
// }
```

**Kiosk vs Admin differences:**
- Kiosk accounts cannot modify forms, artists, clauses, or tenant settings
- Kiosk accounts can only create submissions (read-only access to forms)
- Kiosk accounts inherit the parent tenant's subscription status
- Separate credential store prevents admin password exposure on shared devices

## API Key Authentication

Two-tier API key system:

```javascript
// Login operations use a dedicated key (prevents unauthorized registration)
const LOGIN_API_KEY = process.env.LOGIN_API_KEY;

// All other CRUD operations use a general key
const GENERAL_API_KEY = process.env.GENERAL_API_KEY;

// Middleware checks x-api-key header
function validateApiKey(expectedKey) {
    return (req, res, next) => {
        const provided = req.headers['x-api-key'];
        if (provided === expectedKey) {
            next();
        } else {
            res.status(401).json({ error: 'Unauthorized' });
        }
    };
}
```

## JWT Token Management

```javascript
const jwt = require('jsonwebtoken');

// Token generation on successful login
function generateToken(user) {
    return jwt.sign(
        { id: user._id, role: user.Role },
        process.env.JWT_SECRET,
        { expiresIn: '7d' }
    );
}

// Token verification middleware
function verifyToken(req, res, next) {
    const token = req.headers.authorization?.split(' ')[1];
    if (!token) return res.status(401).json({ error: 'No token provided' });

    jwt.verify(token, process.env.JWT_SECRET, (err, decoded) => {
        if (err) return res.status(401).json({ error: 'Invalid token' });
        req.user = decoded;
        next();
    });
}
```

## Subscription Enforcement

Access control is enforced at multiple points to prevent use after subscription expiry:

```javascript
// 1. Login-time check
async function checkSubscription(tenant) {
    const subDate = new Date(tenant.SubscriptionDate);
    const today = new Date();
    return subDate >= today && tenant.SubscriptionStatus === true;
}

// 2. Pre-submission check (API-level)
router.post('/submissions', async (req, res) => {
    const tenant = await User.findById(req.body.TenantId);
    if (!await checkSubscription(tenant)) {
        return res.status(403).json({
            error: 'Subscription expired',
            renewUrl: '/subscription/renew'
        });
    }
    // ... proceed with submission creation
});

// 3. Client-side gating (Redux state)
// Consent wizard only renders when:
// (userStatus || role === "Kiosk") && subscriptionActive
```

## Cross-Platform Crypto Compatibility

Running PBKDF2 identically across three platforms required specific approaches:

```javascript
// Node.js (API server) — native crypto
const crypto = require('crypto');
crypto.pbkdf2Sync(password, salt, 1000, 64, 'sha512');

// React (web admin) — browser crypto shim
import crypto from 'crypto-browserify';
crypto.pbkdf2Sync(password, salt, 1000, 64, 'sha512');

// React Native (mobile) — requires polyfill chain
// package.json "react-native" field maps:
//   "crypto" → "react-native-crypto"
//   "stream" → "stream-browserify"
//   "buffer" → "buffer"
// Then same API: crypto.pbkdf2Sync(...)
```
