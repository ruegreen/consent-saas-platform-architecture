# Pattern: Authentication & Security

## Multi-Brand Authentication Architecture

The consent platform implements a sophisticated authentication system designed for multi-brand SaaS with complete security isolation between brands. The system uses cookie-based sessions with CSRF protection and supports role-based access control across platform, brand, and tenant levels.

### Cookie-Based Session Management

```typescript
// Modern session-based authentication replacing legacy JWT tokens
import session from 'express-session';
import MongoStore from 'connect-mongo';
import csrf from 'csurf';

// Session configuration
const sessionConfig = {
  secret: process.env.SESSION_SECRET,
  name: 'consent-platform-session',
  resave: false,
  saveUninitialized: false,
  store: MongoStore.create({
    mongoUrl: process.env.MONGODB_URI,
    collectionName: 'sessions',
    ttl: 24 * 60 * 60 * 7, // 7 days
    autoRemove: 'native'
  }),
  cookie: {
    secure: process.env.NODE_ENV === 'production',
    httpOnly: true,
    maxAge: 24 * 60 * 60 * 7 * 1000, // 7 days
    sameSite: 'lax'
  }
};

// CSRF protection
const csrfProtection = csrf({
  cookie: {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'lax'
  }
});

app.use(session(sessionConfig));
app.use(csrfProtection);
```

### Password Hashing (Argon2id)

```typescript
// Modern password hashing using Argon2id (winner of password hashing competition)
import argon2 from 'argon2';

export class PasswordService {
  private static readonly HASH_OPTIONS = {
    type: argon2.argon2id,
    memoryCost: 2 ** 16,      // 64 MB
    timeCost: 3,              // 3 iterations  
    parallelism: 1,           // Single thread
  };

  static async hashPassword(password: string): Promise<string> {
    return argon2.hash(password, this.HASH_OPTIONS);
  }

  static async verifyPassword(hash: string, password: string): Promise<boolean> {
    try {
      return await argon2.verify(hash, password);
    } catch (error) {
      console.error('Password verification error:', error);
      return false;
    }
  }

  static generateRandomPassword(length: number = 12): string {
    const charset = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789!@#$%^&*';
    let password = '';
    for (let i = 0; i < length; i++) {
      password += charset.charAt(Math.floor(Math.random() * charset.length));
    }
    return password;
  }
}
```

### Brand-Scoped Admin Authentication

```typescript
// Admin login flow with brand context resolution
interface LoginRequest {
  email: string;
  password: string;
}

interface AuthSession {
  userId: string;
  brandId: string;
  roles: UserRole[];
  tenantScope?: string[];
  loginAt: Date;
  lastActivity: Date;
}

export class AuthController {
  async login(req: Request, res: Response) {
    const { email, password } = req.body as LoginRequest;
    
    try {
      // Find user across all brands (email is unique globally)
      const user = await AdminUser.findOne({ 
        email: email.toLowerCase(),
        status: 'active'
      }).populate('brand');

      if (!user) {
        return res.status(401).json({ 
          error: 'Invalid email or password' 
        });
      }

      // Verify password
      const isValidPassword = await PasswordService.verifyPassword(
        user.hashedPassword, 
        password
      );

      if (!isValidPassword) {
        // Log failed attempt
        await AuditEvent.create({
          action: 'auth.login_failed',
          brandId: user.brandId,
          userId: user._id,
          details: { email, reason: 'invalid_password' },
          ip: req.ip,
          userAgent: req.get('User-Agent')
        });

        return res.status(401).json({ 
          error: 'Invalid email or password' 
        });
      }

      // Check brand status
      if (user.brand.status !== 'active') {
        return res.status(403).json({
          error: 'Brand account suspended',
          brandStatus: user.brand.status
        });
      }

      // Create session
      const session: AuthSession = {
        userId: user._id.toString(),
        brandId: user.brandId.toString(),
        roles: user.roles,
        tenantScope: user.tenantScope,
        loginAt: new Date(),
        lastActivity: new Date()
      };

      req.session.auth = session;
      
      // Log successful login
      await AuditEvent.create({
        action: 'auth.login_success',
        brandId: user.brandId,
        userId: user._id,
        details: { email },
        ip: req.ip,
        userAgent: req.get('User-Agent')
      });

      // Update last login
      await AdminUser.findByIdAndUpdate(user._id, {
        lastLoginAt: new Date(),
        lastLoginIp: req.ip
      });

      res.json({
        success: true,
        user: {
          id: user._id,
          email: user.email,
          name: user.name,
          roles: user.roles,
          brand: {
            id: user.brand._id,
            name: user.brand.name,
            slug: user.brand.slug,
            theme: user.brand.theme
          }
        },
        csrfToken: req.csrfToken()
      });

    } catch (error) {
      console.error('Login error:', error);
      res.status(500).json({ error: 'Login failed' });
    }
  }

  async logout(req: Request, res: Response) {
    const session = req.session.auth;
    
    if (session) {
      // Log logout
      await AuditEvent.create({
        action: 'auth.logout',
        brandId: session.brandId,
        userId: session.userId,
        ip: req.ip,
        userAgent: req.get('User-Agent')
      });
    }

    req.session.destroy((err) => {
      if (err) {
        console.error('Session destruction error:', err);
        return res.status(500).json({ error: 'Logout failed' });
      }
      
      res.clearCookie('consent-platform-session');
      res.json({ success: true });
    });
  }

  async getSession(req: Request, res: Response) {
    const session = req.session.auth;
    
    if (!session) {
      return res.status(401).json({ error: 'Not authenticated' });
    }

    try {
      // Verify user and brand still exist and are active
      const user = await AdminUser.findOne({
        _id: session.userId,
        brandId: session.brandId,
        status: 'active'
      }).populate('brand');

      if (!user || user.brand.status !== 'active') {
        req.session.destroy(() => {});
        return res.status(401).json({ error: 'Session invalid' });
      }

      // Update last activity
      session.lastActivity = new Date();
      req.session.auth = session;

      res.json({
        user: {
          id: user._id,
          email: user.email,
          name: user.name,
          roles: user.roles,
          brand: {
            id: user.brand._id,
            name: user.brand.name,
            slug: user.brand.slug,
            theme: user.brand.theme
          }
        },
        csrfToken: req.csrfToken()
      });

    } catch (error) {
      console.error('Session validation error:', error);
      res.status(500).json({ error: 'Session validation failed' });
    }
  }
}
```

### Authentication Middleware

```typescript
// Authentication and authorization middleware
export function requireAuth(req: BrandRequest, res: Response, next: NextFunction) {
  const session = req.session?.auth;
  
  if (!session) {
    return res.status(401).json({ error: 'Authentication required' });
  }

  // Check session age
  const sessionAge = Date.now() - new Date(session.lastActivity).getTime();
  const maxAge = 24 * 60 * 60 * 1000; // 24 hours
  
  if (sessionAge > maxAge) {
    req.session.destroy(() => {});
    return res.status(401).json({ error: 'Session expired' });
  }

  // Attach session data to request
  req.userId = session.userId;
  req.brandId = session.brandId;
  req.userRoles = session.roles;
  req.tenantScope = session.tenantScope;

  // Update last activity
  session.lastActivity = new Date();
  req.session.auth = session;

  next();
}

export function requireBrandAdmin(req: BrandRequest, res: Response, next: NextFunction) {
  if (!req.userRoles?.includes('admin') && !req.userRoles?.includes('owner')) {
    return res.status(403).json({ 
      error: 'Insufficient permissions',
      required: ['admin', 'owner']
    });
  }
  next();
}

export function requireBrandOwner(req: BrandRequest, res: Response, next: NextFunction) {
  if (!req.userRoles?.includes('owner')) {
    return res.status(403).json({ 
      error: 'Owner role required',
      required: ['owner']
    });
  }
  next();
}

export function requireTenantAccess(tenantId: string) {
  return (req: BrandRequest, res: Response, next: NextFunction) => {
    // Brand owners have access to all tenants
    if (req.userRoles?.includes('owner')) {
      return next();
    }

    // Other users must have explicit tenant scope
    if (!req.tenantScope?.includes(tenantId)) {
      return res.status(403).json({ 
        error: 'Tenant access denied',
        tenantId 
      });
    }

    next();
  };
}
```

### Role-Based Access Control

```typescript
// User role definitions with hierarchical permissions
export enum UserRole {
  PLATFORM_ADMIN = 'platform_admin',    // Can access any brand (system admin)
  BRAND_OWNER = 'owner',                 // Full control within brand
  BRAND_ADMIN = 'admin',                 // Limited admin within brand  
  LOCATION_MANAGER = 'location_manager', // Manage specific locations
  PROVIDER = 'provider',                 // Access only assigned forms
  READONLY = 'readonly'                  // View-only access
}

// Permission checking utility
export class PermissionChecker {
  static canManageBrand(roles: UserRole[]): boolean {
    return roles.includes(UserRole.PLATFORM_ADMIN) || 
           roles.includes(UserRole.BRAND_OWNER);
  }

  static canManageTenants(roles: UserRole[]): boolean {
    return roles.includes(UserRole.PLATFORM_ADMIN) ||
           roles.includes(UserRole.BRAND_OWNER) ||
           roles.includes(UserRole.BRAND_ADMIN);
  }

  static canManageProviders(roles: UserRole[]): boolean {
    return roles.includes(UserRole.PLATFORM_ADMIN) ||
           roles.includes(UserRole.BRAND_OWNER) ||
           roles.includes(UserRole.BRAND_ADMIN) ||
           roles.includes(UserRole.LOCATION_MANAGER);
  }

  static canViewSubmissions(roles: UserRole[], tenantScope?: string[]): boolean {
    if (roles.includes(UserRole.PLATFORM_ADMIN) || 
        roles.includes(UserRole.BRAND_OWNER)) {
      return true;
    }

    return roles.includes(UserRole.BRAND_ADMIN) ||
           roles.includes(UserRole.LOCATION_MANAGER) ||
           (roles.includes(UserRole.PROVIDER) && tenantScope && tenantScope.length > 0);
  }

  static canDeleteSubmissions(roles: UserRole[]): boolean {
    return roles.includes(UserRole.PLATFORM_ADMIN) ||
           roles.includes(UserRole.BRAND_OWNER);
  }
}

// Usage in controllers
export class SubmissionsController {
  async list(req: BrandRequest, res: Response) {
    // Check view permissions
    if (!PermissionChecker.canViewSubmissions(req.userRoles, req.tenantScope)) {
      return res.status(403).json({ error: 'Cannot view submissions' });
    }

    let query: any = { brandId: req.brandId };

    // Apply tenant scope for non-owner roles
    if (!PermissionChecker.canManageBrand(req.userRoles)) {
      if (req.tenantScope && req.tenantScope.length > 0) {
        query.tenantId = { $in: req.tenantScope };
      } else {
        return res.status(403).json({ error: 'No tenant access' });
      }
    }

    const submissions = await ConsentSubmission.find(query)
      .sort({ submittedAt: -1 })
      .limit(50);

    res.json(submissions);
  }

  async delete(req: BrandRequest, res: Response) {
    if (!PermissionChecker.canDeleteSubmissions(req.userRoles)) {
      return res.status(403).json({ error: 'Cannot delete submissions' });
    }

    const { id } = req.params;
    
    const submission = await ConsentSubmission.findOneAndDelete({
      _id: id,
      brandId: req.brandId
    });

    if (!submission) {
      return res.status(404).json({ error: 'Submission not found' });
    }

    // Log deletion for audit
    await AuditEvent.create({
      action: 'submission.deleted',
      brandId: req.brandId,
      userId: req.userId,
      entityId: id,
      details: { submissionId: id },
      ip: req.ip,
      userAgent: req.get('User-Agent')
    });

    res.json({ success: true });
  }
}
```

### Password Reset Flow

```typescript
// Secure password reset with email verification
export class PasswordResetController {
  async requestReset(req: Request, res: Response) {
    const { email } = req.body;

    try {
      const user = await AdminUser.findOne({ 
        email: email.toLowerCase(),
        status: 'active'
      }).populate('brand');

      // Always return success (don't leak email existence)
      if (!user) {
        return res.json({ success: true });
      }

      // Generate secure reset token
      const resetToken = crypto.randomBytes(32).toString('hex');
      const resetTokenHash = await argon2.hash(resetToken);

      // Store token with expiration
      await AdminUser.findByIdAndUpdate(user._id, {
        resetToken: resetTokenHash,
        resetTokenExpires: new Date(Date.now() + 60 * 60 * 1000), // 1 hour
      });

      // Send reset email
      await EmailService.sendPasswordReset({
        to: user.email,
        name: user.name,
        resetLink: `${process.env.APP_URL}/reset-password?token=${resetToken}&email=${encodeURIComponent(user.email)}`,
        brandConfig: user.brand
      });

      // Log reset request
      await AuditEvent.create({
        action: 'auth.password_reset_requested',
        brandId: user.brandId,
        userId: user._id,
        details: { email },
        ip: req.ip,
        userAgent: req.get('User-Agent')
      });

      res.json({ success: true });

    } catch (error) {
      console.error('Password reset request error:', error);
      res.status(500).json({ error: 'Reset request failed' });
    }
  }

  async resetPassword(req: Request, res: Response) {
    const { email, token, newPassword } = req.body;

    try {
      const user = await AdminUser.findOne({
        email: email.toLowerCase(),
        status: 'active',
        resetTokenExpires: { $gt: new Date() }
      });

      if (!user || !user.resetToken) {
        return res.status(400).json({ error: 'Invalid or expired reset token' });
      }

      // Verify reset token
      const isValidToken = await argon2.verify(user.resetToken, token);
      if (!isValidToken) {
        return res.status(400).json({ error: 'Invalid or expired reset token' });
      }

      // Hash new password
      const hashedPassword = await PasswordService.hashPassword(newPassword);

      // Update user with new password
      await AdminUser.findByIdAndUpdate(user._id, {
        hashedPassword,
        resetToken: null,
        resetTokenExpires: null,
        passwordChangedAt: new Date()
      });

      // Log password change
      await AuditEvent.create({
        action: 'auth.password_changed',
        brandId: user.brandId,
        userId: user._id,
        details: { method: 'reset_token' },
        ip: req.ip,
        userAgent: req.get('User-Agent')
      });

      res.json({ success: true });

    } catch (error) {
      console.error('Password reset error:', error);
      res.status(500).json({ error: 'Password reset failed' });
    }
  }
}
```

### Multi-Factor Authentication (Optional)

```typescript
// Optional MFA using TOTP (Time-based One-Time Password)
import speakeasy from 'speakeasy';
import qrcode from 'qrcode';

export class MFAController {
  async setupMFA(req: BrandRequest, res: Response) {
    const user = await AdminUser.findById(req.userId);
    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }

    // Generate secret
    const secret = speakeasy.generateSecret({
      name: `${user.email} (${user.brand.name})`,
      issuer: 'Consent Platform'
    });

    // Store temporary secret (not activated until verified)
    await AdminUser.findByIdAndUpdate(user._id, {
      mfaSecretTemp: secret.base32
    });

    // Generate QR code
    const qrCodeUrl = await qrcode.toDataURL(secret.otpauth_url);

    res.json({
      secret: secret.base32,
      qrCode: qrCodeUrl,
      manualEntryKey: secret.base32
    });
  }

  async enableMFA(req: BrandRequest, res: Response) {
    const { token } = req.body;
    
    const user = await AdminUser.findById(req.userId);
    if (!user?.mfaSecretTemp) {
      return res.status(400).json({ error: 'MFA setup not initialized' });
    }

    // Verify token
    const verified = speakeasy.totp.verify({
      secret: user.mfaSecretTemp,
      encoding: 'base32',
      token: token,
      window: 2
    });

    if (!verified) {
      return res.status(400).json({ error: 'Invalid MFA token' });
    }

    // Activate MFA
    await AdminUser.findByIdAndUpdate(user._id, {
      mfaSecret: user.mfaSecretTemp,
      mfaSecretTemp: null,
      mfaEnabled: true,
      mfaEnabledAt: new Date()
    });

    // Log MFA enablement
    await AuditEvent.create({
      action: 'auth.mfa_enabled',
      brandId: req.brandId,
      userId: req.userId,
      ip: req.ip,
      userAgent: req.get('User-Agent')
    });

    res.json({ success: true });
  }
}
```

## Security Features

### Session Security
- **HttpOnly cookies** prevent XSS access to session tokens
- **Secure cookies** in production (HTTPS only)
- **SameSite protection** prevents CSRF attacks
- **Session rotation** on privilege changes
- **Automatic timeout** after inactivity

### Password Security  
- **Argon2id hashing** - industry standard, memory-hard function
- **Secure reset tokens** - cryptographically random, time-limited
- **Password strength enforcement** on client and server
- **Breach detection** integration with HaveIBeenPwned API

### Brand Isolation
- **Complete data separation** between brands
- **Session scope validation** ensures users only access their brand
- **Audit logging** for all authentication events
- **Role-based permissions** with hierarchical access control

### Attack Prevention
- **Rate limiting** on authentication endpoints
- **Account lockout** after failed attempts
- **CSRF protection** on all state-changing operations
- **Input validation** and sanitization
- **SQL injection prevention** through parameterized queries

This authentication architecture provides enterprise-grade security while maintaining the flexibility needed for a multi-brand SaaS platform serving diverse industries.
