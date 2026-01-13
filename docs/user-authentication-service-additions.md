# Centralized User Authentication Service - Additional Documentation

> **Version:** 1.2
> **Last Updated:** 2026-01-14

This document contains additional specifications that complement the main documentation.

---

## Table of Contents

1. [API Specification Details](#api-specification-details)
2. [Environment Configuration](#environment-configuration)
3. [Error Codes](#error-codes)
4. [Business Logic Clarifications](#business-logic-clarifications)
   - [Email Verification Flow](#email-verification-flow)
   - [Password Reset Flow](#password-reset-flow)
   - [Session Management](#session-management)
   - [Device Trust & OTP Logic](#device-trust--otp-logic)
   - [Admin-Created Users](#admin-created-users)
   - [Role Assignment Rules](#role-assignment-rules)
   - [Permission Override Handling](#permission-override-handling)
   - [Data Status vs Soft Delete](#data-status-vs-soft-delete)
5. [Testing Strategy](#testing-strategy)
6. [Deployment & DevOps](#deployment--devops)
7. [API Authentication Rules](#api-authentication-rules)
   - [Endpoint Access Matrix](#endpoint-access-matrix)
   - [Permission Check Flow](#permission-check-flow)
   - [Service-to-Service Authentication](#service-to-service-authentication)
8. [Audit Logs Schema](#audit-logs-schema)
9. [Soft Delete Handling](#soft-delete-handling)

---

## API Specification Details

### Authentication Endpoints

#### POST `/api/v1/auth/login`

**Request:**
```json
{
    "login": "johndoe",              // Required - username or email
    "password": "SecurePass123!"     // Required
}
```

**Success Response (200):**
```json
{
    "status": 200,
    "message": "Login successful",
    "data": {
        "user": {
            "uid": "550e8400-e29b-41d4-a716-446655440000",
            "code": "USR-0001",
            "username": "johndoe",
            "email": "john@example.com",
            "email_verified_at": "2026-01-10T10:00:00Z",
            "roles": [
                {
                    "uid": "660e8400-e29b-41d4-a716-446655440001",
                    "name": "user"
                }
            ]
        },
        "access_token": "eyJhbGciOiJIUzI1NiIs...",
        "refresh_token": "eyJhbGciOiJIUzI1NiIs...",
        "token_type": "Bearer",
        "expires_in": 900
    }
}
```

**OTP Required Response (200):**
```json
{
    "status": 200,
    "message": "OTP verification required for new device",
    "data": {
        "requires_otp": true,
        "otp_session_id": "770e8400-e29b-41d4-a716-446655440002",
        "expires_in": 600
    }
}
```

**Error Response (401):**
```json
{
    "status": 401,
    "message": "Invalid credentials",
    "error_code": "AUTH_INVALID_CREDENTIALS"
}
```

**Error Response (423 - Locked):**
```json
{
    "status": 423,
    "message": "Account temporarily locked due to too many failed attempts",
    "error_code": "AUTH_ACCOUNT_LOCKED",
    "data": {
        "locked_until": "2026-01-13T15:30:00Z",
        "remaining_minutes": 45
    }
}
```

---

#### POST `/api/v1/auth/register`

**Request:**
```json
{
    "username": "johndoe",           // Required - min 3, max 100 chars
    "email": "john@example.com",     // Required - valid email format
    "password": "SecurePass123!",    // Required - min 8 chars, complexity rules
    "password_confirmation": "SecurePass123!"  // Required - must match password
}
```

**Success Response (201):**
```json
{
    "status": 201,
    "message": "Registration successful. Please verify your email.",
    "data": {
        "user": {
            "uid": "550e8400-e29b-41d4-a716-446655440000",
            "code": "USR-0001",
            "username": "johndoe",
            "email": "john@example.com",
            "email_verified_at": null
        }
    }
}
```

**Error Response (422):**
```json
{
    "status": 422,
    "message": "Validation failed",
    "error_code": "VALIDATION_ERROR",
    "errors": {
        "email": ["The email has already been taken."],
        "password": ["The password must contain at least one uppercase letter."]
    }
}
```

---

#### POST `/api/v1/auth/logout`

**Headers:**
```
Authorization: Bearer <access_token>
```

**Request:**
```json
{
    "refresh_token": "eyJhbGciOiJIUzI1NiIs..."  // Optional - if not provided, only current session is invalidated
}
```

**Success Response (200):**
```json
{
    "status": 200,
    "message": "Logout successful"
}
```

---

#### POST `/api/v1/auth/forgot-password`

**Request:**
```json
{
    "email": "john@example.com"      // Required
}
```

**Success Response (200):**
```json
{
    "status": 200,
    "message": "If the email exists, a password reset link has been sent."
}
```

> **Note:** Always return success to prevent email enumeration attacks.

---

#### POST `/api/v1/auth/reset-password`

**Request:**
```json
{
    "token": "a1b2c3d4e5f6...",       // Required - from email link
    "password": "NewSecurePass123!",  // Required
    "password_confirmation": "NewSecurePass123!"  // Required
}
```

**Success Response (200):**
```json
{
    "status": 200,
    "message": "Password has been reset successfully"
}
```

**Error Response (400 - Expired):**
```json
{
    "status": 400,
    "message": "Password reset link has expired. Please request a new one.",
    "error_code": "AUTH_RESET_TOKEN_EXPIRED"
}
```

---

#### POST `/api/v1/auth/change-password`

**Headers:**
```
Authorization: Bearer <access_token>
```

**Request:**
```json
{
    "current_password": "OldPass123!",    // Required
    "password": "NewSecurePass456!",      // Required
    "password_confirmation": "NewSecurePass456!"  // Required
}
```

**Success Response (200):**
```json
{
    "status": 200,
    "message": "Password changed successfully"
}
```

**Error Response (400):**
```json
{
    "status": 400,
    "message": "Current password is incorrect",
    "error_code": "AUTH_INVALID_CURRENT_PASSWORD"
}
```

---

#### POST `/api/v1/auth/verify-email/{token}`

**Success Response (200):**
```json
{
    "status": 200,
    "message": "Email verified successfully"
}
```

**Error Response (400):**
```json
{
    "status": 400,
    "message": "Email verification link has expired or is invalid",
    "error_code": "AUTH_EMAIL_VERIFY_INVALID"
}
```

---

#### POST `/api/v1/auth/resend-verification`

**Request:**
```json
{
    "email": "john@example.com"      // Required
}
```

**Success Response (200):**
```json
{
    "status": 200,
    "message": "Verification email has been sent"
}
```

---

#### POST `/api/v1/auth/verify-otp`

**Request:**
```json
{
    "otp_session_id": "770e8400-e29b-41d4-a716-446655440002",  // Required
    "otp_code": "123456"             // Required - 6 digits
}
```

**Success Response (200):**
```json
{
    "status": 200,
    "message": "OTP verified successfully",
    "data": {
        "user": {
            "uid": "550e8400-e29b-41d4-a716-446655440000",
            "code": "USR-0001",
            "username": "johndoe",
            "email": "john@example.com"
        },
        "access_token": "eyJhbGciOiJIUzI1NiIs...",
        "refresh_token": "eyJhbGciOiJIUzI1NiIs...",
        "token_type": "Bearer",
        "expires_in": 900
    }
}
```

**Error Response (400):**
```json
{
    "status": 400,
    "message": "Invalid OTP code",
    "error_code": "AUTH_INVALID_OTP",
    "data": {
        "attempts_remaining": 2
    }
}
```

---

#### POST `/api/v1/auth/resend-otp`

**Request:**
```json
{
    "otp_session_id": "770e8400-e29b-41d4-a716-446655440002"  // Required
}
```

**Success Response (200):**
```json
{
    "status": 200,
    "message": "OTP has been resent",
    "data": {
        "expires_in": 600,
        "resends_remaining": 0
    }
}
```

**Error Response (429):**
```json
{
    "status": 429,
    "message": "Maximum OTP resend limit reached",
    "error_code": "AUTH_OTP_RESEND_LIMIT"
}
```

---

#### POST `/api/v1/auth/refresh-token`

**Request:**
```json
{
    "refresh_token": "eyJhbGciOiJIUzI1NiIs..."  // Required
}
```

**Success Response (200):**
```json
{
    "status": 200,
    "message": "Token refreshed successfully",
    "data": {
        "access_token": "eyJhbGciOiJIUzI1NiIs...",
        "refresh_token": "eyJhbGciOiJIUzI1NiIs...",
        "token_type": "Bearer",
        "expires_in": 900
    }
}
```

**Error Response (401):**
```json
{
    "status": 401,
    "message": "Refresh token is invalid or expired",
    "error_code": "AUTH_INVALID_REFRESH_TOKEN"
}
```

---

#### GET `/api/v1/auth/validate-token`

**Headers:**
```
Authorization: Bearer <access_token>
```

**Success Response (200):**
```json
{
    "status": 200,
    "message": "Token is valid",
    "data": {
        "valid": true,
        "user_uid": "550e8400-e29b-41d4-a716-446655440000",
        "expires_at": "2026-01-13T15:30:00Z"
    }
}
```

---

### User Endpoints

#### GET `/api/v1/users`

**Headers:**
```
Authorization: Bearer <access_token>
```

**Query Parameters:**
| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `page` | integer | No | 1 | Page number |
| `per_page` | integer | No | 15 | Items per page (max 100) |
| `search` | string | No | - | Search by username, email, or code |
| `status` | string | No | - | Filter by status: `active`, `inactive` |
| `is_blocked` | boolean | No | - | Filter by blocked status |
| `role_uid` | uuid | No | - | Filter by role |
| `sort_by` | string | No | `created_at` | Sort field: `username`, `email`, `code`, `created_at` |
| `sort_order` | string | No | `desc` | Sort direction: `asc`, `desc` |
| `created_from` | date | No | - | Filter by creation date (from) |
| `created_to` | date | No | - | Filter by creation date (to) |

**Success Response (200):**
```json
{
    "status": 200,
    "message": "Users retrieved successfully",
    "data": [
        {
            "uid": "550e8400-e29b-41d4-a716-446655440000",
            "code": "USR-0001",
            "username": "johndoe",
            "email": "john@example.com",
            "email_verified_at": "2026-01-10T10:00:00Z",
            "is_blocked": false,
            "status": "active",
            "roles": [
                {
                    "uid": "660e8400-e29b-41d4-a716-446655440001",
                    "name": "user"
                }
            ],
            "created_at": "2026-01-10T08:00:00Z"
        }
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

---

#### GET `/api/v1/users/{uid}`

**Headers:**
```
Authorization: Bearer <access_token>
```

**Success Response (200):**
```json
{
    "status": 200,
    "message": "User retrieved successfully",
    "data": {
        "uid": "550e8400-e29b-41d4-a716-446655440000",
        "code": "USR-0001",
        "username": "johndoe",
        "email": "john@example.com",
        "email_verified_at": "2026-01-10T10:00:00Z",
        "is_blocked": false,
        "blocked_at": null,
        "blocked_reason": null,
        "locked_until": null,
        "status": "active",
        "roles": [
            {
                "uid": "660e8400-e29b-41d4-a716-446655440001",
                "name": "user",
                "description": "Standard user role"
            }
        ],
        "permission_overrides": [
            {
                "uid": "880e8400-e29b-41d4-a716-446655440003",
                "module": {
                    "uid": "990e8400-e29b-41d4-a716-446655440004",
                    "name": "reports",
                    "service_name": "analytics"
                },
                "permission_type": "grant",
                "can_create": false,
                "can_read": true,
                "can_update": false,
                "can_delete": false,
                "expires_at": "2026-02-10T00:00:00Z",
                "reason": "Temporary access for Q1 reporting"
            }
        ],
        "created_at": "2026-01-10T08:00:00Z",
        "created_by": "550e8400-e29b-41d4-a716-446655440099",
        "updated_at": "2026-01-12T14:30:00Z",
        "updated_by": "550e8400-e29b-41d4-a716-446655440099"
    }
}
```

**Error Response (404):**
```json
{
    "status": 404,
    "message": "User not found",
    "error_code": "USER_NOT_FOUND"
}
```

---

#### POST `/api/v1/users`

**Headers:**
```
Authorization: Bearer <access_token>
```

**Request:**
```json
{
    "username": "janedoe",           // Required - min 3, max 100 chars
    "email": "jane@example.com",     // Required - valid email format
    "password": "SecurePass123!",    // Required - min 8 chars, complexity rules
    "role_uids": [                   // Required - at least one role
        "660e8400-e29b-41d4-a716-446655440001"
    ],
    "status": "active"               // Optional - default: active
}
```

**Success Response (201):**
```json
{
    "status": 201,
    "message": "User created successfully",
    "data": {
        "uid": "550e8400-e29b-41d4-a716-446655440010",
        "code": "USR-0002",
        "username": "janedoe",
        "email": "jane@example.com",
        "email_verified_at": null,
        "is_blocked": false,
        "status": "active",
        "roles": [
            {
                "uid": "660e8400-e29b-41d4-a716-446655440001",
                "name": "user"
            }
        ],
        "created_at": "2026-01-13T10:00:00Z"
    }
}
```

---

#### PUT `/api/v1/users/{uid}`

**Headers:**
```
Authorization: Bearer <access_token>
```

**Request:**
```json
{
    "username": "janedoe_updated",   // Optional
    "email": "jane.new@example.com", // Optional
    "role_uids": [                   // Optional
        "660e8400-e29b-41d4-a716-446655440001",
        "660e8400-e29b-41d4-a716-446655440002"
    ],
    "status": "active"               // Optional
}
```

**Success Response (200):**
```json
{
    "status": 200,
    "message": "User updated successfully",
    "data": {
        "uid": "550e8400-e29b-41d4-a716-446655440010",
        "code": "USR-0002",
        "username": "janedoe_updated",
        "email": "jane.new@example.com",
        "status": "active",
        "roles": [
            {
                "uid": "660e8400-e29b-41d4-a716-446655440001",
                "name": "user"
            },
            {
                "uid": "660e8400-e29b-41d4-a716-446655440002",
                "name": "manager"
            }
        ],
        "updated_at": "2026-01-13T11:00:00Z"
    }
}
```

---

#### DELETE `/api/v1/users/{uid}`

**Headers:**
```
Authorization: Bearer <access_token>
```

**Success Response (200):**
```json
{
    "status": 200,
    "message": "User deleted successfully"
}
```

**Error Response (400):**
```json
{
    "status": 400,
    "message": "Cannot delete user with active sessions. Please revoke all sessions first.",
    "error_code": "USER_HAS_ACTIVE_SESSIONS"
}
```

---

#### POST `/api/v1/users/{uid}/block`

**Headers:**
```
Authorization: Bearer <access_token>
```

**Request:**
```json
{
    "reason": "Violation of terms of service"  // Optional
}
```

**Success Response (200):**
```json
{
    "status": 200,
    "message": "User blocked successfully",
    "data": {
        "uid": "550e8400-e29b-41d4-a716-446655440010",
        "is_blocked": true,
        "blocked_at": "2026-01-13T12:00:00Z",
        "blocked_reason": "Violation of terms of service"
    }
}
```

---

#### POST `/api/v1/users/{uid}/unblock`

**Headers:**
```
Authorization: Bearer <access_token>
```

**Success Response (200):**
```json
{
    "status": 200,
    "message": "User unblocked successfully",
    "data": {
        "uid": "550e8400-e29b-41d4-a716-446655440010",
        "is_blocked": false
    }
}
```

---

#### POST `/api/v1/users/{uid}/unlock`

**Headers:**
```
Authorization: Bearer <access_token>
```

**Success Response (200):**
```json
{
    "status": 200,
    "message": "User unlocked successfully",
    "data": {
        "uid": "550e8400-e29b-41d4-a716-446655440010",
        "locked_until": null
    }
}
```

---

#### GET `/api/v1/users/{uid}/sessions`

**Headers:**
```
Authorization: Bearer <access_token>
```

**Query Parameters:**
| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `page` | integer | No | 1 | Page number |
| `per_page` | integer | No | 15 | Items per page |
| `is_active` | boolean | No | - | Filter active/revoked sessions |

**Success Response (200):**
```json
{
    "status": 200,
    "message": "Sessions retrieved successfully",
    "data": [
        {
            "uid": "aa0e8400-e29b-41d4-a716-446655440020",
            "ip_address": "192.168.1.100",
            "device_name": "Chrome on Windows",
            "user_agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64)...",
            "is_trusted": true,
            "is_current": true,
            "last_activity": "2026-01-13T10:30:00Z",
            "expires_at": "2026-01-20T10:00:00Z",
            "created_at": "2026-01-13T10:00:00Z"
        }
    ],
    "meta": {
        "current_page": 1,
        "per_page": 15,
        "total": 3,
        "total_pages": 1,
        "has_more": false
    }
}
```

---

#### DELETE `/api/v1/users/{uid}/sessions`

**Headers:**
```
Authorization: Bearer <access_token>
```

**Request:**
```json
{
    "except_current": true           // Optional - default: false, keep current session
}
```

**Success Response (200):**
```json
{
    "status": 200,
    "message": "All sessions revoked successfully",
    "data": {
        "revoked_count": 5
    }
}
```

---

#### POST `/api/v1/users/{uid}/resend-verification`

**Headers:**
```
Authorization: Bearer <access_token>
```

**Success Response (200):**
```json
{
    "status": 200,
    "message": "Verification email sent successfully"
}
```

**Error Response (400):**
```json
{
    "status": 400,
    "message": "User email is already verified",
    "error_code": "USER_EMAIL_ALREADY_VERIFIED"
}
```

---

### Role Endpoints

#### GET `/api/v1/roles`

**Headers:**
```
Authorization: Bearer <access_token>
```

**Query Parameters:**
| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `page` | integer | No | 1 | Page number |
| `per_page` | integer | No | 15 | Items per page |
| `search` | string | No | - | Search by name |
| `status` | string | No | - | Filter by status |
| `is_system` | boolean | No | - | Filter system roles |
| `sort_by` | string | No | `name` | Sort field: `name`, `created_at` |
| `sort_order` | string | No | `asc` | Sort direction |

**Success Response (200):**
```json
{
    "status": 200,
    "message": "Roles retrieved successfully",
    "data": [
        {
            "uid": "660e8400-e29b-41d4-a716-446655440001",
            "name": "admin",
            "description": "Administrator with full access",
            "is_system": true,
            "status": "active",
            "user_count": 5,
            "created_at": "2026-01-01T00:00:00Z"
        },
        {
            "uid": "660e8400-e29b-41d4-a716-446655440002",
            "name": "user",
            "description": "Standard user role",
            "is_system": true,
            "status": "active",
            "user_count": 150,
            "created_at": "2026-01-01T00:00:00Z"
        }
    ],
    "meta": {
        "current_page": 1,
        "per_page": 15,
        "total": 2,
        "total_pages": 1,
        "has_more": false
    }
}
```

---

#### GET `/api/v1/roles/{uid}`

**Headers:**
```
Authorization: Bearer <access_token>
```

**Success Response (200):**
```json
{
    "status": 200,
    "message": "Role retrieved successfully",
    "data": {
        "uid": "660e8400-e29b-41d4-a716-446655440001",
        "name": "admin",
        "description": "Administrator with full access",
        "is_system": true,
        "status": "active",
        "permissions": [
            {
                "module_uid": "990e8400-e29b-41d4-a716-446655440004",
                "module_name": "users",
                "service_name": "auth",
                "can_create": true,
                "can_read": true,
                "can_update": true,
                "can_delete": true
            }
        ],
        "user_count": 5,
        "created_at": "2026-01-01T00:00:00Z",
        "created_by": null,
        "updated_at": null,
        "updated_by": null
    }
}
```

---

#### POST `/api/v1/roles`

**Headers:**
```
Authorization: Bearer <access_token>
```

**Request:**
```json
{
    "name": "manager",               // Required - unique, max 100 chars
    "description": "Manager role with limited admin access",  // Optional
    "status": "active"               // Optional - default: active
}
```

**Success Response (201):**
```json
{
    "status": 201,
    "message": "Role created successfully",
    "data": {
        "uid": "660e8400-e29b-41d4-a716-446655440003",
        "name": "manager",
        "description": "Manager role with limited admin access",
        "is_system": false,
        "status": "active",
        "created_at": "2026-01-13T10:00:00Z"
    }
}
```

---

#### PUT `/api/v1/roles/{uid}`

**Headers:**
```
Authorization: Bearer <access_token>
```

**Request:**
```json
{
    "name": "senior_manager",        // Optional
    "description": "Updated description",  // Optional
    "status": "active"               // Optional
}
```

**Success Response (200):**
```json
{
    "status": 200,
    "message": "Role updated successfully",
    "data": {
        "uid": "660e8400-e29b-41d4-a716-446655440003",
        "name": "senior_manager",
        "description": "Updated description",
        "is_system": false,
        "status": "active",
        "updated_at": "2026-01-13T11:00:00Z"
    }
}
```

**Error Response (400):**
```json
{
    "status": 400,
    "message": "Cannot modify system role name",
    "error_code": "ROLE_SYSTEM_PROTECTED"
}
```

---

#### DELETE `/api/v1/roles/{uid}`

**Headers:**
```
Authorization: Bearer <access_token>
```

**Success Response (200):**
```json
{
    "status": 200,
    "message": "Role deleted successfully"
}
```

**Error Response (400 - Has Users):**
```json
{
    "status": 400,
    "message": "Cannot delete role that has assigned users",
    "error_code": "ROLE_HAS_USERS",
    "data": {
        "user_count": 15
    }
}
```

**Error Response (400 - System Role):**
```json
{
    "status": 400,
    "message": "Cannot delete system role",
    "error_code": "ROLE_SYSTEM_PROTECTED"
}
```

---

#### PUT `/api/v1/roles/{uid}/permissions`

**Headers:**
```
Authorization: Bearer <access_token>
```

**Request:**
```json
{
    "permissions": [
        {
            "module_uid": "990e8400-e29b-41d4-a716-446655440004",
            "can_create": true,
            "can_read": true,
            "can_update": true,
            "can_delete": false
        },
        {
            "module_uid": "990e8400-e29b-41d4-a716-446655440005",
            "can_create": false,
            "can_read": true,
            "can_update": false,
            "can_delete": false
        }
    ]
}
```

**Success Response (200):**
```json
{
    "status": 200,
    "message": "Role permissions updated successfully",
    "data": {
        "uid": "660e8400-e29b-41d4-a716-446655440003",
        "name": "manager",
        "permissions": [
            {
                "module_uid": "990e8400-e29b-41d4-a716-446655440004",
                "module_name": "users",
                "service_name": "auth",
                "can_create": true,
                "can_read": true,
                "can_update": true,
                "can_delete": false
            }
        ]
    }
}
```

---

### Service Endpoints

#### GET `/api/v1/services`

**Headers:**
```
Authorization: Bearer <access_token>
```

**Query Parameters:**
| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `page` | integer | No | 1 | Page number |
| `per_page` | integer | No | 15 | Items per page |
| `search` | string | No | - | Search by name or code |
| `status` | string | No | - | Filter by status |
| `sort_by` | string | No | `name` | Sort field |
| `sort_order` | string | No | `asc` | Sort direction |

**Success Response (200):**
```json
{
    "status": 200,
    "message": "Services retrieved successfully",
    "data": [
        {
            "uid": "bb0e8400-e29b-41d4-a716-446655440030",
            "name": "Authentication Service",
            "code": "auth",
            "description": "Handles user authentication and authorization",
            "base_url": "https://auth.example.com",
            "status": "active",
            "module_count": 5,
            "created_at": "2026-01-01T00:00:00Z"
        }
    ],
    "meta": {
        "current_page": 1,
        "per_page": 15,
        "total": 1,
        "total_pages": 1,
        "has_more": false
    }
}
```

---

#### GET `/api/v1/services/{uid}`

**Headers:**
```
Authorization: Bearer <access_token>
```

**Success Response (200):**
```json
{
    "status": 200,
    "message": "Service retrieved successfully",
    "data": {
        "uid": "bb0e8400-e29b-41d4-a716-446655440030",
        "name": "Authentication Service",
        "code": "auth",
        "description": "Handles user authentication and authorization",
        "base_url": "https://auth.example.com",
        "status": "active",
        "modules": [
            {
                "uid": "990e8400-e29b-41d4-a716-446655440004",
                "name": "users",
                "code": "users",
                "description": "User management module"
            },
            {
                "uid": "990e8400-e29b-41d4-a716-446655440005",
                "name": "roles",
                "code": "roles",
                "description": "Role management module"
            }
        ],
        "created_at": "2026-01-01T00:00:00Z",
        "created_by": null,
        "updated_at": null,
        "updated_by": null
    }
}
```

---

#### POST `/api/v1/services`

**Headers:**
```
Authorization: Bearer <access_token>
```

**Request:**
```json
{
    "name": "Inventory Service",     // Required - unique, max 100 chars
    "code": "inventory",             // Required - unique, max 50 chars, lowercase
    "description": "Manages product inventory",  // Optional
    "base_url": "https://inventory.example.com", // Optional
    "status": "active"               // Optional - default: active
}
```

**Success Response (201):**
```json
{
    "status": 201,
    "message": "Service created successfully",
    "data": {
        "uid": "bb0e8400-e29b-41d4-a716-446655440031",
        "name": "Inventory Service",
        "code": "inventory",
        "description": "Manages product inventory",
        "base_url": "https://inventory.example.com",
        "status": "active",
        "created_at": "2026-01-13T10:00:00Z"
    }
}
```

---

#### PUT `/api/v1/services/{uid}`

**Headers:**
```
Authorization: Bearer <access_token>
```

**Request:**
```json
{
    "name": "Inventory Management Service",  // Optional
    "description": "Updated description",    // Optional
    "base_url": "https://inv.example.com",   // Optional
    "status": "active"                       // Optional
}
```

**Success Response (200):**
```json
{
    "status": 200,
    "message": "Service updated successfully",
    "data": {
        "uid": "bb0e8400-e29b-41d4-a716-446655440031",
        "name": "Inventory Management Service",
        "code": "inventory",
        "description": "Updated description",
        "base_url": "https://inv.example.com",
        "status": "active",
        "updated_at": "2026-01-13T11:00:00Z"
    }
}
```

---

#### DELETE `/api/v1/services/{uid}`

**Headers:**
```
Authorization: Bearer <access_token>
```

**Success Response (200):**
```json
{
    "status": 200,
    "message": "Service deleted successfully"
}
```

**Error Response (400):**
```json
{
    "status": 400,
    "message": "Cannot delete service that has modules",
    "error_code": "SERVICE_HAS_MODULES",
    "data": {
        "module_count": 5
    }
}
```

---

### Module Endpoints

#### GET `/api/v1/modules`

**Headers:**
```
Authorization: Bearer <access_token>
```

**Query Parameters:**
| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `page` | integer | No | 1 | Page number |
| `per_page` | integer | No | 15 | Items per page |
| `search` | string | No | - | Search by name or code |
| `service_uid` | uuid | No | - | Filter by service |
| `status` | string | No | - | Filter by status |
| `sort_by` | string | No | `name` | Sort field |
| `sort_order` | string | No | `asc` | Sort direction |

**Success Response (200):**
```json
{
    "status": 200,
    "message": "Modules retrieved successfully",
    "data": [
        {
            "uid": "990e8400-e29b-41d4-a716-446655440004",
            "name": "users",
            "code": "users",
            "description": "User management module",
            "service": {
                "uid": "bb0e8400-e29b-41d4-a716-446655440030",
                "name": "Authentication Service",
                "code": "auth"
            },
            "status": "active",
            "created_at": "2026-01-01T00:00:00Z"
        }
    ],
    "meta": {
        "current_page": 1,
        "per_page": 15,
        "total": 5,
        "total_pages": 1,
        "has_more": false
    }
}
```

---

#### GET `/api/v1/modules/{uid}`

**Headers:**
```
Authorization: Bearer <access_token>
```

**Success Response (200):**
```json
{
    "status": 200,
    "message": "Module retrieved successfully",
    "data": {
        "uid": "990e8400-e29b-41d4-a716-446655440004",
        "name": "users",
        "code": "users",
        "description": "User management module",
        "service": {
            "uid": "bb0e8400-e29b-41d4-a716-446655440030",
            "name": "Authentication Service",
            "code": "auth"
        },
        "status": "active",
        "created_at": "2026-01-01T00:00:00Z",
        "created_by": null,
        "updated_at": null,
        "updated_by": null
    }
}
```

---

#### POST `/api/v1/modules`

**Headers:**
```
Authorization: Bearer <access_token>
```

**Request:**
```json
{
    "service_uid": "bb0e8400-e29b-41d4-a716-446655440030",  // Required
    "name": "reports",               // Required - unique within service, max 100 chars
    "code": "reports",               // Required - unique within service, max 50 chars
    "description": "Report generation module",  // Optional
    "status": "active"               // Optional - default: active
}
```

**Success Response (201):**
```json
{
    "status": 201,
    "message": "Module created successfully",
    "data": {
        "uid": "990e8400-e29b-41d4-a716-446655440006",
        "name": "reports",
        "code": "reports",
        "description": "Report generation module",
        "service": {
            "uid": "bb0e8400-e29b-41d4-a716-446655440030",
            "name": "Authentication Service",
            "code": "auth"
        },
        "status": "active",
        "created_at": "2026-01-13T10:00:00Z"
    }
}
```

---

#### PUT `/api/v1/modules/{uid}`

**Headers:**
```
Authorization: Bearer <access_token>
```

**Request:**
```json
{
    "name": "analytics_reports",     // Optional
    "description": "Updated description",  // Optional
    "status": "active"               // Optional
}
```

**Success Response (200):**
```json
{
    "status": 200,
    "message": "Module updated successfully",
    "data": {
        "uid": "990e8400-e29b-41d4-a716-446655440006",
        "name": "analytics_reports",
        "code": "reports",
        "description": "Updated description",
        "status": "active",
        "updated_at": "2026-01-13T11:00:00Z"
    }
}
```

---

#### DELETE `/api/v1/modules/{uid}`

**Headers:**
```
Authorization: Bearer <access_token>
```

**Success Response (200):**
```json
{
    "status": 200,
    "message": "Module deleted successfully"
}
```

**Error Response (400):**
```json
{
    "status": 400,
    "message": "Cannot delete module that has role permissions assigned",
    "error_code": "MODULE_HAS_PERMISSIONS",
    "data": {
        "permission_count": 3
    }
}
```

---

### Permission Override Endpoints

#### POST `/api/v1/users/{uid}/permission-overrides`

**Headers:**
```
Authorization: Bearer <access_token>
```

**Request:**
```json
{
    "module_uid": "990e8400-e29b-41d4-a716-446655440004",  // Required
    "permission_type": "grant",      // Required - 'grant' or 'deny'
    "can_create": true,              // Optional - default: false
    "can_read": true,                // Optional - default: false
    "can_update": true,              // Optional - default: false
    "can_delete": false,             // Optional - default: false
    "expires_at": "2026-02-13T00:00:00Z",  // Optional - null for permanent
    "reason": "Temporary access for project X"  // Optional
}
```

**Success Response (201):**
```json
{
    "status": 201,
    "message": "Permission override created successfully",
    "data": {
        "uid": "880e8400-e29b-41d4-a716-446655440003",
        "user_uid": "550e8400-e29b-41d4-a716-446655440000",
        "module": {
            "uid": "990e8400-e29b-41d4-a716-446655440004",
            "name": "users",
            "service_name": "auth"
        },
        "permission_type": "grant",
        "can_create": true,
        "can_read": true,
        "can_update": true,
        "can_delete": false,
        "expires_at": "2026-02-13T00:00:00Z",
        "reason": "Temporary access for project X",
        "created_at": "2026-01-13T10:00:00Z"
    }
}
```

---

#### GET `/api/v1/users/{uid}/permission-overrides`

**Headers:**
```
Authorization: Bearer <access_token>
```

**Query Parameters:**
| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `page` | integer | No | 1 | Page number |
| `per_page` | integer | No | 15 | Items per page |
| `permission_type` | string | No | - | Filter by type: `grant`, `deny` |
| `module_uid` | uuid | No | - | Filter by module |
| `include_expired` | boolean | No | false | Include expired overrides |

**Success Response (200):**
```json
{
    "status": 200,
    "message": "Permission overrides retrieved successfully",
    "data": [
        {
            "uid": "880e8400-e29b-41d4-a716-446655440003",
            "module": {
                "uid": "990e8400-e29b-41d4-a716-446655440004",
                "name": "users",
                "service_name": "auth"
            },
            "permission_type": "grant",
            "can_create": true,
            "can_read": true,
            "can_update": true,
            "can_delete": false,
            "expires_at": "2026-02-13T00:00:00Z",
            "is_expired": false,
            "reason": "Temporary access for project X",
            "created_at": "2026-01-13T10:00:00Z"
        }
    ],
    "meta": {
        "current_page": 1,
        "per_page": 15,
        "total": 1,
        "total_pages": 1,
        "has_more": false
    }
}
```

---

#### DELETE `/api/v1/users/{uid}/permission-overrides/{override_uid}`

**Headers:**
```
Authorization: Bearer <access_token>
```

**Success Response (200):**
```json
{
    "status": 200,
    "message": "Permission override removed successfully"
}
```

---

#### GET `/api/v1/permissions/check`

**Headers:**
```
Authorization: Bearer <access_token>
```

**Query Parameters:**
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `user_uid` | uuid | Yes | User to check |
| `service_code` | string | Yes | Service code (e.g., `auth`) |
| `module_code` | string | Yes | Module code (e.g., `users`) |
| `action` | string | Yes | Action: `create`, `read`, `update`, `delete` |

**Success Response (200):**
```json
{
    "status": 200,
    "message": "Permission check completed",
    "data": {
        "has_permission": true,
        "source": "role",
        "role_name": "admin"
    }
}
```

**Success Response (200 - Via Override):**
```json
{
    "status": 200,
    "message": "Permission check completed",
    "data": {
        "has_permission": true,
        "source": "override",
        "override_type": "grant",
        "expires_at": "2026-02-13T00:00:00Z"
    }
}
```

**Success Response (200 - Denied):**
```json
{
    "status": 200,
    "message": "Permission check completed",
    "data": {
        "has_permission": false,
        "source": "none"
    }
}
```

---

## Environment Configuration

### `.env` Variables

```bash
#--------------------------------------------------------------
# Application
#--------------------------------------------------------------
APP_NAME="Auth Service"
APP_ENV=local
APP_KEY=base64:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
APP_DEBUG=true
APP_URL=http://localhost:8000
APP_TIMEZONE=UTC

#--------------------------------------------------------------
# Database
#--------------------------------------------------------------
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=auth_service
DB_USERNAME=root
DB_PASSWORD=secret
DB_CHARSET=utf8mb4
DB_COLLATION=utf8mb4_unicode_ci

#--------------------------------------------------------------
# Redis
#--------------------------------------------------------------
REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379
REDIS_DB=0
REDIS_CACHE_DB=1

#--------------------------------------------------------------
# JWT Configuration
#--------------------------------------------------------------
JWT_SECRET=your-256-bit-secret-key-here
JWT_ACCESS_TOKEN_TTL=15              # Access token TTL in minutes
JWT_REFRESH_TOKEN_TTL=10080          # Refresh token TTL in minutes (7 days)
JWT_ALGORITHM=HS256

#--------------------------------------------------------------
# Authentication Settings
#--------------------------------------------------------------
AUTH_MAX_LOGIN_ATTEMPTS=3            # Max failed login attempts
AUTH_LOCKOUT_DURATION=60             # Lockout duration in minutes
AUTH_OTP_TTL=10                      # OTP validity in minutes
AUTH_OTP_MAX_RESENDS=1               # Max OTP resends per session
AUTH_OTP_LENGTH=6                    # OTP code length
AUTH_REQUIRE_EMAIL_VERIFICATION=true # Require email verification to login

#--------------------------------------------------------------
# Password Policy
#--------------------------------------------------------------
PASSWORD_MIN_LENGTH=8
PASSWORD_REQUIRE_UPPERCASE=true
PASSWORD_REQUIRE_LOWERCASE=true
PASSWORD_REQUIRE_NUMBER=true
PASSWORD_REQUIRE_SPECIAL=true
PASSWORD_BCRYPT_ROUNDS=12

#--------------------------------------------------------------
# Password Reset
#--------------------------------------------------------------
PASSWORD_RESET_TOKEN_TTL=60          # Token validity in minutes (1 hour)

#--------------------------------------------------------------
# Email Verification
#--------------------------------------------------------------
EMAIL_VERIFICATION_TOKEN_TTL=1440    # Token validity in minutes (24 hours)

#--------------------------------------------------------------
# Session Settings
#--------------------------------------------------------------
SESSION_DRIVER=redis
SESSION_LIFETIME=120
SESSION_ENCRYPT=false

#--------------------------------------------------------------
# Mail Configuration
#--------------------------------------------------------------
MAIL_MAILER=smtp
MAIL_HOST=smtp.mailgun.org
MAIL_PORT=587
MAIL_USERNAME=null
MAIL_PASSWORD=null
MAIL_ENCRYPTION=tls
MAIL_FROM_ADDRESS="noreply@example.com"
MAIL_FROM_NAME="${APP_NAME}"

# Mailgun specific (if using Mailgun)
MAILGUN_DOMAIN=mg.example.com
MAILGUN_SECRET=key-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

#--------------------------------------------------------------
# Queue Configuration
#--------------------------------------------------------------
QUEUE_CONNECTION=redis

#--------------------------------------------------------------
# Rate Limiting
#--------------------------------------------------------------
RATE_LIMIT_PER_MINUTE=60             # API requests per minute per user
RATE_LIMIT_LOGIN_PER_MINUTE=5        # Login attempts per minute per IP

#--------------------------------------------------------------
# CORS Configuration
#--------------------------------------------------------------
CORS_ALLOWED_ORIGINS=http://localhost:3000,https://app.example.com
CORS_ALLOWED_METHODS=GET,POST,PUT,DELETE,OPTIONS
CORS_ALLOWED_HEADERS=Content-Type,Authorization,X-Requested-With

#--------------------------------------------------------------
# Logging
#--------------------------------------------------------------
LOG_CHANNEL=stack
LOG_DEPRECATIONS_CHANNEL=null
LOG_LEVEL=debug

#--------------------------------------------------------------
# Cache
#--------------------------------------------------------------
CACHE_DRIVER=redis
CACHE_PREFIX=auth_service

#--------------------------------------------------------------
# User Code Generation
#--------------------------------------------------------------
USER_CODE_PREFIX=USR
USER_CODE_PAD_LENGTH=4               # USR-0001, USR-0002, etc. (continues to 10000+)

#--------------------------------------------------------------
# Service-to-Service Authentication
#--------------------------------------------------------------
SERVICE_SECRET_TOKEN=Abc123!@#XyZ456  # 16 chars: letters, numbers, special chars
                                      # Must match across all microservices
```

---

### Configuration Constants

#### `app/Constants/StatusConstants.php`

```php
<?php

namespace App\Constants;

class StatusConstants
{
    // Entity Status
    public const ACTIVE = 'active';
    public const INACTIVE = 'inactive';

    // All statuses
    public const ALL_STATUSES = [
        self::ACTIVE,
        self::INACTIVE,
    ];

    // User Block Status
    public const BLOCKED = true;
    public const UNBLOCKED = false;

    // Session Status
    public const SESSION_ACTIVE = 'active';
    public const SESSION_REVOKED = 'revoked';
    public const SESSION_EXPIRED = 'expired';
}
```

#### `app/Constants/PermissionConstants.php`

```php
<?php

namespace App\Constants;

class PermissionConstants
{
    // Permission Actions
    public const ACTION_CREATE = 'create';
    public const ACTION_READ = 'read';
    public const ACTION_UPDATE = 'update';
    public const ACTION_DELETE = 'delete';

    public const ALL_ACTIONS = [
        self::ACTION_CREATE,
        self::ACTION_READ,
        self::ACTION_UPDATE,
        self::ACTION_DELETE,
    ];

    // Permission Override Types
    public const OVERRIDE_GRANT = 'grant';
    public const OVERRIDE_DENY = 'deny';

    public const ALL_OVERRIDE_TYPES = [
        self::OVERRIDE_GRANT,
        self::OVERRIDE_DENY,
    ];

    // System Roles
    public const ROLE_ADMIN = 'admin';
    public const ROLE_USER = 'user';

    public const SYSTEM_ROLES = [
        self::ROLE_ADMIN,
        self::ROLE_USER,
    ];
}
```

#### `app/Constants/ErrorCodeConstants.php`

```php
<?php

namespace App\Constants;

class ErrorCodeConstants
{
    // Authentication Errors (AUTH_*)
    public const AUTH_INVALID_CREDENTIALS = 'AUTH_INVALID_CREDENTIALS';
    public const AUTH_ACCOUNT_LOCKED = 'AUTH_ACCOUNT_LOCKED';
    public const AUTH_ACCOUNT_BLOCKED = 'AUTH_ACCOUNT_BLOCKED';
    public const AUTH_ACCOUNT_INACTIVE = 'AUTH_ACCOUNT_INACTIVE';
    public const AUTH_EMAIL_NOT_VERIFIED = 'AUTH_EMAIL_NOT_VERIFIED';
    public const AUTH_INVALID_TOKEN = 'AUTH_INVALID_TOKEN';
    public const AUTH_TOKEN_EXPIRED = 'AUTH_TOKEN_EXPIRED';
    public const AUTH_INVALID_REFRESH_TOKEN = 'AUTH_INVALID_REFRESH_TOKEN';
    public const AUTH_REFRESH_TOKEN_REVOKED = 'AUTH_REFRESH_TOKEN_REVOKED';
    public const AUTH_INVALID_OTP = 'AUTH_INVALID_OTP';
    public const AUTH_OTP_EXPIRED = 'AUTH_OTP_EXPIRED';
    public const AUTH_OTP_RESEND_LIMIT = 'AUTH_OTP_RESEND_LIMIT';
    public const AUTH_OTP_MAX_ATTEMPTS = 'AUTH_OTP_MAX_ATTEMPTS';
    public const AUTH_RESET_TOKEN_EXPIRED = 'AUTH_RESET_TOKEN_EXPIRED';
    public const AUTH_RESET_TOKEN_INVALID = 'AUTH_RESET_TOKEN_INVALID';
    public const AUTH_RESET_TOKEN_USED = 'AUTH_RESET_TOKEN_USED';
    public const AUTH_EMAIL_VERIFY_INVALID = 'AUTH_EMAIL_VERIFY_INVALID';
    public const AUTH_EMAIL_VERIFY_EXPIRED = 'AUTH_EMAIL_VERIFY_EXPIRED';
    public const AUTH_INVALID_CURRENT_PASSWORD = 'AUTH_INVALID_CURRENT_PASSWORD';
    public const AUTH_REQUIRES_OTP = 'AUTH_REQUIRES_OTP';

    // User Errors (USER_*)
    public const USER_NOT_FOUND = 'USER_NOT_FOUND';
    public const USER_ALREADY_EXISTS = 'USER_ALREADY_EXISTS';
    public const USER_EMAIL_TAKEN = 'USER_EMAIL_TAKEN';
    public const USER_USERNAME_TAKEN = 'USER_USERNAME_TAKEN';
    public const USER_ALREADY_BLOCKED = 'USER_ALREADY_BLOCKED';
    public const USER_NOT_BLOCKED = 'USER_NOT_BLOCKED';
    public const USER_ALREADY_UNLOCKED = 'USER_ALREADY_UNLOCKED';
    public const USER_NOT_LOCKED = 'USER_NOT_LOCKED';
    public const USER_HAS_ACTIVE_SESSIONS = 'USER_HAS_ACTIVE_SESSIONS';
    public const USER_EMAIL_ALREADY_VERIFIED = 'USER_EMAIL_ALREADY_VERIFIED';
    public const USER_CANNOT_DELETE_SELF = 'USER_CANNOT_DELETE_SELF';
    public const USER_CANNOT_BLOCK_SELF = 'USER_CANNOT_BLOCK_SELF';
    public const USER_MUST_HAVE_ROLE = 'USER_MUST_HAVE_ROLE';
    public const CANNOT_REMOVE_LAST_ADMIN = 'CANNOT_REMOVE_LAST_ADMIN';
    public const CANNOT_DELETE_LAST_ADMIN = 'CANNOT_DELETE_LAST_ADMIN';
    public const CANNOT_DEACTIVATE_LAST_ADMIN = 'CANNOT_DEACTIVATE_LAST_ADMIN';

    // Role Errors (ROLE_*)
    public const ROLE_NOT_FOUND = 'ROLE_NOT_FOUND';
    public const ROLE_ALREADY_EXISTS = 'ROLE_ALREADY_EXISTS';
    public const ROLE_NAME_TAKEN = 'ROLE_NAME_TAKEN';
    public const ROLE_HAS_USERS = 'ROLE_HAS_USERS';
    public const ROLE_SYSTEM_PROTECTED = 'ROLE_SYSTEM_PROTECTED';
    public const ROLE_CANNOT_DELETE_SYSTEM = 'ROLE_CANNOT_DELETE_SYSTEM';

    // Service Errors (SERVICE_*)
    public const SERVICE_NOT_FOUND = 'SERVICE_NOT_FOUND';
    public const SERVICE_ALREADY_EXISTS = 'SERVICE_ALREADY_EXISTS';
    public const SERVICE_NAME_TAKEN = 'SERVICE_NAME_TAKEN';
    public const SERVICE_CODE_TAKEN = 'SERVICE_CODE_TAKEN';
    public const SERVICE_HAS_MODULES = 'SERVICE_HAS_MODULES';

    // Module Errors (MODULE_*)
    public const MODULE_NOT_FOUND = 'MODULE_NOT_FOUND';
    public const MODULE_ALREADY_EXISTS = 'MODULE_ALREADY_EXISTS';
    public const MODULE_NAME_TAKEN = 'MODULE_NAME_TAKEN';
    public const MODULE_CODE_TAKEN = 'MODULE_CODE_TAKEN';
    public const MODULE_HAS_PERMISSIONS = 'MODULE_HAS_PERMISSIONS';

    // Permission Errors (PERMISSION_*)
    public const PERMISSION_NOT_FOUND = 'PERMISSION_NOT_FOUND';
    public const PERMISSION_DENIED = 'PERMISSION_DENIED';
    public const PERMISSION_OVERRIDE_NOT_FOUND = 'PERMISSION_OVERRIDE_NOT_FOUND';
    public const PERMISSION_OVERRIDE_EXISTS = 'PERMISSION_OVERRIDE_EXISTS';

    // Session Errors (SESSION_*)
    public const SESSION_NOT_FOUND = 'SESSION_NOT_FOUND';
    public const SESSION_EXPIRED = 'SESSION_EXPIRED';
    public const SESSION_REVOKED = 'SESSION_REVOKED';

    // Validation Errors (VALIDATION_*)
    public const VALIDATION_ERROR = 'VALIDATION_ERROR';
    public const VALIDATION_INVALID_UUID = 'VALIDATION_INVALID_UUID';
    public const VALIDATION_INVALID_EMAIL = 'VALIDATION_INVALID_EMAIL';
    public const VALIDATION_PASSWORD_WEAK = 'VALIDATION_PASSWORD_WEAK';

    // Rate Limit Errors (RATE_*)
    public const RATE_LIMIT_EXCEEDED = 'RATE_LIMIT_EXCEEDED';
    public const RATE_LIMIT_LOGIN_EXCEEDED = 'RATE_LIMIT_LOGIN_EXCEEDED';

    // Service-to-Service Errors (SERVICE_AUTH_*)
    public const INVALID_SERVICE_TOKEN = 'INVALID_SERVICE_TOKEN';
    public const MISSING_SERVICE_TOKEN = 'MISSING_SERVICE_TOKEN';

    // General Errors (GENERAL_*)
    public const GENERAL_SERVER_ERROR = 'GENERAL_SERVER_ERROR';
    public const GENERAL_NOT_FOUND = 'GENERAL_NOT_FOUND';
    public const GENERAL_UNAUTHORIZED = 'GENERAL_UNAUTHORIZED';
    public const GENERAL_FORBIDDEN = 'GENERAL_FORBIDDEN';
    public const GENERAL_BAD_REQUEST = 'GENERAL_BAD_REQUEST';
    public const GENERAL_CONFLICT = 'GENERAL_CONFLICT';
    public const GENERAL_UNPROCESSABLE = 'GENERAL_UNPROCESSABLE';
}
```

#### `app/Constants/OtpConstants.php`

```php
<?php

namespace App\Constants;

class OtpConstants
{
    // OTP Types
    public const TYPE_LOGIN = 'login';
    public const TYPE_EMAIL_VERIFY = 'email_verify';
    public const TYPE_PASSWORD_RESET = 'password_reset';

    public const ALL_TYPES = [
        self::TYPE_LOGIN,
        self::TYPE_EMAIL_VERIFY,
        self::TYPE_PASSWORD_RESET,
    ];

    // OTP Configuration
    public const MAX_ATTEMPTS = 3;
    public const MAX_RESENDS = 1;
}
```

#### `app/Constants/AuditConstants.php`

```php
<?php

namespace App\Constants;

class AuditConstants
{
    // Audit Actions
    public const ACTION_CREATE = 'create';
    public const ACTION_UPDATE = 'update';
    public const ACTION_DELETE = 'delete';
    public const ACTION_LOGIN = 'login';
    public const ACTION_LOGOUT = 'logout';
    public const ACTION_LOGIN_FAILED = 'login_failed';
    public const ACTION_PASSWORD_RESET = 'password_reset';
    public const ACTION_PASSWORD_CHANGE = 'password_change';
    public const ACTION_EMAIL_VERIFY = 'email_verify';
    public const ACTION_OTP_VERIFY = 'otp_verify';
    public const ACTION_BLOCK = 'block';
    public const ACTION_UNBLOCK = 'unblock';
    public const ACTION_LOCK = 'lock';
    public const ACTION_UNLOCK = 'unlock';
    public const ACTION_SESSION_REVOKE = 'session_revoke';
    public const ACTION_PERMISSION_GRANT = 'permission_grant';
    public const ACTION_PERMISSION_REVOKE = 'permission_revoke';

    // Entity Types
    public const ENTITY_USER = 'user';
    public const ENTITY_ROLE = 'role';
    public const ENTITY_SERVICE = 'service';
    public const ENTITY_MODULE = 'module';
    public const ENTITY_SESSION = 'session';
    public const ENTITY_PERMISSION = 'permission';
}
```

---

## Error Codes

### Complete Error Code Reference

| Error Code | HTTP Status | Description |
|------------|-------------|-------------|
| **Authentication** |||
| `AUTH_INVALID_CREDENTIALS` | 401 | Invalid username/email or password |
| `AUTH_ACCOUNT_LOCKED` | 423 | Account locked due to failed login attempts |
| `AUTH_ACCOUNT_BLOCKED` | 403 | Account has been blocked by admin |
| `AUTH_ACCOUNT_INACTIVE` | 403 | Account is inactive |
| `AUTH_EMAIL_NOT_VERIFIED` | 403 | Email address not verified |
| `AUTH_INVALID_TOKEN` | 401 | JWT token is invalid |
| `AUTH_TOKEN_EXPIRED` | 401 | JWT token has expired |
| `AUTH_INVALID_REFRESH_TOKEN` | 401 | Refresh token is invalid |
| `AUTH_REFRESH_TOKEN_REVOKED` | 401 | Refresh token has been revoked |
| `AUTH_INVALID_OTP` | 400 | OTP code is incorrect |
| `AUTH_OTP_EXPIRED` | 400 | OTP code has expired |
| `AUTH_OTP_RESEND_LIMIT` | 429 | Maximum OTP resend limit reached |
| `AUTH_OTP_MAX_ATTEMPTS` | 429 | Maximum OTP verification attempts exceeded |
| `AUTH_RESET_TOKEN_EXPIRED` | 400 | Password reset token has expired |
| `AUTH_RESET_TOKEN_INVALID` | 400 | Password reset token is invalid |
| `AUTH_RESET_TOKEN_USED` | 400 | Password reset token already used |
| `AUTH_EMAIL_VERIFY_INVALID` | 400 | Email verification token is invalid |
| `AUTH_EMAIL_VERIFY_EXPIRED` | 400 | Email verification token has expired |
| `AUTH_INVALID_CURRENT_PASSWORD` | 400 | Current password is incorrect |
| `AUTH_REQUIRES_OTP` | 200 | OTP verification required (new device) |
| **User** |||
| `USER_NOT_FOUND` | 404 | User does not exist |
| `USER_ALREADY_EXISTS` | 409 | User already exists |
| `USER_EMAIL_TAKEN` | 422 | Email address already in use |
| `USER_USERNAME_TAKEN` | 422 | Username already in use |
| `USER_ALREADY_BLOCKED` | 400 | User is already blocked |
| `USER_NOT_BLOCKED` | 400 | User is not blocked |
| `USER_ALREADY_UNLOCKED` | 400 | User is already unlocked |
| `USER_NOT_LOCKED` | 400 | User is not locked |
| `USER_HAS_ACTIVE_SESSIONS` | 400 | User has active sessions |
| `USER_EMAIL_ALREADY_VERIFIED` | 400 | Email is already verified |
| `USER_CANNOT_DELETE_SELF` | 400 | Cannot delete your own account |
| `USER_CANNOT_BLOCK_SELF` | 400 | Cannot block your own account |
| `USER_MUST_HAVE_ROLE` | 400 | User must have at least one role |
| `CANNOT_REMOVE_LAST_ADMIN` | 400 | Cannot remove admin role from the last admin |
| `CANNOT_DELETE_LAST_ADMIN` | 400 | Cannot delete the last admin user |
| `CANNOT_DEACTIVATE_LAST_ADMIN` | 400 | Cannot deactivate the last admin user |
| **Role** |||
| `ROLE_NOT_FOUND` | 404 | Role does not exist |
| `ROLE_ALREADY_EXISTS` | 409 | Role already exists |
| `ROLE_NAME_TAKEN` | 422 | Role name already in use |
| `ROLE_HAS_USERS` | 400 | Role has assigned users |
| `ROLE_SYSTEM_PROTECTED` | 400 | System role cannot be modified |
| `ROLE_CANNOT_DELETE_SYSTEM` | 400 | System role cannot be deleted |
| **Service** |||
| `SERVICE_NOT_FOUND` | 404 | Service does not exist |
| `SERVICE_ALREADY_EXISTS` | 409 | Service already exists |
| `SERVICE_NAME_TAKEN` | 422 | Service name already in use |
| `SERVICE_CODE_TAKEN` | 422 | Service code already in use |
| `SERVICE_HAS_MODULES` | 400 | Service has assigned modules |
| **Module** |||
| `MODULE_NOT_FOUND` | 404 | Module does not exist |
| `MODULE_ALREADY_EXISTS` | 409 | Module already exists |
| `MODULE_NAME_TAKEN` | 422 | Module name already in use (within service) |
| `MODULE_CODE_TAKEN` | 422 | Module code already in use (within service) |
| `MODULE_HAS_PERMISSIONS` | 400 | Module has assigned permissions |
| **Permission** |||
| `PERMISSION_NOT_FOUND` | 404 | Permission does not exist |
| `PERMISSION_DENIED` | 403 | User does not have required permission |
| `PERMISSION_OVERRIDE_NOT_FOUND` | 404 | Permission override does not exist |
| `PERMISSION_OVERRIDE_EXISTS` | 409 | Permission override already exists |
| **Session** |||
| `SESSION_NOT_FOUND` | 404 | Session does not exist |
| `SESSION_EXPIRED` | 401 | Session has expired |
| `SESSION_REVOKED` | 401 | Session has been revoked |
| **Validation** |||
| `VALIDATION_ERROR` | 422 | Input validation failed |
| `VALIDATION_INVALID_UUID` | 422 | Invalid UUID format |
| `VALIDATION_INVALID_EMAIL` | 422 | Invalid email format |
| `VALIDATION_PASSWORD_WEAK` | 422 | Password does not meet requirements |
| **Rate Limiting** |||
| `RATE_LIMIT_EXCEEDED` | 429 | API rate limit exceeded |
| `RATE_LIMIT_LOGIN_EXCEEDED` | 429 | Login rate limit exceeded |
| **Service-to-Service** |||
| `INVALID_SERVICE_TOKEN` | 401 | Service token is invalid |
| `MISSING_SERVICE_TOKEN` | 401 | Service token header is missing |
| **General** |||
| `GENERAL_SERVER_ERROR` | 500 | Internal server error |
| `GENERAL_NOT_FOUND` | 404 | Resource not found |
| `GENERAL_UNAUTHORIZED` | 401 | Authentication required |
| `GENERAL_FORBIDDEN` | 403 | Access forbidden |
| `GENERAL_BAD_REQUEST` | 400 | Bad request |
| `GENERAL_CONFLICT` | 409 | Resource conflict |
| `GENERAL_UNPROCESSABLE` | 422 | Unprocessable entity |

---

## Business Logic Clarifications

### Email Verification Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                   Email Verification Flow                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. User Registers                                               │
│     └── System generates verification token (24h expiry)         │
│     └── System sends verification email                          │
│                                                                  │
│  2. User Clicks Verification Link                                │
│     ├── Token Valid → Email verified, can login                  │
│     └── Token Expired/Invalid → Show error page                  │
│                                                                  │
│  3. Resend Verification (Admin Only via User Management)         │
│     └── POST /api/v1/users/{uid}/resend-verification             │
│     └── Generates new token, invalidates old one                 │
│                                                                  │
│  4. Login Attempt (if AUTH_REQUIRE_EMAIL_VERIFICATION=true)      │
│     ├── Email Verified → Proceed with login                      │
│     └── Email Not Verified → Return AUTH_EMAIL_NOT_VERIFIED      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Rules:**
- Verification token expires after 24 hours
- Only admins can resend verification emails via user management
- Users can use `/api/v1/auth/resend-verification` if they know their email
- Each new token invalidates the previous one
- Login blocked until verified (if configured)

---

### Password Reset Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                    Password Reset Flow                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. User Requests Reset                                          │
│     └── POST /api/v1/auth/forgot-password                        │
│     └── Always returns success (prevent email enumeration)       │
│     └── If email exists: generate token (1h expiry), send email  │
│                                                                  │
│  2. User Clicks Reset Link                                       │
│     └── POST /api/v1/auth/reset-password with token              │
│     ├── Token Valid → Password updated, token marked used        │
│     ├── Token Expired → AUTH_RESET_TOKEN_EXPIRED                 │
│     │   └── Message: "Please request a new password reset link"  │
│     └── Token Invalid/Used → AUTH_RESET_TOKEN_INVALID            │
│                                                                  │
│  3. No Attempt Limits                                             │
│     └── Users can request unlimited reset links                  │
│     └── Each new request generates new token                     │
│     └── Old tokens remain valid until expiry or use              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Rules:**
- Reset token expires after 1 hour
- No limit on password reset requests
- Token is single-use (marked as used after successful reset)
- All user sessions are revoked after password reset

---

### Session Management

```
┌─────────────────────────────────────────────────────────────────┐
│                   Session Management                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Session Limits: NONE (unlimited sessions per user)              │
│                                                                  │
│  Session Lifecycle:                                              │
│  1. Created on successful login (after OTP if required)          │
│  2. Contains: refresh_token, device_hash, IP, user_agent         │
│  3. Refresh token validity: 7 days                               │
│  4. Session updates last_activity on token refresh               │
│                                                                  │
│  Refresh Token Rotation:                                         │
│  - When refresh token is used, it is INVALIDATED immediately     │
│  - A new refresh token is issued with each refresh               │
│  - Old refresh tokens cannot be reused                           │
│                                                                  │
│  Session Termination:                                            │
│  - User logout → Current session revoked                         │
│  - Admin revokes all → All user sessions revoked                 │
│  - Password change → ALL sessions revoked (including current)    │
│  - Password reset → All sessions revoked                         │
│  - User blocked → All sessions revoked                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

### Device Trust & OTP Logic

```
┌─────────────────────────────────────────────────────────────────┐
│                   Device Trust & OTP Logic                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Device Identification:                                          │
│  - device_hash = hash(IP + User-Agent)                          │
│  - Used to identify returning devices                            │
│                                                                  │
│  OTP Requirement:                                                │
│  - OTP is required ONLY when BOTH conditions are true:          │
│    1. New device (device_hash not found in sessions)             │
│    2. New IP address                                             │
│  - If device_hash matches existing session → No OTP needed       │
│                                                                  │
│  Device Trust Persistence:                                       │
│  - Trust is tied to the session record                           │
│  - When session is created after OTP, is_trusted = true          │
│  - Trust persists as long as the session exists                  │
│  - Same device_hash on future login → Skip OTP                   │
│                                                                  │
│  Trust Reset Events:                                             │
│  - Password change → ALL sessions deleted (trust reset)          │
│  - Password reset → ALL sessions deleted (trust reset)           │
│  - User must OTP verify again on next login                      │
│                                                                  │
│  Flow Example:                                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ Login from device_hash=ABC123, IP=192.168.1.1              │ │
│  │ ├── Session with ABC123 exists? → Login directly           │ │
│  │ └── No session with ABC123?                                │ │
│  │     └── New IP? → Require OTP → Create trusted session     │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

### Admin-Created Users

```
┌─────────────────────────────────────────────────────────────────┐
│                   Admin-Created Users Flow                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  When admin creates a user via POST /api/v1/users:               │
│                                                                  │
│  1. Email is AUTO-VERIFIED                                       │
│     └── email_verified_at = NOW()                                │
│     └── No verification email sent                               │
│                                                                  │
│  2. Welcome Email with Temporary Password                        │
│     └── System generates temporary password                      │
│     └── Sends welcome email to user                              │
│     └── Email contains: username, temporary password             │
│     └── User should change password on first login               │
│                                                                  │
│  3. Role Assignment                                              │
│     └── Admin must assign at least 1 role                        │
│     └── role_uids[] is REQUIRED in request                       │
│                                                                  │
│  Note: This differs from self-registration where:                │
│  - Email verification is required                                │
│  - User sets their own password                                  │
│  - Default 'user' role may be auto-assigned                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

### Role Assignment Rules

```
┌─────────────────────────────────────────────────────────────────┐
│                   Role Assignment Rules                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Minimum Roles:                                                  │
│  - Every user MUST have at least 1 role assigned                 │
│  - Cannot remove last role from user                             │
│  - API validates: role_uids[] must have min 1 item               │
│                                                                  │
│  Last Admin Protection:                                          │
│  - System tracks admin role assignments                          │
│  - Cannot remove admin role if user is the LAST admin            │
│  - Cannot delete the last admin user                             │
│  - Cannot deactivate the last admin user                         │
│                                                                  │
│  Error Responses:                                                │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ Action                      │ Error Code                   │ │
│  ├────────────────────────────────────────────────────────────┤ │
│  │ Remove last role from user  │ USER_MUST_HAVE_ROLE          │ │
│  │ Remove admin from last admin│ CANNOT_REMOVE_LAST_ADMIN     │ │
│  │ Delete last admin user      │ CANNOT_DELETE_LAST_ADMIN     │ │
│  │ Deactivate last admin       │ CANNOT_DEACTIVATE_LAST_ADMIN │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

### Permission Override Handling

```
┌─────────────────────────────────────────────────────────────────┐
│                Permission Override Handling                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Expiry Behavior:                                                │
│  - expires_at field is OPTIONAL (can be NULL for permanent)      │
│  - NO scheduled cleanup job for expired overrides                │
│  - Expired overrides are IGNORED during permission checks        │
│  - Check logic: WHERE expires_at IS NULL OR expires_at > NOW()   │
│                                                                  │
│  Permission Check Order:                                         │
│  1. Check user_permission_overrides (non-expired only)           │
│     ├── GRANT override exists → ALLOW                            │
│     └── DENY override exists → DENY                              │
│  2. Check role_permissions for ALL user roles                    │
│     └── ANY role has permission → ALLOW                          │
│  3. Default → DENY                                               │
│                                                                  │
│  Manual Cleanup (Optional):                                      │
│  - Admin can manually delete expired overrides                   │
│  - Or run periodic cleanup via artisan command if desired        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

### Data Status vs Soft Delete

```
┌─────────────────────────────────────────────────────────────────┐
│              Status vs Soft Delete vs Archive                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  STATUS (active/inactive)                                        │
│  ├── Purpose: Temporary enable/disable without removal           │
│  ├── Can be reactivated: YES                                     │
│  ├── deleted_at: NULL                                            │
│  └── archived: FALSE                                             │
│                                                                  │
│  SOFT DELETE                                                     │
│  ├── Purpose: Permanent removal while keeping audit trail        │
│  ├── Can be restored: NO (manual DB only if needed)              │
│  ├── deleted_at: TIMESTAMP (when deleted)                        │
│  └── archived: TRUE                                              │
│                                                                  │
│  Example Operations:                                             │
│  ┌────────────────┬──────────┬────────────┬──────────┐          │
│  │ Action         │ status   │ deleted_at │ archived │          │
│  ├────────────────┼──────────┼────────────┼──────────┤          │
│  │ Create         │ active   │ NULL       │ false    │          │
│  │ Deactivate     │ inactive │ NULL       │ false    │          │
│  │ Reactivate     │ active   │ NULL       │ false    │          │
│  │ Delete         │ inactive │ TIMESTAMP  │ true     │          │
│  └────────────────┴──────────┴────────────┴──────────┘          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Rules:**
- Deactivated records can be reactivated
- Deleted records CANNOT be restored via API (manual DB operation only)
- Queries by default exclude deleted records (`deleted_at IS NULL`)
- Uniqueness constraints (username, email) only apply to non-deleted records

---

## Testing Strategy

### Testing Stack

| Tool | Purpose |
|------|---------|
| **PHPUnit** | Unit and Feature testing (Laravel default) |
| **Laravel Sanctum/JWT** | API authentication testing |
| **Faker** | Test data generation |
| **RefreshDatabase** | Database reset between tests |
| **Mockery** | Mocking external services |

### Test Structure

```
tests/
├── Unit/
│   ├── Services/
│   │   ├── AuthServiceTest.php
│   │   ├── UserServiceTest.php
│   │   ├── RoleServiceTest.php
│   │   ├── PermissionServiceTest.php
│   │   └── ...
│   ├── Utils/
│   │   ├── UuidHelperTest.php
│   │   ├── CodeGeneratorTest.php
│   │   └── PasswordValidatorTest.php
│   └── Repositories/
│       └── ...
├── Feature/
│   └── Api/
│       └── V1/
│           ├── Auth/
│           │   ├── LoginTest.php
│           │   ├── RegisterTest.php
│           │   ├── LogoutTest.php
│           │   ├── ForgotPasswordTest.php
│           │   ├── ResetPasswordTest.php
│           │   ├── ChangePasswordTest.php
│           │   ├── VerifyEmailTest.php
│           │   ├── OtpVerificationTest.php
│           │   ├── RefreshTokenTest.php
│           │   └── ValidateTokenTest.php
│           ├── User/
│           │   ├── ListUsersTest.php
│           │   ├── ShowUserTest.php
│           │   ├── CreateUserTest.php
│           │   ├── UpdateUserTest.php
│           │   ├── DeleteUserTest.php
│           │   ├── BlockUserTest.php
│           │   ├── UnblockUserTest.php
│           │   ├── UnlockUserTest.php
│           │   ├── UserSessionsTest.php
│           │   └── ResendVerificationTest.php
│           ├── Role/
│           │   ├── ListRolesTest.php
│           │   ├── ShowRoleTest.php
│           │   ├── CreateRoleTest.php
│           │   ├── UpdateRoleTest.php
│           │   ├── DeleteRoleTest.php
│           │   └── UpdateRolePermissionsTest.php
│           ├── Service/
│           │   ├── ListServicesTest.php
│           │   ├── ShowServiceTest.php
│           │   ├── CreateServiceTest.php
│           │   ├── UpdateServiceTest.php
│           │   └── DeleteServiceTest.php
│           ├── Module/
│           │   ├── ListModulesTest.php
│           │   ├── ShowModuleTest.php
│           │   ├── CreateModuleTest.php
│           │   ├── UpdateModuleTest.php
│           │   └── DeleteModuleTest.php
│           └── Permission/
│               ├── CreateOverrideTest.php
│               ├── ListOverridesTest.php
│               ├── DeleteOverrideTest.php
│               └── CheckPermissionTest.php
└── TestCase.php
```

### Test Cases (1 Positive + 1 Negative per API)

#### Authentication Tests

```php
// tests/Feature/Api/V1/Auth/LoginTest.php

class LoginTest extends TestCase
{
    use RefreshDatabase;

    /** @test */
    public function user_can_login_with_valid_credentials()
    {
        $user = User::factory()->create([
            'email' => 'test@example.com',
            'password' => Hash::make('SecurePass123!'),
            'email_verified_at' => now(),
        ]);

        $response = $this->postJson('/api/v1/auth/login', [
            'login' => 'test@example.com',
            'password' => 'SecurePass123!',
        ]);

        $response->assertStatus(200)
            ->assertJsonStructure([
                'status',
                'message',
                'data' => [
                    'user' => ['uid', 'username', 'email'],
                    'access_token',
                    'refresh_token',
                    'token_type',
                    'expires_in',
                ],
            ]);
    }

    /** @test */
    public function user_cannot_login_with_invalid_credentials()
    {
        $user = User::factory()->create([
            'email' => 'test@example.com',
            'password' => Hash::make('SecurePass123!'),
        ]);

        $response = $this->postJson('/api/v1/auth/login', [
            'login' => 'test@example.com',
            'password' => 'WrongPassword123!',
        ]);

        $response->assertStatus(401)
            ->assertJson([
                'status' => 401,
                'error_code' => 'AUTH_INVALID_CREDENTIALS',
            ]);
    }
}
```

```php
// tests/Feature/Api/V1/Auth/RegisterTest.php

class RegisterTest extends TestCase
{
    use RefreshDatabase;

    /** @test */
    public function user_can_register_with_valid_data()
    {
        $response = $this->postJson('/api/v1/auth/register', [
            'username' => 'newuser',
            'email' => 'newuser@example.com',
            'password' => 'SecurePass123!',
            'password_confirmation' => 'SecurePass123!',
        ]);

        $response->assertStatus(201)
            ->assertJsonStructure([
                'status',
                'message',
                'data' => [
                    'user' => ['uid', 'code', 'username', 'email'],
                ],
            ]);

        $this->assertDatabaseHas('users', [
            'username' => 'newuser',
            'email' => 'newuser@example.com',
        ]);
    }

    /** @test */
    public function user_cannot_register_with_existing_email()
    {
        User::factory()->create(['email' => 'existing@example.com']);

        $response = $this->postJson('/api/v1/auth/register', [
            'username' => 'newuser',
            'email' => 'existing@example.com',
            'password' => 'SecurePass123!',
            'password_confirmation' => 'SecurePass123!',
        ]);

        $response->assertStatus(422)
            ->assertJsonValidationErrors(['email']);
    }
}
```

```php
// tests/Feature/Api/V1/Auth/ForgotPasswordTest.php

class ForgotPasswordTest extends TestCase
{
    use RefreshDatabase;

    /** @test */
    public function user_can_request_password_reset()
    {
        $user = User::factory()->create(['email' => 'test@example.com']);

        Mail::fake();

        $response = $this->postJson('/api/v1/auth/forgot-password', [
            'email' => 'test@example.com',
        ]);

        $response->assertStatus(200)
            ->assertJson([
                'message' => 'If the email exists, a password reset link has been sent.',
            ]);

        Mail::assertSent(PasswordResetMail::class);
    }

    /** @test */
    public function forgot_password_returns_success_for_nonexistent_email()
    {
        // Security: prevent email enumeration
        $response = $this->postJson('/api/v1/auth/forgot-password', [
            'email' => 'nonexistent@example.com',
        ]);

        $response->assertStatus(200)
            ->assertJson([
                'message' => 'If the email exists, a password reset link has been sent.',
            ]);
    }
}
```

```php
// tests/Feature/Api/V1/Auth/ResetPasswordTest.php

class ResetPasswordTest extends TestCase
{
    use RefreshDatabase;

    /** @test */
    public function user_can_reset_password_with_valid_token()
    {
        $user = User::factory()->create();
        $token = Str::random(64);

        PasswordReset::create([
            'uid' => Str::uuid()->getBytes(),
            'user_uid' => $user->uid,
            'token' => $token,
            'expires_at' => now()->addHour(),
        ]);

        $response = $this->postJson('/api/v1/auth/reset-password', [
            'token' => $token,
            'password' => 'NewSecurePass456!',
            'password_confirmation' => 'NewSecurePass456!',
        ]);

        $response->assertStatus(200)
            ->assertJson([
                'message' => 'Password has been reset successfully',
            ]);
    }

    /** @test */
    public function user_cannot_reset_password_with_expired_token()
    {
        $user = User::factory()->create();
        $token = Str::random(64);

        PasswordReset::create([
            'uid' => Str::uuid()->getBytes(),
            'user_uid' => $user->uid,
            'token' => $token,
            'expires_at' => now()->subHour(), // Expired
        ]);

        $response = $this->postJson('/api/v1/auth/reset-password', [
            'token' => $token,
            'password' => 'NewSecurePass456!',
            'password_confirmation' => 'NewSecurePass456!',
        ]);

        $response->assertStatus(400)
            ->assertJson([
                'error_code' => 'AUTH_RESET_TOKEN_EXPIRED',
            ]);
    }
}
```

#### User Tests

```php
// tests/Feature/Api/V1/User/CreateUserTest.php

class CreateUserTest extends TestCase
{
    use RefreshDatabase;

    /** @test */
    public function admin_can_create_user()
    {
        $admin = User::factory()->admin()->create();
        $role = Role::factory()->create();

        $response = $this->actingAs($admin)
            ->postJson('/api/v1/users', [
                'username' => 'newuser',
                'email' => 'newuser@example.com',
                'password' => 'SecurePass123!',
                'role_uids' => [$role->uid_string],
            ]);

        $response->assertStatus(201)
            ->assertJsonStructure([
                'data' => ['uid', 'code', 'username', 'email'],
            ]);
    }

    /** @test */
    public function non_admin_cannot_create_user_without_permission()
    {
        $user = User::factory()->create();
        $role = Role::factory()->create();

        $response = $this->actingAs($user)
            ->postJson('/api/v1/users', [
                'username' => 'newuser',
                'email' => 'newuser@example.com',
                'password' => 'SecurePass123!',
                'role_uids' => [$role->uid_string],
            ]);

        $response->assertStatus(403)
            ->assertJson([
                'error_code' => 'PERMISSION_DENIED',
            ]);
    }
}
```

```php
// tests/Feature/Api/V1/User/DeleteUserTest.php

class DeleteUserTest extends TestCase
{
    use RefreshDatabase;

    /** @test */
    public function admin_can_delete_user()
    {
        $admin = User::factory()->admin()->create();
        $user = User::factory()->create();

        $response = $this->actingAs($admin)
            ->deleteJson("/api/v1/users/{$user->uid_string}");

        $response->assertStatus(200);

        $this->assertSoftDeleted('users', ['id' => $user->id]);
    }

    /** @test */
    public function admin_cannot_delete_self()
    {
        $admin = User::factory()->admin()->create();

        $response = $this->actingAs($admin)
            ->deleteJson("/api/v1/users/{$admin->uid_string}");

        $response->assertStatus(400)
            ->assertJson([
                'error_code' => 'USER_CANNOT_DELETE_SELF',
            ]);
    }
}
```

#### Role Tests

```php
// tests/Feature/Api/V1/Role/DeleteRoleTest.php

class DeleteRoleTest extends TestCase
{
    use RefreshDatabase;

    /** @test */
    public function admin_can_delete_role_without_users()
    {
        $admin = User::factory()->admin()->create();
        $role = Role::factory()->create(['is_system' => false]);

        $response = $this->actingAs($admin)
            ->deleteJson("/api/v1/roles/{$role->uid_string}");

        $response->assertStatus(200);
        $this->assertSoftDeleted('roles', ['id' => $role->id]);
    }

    /** @test */
    public function admin_cannot_delete_role_with_assigned_users()
    {
        $admin = User::factory()->admin()->create();
        $role = Role::factory()->create(['is_system' => false]);
        $user = User::factory()->create();
        $user->roles()->attach($role);

        $response = $this->actingAs($admin)
            ->deleteJson("/api/v1/roles/{$role->uid_string}");

        $response->assertStatus(400)
            ->assertJson([
                'error_code' => 'ROLE_HAS_USERS',
            ]);
    }
}
```

---

## Deployment & DevOps

### Database Seeders

#### `database/seeders/DatabaseSeeder.php`

```php
<?php

namespace Database\Seeders;

use Illuminate\Database\Seeder;

class DatabaseSeeder extends Seeder
{
    public function run(): void
    {
        $this->call([
            ServiceSeeder::class,
            ModuleSeeder::class,
            RoleSeeder::class,
            RolePermissionSeeder::class,
            AdminUserSeeder::class,
        ]);
    }
}
```

#### `database/seeders/ServiceSeeder.php`

```php
<?php

namespace Database\Seeders;

use App\Models\Service;
use App\Utils\UuidHelper;
use Illuminate\Database\Seeder;

class ServiceSeeder extends Seeder
{
    public function run(): void
    {
        $services = [
            [
                'uid' => UuidHelper::generateBinary(),
                'name' => 'Authentication Service',
                'code' => 'auth',
                'description' => 'Handles user authentication and authorization',
                'base_url' => config('app.url'),
                'status' => 'active',
            ],
        ];

        foreach ($services as $service) {
            Service::firstOrCreate(
                ['code' => $service['code']],
                $service
            );
        }
    }
}
```

#### `database/seeders/ModuleSeeder.php`

```php
<?php

namespace Database\Seeders;

use App\Models\Module;
use App\Models\Service;
use App\Utils\UuidHelper;
use Illuminate\Database\Seeder;

class ModuleSeeder extends Seeder
{
    public function run(): void
    {
        $authService = Service::where('code', 'auth')->first();

        $modules = [
            [
                'uid' => UuidHelper::generateBinary(),
                'service_uid' => $authService->uid,
                'name' => 'Users',
                'code' => 'users',
                'description' => 'User management module',
                'status' => 'active',
            ],
            [
                'uid' => UuidHelper::generateBinary(),
                'service_uid' => $authService->uid,
                'name' => 'Roles',
                'code' => 'roles',
                'description' => 'Role management module',
                'status' => 'active',
            ],
            [
                'uid' => UuidHelper::generateBinary(),
                'service_uid' => $authService->uid,
                'name' => 'Services',
                'code' => 'services',
                'description' => 'Service management module',
                'status' => 'active',
            ],
            [
                'uid' => UuidHelper::generateBinary(),
                'service_uid' => $authService->uid,
                'name' => 'Modules',
                'code' => 'modules',
                'description' => 'Module management module',
                'status' => 'active',
            ],
            [
                'uid' => UuidHelper::generateBinary(),
                'service_uid' => $authService->uid,
                'name' => 'Permissions',
                'code' => 'permissions',
                'description' => 'Permission management module',
                'status' => 'active',
            ],
        ];

        foreach ($modules as $module) {
            Module::firstOrCreate(
                [
                    'service_uid' => $module['service_uid'],
                    'code' => $module['code'],
                ],
                $module
            );
        }
    }
}
```

#### `database/seeders/RoleSeeder.php`

```php
<?php

namespace Database\Seeders;

use App\Models\Role;
use App\Utils\UuidHelper;
use Illuminate\Database\Seeder;

class RoleSeeder extends Seeder
{
    public function run(): void
    {
        $roles = [
            [
                'uid' => UuidHelper::generateBinary(),
                'name' => 'admin',
                'description' => 'Administrator with full access to all modules',
                'is_system' => true,
                'status' => 'active',
            ],
            [
                'uid' => UuidHelper::generateBinary(),
                'name' => 'user',
                'description' => 'Standard user with limited access',
                'is_system' => true,
                'status' => 'active',
            ],
        ];

        foreach ($roles as $role) {
            Role::firstOrCreate(
                ['name' => $role['name']],
                $role
            );
        }
    }
}
```

#### `database/seeders/RolePermissionSeeder.php`

```php
<?php

namespace Database\Seeders;

use App\Models\Module;
use App\Models\Role;
use App\Models\RolePermission;
use App\Utils\UuidHelper;
use Illuminate\Database\Seeder;

class RolePermissionSeeder extends Seeder
{
    public function run(): void
    {
        $adminRole = Role::where('name', 'admin')->first();
        $userRole = Role::where('name', 'user')->first();
        $modules = Module::all();

        // Admin gets full permissions on all modules
        foreach ($modules as $module) {
            RolePermission::firstOrCreate(
                [
                    'role_uid' => $adminRole->uid,
                    'module_uid' => $module->uid,
                ],
                [
                    'uid' => UuidHelper::generateBinary(),
                    'role_uid' => $adminRole->uid,
                    'module_uid' => $module->uid,
                    'can_create' => true,
                    'can_read' => true,
                    'can_update' => true,
                    'can_delete' => true,
                    'status' => 'active',
                ]
            );
        }

        // User role gets NO default permissions
        // Admin must explicitly grant permissions to users
        // This is intentional - users need admin to enable their permissions
    }
}
```

#### `database/seeders/AdminUserSeeder.php`

```php
<?php

namespace Database\Seeders;

use App\Models\Role;
use App\Models\User;
use App\Models\UserRole;
use App\Utils\UuidHelper;
use Illuminate\Database\Seeder;
use Illuminate\Support\Facades\Hash;

class AdminUserSeeder extends Seeder
{
    public function run(): void
    {
        // Create admin user
        $adminUser = User::firstOrCreate(
            ['email' => 'admin@example.com'],
            [
                'uid' => UuidHelper::generateBinary(),
                'code' => 'USR-0001',
                'username' => 'admin',
                'email' => 'admin@example.com',
                'password' => Hash::make('Admin@123!'), // Change in production!
                'email_verified_at' => now(),
                'is_blocked' => false,
                'status' => 'active',
            ]
        );

        // Assign admin role
        $adminRole = Role::where('name', 'admin')->first();

        UserRole::firstOrCreate(
            [
                'user_uid' => $adminUser->uid,
                'role_uid' => $adminRole->uid,
            ],
            [
                'uid' => UuidHelper::generateBinary(),
                'user_uid' => $adminUser->uid,
                'role_uid' => $adminRole->uid,
                'status' => 'active',
            ]
        );

        $this->command->info('Admin user created:');
        $this->command->info('  Email: admin@example.com');
        $this->command->info('  Password: Admin@123!');
        $this->command->warn('  ⚠️  Please change the password in production!');
    }
}
```

---

## API Authentication Rules

### Endpoint Access Matrix

```
┌─────────────────────────────────────────────────────────────────┐
│                    Authentication Requirements                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  PUBLIC ENDPOINTS (No Authentication Required)                   │
│  ├── POST /api/v1/auth/login                                    │
│  ├── POST /api/v1/auth/register                                 │
│  ├── POST /api/v1/auth/forgot-password                          │
│  ├── POST /api/v1/auth/reset-password                           │
│  ├── POST /api/v1/auth/verify-email/{token}                     │
│  ├── POST /api/v1/auth/resend-verification                      │
│  ├── POST /api/v1/auth/verify-otp                               │
│  ├── POST /api/v1/auth/resend-otp                               │
│  └── POST /api/v1/auth/refresh-token                            │
│                                                                  │
│  AUTHENTICATED ENDPOINTS (Token Required)                        │
│  └── All other endpoints                                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Permission Check Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                   Permission Check Middleware                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. Validate JWT Token                                          │
│     └── Invalid/Expired → 401 Unauthorized                      │
│                                                                  │
│  2. Check if User is Admin Role                                 │
│     └── YES → ALLOW (bypass permission check)                   │
│     └── NO → Continue to step 3                                 │
│                                                                  │
│  3. Check User Permission Overrides                             │
│     ├── GRANT override exists (not expired) → ALLOW             │
│     └── DENY override exists (not expired) → DENY (403)         │
│                                                                  │
│  4. Check Role Permissions                                      │
│     ├── ANY role has permission → ALLOW                         │
│     └── NO role has permission → DENY (403)                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Service-to-Service Authentication

```
┌─────────────────────────────────────────────────────────────────┐
│              Service-to-Service Authentication                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Purpose:                                                        │
│  - Allow other microservices to validate tokens/permissions      │
│  - Used by: GET /api/v1/auth/validate-token                     │
│            GET /api/v1/permissions/check                         │
│                                                                  │
│  Authentication Method:                                          │
│  - Each service has SERVICE_SECRET_TOKEN in .env                 │
│  - Token: 16 characters (letters, numbers, special chars)        │
│  - All services must use the SAME secret token                   │
│                                                                  │
│  Request Header:                                                 │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ X-Service-Token: Abc123!@#XyZ456                           │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  Validation Flow:                                                │
│  1. Check X-Service-Token header exists                          │
│  2. Compare with SERVICE_SECRET_TOKEN from .env                  │
│  3. If match → Process request                                   │
│  4. If no match → 401 Unauthorized (INVALID_SERVICE_TOKEN)       │
│                                                                  │
│  Security Notes:                                                 │
│  - Token should be rotated periodically                          │
│  - Never log the service token                                   │
│  - Use HTTPS in production                                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Service Token Endpoints:**

| Endpoint | Description |
|----------|-------------|
| `GET /api/v1/auth/validate-token` | Validate user JWT token |
| `GET /api/v1/permissions/check` | Check user permission |

**Example Request from Other Service:**
```bash
curl -X GET "https://auth.example.com/api/v1/auth/validate-token" \
  -H "X-Service-Token: Abc123!@#XyZ456" \
  -H "Authorization: Bearer <user_jwt_token>"
```

---

### Middleware Implementation

```php
// app/Http/Middleware/CheckPermission.php

<?php

namespace App\Http\Middleware;

use App\Constants\ErrorCodeConstants;
use App\Constants\PermissionConstants;
use App\Services\PermissionService;
use Closure;
use Illuminate\Http\Request;

class CheckPermission
{
    public function __construct(
        private PermissionService $permissionService
    ) {}

    public function handle(Request $request, Closure $next, string $serviceCode, string $moduleCode, string $action)
    {
        $user = $request->user();

        // Check if user has admin role (bypass all permission checks)
        if ($this->permissionService->isAdmin($user)) {
            return $next($request);
        }

        // Check permission
        $hasPermission = $this->permissionService->checkPermission(
            $user->uid,
            $serviceCode,
            $moduleCode,
            $action
        );

        if (!$hasPermission) {
            return response()->json([
                'status' => 403,
                'message' => 'You do not have permission to perform this action',
                'error_code' => ErrorCodeConstants::PERMISSION_DENIED,
            ], 403);
        }

        return $next($request);
    }
}
```

```php
// app/Http/Middleware/ValidateServiceToken.php

<?php

namespace App\Http\Middleware;

use App\Constants\ErrorCodeConstants;
use Closure;
use Illuminate\Http\Request;

class ValidateServiceToken
{
    public function handle(Request $request, Closure $next)
    {
        $serviceToken = $request->header('X-Service-Token');

        if (!$serviceToken) {
            return response()->json([
                'status' => 401,
                'message' => 'Service token is required',
                'error_code' => ErrorCodeConstants::MISSING_SERVICE_TOKEN,
            ], 401);
        }

        if ($serviceToken !== config('auth.service_secret_token')) {
            return response()->json([
                'status' => 401,
                'message' => 'Invalid service token',
                'error_code' => ErrorCodeConstants::INVALID_SERVICE_TOKEN,
            ], 401);
        }

        return $next($request);
    }
}
```

### Route Configuration Example

```php
// routes/api.php

Route::prefix('v1')->group(function () {

    // Public routes
    Route::prefix('auth')->group(function () {
        Route::post('login', [AuthController::class, 'login']);
        Route::post('register', [AuthController::class, 'register']);
        Route::post('forgot-password', [AuthController::class, 'forgotPassword']);
        Route::post('reset-password', [AuthController::class, 'resetPassword']);
        Route::post('verify-email/{token}', [AuthController::class, 'verifyEmail']);
        Route::post('resend-verification', [AuthController::class, 'resendVerification']);
        Route::post('verify-otp', [AuthController::class, 'verifyOtp']);
        Route::post('resend-otp', [AuthController::class, 'resendOtp']);
        Route::post('refresh-token', [AuthController::class, 'refreshToken']);
    });

    // Service-to-Service routes (requires X-Service-Token header)
    Route::middleware('service.token')->group(function () {
        Route::get('auth/validate-token', [AuthController::class, 'validateToken']);
        Route::get('permissions/check', [PermissionController::class, 'check']);
    });

    // Protected routes (requires user JWT token)
    Route::middleware('auth:api')->group(function () {

        // Auth routes (authenticated)
        Route::prefix('auth')->group(function () {
            Route::post('logout', [AuthController::class, 'logout']);
            Route::post('change-password', [AuthController::class, 'changePassword']);
        });

        // Users
        Route::prefix('users')->group(function () {
            Route::get('/', [UserController::class, 'index'])
                ->middleware('permission:auth,users,read');
            Route::get('{uid}', [UserController::class, 'show'])
                ->middleware('permission:auth,users,read');
            Route::post('/', [UserController::class, 'store'])
                ->middleware('permission:auth,users,create');
            Route::put('{uid}', [UserController::class, 'update'])
                ->middleware('permission:auth,users,update');
            Route::delete('{uid}', [UserController::class, 'destroy'])
                ->middleware('permission:auth,users,delete');
            Route::post('{uid}/block', [UserController::class, 'block'])
                ->middleware('permission:auth,users,update');
            Route::post('{uid}/unblock', [UserController::class, 'unblock'])
                ->middleware('permission:auth,users,update');
            Route::post('{uid}/unlock', [UserController::class, 'unlock'])
                ->middleware('permission:auth,users,update');
            Route::get('{uid}/sessions', [UserController::class, 'sessions'])
                ->middleware('permission:auth,users,read');
            Route::delete('{uid}/sessions', [UserController::class, 'revokeSessions'])
                ->middleware('permission:auth,users,update');
            Route::post('{uid}/resend-verification', [UserController::class, 'resendVerification'])
                ->middleware('permission:auth,users,update');
            Route::post('{uid}/permission-overrides', [PermissionOverrideController::class, 'store'])
                ->middleware('permission:auth,permissions,create');
            Route::get('{uid}/permission-overrides', [PermissionOverrideController::class, 'index'])
                ->middleware('permission:auth,permissions,read');
            Route::delete('{uid}/permission-overrides/{overrideUid}', [PermissionOverrideController::class, 'destroy'])
                ->middleware('permission:auth,permissions,delete');
        });

        // Roles
        Route::prefix('roles')->group(function () {
            Route::get('/', [RoleController::class, 'index'])
                ->middleware('permission:auth,roles,read');
            Route::get('{uid}', [RoleController::class, 'show'])
                ->middleware('permission:auth,roles,read');
            Route::post('/', [RoleController::class, 'store'])
                ->middleware('permission:auth,roles,create');
            Route::put('{uid}', [RoleController::class, 'update'])
                ->middleware('permission:auth,roles,update');
            Route::delete('{uid}', [RoleController::class, 'destroy'])
                ->middleware('permission:auth,roles,delete');
            Route::put('{uid}/permissions', [RoleController::class, 'updatePermissions'])
                ->middleware('permission:auth,roles,update');
        });

        // Services
        Route::prefix('services')->group(function () {
            Route::get('/', [ServiceController::class, 'index'])
                ->middleware('permission:auth,services,read');
            Route::get('{uid}', [ServiceController::class, 'show'])
                ->middleware('permission:auth,services,read');
            Route::post('/', [ServiceController::class, 'store'])
                ->middleware('permission:auth,services,create');
            Route::put('{uid}', [ServiceController::class, 'update'])
                ->middleware('permission:auth,services,update');
            Route::delete('{uid}', [ServiceController::class, 'destroy'])
                ->middleware('permission:auth,services,delete');
        });

        // Modules
        Route::prefix('modules')->group(function () {
            Route::get('/', [ModuleController::class, 'index'])
                ->middleware('permission:auth,modules,read');
            Route::get('{uid}', [ModuleController::class, 'show'])
                ->middleware('permission:auth,modules,read');
            Route::post('/', [ModuleController::class, 'store'])
                ->middleware('permission:auth,modules,create');
            Route::put('{uid}', [ModuleController::class, 'update'])
                ->middleware('permission:auth,modules,update');
            Route::delete('{uid}', [ModuleController::class, 'destroy'])
                ->middleware('permission:auth,modules,delete');
        });

    });
});
```

---

## Audit Logs Schema

### Table: `audit_logs`

```sql
CREATE TABLE audit_logs (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    uid             BINARY(16) UNIQUE NOT NULL,

    -- Who performed the action
    user_uid        BINARY(16) NULL,                   -- NULL for system actions or unauthenticated
    user_email      VARCHAR(255) NULL,                 -- Denormalized for historical reference

    -- What action was performed
    action          VARCHAR(50) NOT NULL,              -- create, update, delete, login, etc.
    entity_type     VARCHAR(50) NOT NULL,              -- user, role, service, module, session
    entity_uid      BINARY(16) NULL,                   -- UID of affected entity
    entity_code     VARCHAR(50) NULL,                  -- Code of affected entity (e.g., USR-0001)

    -- Change details
    old_values      JSON NULL,                         -- Previous state (for updates)
    new_values      JSON NULL,                         -- New state (for creates/updates)

    -- Request context
    ip_address      VARCHAR(45) NULL,                  -- Client IP (supports IPv6)
    user_agent      TEXT NULL,                         -- Browser/client info
    request_id      VARCHAR(36) NULL,                  -- Request correlation ID

    -- Additional context
    description     TEXT NULL,                         -- Human-readable description
    metadata        JSON NULL,                         -- Additional context data

    -- Timestamps
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    -- Indexes for common queries
    INDEX idx_user (user_uid),
    INDEX idx_action (action),
    INDEX idx_entity (entity_type, entity_uid),
    INDEX idx_created (created_at),
    INDEX idx_ip (ip_address)
);
```

### Audit Log Examples

**User Login:**
```json
{
    "uid": "binary...",
    "user_uid": "binary...",
    "user_email": "john@example.com",
    "action": "login",
    "entity_type": "session",
    "entity_uid": "binary...",
    "old_values": null,
    "new_values": {
        "device_name": "Chrome on Windows",
        "is_trusted": false
    },
    "ip_address": "192.168.1.100",
    "user_agent": "Mozilla/5.0...",
    "description": "User logged in from new device",
    "metadata": {
        "requires_otp": true
    }
}
```

**User Created:**
```json
{
    "uid": "binary...",
    "user_uid": "binary...",
    "user_email": "admin@example.com",
    "action": "create",
    "entity_type": "user",
    "entity_uid": "binary...",
    "entity_code": "USR-0002",
    "old_values": null,
    "new_values": {
        "username": "newuser",
        "email": "newuser@example.com",
        "roles": ["user"]
    },
    "ip_address": "192.168.1.100",
    "description": "Created new user account"
}
```

**Role Updated:**
```json
{
    "uid": "binary...",
    "user_uid": "binary...",
    "user_email": "admin@example.com",
    "action": "update",
    "entity_type": "role",
    "entity_uid": "binary...",
    "old_values": {
        "name": "manager",
        "description": "Old description"
    },
    "new_values": {
        "name": "senior_manager",
        "description": "New description"
    },
    "ip_address": "192.168.1.100",
    "description": "Updated role details"
}
```

**User Blocked:**
```json
{
    "uid": "binary...",
    "user_uid": "binary...",
    "user_email": "admin@example.com",
    "action": "block",
    "entity_type": "user",
    "entity_uid": "binary...",
    "entity_code": "USR-0005",
    "old_values": {
        "is_blocked": false
    },
    "new_values": {
        "is_blocked": true,
        "blocked_reason": "Violation of terms"
    },
    "ip_address": "192.168.1.100",
    "description": "Blocked user account"
}
```

---

## Soft Delete Handling

### Deletion Rules

```
┌─────────────────────────────────────────────────────────────────┐
│                      Soft Delete Rules                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  GENERAL RULES:                                                  │
│  • Soft delete sets: deleted_at = NOW(), archived = TRUE        │
│  • Status is set to 'inactive' on deletion                       │
│  • NO API endpoint for restoration                               │
│  • Restoration requires manual database intervention             │
│                                                                  │
│  DEPENDENCY CHECKS (Cannot delete if has dependencies):          │
│                                                                  │
│  ┌────────────┬────────────────────────────────────────────┐    │
│  │ Entity     │ Cannot Delete If...                        │    │
│  ├────────────┼────────────────────────────────────────────┤    │
│  │ User       │ - (No restrictions, can always delete)     │    │
│  │            │ - All sessions will be revoked             │    │
│  ├────────────┼────────────────────────────────────────────┤    │
│  │ Role       │ - Has assigned users (user_roles)          │    │
│  │            │ - Is a system role (is_system = true)      │    │
│  ├────────────┼────────────────────────────────────────────┤    │
│  │ Service    │ - Has modules                              │    │
│  ├────────────┼────────────────────────────────────────────┤    │
│  │ Module     │ - Has role_permissions assigned            │    │
│  │            │ - Has user_permission_overrides assigned   │    │
│  └────────────┴────────────────────────────────────────────┘    │
│                                                                  │
│  CASCADING BEHAVIOR:                                             │
│  • Deleting User → Cascades to: user_roles, user_permission_    │
│    overrides, sessions (all marked deleted/revoked)              │
│  • Deleting Role → Does NOT cascade (blocked if has users)      │
│  • Deleting Service → Does NOT cascade (blocked if has modules) │
│  • Deleting Module → Does NOT cascade (blocked if has perms)    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Uniqueness with Soft Deletes

```
┌─────────────────────────────────────────────────────────────────┐
│                 Uniqueness Constraint Behavior                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  UNIQUE AMONG ACTIVE RECORDS ONLY:                               │
│  • users.username   - Can reuse if original is soft-deleted     │
│  • users.email      - Can reuse if original is soft-deleted     │
│                                                                  │
│  UNIQUE ACROSS ALL RECORDS (including deleted):                  │
│  • users.code       - USR-XXXX is never reused                  │
│  • services.code    - Service code is never reused              │
│  • roles.name       - Role name is never reused                 │
│                                                                  │
│  Implementation Note:                                            │
│  For "active only" uniqueness, validation should check:          │
│  WHERE (column = value) AND deleted_at IS NULL                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Document Changes from Original

### Removed

1. **Table: `email_verifications`** - Removed from database schema diagram. OTP verifications table handles email verification OTPs.

### Added

1. **API Specification Details** - Complete request/response examples for all endpoints
2. **Environment Configuration** - Full `.env` template and configuration constants
3. **Error Codes** - Complete error code reference with HTTP status codes
4. **Business Logic Clarifications** - Email verification, password reset, session management
5. **Testing Strategy** - PHPUnit test structure with positive/negative test examples
6. **Deployment & DevOps** - Database seeders for initial data
7. **API Authentication Rules** - Endpoint access matrix and permission middleware
8. **Audit Logs Schema** - Complete audit logging table design
9. **Soft Delete Handling** - Deletion rules and dependency checks

### Added in v1.2

1. **Device Trust & OTP Logic** - Detailed device identification and OTP trigger conditions
2. **Refresh Token Rotation** - Token invalidation on use
3. **Admin-Created Users Flow** - Auto-verified email, welcome email with temp password
4. **Role Assignment Rules** - Minimum 1 role, last admin protection
5. **Permission Override Handling** - No cleanup job, expired overrides ignored in checks
6. **Service-to-Service Authentication** - X-Service-Token header for inter-service calls
7. **User Code Continuation** - USR-9999 continues to USR-10000
8. **New Error Codes** - USER_MUST_HAVE_ROLE, CANNOT_REMOVE_LAST_ADMIN, CANNOT_DELETE_LAST_ADMIN, CANNOT_DEACTIVATE_LAST_ADMIN, INVALID_SERVICE_TOKEN, MISSING_SERVICE_TOKEN

---

## Revision History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-01-10 | Initial documentation |
| 1.1 | 2026-01-13 | Added API specifications, environment config, error codes, business logic, testing, seeders, auth rules, audit logs, soft delete handling |
| 1.2 | 2026-01-14 | Added device trust logic, refresh token rotation, admin-created users flow, role assignment rules, service-to-service auth, permission override handling |
