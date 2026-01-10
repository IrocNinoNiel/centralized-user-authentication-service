# Centralized User Authentication Service

> **Version:** 1.0
> **Last Updated:** 2026-01-10

This service handles the single source of truth for user authentication and authorization across all microservices. It manages users, roles, permissions, services, and modules.

---

## Table of Contents

1. [API Endpoints](#api-endpoints)
2. [Business Logic](#business-logic)
3. [Database Schema](#database-schema)
4. [Tech Stack](#tech-stack)
5. [Code Patterns](#code-patterns)
6. [Response Structure](#response-structure)
7. [Security Considerations](#security-considerations)

---

## API Endpoints

### Authentication

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/auth/login` | User login |
| POST | `/api/v1/auth/logout` | User logout (invalidate token) |
| POST | `/api/v1/auth/register` | User registration |
| POST | `/api/v1/auth/forgot-password` | Request password reset |
| POST | `/api/v1/auth/reset-password` | **[ADDED]** Reset password with token |
| POST | `/api/v1/auth/change-password` | **[ADDED]** Change password (authenticated) |
| POST | `/api/v1/auth/verify-email/{token}` | **[ADDED]** Verify email address |
| POST | `/api/v1/auth/resend-verification` | **[ADDED]** Resend verification email |
| POST | `/api/v1/auth/verify-otp` | **[ADDED]** Verify OTP for new device |
| POST | `/api/v1/auth/resend-otp` | **[ADDED]** Resend OTP (limited) |
| POST | `/api/v1/auth/refresh-token` | **[ADDED]** Refresh access token |
| GET | `/api/v1/auth/validate-token` | **[ADDED]** Validate token (for other services) |

### Users

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/users` | Get all users (paginated) |
| GET | `/api/v1/users/{uid}` | Get one user |
| POST | `/api/v1/users` | Create user (admin) |
| PUT | `/api/v1/users/{uid}` | Update user |
| DELETE | `/api/v1/users/{uid}` | Delete user (soft delete) |
| POST | `/api/v1/users/{uid}/block` | Block user |
| POST | `/api/v1/users/{uid}/unblock` | **[ADDED]** Unblock user |
| POST | `/api/v1/users/{uid}/unlock` | **[ADDED]** Unlock user after lockout |
| GET | `/api/v1/users/{uid}/sessions` | **[ADDED]** Get user active sessions |
| DELETE | `/api/v1/users/{uid}/sessions` | **[ADDED]** Revoke all user sessions |

### Roles

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/roles` | Get all roles |
| GET | `/api/v1/roles/{uid}` | Get one role |
| POST | `/api/v1/roles` | Create role |
| PUT | `/api/v1/roles/{uid}` | Update role |
| DELETE | `/api/v1/roles/{uid}` | Delete role (soft delete) |
| PUT | `/api/v1/roles/{uid}/permissions` | Update role permissions |

### Services

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/services` | Get all services |
| GET | `/api/v1/services/{uid}` | Get one service |
| POST | `/api/v1/services` | Create service |
| PUT | `/api/v1/services/{uid}` | Update service |
| DELETE | `/api/v1/services/{uid}` | Delete service (soft delete) |

### Modules

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/modules` | Get all modules |
| GET | `/api/v1/modules/{uid}` | Get one module |
| POST | `/api/v1/modules` | Create module |
| PUT | `/api/v1/modules/{uid}` | Update module |
| DELETE | `/api/v1/modules/{uid}` | Delete module (soft delete) |

### Permission Overrides

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/users/{uid}/permission-overrides` | Add user permission override |
| GET | `/api/v1/users/{uid}/permission-overrides` | **[ADDED]** Get user permission overrides |
| DELETE | `/api/v1/users/{uid}/permission-overrides/{override_uid}` | **[ADDED]** Remove permission override |
| GET | `/api/v1/permissions/check` | **[ADDED]** Check if user has permission (for other services) |

---

## Business Logic

### Login Flow

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐     ┌──────────────┐
│   Submit    │────▶│   Validate   │────▶│ Check Device│────▶│   Generate   │
│ Credentials │     │  User/Pass   │     │   & IP      │     │    Token     │
└─────────────┘     └──────────────┘     └─────────────┘     └──────────────┘
                           │                    │
                           ▼                    ▼
                    ┌──────────────┐     ┌─────────────┐
                    │   Increment  │     │  Send OTP   │
                    │   Attempts   │     │ (New Device)│
                    └──────────────┘     └─────────────┘
```

**Rules:**
- Must be correct username/email and password
- Maximum **3 login attempts** before lockout
- **[SUGGESTION]** Track attempts per IP + username combination
- If User IP + Device is new → require OTP verification
- OTP expiry: **10 minutes** ~~30 minutes~~ **[CHANGED - 30 min is too long for security]**
- Maximum **1 OTP resend** per session
- If max attempts OR max OTP resends reached → **1-hour lockout**
- Generate JWT access token (15 min expiry) + refresh token (7 days expiry) **[ADDED]**

**[ADDED] Token Strategy:**
- Use **JWT** for stateless authentication
- Access Token: Short-lived (15 minutes)
- Refresh Token: Longer-lived (7 days), stored in `sessions` table
- On logout: Blacklist refresh token

### Permission Logic

```
┌─────────────────────────────────────────────────────────────────┐
│                    Permission Check Flow                         │
├─────────────────────────────────────────────────────────────────┤
│  1. Check user_permission_overrides (highest priority)          │
│     ├── If GRANTED override exists → ALLOW                      │
│     └── If DENIED override exists → DENY                        │
│                                                                  │
│  2. Check role_permissions for ALL user roles                   │
│     └── If ANY role has permission → ALLOW                      │
│                                                                  │
│  3. Default → DENY                                              │
└─────────────────────────────────────────────────────────────────┘
```

**Rules:**
- User can be assigned to **1 or many roles**
- If **any** role has the required permission → action proceeds
- Admin can add **permission overrides** per user
  - Override can GRANT or DENY **[ADDED - clarification needed]**
  - Override can have **optional expiry date** for temporary permissions **[ADDED]**
- **[ADDED]** Permission format: `{service}.{module}.{action}` (e.g., `auth.users.create`)

### Roles Business Logic

**Rules:**
- Role name must be **unique** (case-insensitive recommended)
- Roles use **soft delete** (`deleted_at` populated)
- Role **cannot be deleted** if users are assigned (active OR inactive)
- **[ADDED]** Consider having system roles that cannot be deleted (e.g., `super-admin`)

### User Registration & Update

**Code Generation:**
- Format: `USR-XXXX` (auto-increment)
- Must be unique across **ALL users** (active and inactive)

**Uniqueness Constraints:**
- `username` - unique among **active users only**
- `email` - unique among **active users only**
- `code` - unique among **ALL users**

**Password Requirements:**
- Minimum **8 characters** ~~digits~~ **[CLARIFICATION]**
- Must contain:
  - Lowercase letter (a-z)
  - Uppercase letter (A-Z)
  - Number (0-9)
  - Special character (!@#$%^&*, etc.)
- **[ADDED]** Password hashing: Use **bcrypt** (cost factor 12) or **Argon2id**

**Email Verification:**
- Send verification email on registration
- **[ADDED]** Verification token expiry: 24 hours
- **[ADDED]** User cannot login until email is verified (configurable)

### Services CRUD

**Rules:**
- Service `name` must be **unique**
- Use **soft delete**
- **[ADDED]** Service has a unique `code` for API identification
- Introduces dependency on other services (accepted trade-off)

### Modules CRUD

**Rules:**
- Module `name` must be **unique within a service** **[CLARIFICATION NEEDED]**
- Use **soft delete**
- **[ADDED]** Module belongs to a Service (foreign key)

---

## Database Schema

### Tables Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                         RELATIONSHIPS                             │
├──────────────────────────────────────────────────────────────────┤
│  users ──────┬───── user_roles ─────┬───── roles                 │
│              │                      │                             │
│              │                      └───── role_permissions ──┐   │
│              │                                                │   │
│              └───── user_permission_overrides ────────────────┤   │
│                                                               │   │
│  services ─────── modules ────────────────────────────────────┘   │
│                                                                   │
│  [ADDED TABLES]                                                   │
│  sessions (device tracking & refresh tokens)                      │
│  login_attempts (rate limiting)                                   │
│  password_resets                                                  │
│  email_verifications                                              │
└──────────────────────────────────────────────────────────────────┘
```

### Common Columns (All Tables)

```sql
-- Primary Keys
id          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY
uid         BINARY(16) UNIQUE NOT NULL  -- UUID stored as binary

-- Audit Trail
created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
created_by  BINARY(16) NULL  -- References users.uid
updated_at  TIMESTAMP NULL ON UPDATE CURRENT_TIMESTAMP
updated_by  BINARY(16) NULL  -- References users.uid
deleted_at  TIMESTAMP NULL   -- Soft delete
status      ENUM('active', 'inactive') DEFAULT 'active'
archived    BOOLEAN DEFAULT FALSE
```

### Table: `users`

```sql
CREATE TABLE users (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    uid             BINARY(16) UNIQUE NOT NULL,
    code            VARCHAR(20) UNIQUE NOT NULL,           -- USR-XXXX
    username        VARCHAR(100) NOT NULL,
    email           VARCHAR(255) NOT NULL,
    password        VARCHAR(255) NOT NULL,                 -- bcrypt hash
    email_verified_at TIMESTAMP NULL,                      -- [ADDED]
    is_blocked      BOOLEAN DEFAULT FALSE,
    blocked_at      TIMESTAMP NULL,                        -- [ADDED]
    blocked_by      BINARY(16) NULL,                       -- [ADDED]
    blocked_reason  TEXT NULL,                             -- [ADDED]
    locked_until    TIMESTAMP NULL,                        -- [ADDED] Login lockout
    -- Audit columns
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by      BINARY(16) NULL,
    updated_at      TIMESTAMP NULL,
    updated_by      BINARY(16) NULL,
    deleted_at      TIMESTAMP NULL,
    status          ENUM('active', 'inactive') DEFAULT 'active',
    archived        BOOLEAN DEFAULT FALSE,

    INDEX idx_username (username),
    INDEX idx_email (email),
    INDEX idx_status (status),
    INDEX idx_code (code)
);
```

### Table: `roles`

```sql
CREATE TABLE roles (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    uid             BINARY(16) UNIQUE NOT NULL,
    name            VARCHAR(100) UNIQUE NOT NULL,
    description     TEXT NULL,
    is_system       BOOLEAN DEFAULT FALSE,                 -- [ADDED] System roles can't be deleted
    -- Audit columns
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by      BINARY(16) NULL,
    updated_at      TIMESTAMP NULL,
    updated_by      BINARY(16) NULL,
    deleted_at      TIMESTAMP NULL,
    status          ENUM('active', 'inactive') DEFAULT 'active',
    archived        BOOLEAN DEFAULT FALSE
);
```

### Table: `services`

```sql
CREATE TABLE services (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    uid             BINARY(16) UNIQUE NOT NULL,
    name            VARCHAR(100) UNIQUE NOT NULL,
    code            VARCHAR(50) UNIQUE NOT NULL,           -- [ADDED] e.g., 'auth', 'inventory'
    description     TEXT NULL,
    base_url        VARCHAR(255) NULL,                     -- [ADDED] Service URL
    -- Audit columns
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by      BINARY(16) NULL,
    updated_at      TIMESTAMP NULL,
    updated_by      BINARY(16) NULL,
    deleted_at      TIMESTAMP NULL,
    status          ENUM('active', 'inactive') DEFAULT 'active',
    archived        BOOLEAN DEFAULT FALSE
);
```

### Table: `modules`

```sql
CREATE TABLE modules (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    uid             BINARY(16) UNIQUE NOT NULL,
    service_uid     BINARY(16) NOT NULL,                   -- [ADDED] FK to services
    name            VARCHAR(100) NOT NULL,
    code            VARCHAR(50) NOT NULL,                  -- [ADDED] e.g., 'users', 'roles'
    description     TEXT NULL,
    -- Audit columns
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by      BINARY(16) NULL,
    updated_at      TIMESTAMP NULL,
    updated_by      BINARY(16) NULL,
    deleted_at      TIMESTAMP NULL,
    status          ENUM('active', 'inactive') DEFAULT 'active',
    archived        BOOLEAN DEFAULT FALSE,

    UNIQUE KEY unique_service_module (service_uid, name),
    FOREIGN KEY (service_uid) REFERENCES services(uid)
);
```

### Table: `user_roles`

```sql
CREATE TABLE user_roles (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    uid             BINARY(16) UNIQUE NOT NULL,
    user_uid        BINARY(16) NOT NULL,
    role_uid        BINARY(16) NOT NULL,
    -- Audit columns
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by      BINARY(16) NULL,
    updated_at      TIMESTAMP NULL,
    updated_by      BINARY(16) NULL,
    deleted_at      TIMESTAMP NULL,
    status          ENUM('active', 'inactive') DEFAULT 'active',
    archived        BOOLEAN DEFAULT FALSE,

    UNIQUE KEY unique_user_role (user_uid, role_uid),
    FOREIGN KEY (user_uid) REFERENCES users(uid),
    FOREIGN KEY (role_uid) REFERENCES roles(uid)
);
```

### Table: `role_permissions`

```sql
CREATE TABLE role_permissions (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    uid             BINARY(16) UNIQUE NOT NULL,
    role_uid        BINARY(16) NOT NULL,
    module_uid      BINARY(16) NOT NULL,
    can_create      BOOLEAN DEFAULT FALSE,
    can_read        BOOLEAN DEFAULT FALSE,
    can_update      BOOLEAN DEFAULT FALSE,
    can_delete      BOOLEAN DEFAULT FALSE,
    -- Audit columns
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by      BINARY(16) NULL,
    updated_at      TIMESTAMP NULL,
    updated_by      BINARY(16) NULL,
    deleted_at      TIMESTAMP NULL,
    status          ENUM('active', 'inactive') DEFAULT 'active',
    archived        BOOLEAN DEFAULT FALSE,

    UNIQUE KEY unique_role_module (role_uid, module_uid),
    FOREIGN KEY (role_uid) REFERENCES roles(uid),
    FOREIGN KEY (module_uid) REFERENCES modules(uid)
);
```

### Table: `user_permission_overrides`

```sql
CREATE TABLE user_permission_overrides (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    uid             BINARY(16) UNIQUE NOT NULL,
    user_uid        BINARY(16) NOT NULL,
    module_uid      BINARY(16) NOT NULL,
    permission_type ENUM('grant', 'deny') NOT NULL,        -- [ADDED]
    can_create      BOOLEAN DEFAULT FALSE,
    can_read        BOOLEAN DEFAULT FALSE,
    can_update      BOOLEAN DEFAULT FALSE,
    can_delete      BOOLEAN DEFAULT FALSE,
    expires_at      TIMESTAMP NULL,                        -- [ADDED] For temporary permissions
    reason          TEXT NULL,                             -- [ADDED] Why override was granted
    -- Audit columns
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by      BINARY(16) NULL,
    updated_at      TIMESTAMP NULL,
    updated_by      BINARY(16) NULL,
    deleted_at      TIMESTAMP NULL,
    status          ENUM('active', 'inactive') DEFAULT 'active',
    archived        BOOLEAN DEFAULT FALSE,

    FOREIGN KEY (user_uid) REFERENCES users(uid),
    FOREIGN KEY (module_uid) REFERENCES modules(uid)
);
```

### **[ADDED]** Table: `sessions`

```sql
CREATE TABLE sessions (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    uid             BINARY(16) UNIQUE NOT NULL,
    user_uid        BINARY(16) NOT NULL,
    refresh_token   VARCHAR(500) NOT NULL,
    ip_address      VARCHAR(45) NOT NULL,                  -- Supports IPv6
    user_agent      TEXT NULL,
    device_name     VARCHAR(255) NULL,
    device_hash     VARCHAR(64) NOT NULL,                  -- Hash of IP + User-Agent
    is_trusted      BOOLEAN DEFAULT FALSE,                 -- Device has been OTP verified
    last_activity   TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at      TIMESTAMP NOT NULL,
    revoked_at      TIMESTAMP NULL,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_user (user_uid),
    INDEX idx_device_hash (device_hash),
    INDEX idx_refresh_token (refresh_token(255)),
    FOREIGN KEY (user_uid) REFERENCES users(uid)
);
```

### **[ADDED]** Table: `login_attempts`

```sql
CREATE TABLE login_attempts (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    uid             BINARY(16) UNIQUE NOT NULL,
    user_uid        BINARY(16) NULL,                       -- NULL if user doesn't exist
    username_tried  VARCHAR(100) NOT NULL,
    ip_address      VARCHAR(45) NOT NULL,
    user_agent      TEXT NULL,
    success         BOOLEAN DEFAULT FALSE,
    failure_reason  VARCHAR(100) NULL,                     -- 'invalid_password', 'user_blocked', etc.
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_user (user_uid),
    INDEX idx_ip (ip_address),
    INDEX idx_created (created_at)
);
```

### **[ADDED]** Table: `password_resets`

```sql
CREATE TABLE password_resets (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    uid             BINARY(16) UNIQUE NOT NULL,
    user_uid        BINARY(16) NOT NULL,
    token           VARCHAR(255) NOT NULL,
    expires_at      TIMESTAMP NOT NULL,
    used_at         TIMESTAMP NULL,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_token (token),
    FOREIGN KEY (user_uid) REFERENCES users(uid)
);
```

### **[ADDED]** Table: `otp_verifications`

```sql
CREATE TABLE otp_verifications (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    uid             BINARY(16) UNIQUE NOT NULL,
    user_uid        BINARY(16) NOT NULL,
    otp_code        VARCHAR(6) NOT NULL,                   -- 6-digit OTP
    type            ENUM('login', 'email_verify', 'password_reset') NOT NULL,
    ip_address      VARCHAR(45) NOT NULL,
    device_hash     VARCHAR(64) NOT NULL,
    attempts        TINYINT DEFAULT 0,
    resend_count    TINYINT DEFAULT 0,
    expires_at      TIMESTAMP NOT NULL,
    verified_at     TIMESTAMP NULL,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_user (user_uid),
    INDEX idx_otp (otp_code),
    FOREIGN KEY (user_uid) REFERENCES users(uid)
);
```

---

## Tech Stack

### Backend Framework
- **Laravel 11** (PHP 8.2+)

### Database
- **MySQL 8.0** or **PostgreSQL 15**

### Cache & Session
- **[ADDED] Redis** - For token blacklisting, rate limiting, and caching

### Email Service (Recommendations)

| Service | Pros | Cons | Free Tier |
|---------|------|------|-----------|
| **Mailgun** (Recommended) | Great Laravel integration, reliable | Limited dashboard | 5,000/month |
| **Amazon SES** | Cheapest at scale | AWS setup required | 62,000/month (EC2) |
| **SendGrid** | Good analytics | Can be complex | 100/day |
| **Postmark** | Best deliverability | Higher cost | 100/month |

**Recommendation:** Start with **Mailgun** for development/MVP, migrate to **Amazon SES** at scale.

### **[ADDED]** Queue System
- **Laravel Queue with Redis** - For async email sending

---

## Code Patterns

### Architecture: Service Layer Pattern

```
┌─────────────────────────────────────────────────────────────┐
│                      HTTP Request                           │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  Controller                                                  │
│  - Receives request                                          │
│  - Calls FormRequest for validation                          │
│  - Delegates to Service                                      │
│  - Returns Resource response                                 │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  FormRequest                                                 │
│  - Input validation                                          │
│  - Authorization check                                       │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  Service                                                     │
│  - Business logic                                            │
│  - Orchestrates repositories                                 │
│  - Transaction management                                    │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  Interface                                                   │
│  - Contract for repository                                   │
│  - Enables dependency injection                              │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  Repository                                                  │
│  - Data access layer                                         │
│  - Database queries                                          │
│  - Model interactions                                        │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  Resource                                                    │
│  - Response transformation                                   │
│  - Data formatting                                           │
│  - UUID conversion                                           │
└─────────────────────────────────────────────────────────────┘
```

### Directory Structure

```
app/
├── Http/
│   ├── Controllers/
│   │   └── Api/
│   │       └── V1/
│   │           ├── AuthController.php
│   │           ├── UserController.php
│   │           ├── RoleController.php
│   │           ├── ServiceController.php
│   │           └── ModuleController.php
│   ├── Requests/
│   │   └── Api/
│   │       └── V1/
│   │           ├── Auth/
│   │           │   ├── LoginRequest.php
│   │           │   ├── RegisterRequest.php
│   │           │   └── ...
│   │           ├── User/
│   │           └── ...
│   ├── Resources/
│   │   └── Api/
│   │       └── V1/
│   │           ├── UserResource.php
│   │           ├── UserCollection.php
│   │           └── ...
│   └── Middleware/
│       ├── ValidateJwtToken.php
│       └── CheckPermission.php
├── Services/
│   ├── Interfaces/
│   │   ├── AuthServiceInterface.php
│   │   ├── UserServiceInterface.php
│   │   └── ...
│   ├── AuthService.php
│   ├── UserService.php
│   ├── RoleService.php
│   ├── PermissionService.php
│   └── ...
├── Repositories/
│   ├── Interfaces/
│   │   ├── UserRepositoryInterface.php
│   │   └── ...
│   ├── UserRepository.php
│   └── ...
├── Models/
│   ├── User.php
│   ├── Role.php
│   └── ...
├── Utils/                                    -- [ADDED] Utilities
│   ├── UuidHelper.php                        -- UUID <-> Binary conversion
│   ├── CodeGenerator.php                     -- USR-XXXX generation
│   └── PasswordValidator.php
└── Constants/                                -- [ADDED] Constants
    ├── StatusConstants.php
    ├── PermissionConstants.php
    └── ErrorCodeConstants.php
```

### UUID Helper Utility

```php
<?php
// app/Utils/UuidHelper.php

namespace App\Utils;

use Ramsey\Uuid\Uuid;

class UuidHelper
{
    /**
     * Convert UUID string to binary(16) for database storage
     */
    public static function toBinary(string $uuid): string
    {
        return Uuid::fromString($uuid)->getBytes();
    }

    /**
     * Convert binary(16) to UUID string for API response
     */
    public static function toUuid(string $binary): string
    {
        return Uuid::fromBytes($binary)->toString();
    }

    /**
     * Generate new UUID as binary
     */
    public static function generateBinary(): string
    {
        return Uuid::uuid4()->getBytes();
    }

    /**
     * Generate new UUID as string
     */
    public static function generate(): string
    {
        return Uuid::uuid4()->toString();
    }
}
```

---

## Response Structure

### Success Response

```json
{
    "status": 200,
    "message": "Data successfully retrieved",
    "data": {
        "id": "550e8400-e29b-41d4-a716-446655440000",
        "username": "johndoe",
        "email": "john@example.com"
    }
}
```

### Success Response (Collection with Pagination) **[ADDED]**

```json
{
    "status": 200,
    "message": "Data successfully retrieved",
    "data": [
        { "id": "...", "username": "user1" },
        { "id": "...", "username": "user2" }
    ],
    "meta": {
        "current_page": 1,
        "per_page": 15,
        "total": 100,
        "total_pages": 7,
        "has_more": true
    }
}
```

### Error Response **[ADDED]**

```json
{
    "status": 422,
    "message": "Validation failed",
    "errors": {
        "email": ["The email field is required."],
        "password": ["The password must be at least 8 characters."]
    }
}
```

### Error Response (Single Error) **[ADDED]**

```json
{
    "status": 401,
    "message": "Invalid credentials",
    "error_code": "AUTH_INVALID_CREDENTIALS"
}
```

---

## Security Considerations

### **[ADDED]** Security Checklist

- [ ] **Rate Limiting**: Implement per-IP and per-user rate limits
- [ ] **CORS**: Configure allowed origins for API access
- [ ] **HTTPS**: Enforce HTTPS in production
- [ ] **Password Hashing**: Use bcrypt (cost 12) or Argon2id
- [ ] **JWT Security**:
  - Short-lived access tokens (15 min)
  - HTTP-only refresh token cookies (optional)
  - Token blacklisting on logout
- [ ] **SQL Injection**: Use parameterized queries (Eloquent ORM)
- [ ] **XSS Prevention**: Escape output, validate input
- [ ] **CSRF Protection**: Laravel's built-in CSRF for web routes
- [ ] **Audit Logging**: Log all authentication events
- [ ] **Sensitive Data**: Never log passwords or tokens

### **[ADDED]** Inter-Service Authentication

For other microservices to validate tokens/permissions:

```
Option 1: REST API Call (Recommended for simplicity)
┌─────────────┐     GET /api/v1/auth/validate-token      ┌──────────────┐
│  Service A  │ ──────────────────────────────────────▶  │  Auth Service│
└─────────────┘     Authorization: Bearer <token>        └──────────────┘

Option 2: JWT Self-Validation (Recommended for performance)
- Share public key with all services
- Services decode and validate JWT locally
- Only call Auth Service for permission checks
```

---

## Revision History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-01-10 | Initial documentation with integrated feedback |

---

## Feedback Summary

### Original Items Preserved
- All 25 original API endpoints
- Login business logic (3 attempts, OTP, 1-hour lockout)
- Permission logic (multi-role, override capability)
- Role soft delete rules
- User code uniqueness rules
- Password requirements
- Service layer architecture
- All 7 original tables
- Audit trail columns
- UUID/binary(16) pattern

### Additions & Suggestions Made
1. **APIs Added**: Token refresh, validate, password reset, email verification, OTP endpoints, session management, permission check
2. **Security**: Reduced OTP expiry from 30 to 10 minutes, added token strategy (JWT), added password hashing specification
3. **Tables Added**: sessions, login_attempts, password_resets, otp_verifications
4. **Schema Enhancements**: Added expires_at to permission overrides, added is_system to roles, added device tracking fields
5. **Response Structure**: Added pagination meta, error response format, error codes
6. **Code Patterns**: Added directory structure, UUID helper implementation
7. **Mailer Recommendation**: Mailgun for MVP, SES for scale
8. **Inter-service Communication**: Added validation strategies
