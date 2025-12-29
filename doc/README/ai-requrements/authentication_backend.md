---
order: 2
title: ИИ ТРЕБОВАНИЯ по авторизации
---
# Backend Logic: Authentication System

## 1. General Overview
**Purpose:** Secure user authentication and authorization for a web/mobile application with support for multiple authentication methods, session management, and security features.
**High-level logic:** User registration → Credential validation → Token generation → Access control → Session management → Logout/Token invalidation.

## 2. Input Data
### 2.1. Registration Request
```typescript
interface RegistrationRequest {
  email: string;           // Valid email format
  password: string;        // 8+ characters, at least one letter, one number, one special character
  firstName: string;       // 2-50 characters
  lastName: string;        // 2-50 characters
  phone?: string;          // Optional, international format
}
```

### 2.2. Login Request
```typescript
interface LoginRequest {
  email: string;           // Registered email
  password: string;        // User password
  deviceInfo?: {           // Optional device fingerprint
    userAgent: string;
    ipAddress: string;
  };
}
```

### 2.3. Token Refresh Request
```typescript
interface RefreshTokenRequest {
  refreshToken: string;    // Valid refresh token
}
```

### 2.4. Password Reset Request
```typescript
interface PasswordResetRequest {
  email: string;           // Registered email
  newPassword: string;     // New password meeting complexity rules
  resetToken: string;      // Token from reset email
}
```

## 3. Validations
### 3.1. Structural Validations
- **Email**: RFC 5322 format, maximum 255 characters, domain exists (MX record check)
- **Password**: Minimum 8 characters, at least one uppercase, one lowercase, one digit, one special character
- **Name fields**: Only letters, hyphens, apostrophes allowed; length 2-50 characters
- **Phone**: E.164 format if provided

### 3.2. Business Validations
- **Registration**: Email must not already exist in system
- **Login**: Account must be active (not suspended, deleted, or locked)
- **Password reset**: Reset token must be valid and not expired (24-hour TTL)
- **Device**: Suspicious device detection (new location, unusual pattern)

### 3.3. Security Validations
- **Rate limiting**: Maximum 5 login attempts per 15 minutes per IP
- **Password strength**: Check against known breached passwords (HaveIBeenPwned API)
- **Token validation**: JWT signature, issuer, audience, expiration

## 4. Main Logic
### 4.1. Registration Flow
**Step 1: Input Validation**
- Validate all fields structurally
- Check email uniqueness via database query

**Step 2: Password Processing**
```sql
-- Check if email exists
SELECT id FROM users WHERE email = ? AND deleted_at IS NULL;
```
- Hash password using bcrypt with cost factor 12
- Generate email verification token (UUID) with 24-hour expiry

**Step 3: User Creation**
```sql
BEGIN TRANSACTION;
INSERT INTO users (email, password_hash, first_name, last_name, phone, status, created_at) 
VALUES (?, ?, ?, ?, ?, 'PENDING_VERIFICATION', NOW());

INSERT INTO email_verifications (user_id, token, expires_at) 
VALUES (LAST_INSERT_ID(), ?, DATE_ADD(NOW(), INTERVAL 24 HOUR));
COMMIT;
```

**Step 4: Notification**
- Send verification email with link containing token
- Log registration event for analytics

### 4.2. Login Flow
**Step 1: Credential Validation**
- Find user by email (case-insensitive search)
- Verify account status (must be 'ACTIVE' or 'PENDING_VERIFICATION')

**Step 2: Password Verification**
```typescript
const isValid = await bcrypt.compare(password, storedHash);
if (!isValid) {
  incrementFailedAttempts(userId);
  if (failedAttempts >= 5) lockAccount(userId);
  throw new AuthenticationError('Invalid credentials');
}
```

**Step 3: Token Generation**
- Generate JWT access token (15-minute expiry) with claims:
  ```json
  {
    "sub": "user-id",
    "email": "user@example.com",
    "roles": ["USER"],
    "iat": 1672531200,
    "exp": 1672532100
  }
  ```
- Generate refresh token (7-day expiry, stored in database)
- Record login session with device info

**Step 4: Response Preparation**
- Return tokens in HTTP-only cookies (or Authorization header)
- Update last login timestamp

### 4.3. Token Refresh Flow
**Step 1: Refresh Token Validation**
- Verify token exists in database and is not revoked
- Check expiration date
- Validate associated user is still active

**Step 2: New Token Generation**
- Issue new access token with same claims
- Optionally rotate refresh token (depending on security policy)

**Step 3: Session Update**
- Update token usage timestamp
- Log refresh event for security monitoring

### 4.4. Password Reset Flow
**Step 1: Request Initiation**
- Validate email exists
- Generate reset token (cryptographically random)
- Store token hash with expiry (1 hour)
- Send email with reset link

**Step 2: Password Update**
- Verify token matches hash and is not expired
- Validate new password meets complexity requirements
- Update password hash in database
- Invalidate all existing sessions and tokens for the user
- Send confirmation email

## 5. Integrations
### 5.1. Database
- **Read operations**: users, sessions, tokens, audit_logs
- **Write operations**: users (password hash), sessions, tokens, audit_logs

### 5.2. External Services
- **Email Service**: SendGrid/Mailgun for transactional emails (verification, reset)
- **SMS Service**: Twilio for 2FA codes (if enabled)
- **Security Service**: Fraud detection, IP reputation check
- **Monitoring**: Log authentication events to SIEM

### 5.3. Cache (Redis)
- **Rate limiting**: Store attempt counts per IP/user
- **Token blacklist**: Revoked tokens (until expiry)
- **Session data**: Active sessions for quick validation

## 6. Exceptional Situations
### 6.1. Validation Errors (HTTP 400)
- Invalid email format → "Please provide a valid email address"
- Weak password → "Password must contain at least 8 characters with uppercase, lowercase, number, and special character"
- Email already registered → "An account with this email already exists"

### 6.2. Authentication Errors (HTTP 401)
- Invalid credentials → "Email or password is incorrect"
- Expired token → "Session expired, please login again"
- Revoked token → "Invalid authentication token"

### 6.3. Authorization Errors (HTTP 403)
- Insufficient permissions → "You don't have permission to access this resource"
- Account locked → "Account temporarily locked due to multiple failed attempts"

### 6.4. Server Errors (HTTP 500)
- Database connection failure → Retry with exponential backoff
- External service unavailable → Fallback to local validation, queue email for later

### 6.5. Recovery Strategies
- **Transactional rollback**: On any error during user creation, rollback entire transaction
- **Compensating actions**: If email sending fails, mark for retry in queue
- **Graceful degradation**: If Redis cache unavailable, use database with performance impact

## 7. Output Data
### 7.1. Successful Login Response (200)
```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIs...",
  "refreshToken": "dGhpcyBpcyBhIHJlZnJlc2ggdG9rZW4",
  "expiresIn": 900,
  "tokenType": "Bearer",
  "user": {
    "id": "user-123",
    "email": "user@example.com",
    "firstName": "John",
    "lastName": "Doe",
    "roles": ["USER"]
  }
}
```

### 7.2. Error Response (400)
```json
{
  "error": "VALIDATION_ERROR",
  "message": "Invalid input data",
  "details": [
    {
      "field": "password",
      "message": "Must contain at least one special character"
    }
  ]
}
```

## 8. Performance
### 8.1. Optimizations
- **Database indexing**: Index on email (unique), foreign keys for relationships
- **Caching**: Frequently accessed user data, token validation results
- **Connection pooling**: Database and Redis connection reuse
- **Asynchronous operations**: Email sending, audit logging via message queue

### 8.2. Limitations
- **Maximum concurrent users**: 10,000 based on token validation throughput
- **Response time**: < 200ms for login (95th percentile)
- **Availability**: 99.95% uptime SLA
- **Geographic latency**: Use regional token validation caches for global users