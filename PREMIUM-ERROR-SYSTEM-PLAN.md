# PostEngage.ai — Premium Error & Exception System (From Scratch)

**Date:** March 1, 2026
**Approach:** Build from zero — no old code, no legacy patterns
**Stack:** NestJS v11 Monorepo | MongoDB (Mongoose v9) | BullMQ | Socket.io | Redis

---

## Design Philosophy — Inspired by Industry Leaders

After deep-diving into how Stripe, GitHub, Twilio, Shopify, Auth0, and Cloudflare build their error systems, here's what we're adopting:

| Pattern | Inspiration | What We Take |
|---------|-------------|--------------|
| `type` + `code` + `param` | Stripe | Machine-readable codes + field-level validation |
| `errors[]` array with `resource` + `field` | GitHub | Multiple validation errors per request |
| Numeric error codes + `more_info` URL | Twilio | Documentation URL per error code |
| `success: boolean` envelope | Cloudflare | Explicit success flag (not just HTTP status) |
| `error_description` for auth | Auth0 | OAuth-standard auth error format |
| Consistent shape across ALL endpoints | All of them | One shape to rule them all |

---

## The Unified Error Response Shape

Every error — HTTP, WebSocket, Queue, RPC — produces this exact shape:

```typescript
interface PostEngageErrorResponse {
  success: false;

  error: {
    type: string;                    // "validation_error" | "authentication_error" | "rate_limit_error" | "api_error" | "server_error"
    code: string;                    // "PE-AUTH-001" — machine-readable, never changes
    message: string;                 // "Invalid credentials" — human-readable, can change
    param?: string;                  // "email" — which field caused this (Stripe pattern)
    details?: ErrorDetail[];         // Multiple field errors (GitHub pattern)
    doc_url?: string;                // "https://docs.postengage.ai/errors/PE-AUTH-001" (Twilio pattern)
  };

  meta: {
    request_id: string;              // "req_01HXYZ..." — ULID-based
    timestamp: string;               // ISO 8601
    path: string;                    // "/api/v1/auth/login"
    status: number;                  // HTTP status code
  };

  // Only in development/staging — NEVER in production
  debug?: {
    stack?: string;
    cause?: string;
    service?: string;                // "gatekeeper" | "pulse" | "worker" | "scheduler"
    duration_ms?: number;
  };
}

interface ErrorDetail {
  field: string;                     // "email" | "password" | "metadata.name"
  message: string;                   // "must be a valid email address"
  code: string;                      // "invalid_format" | "required" | "too_long"
}
```

---

## Error Type System (Stripe-inspired)

Instead of treating every error the same, we categorize by _type_ — this tells the client HOW to handle it:

| Type | HTTP Range | Client Behavior |
|------|-----------|-----------------|
| `validation_error` | 400, 422 | Show field-level errors in form |
| `authentication_error` | 401 | Redirect to login / refresh token |
| `authorization_error` | 403 | Show "no permission" UI |
| `not_found_error` | 404 | Show 404 page or "resource deleted" |
| `conflict_error` | 409 | Show "already exists" or retry |
| `rate_limit_error` | 429 | Show "slow down" + retry after X seconds |
| `api_error` | 400 | Bad API usage (wrong method, bad params) |
| `payment_error` | 402 | Show payment failure + retry |
| `server_error` | 500+ | Show "something went wrong" + report |
| `platform_error` | 502, 503 | Show "Instagram/Facebook is down" + retry |

---

## Error Code Taxonomy

```
PE-{DOMAIN}-{SEQUENCE}

Domains:
  AUTH   — Authentication & Authorization        (PE-AUTH-001 to PE-AUTH-999)
  USR    — User Account & Profile                (PE-USR-001 to PE-USR-999)
  VAL    — Validation & Input                    (PE-VAL-001 to PE-VAL-999)
  RATE   — Rate Limiting & Quotas                (PE-RATE-001 to PE-RATE-999)
  PAY    — Payment & Subscription                (PE-PAY-001 to PE-PAY-999)
  SOC    — Social Media Platform                 (PE-SOC-001 to PE-SOC-999)
  ENG    — Engagement & Automation               (PE-ENG-001 to PE-ENG-999)
  FILE   — File Upload & Media                   (PE-FILE-001 to PE-FILE-999)
  WS     — WebSocket & Real-time                 (PE-WS-001 to PE-WS-999)
  QUEUE  — Background Jobs                       (PE-QUEUE-001 to PE-QUEUE-999)
  INT    — Internal / Server                     (PE-INT-001 to PE-INT-999)
  EXT    — External Service                      (PE-EXT-001 to PE-EXT-999)
  API    — API Contract (versioning, endpoints)  (PE-API-001 to PE-API-999)
  DB     — Database                              (PE-DB-001 to PE-DB-999)
```

---

## NestJS Library Structure

We create a proper NestJS library: `libs/errors/`

```
libs/errors/
├── tsconfig.lib.json
└── src/
    ├── index.ts                                    # Barrel exports
    ├── errors.module.ts                            # NestJS Module (forRoot/forRootAsync)
    │
    ├── core/                                       # Core exception classes
    │   ├── postengage.exception.ts                 # Main exception class
    │   ├── validation.exception.ts                 # Validation-specific exception
    │   └── platform.exception.ts                   # External platform errors (Instagram, etc.)
    │
    ├── types/                                      # Type definitions
    │   ├── error-response.types.ts                 # PostEngageErrorResponse, ErrorDetail, etc.
    │   ├── error-code.types.ts                     # ErrorCode union type, ErrorType enum
    │   └── error-definition.types.ts               # ErrorCodeDefinition interface
    │
    ├── codes/                                      # Error code registry
    │   ├── index.ts                                # Aggregates all codes + lookup functions
    │   ├── auth.codes.ts                           # PE-AUTH-001 to PE-AUTH-xxx
    │   ├── user.codes.ts                           # PE-USR-001 to PE-USR-xxx
    │   ├── validation.codes.ts                     # PE-VAL-001 to PE-VAL-xxx
    │   ├── rate-limit.codes.ts                     # PE-RATE-001 to PE-RATE-xxx
    │   ├── payment.codes.ts                        # PE-PAY-001 to PE-PAY-xxx
    │   ├── social.codes.ts                         # PE-SOC-001 to PE-SOC-xxx
    │   ├── engagement.codes.ts                     # PE-ENG-001 to PE-ENG-xxx
    │   ├── file.codes.ts                           # PE-FILE-001 to PE-FILE-xxx
    │   ├── websocket.codes.ts                      # PE-WS-001 to PE-WS-xxx
    │   ├── queue.codes.ts                          # PE-QUEUE-001 to PE-QUEUE-xxx
    │   ├── internal.codes.ts                       # PE-INT-001 to PE-INT-xxx
    │   ├── external.codes.ts                       # PE-EXT-001 to PE-EXT-xxx
    │   ├── api.codes.ts                            # PE-API-001 to PE-API-xxx
    │   └── database.codes.ts                       # PE-DB-001 to PE-DB-xxx
    │
    ├── filters/                                    # Exception filters (NestJS @Catch)
    │   ├── global-exception.filter.ts              # HTTP — catches everything
    │   ├── ws-exception.filter.ts                  # WebSocket — for Pulse gateway
    │   └── rpc-exception.filter.ts                 # RPC — for Worker/Scheduler queue jobs
    │
    ├── handlers/                                   # Error-specific handlers
    │   ├── mongoose-error.handler.ts               # E11000, CastError, ValidationError, etc.
    │   ├── throttler-error.handler.ts              # @nestjs/throttler → PE-RATE-001
    │   └── validation-pipe-error.handler.ts        # class-validator → PE-VAL-xxx
    │
    ├── sanitizers/                                 # Production safety
    │   ├── error-sanitizer.ts                      # Strip stack traces, redact secrets
    │   └── sensitive-fields.ts                     # List of fields to redact
    │
    └── utils/                                      # Helpers
        ├── error-response.builder.ts               # Builds PostEngageErrorResponse
        ├── error-message.utils.ts                  # getErrorMessage, getErrorInfo
        └── http-status.mapper.ts                   # Maps error types → HTTP status codes
```

---

## PHASE 1: Core Library Setup

**Goal:** Create the `libs/errors/` NestJS library with core types and exception classes.

### Task 1.1: Generate NestJS Library

Since `nest-cli.json` and `tsconfig.json` already have the `errors` library config (`@app/errors` path alias), we just need to create the directory structure and files. The config already maps:
- `@app/errors` → `libs/errors/src`
- `@app/errors/*` → `libs/errors/src/*`

**Files to create:**
- `libs/errors/tsconfig.lib.json`
- `libs/errors/src/index.ts`
- `libs/errors/src/errors.module.ts`

### Task 1.2: Define Type System

**File: `libs/errors/src/types/error-response.types.ts`**

```typescript
// The unified error response — every error everywhere uses this shape
export interface PostEngageErrorResponse {
  success: false;
  error: {
    type: ErrorType;
    code: string;
    message: string;
    param?: string;
    details?: ErrorDetail[];
    doc_url?: string;
  };
  meta: ResponseMeta;
  debug?: DebugInfo;
}

export interface ErrorDetail {
  field: string;
  message: string;
  code: string;
}

export interface ResponseMeta {
  request_id: string;
  timestamp: string;
  path: string;
  status: number;
}

export interface DebugInfo {
  stack?: string;
  cause?: string;
  service?: string;
  duration_ms?: number;
}
```

**File: `libs/errors/src/types/error-code.types.ts`**

```typescript
export enum ErrorType {
  VALIDATION = 'validation_error',
  AUTHENTICATION = 'authentication_error',
  AUTHORIZATION = 'authorization_error',
  NOT_FOUND = 'not_found_error',
  CONFLICT = 'conflict_error',
  RATE_LIMIT = 'rate_limit_error',
  PAYMENT = 'payment_error',
  API = 'api_error',
  PLATFORM = 'platform_error',
  SERVER = 'server_error',
}

export enum ErrorDomain {
  AUTH = 'AUTH',
  USR = 'USR',
  VAL = 'VAL',
  RATE = 'RATE',
  PAY = 'PAY',
  SOC = 'SOC',
  ENG = 'ENG',
  FILE = 'FILE',
  WS = 'WS',
  QUEUE = 'QUEUE',
  INT = 'INT',
  EXT = 'EXT',
  API = 'API',
  DB = 'DB',
}

export enum ErrorSeverity {
  LOW = 'low',
  MEDIUM = 'medium',
  HIGH = 'high',
  CRITICAL = 'critical',
}
```

**File: `libs/errors/src/types/error-definition.types.ts`**

```typescript
import { ErrorDomain, ErrorSeverity, ErrorType } from './error-code.types.js';

export interface ErrorCodeDefinition {
  code: string;                      // "PE-AUTH-001"
  type: ErrorType;                   // "authentication_error"
  domain: ErrorDomain;               // "AUTH"
  httpStatus: number;                // 401
  message: string;                   // User-facing message
  developerMessage: string;          // Internal message for logs
  severity: ErrorSeverity;           // "medium"
  retryable: boolean;                // Can the client retry?
  retryAfterSeconds?: number;        // If retryable, how long to wait
  docUrl?: string;                   // Documentation URL
}
```

### Task 1.3: Build PostEngageException Class

**File: `libs/errors/src/core/postengage.exception.ts`**

```typescript
import { HttpException } from '@nestjs/common';

export interface PostEngageExceptionOptions {
  message?: string;                  // Override default message
  param?: string;                    // Field that caused the error (Stripe pattern)
  details?: ErrorDetail[];           // Multiple field errors (GitHub pattern)
  cause?: Error;                     // Original error (for error chaining)
  metadata?: Record<string, unknown>; // Extra context for logging
}

export class PostEngageException extends HttpException {
  public readonly errorCode: string;
  public readonly errorType: ErrorType;
  public readonly param?: string;
  public readonly details?: ErrorDetail[];
  public readonly metadata?: Record<string, unknown>;
  public readonly originalCause?: Error;

  constructor(code: string, options?: PostEngageExceptionOptions) {
    const definition = getErrorDefinition(code); // Lookup from registry
    const message = options?.message ?? definition.message;
    const status = definition.httpStatus;

    super(message, status, { cause: options?.cause });

    this.errorCode = code;
    this.errorType = definition.type;
    this.param = options?.param;
    this.details = options?.details;
    this.metadata = options?.metadata;
    this.originalCause = options?.cause;
  }
}
```

### Task 1.4: Build ValidationException

**File: `libs/errors/src/core/validation.exception.ts`**

Specialized for class-validator pipe errors — auto-converts validation array to `ErrorDetail[]`:

```typescript
export class ValidationException extends PostEngageException {
  constructor(errors: ErrorDetail[]) {
    super('PE-VAL-001', {
      message: 'Validation failed',
      details: errors,
    });
  }
}
```

### Task 1.5: Build PlatformException

**File: `libs/errors/src/core/platform.exception.ts`**

For external platform errors (Instagram, Facebook, etc.):

```typescript
export class PlatformException extends PostEngageException {
  public readonly platform: string;
  public readonly platformCode?: string | number;
  public readonly platformMessage?: string;

  constructor(platform: string, code: string, options?) {
    super(code, options);
    this.platform = platform;
    // ...
  }
}
```

**Phase 1 TODO Checklist:**
- [x] Create `libs/errors/tsconfig.lib.json`
- [x] Create `libs/errors/src/errors.module.ts`
- [x] Create `libs/errors/src/types/error-response.types.ts`
- [x] Create `libs/errors/src/types/error-code.types.ts`
- [x] Create `libs/errors/src/types/error-definition.types.ts`
- [x] Create `libs/errors/src/core/postengage.exception.ts`
- [x] Create `libs/errors/src/core/validation.exception.ts`
- [x] Create `libs/errors/src/core/platform.exception.ts`
- [x] Create `libs/errors/src/index.ts` barrel exports
- [x] Verify TypeScript compilation

---

## PHASE 2: Error Code Registry

**Goal:** Define all error codes across 14 domains.

### Task 2.1: Error Code Definition Files

Each domain gets its own file. Example format:

**File: `libs/errors/src/codes/auth.codes.ts`**

```typescript
import { ErrorCodeDefinition } from '../types/error-definition.types.js';
import { ErrorDomain, ErrorSeverity, ErrorType } from '../types/error-code.types.js';

export const AUTH_ERROR_CODES = {
  'PE-AUTH-001': {
    code: 'PE-AUTH-001',
    type: ErrorType.AUTHENTICATION,
    domain: ErrorDomain.AUTH,
    httpStatus: 401,
    message: 'Invalid email or password.',
    developerMessage: 'Login credentials incorrect',
    severity: ErrorSeverity.MEDIUM,
    retryable: false,
  },
  'PE-AUTH-002': {
    code: 'PE-AUTH-002',
    type: ErrorType.AUTHENTICATION,
    domain: ErrorDomain.AUTH,
    httpStatus: 401,
    message: 'Your session has expired. Please log in again.',
    developerMessage: 'JWT token expired',
    severity: ErrorSeverity.MEDIUM,
    retryable: false,
  },
  // ... more codes
} as const satisfies Record<string, ErrorCodeDefinition>;
```

### Task 2.2: Error Codes Aggregator + Lookup Functions

**File: `libs/errors/src/codes/index.ts`**

```typescript
import { AUTH_ERROR_CODES } from './auth.codes.js';
import { USER_ERROR_CODES } from './user.codes.js';
// ... all domains

export const ERROR_REGISTRY = {
  ...AUTH_ERROR_CODES,
  ...USER_ERROR_CODES,
  ...VALIDATION_ERROR_CODES,
  ...RATE_LIMIT_ERROR_CODES,
  ...PAYMENT_ERROR_CODES,
  ...SOCIAL_ERROR_CODES,
  ...ENGAGEMENT_ERROR_CODES,
  ...FILE_ERROR_CODES,
  ...WEBSOCKET_ERROR_CODES,
  ...QUEUE_ERROR_CODES,
  ...INTERNAL_ERROR_CODES,
  ...EXTERNAL_ERROR_CODES,
  ...API_ERROR_CODES,
  ...DATABASE_ERROR_CODES,
} as const;

export type ErrorCode = keyof typeof ERROR_REGISTRY;

export function getErrorDefinition(code: string): ErrorCodeDefinition {
  const definition = ERROR_REGISTRY[code as ErrorCode];
  if (!definition) {
    // Fallback to generic server error
    return {
      code,
      type: ErrorType.SERVER,
      domain: ErrorDomain.INT,
      httpStatus: 500,
      message: 'An unexpected error occurred.',
      developerMessage: `Unknown error code: ${code}`,
      severity: ErrorSeverity.HIGH,
      retryable: false,
    };
  }
  return definition;
}

export function getErrorCodesByDomain(domain: ErrorDomain): ErrorCodeDefinition[] {
  return Object.values(ERROR_REGISTRY).filter(e => e.domain === domain);
}
```

### Complete Error Code List Per Domain

Below is every error code we need. Each one maps to a specific business scenario in PostEngage:

**AUTH (PE-AUTH-001 to PE-AUTH-020)**

| Code | HTTP | Message | Developer Message |
|------|------|---------|-------------------|
| PE-AUTH-001 | 401 | Invalid email or password. | Login credentials incorrect |
| PE-AUTH-002 | 401 | Your session has expired. Please log in again. | JWT token expired |
| PE-AUTH-003 | 401 | Authentication required. | Missing authentication token |
| PE-AUTH-004 | 401 | Your session has been revoked. | Token revoked or blacklisted |
| PE-AUTH-005 | 423 | Account locked due to multiple failed attempts. | Account locked after N failures |
| PE-AUTH-006 | 403 | Your account has been suspended. | Account suspended by admin |
| PE-AUTH-007 | 403 | Please verify your email to continue. | Email not verified |
| PE-AUTH-008 | 401 | Password reset required. | Forced password reset |
| PE-AUTH-009 | 401 | Social login failed. Please try again. | OAuth flow failed |
| PE-AUTH-010 | 409 | This social account is already linked to another user. | OAuth account conflict |
| PE-AUTH-011 | 429 | Too many login attempts. Try again later. | Auth rate limit exceeded |
| PE-AUTH-012 | 403 | Security violation detected. | Suspicious activity flagged |
| PE-AUTH-013 | 409 | An account with this email already exists. | Duplicate email on registration |
| PE-AUTH-014 | 401 | Invalid verification token. | Token malformed or tampered |
| PE-AUTH-015 | 401 | Verification token has expired. | Token TTL exceeded |
| PE-AUTH-016 | 409 | Email is already verified. | Duplicate verification attempt |
| PE-AUTH-017 | 409 | A verification email was already sent. | Active token exists, cooldown |
| PE-AUTH-018 | 403 | You do not have permission for this action. | Missing required permissions |
| PE-AUTH-019 | 403 | This action requires a higher plan. | Role/plan insufficient |
| PE-AUTH-020 | 401 | Invalid or expired refresh token. | Refresh token validation failed |

**USR (PE-USR-001 to PE-USR-008)**

| Code | HTTP | Message |
|------|------|---------|
| PE-USR-001 | 404 | User not found. |
| PE-USR-002 | 409 | This email is already in use. |
| PE-USR-003 | 409 | This username is already taken. |
| PE-USR-004 | 400 | Invalid profile data. |
| PE-USR-005 | 403 | Account deactivated. |
| PE-USR-006 | 400 | Password does not meet requirements. |
| PE-USR-007 | 404 | Session not found. |
| PE-USR-008 | 409 | Social account already connected. |

**VAL (PE-VAL-001 to PE-VAL-008)**

| Code | HTTP | Message |
|------|------|---------|
| PE-VAL-001 | 400 | Validation failed. |
| PE-VAL-002 | 400 | Required field missing. |
| PE-VAL-003 | 400 | Invalid format. |
| PE-VAL-004 | 400 | Value out of allowed range. |
| PE-VAL-005 | 400 | Invalid enum value. |
| PE-VAL-006 | 400 | String too long. |
| PE-VAL-007 | 400 | Invalid pagination parameters. |
| PE-VAL-008 | 415 | Unsupported content type. |

**RATE (PE-RATE-001 to PE-RATE-005)**

| Code | HTTP | Message |
|------|------|---------|
| PE-RATE-001 | 429 | Too many requests. Please slow down. |
| PE-RATE-002 | 429 | API rate limit exceeded. |
| PE-RATE-003 | 429 | Too many login attempts. |
| PE-RATE-004 | 429 | Too many signup attempts. |
| PE-RATE-005 | 403 | Daily quota exceeded. Upgrade your plan. |

**PAY (PE-PAY-001 to PE-PAY-008)**

| Code | HTTP | Message |
|------|------|---------|
| PE-PAY-001 | 402 | Payment processing failed. |
| PE-PAY-002 | 402 | Payment verification failed. |
| PE-PAY-003 | 403 | Your subscription has expired. |
| PE-PAY-004 | 403 | Plan limit reached. Please upgrade. |
| PE-PAY-005 | 500 | Refund processing failed. |
| PE-PAY-006 | 404 | Order not found. |
| PE-PAY-007 | 400 | Invalid payment method. |
| PE-PAY-008 | 409 | Subscription already active. |

**SOC (PE-SOC-001 to PE-SOC-008)**

| Code | HTTP | Message |
|------|------|---------|
| PE-SOC-001 | 401 | Social account authorization expired. Reconnect required. |
| PE-SOC-002 | 429 | Social platform rate limit reached. |
| PE-SOC-003 | 400 | Post rejected by platform. |
| PE-SOC-004 | 403 | Social account suspended by platform. |
| PE-SOC-005 | 503 | Social platform temporarily unavailable. |
| PE-SOC-006 | 404 | Social account not found. |
| PE-SOC-007 | 400 | Invalid media format for platform. |
| PE-SOC-008 | 409 | Duplicate post detected by platform. |

**ENG (PE-ENG-001 to PE-ENG-008)**

| Code | HTTP | Message |
|------|------|---------|
| PE-ENG-001 | 404 | Automation not found. |
| PE-ENG-002 | 403 | Automation limit reached for your plan. |
| PE-ENG-003 | 400 | Invalid automation configuration. |
| PE-ENG-004 | 409 | Automation already running. |
| PE-ENG-005 | 500 | Automation execution failed. |
| PE-ENG-006 | 404 | Lead capture form not found. |
| PE-ENG-007 | 400 | Invalid trigger condition. |
| PE-ENG-008 | 500 | Notification delivery failed. |

**FILE (PE-FILE-001 to PE-FILE-006)**

| Code | HTTP | Message |
|------|------|---------|
| PE-FILE-001 | 413 | File exceeds maximum size. |
| PE-FILE-002 | 415 | Unsupported file type. |
| PE-FILE-003 | 500 | File upload failed. |
| PE-FILE-004 | 507 | Storage quota exceeded. |
| PE-FILE-005 | 404 | Media not found. |
| PE-FILE-006 | 400 | Invalid image dimensions. |

**WS (PE-WS-001 to PE-WS-005)**

| Code | HTTP | Message |
|------|------|---------|
| PE-WS-001 | 401 | WebSocket authentication failed. |
| PE-WS-002 | 404 | Channel not found. |
| PE-WS-003 | 413 | Message exceeds size limit. |
| PE-WS-004 | 429 | Too many concurrent connections. |
| PE-WS-005 | 400 | Invalid WebSocket event. |

**QUEUE (PE-QUEUE-001 to PE-QUEUE-006)**

| Code | HTTP | Message |
|------|------|---------|
| PE-QUEUE-001 | 500 | Background job failed. |
| PE-QUEUE-002 | 408 | Background job timed out. |
| PE-QUEUE-003 | 503 | Job queue is full. |
| PE-QUEUE-004 | 404 | Job not found. |
| PE-QUEUE-005 | 409 | Job already exists. |
| PE-QUEUE-006 | 500 | Max retries exceeded. |

**INT (PE-INT-001 to PE-INT-005)**

| Code | HTTP | Message |
|------|------|---------|
| PE-INT-001 | 500 | An unexpected error occurred. |
| PE-INT-002 | 408 | Request timed out. |
| PE-INT-003 | 503 | Service temporarily unavailable. |
| PE-INT-004 | 500 | Configuration error. |
| PE-INT-005 | 500 | Uncaught exception. |

**EXT (PE-EXT-001 to PE-EXT-005)**

| Code | HTTP | Message |
|------|------|---------|
| PE-EXT-001 | 502 | External service returned an error. |
| PE-EXT-002 | 504 | External service timed out. |
| PE-EXT-003 | 503 | External service unavailable. |
| PE-EXT-004 | 502 | Invalid response from external service. |
| PE-EXT-005 | 500 | External service configuration error. |

**API (PE-API-001 to PE-API-005)**

| Code | HTTP | Message |
|------|------|---------|
| PE-API-001 | 404 | Endpoint not found. |
| PE-API-002 | 405 | HTTP method not allowed. |
| PE-API-003 | 406 | Not acceptable. |
| PE-API-004 | 400 | Invalid API version. |
| PE-API-005 | 400 | Malformed request body. |

**DB (PE-DB-001 to PE-DB-006)**

| Code | HTTP | Message |
|------|------|---------|
| PE-DB-001 | 409 | Duplicate record. |
| PE-DB-002 | 400 | Invalid ID format. |
| PE-DB-003 | 400 | Schema validation failed. |
| PE-DB-004 | 404 | Document not found. |
| PE-DB-005 | 409 | Concurrent modification conflict. |
| PE-DB-006 | 503 | Database connection error. |

**Phase 2 TODO Checklist:**
- [x] Create `libs/errors/src/codes/auth.codes.ts` (20 codes)
- [x] Create `libs/errors/src/codes/user.codes.ts` (8 codes)
- [x] Create `libs/errors/src/codes/validation.codes.ts` (8 codes)
- [x] Create `libs/errors/src/codes/rate-limit.codes.ts` (5 codes)
- [x] Create `libs/errors/src/codes/payment.codes.ts` (8 codes)
- [x] Create `libs/errors/src/codes/social.codes.ts` (8 codes)
- [x] Create `libs/errors/src/codes/engagement.codes.ts` (8 codes)
- [x] Create `libs/errors/src/codes/file.codes.ts` (6 codes)
- [x] Create `libs/errors/src/codes/websocket.codes.ts` (5 codes)
- [x] Create `libs/errors/src/codes/queue.codes.ts` (6 codes)
- [x] Create `libs/errors/src/codes/internal.codes.ts` (5 codes)
- [x] Create `libs/errors/src/codes/external.codes.ts` (5 codes)
- [x] Create `libs/errors/src/codes/api.codes.ts` (5 codes)
- [x] Create `libs/errors/src/codes/database.codes.ts` (6 codes)
- [x] Create `libs/errors/src/codes/index.ts` aggregator + lookup functions
- [x] Create `libs/errors/src/codes/constants.ts` — ERROR_CODES namespace (added per requirement)
- [x] Verify TypeScript compilation

---

## PHASE 3: Exception Filters & Error Handlers

**Goal:** Build the `GlobalExceptionFilter`, specialized handlers, and error sanitization.

### Task 3.1: Error Response Builder

**File: `libs/errors/src/utils/error-response.builder.ts`**

Single function that builds `PostEngageErrorResponse` from any exception:

```typescript
export class ErrorResponseBuilder {
  static build(params: {
    exception: unknown;
    requestId: string;
    path: string;
    service?: string;
    startTime?: number;
  }): { response: PostEngageErrorResponse; httpStatus: number } {
    // 1. If PostEngageException → use its errorCode, type, details, param
    // 2. If HttpException → map status to ErrorType + generic code
    // 3. If Mongoose error → delegate to MongooseErrorHandler
    // 4. If ThrottlerException → PE-RATE-001
    // 5. Unknown → PE-INT-001
    //
    // Always returns { response, httpStatus }
  }
}
```

### Task 3.2: Global Exception Filter (HTTP)

**File: `libs/errors/src/filters/global-exception.filter.ts`**

```typescript
@Catch()
export class GlobalExceptionFilter implements ExceptionFilter {
  private readonly logger = new Logger('ExceptionFilter');

  catch(exception: unknown, host: ArgumentsHost): void {
    const ctx = host.switchToHttp();
    const request = ctx.getRequest();
    const response = ctx.getResponse();

    const requestId = request.id ?? request.headers['x-request-id'] ?? generateRequestId();
    const startTime = request.startTime ?? Date.now();

    // Build unified response
    const { response: errorResponse, httpStatus } = ErrorResponseBuilder.build({
      exception,
      requestId,
      path: request.url,
      service: process.env.SERVICE_NAME,
      startTime,
    });

    // Log — 4xx = warn, 5xx = error
    if (httpStatus >= 500) {
      this.logger.error(errorResponse.error.message, {
        ...errorResponse,
        stack: exception instanceof Error ? exception.stack : undefined,
      });
    } else {
      this.logger.warn(errorResponse.error.message, errorResponse);
    }

    // Strip debug info in production
    if (process.env.NODE_ENV === 'production') {
      delete errorResponse.debug;
    }

    response.status(httpStatus).json(errorResponse);
  }
}
```

### Task 3.3: WebSocket Exception Filter

**File: `libs/errors/src/filters/ws-exception.filter.ts`**

For Pulse service — catches errors in Socket.io gateway handlers:

```typescript
@Catch()
export class WsExceptionFilter extends BaseWsExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost): void {
    const client = host.switchToWs().getClient<Socket>();
    // Build same PostEngageErrorResponse shape
    // Emit to client: client.emit('error', errorResponse)
    // Log the error
  }
}
```

### Task 3.4: RPC Exception Filter

**File: `libs/errors/src/filters/rpc-exception.filter.ts`**

For Worker/Scheduler BullMQ processors:

```typescript
@Catch()
export class RpcExceptionFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost): Observable<never> {
    // Build PostEngageErrorResponse shape
    // Log with queue/job context
    // Return throwError(() => new RpcException(errorResponse))
  }
}
```

### Task 3.5: Mongoose Error Handler

**File: `libs/errors/src/handlers/mongoose-error.handler.ts`**

Maps Mongoose-specific errors to `PostEngageException`:

```typescript
export class MongooseErrorHandler {
  static handle(error: unknown): PostEngageException | null {
    // MongoServerError code 11000 → PE-DB-001 (duplicate key)
    // CastError → PE-DB-002 (invalid ObjectId)
    // Mongoose ValidationError → PE-DB-003 (schema validation)
    // DocumentNotFoundError → PE-DB-004
    // VersionError → PE-DB-005 (optimistic locking conflict)
    // MongoNetworkError → PE-DB-006 (connection error)
    // Returns null if not a Mongoose error
  }
}
```

### Task 3.6: Throttler Error Handler

**File: `libs/errors/src/handlers/throttler-error.handler.ts`**

```typescript
export class ThrottlerErrorHandler {
  static handle(error: unknown): PostEngageException | null {
    // ThrottlerException → PE-RATE-001 with retry-after header info
  }
}
```

### Task 3.7: Validation Pipe Error Handler

**File: `libs/errors/src/handlers/validation-pipe-error.handler.ts`**

Converts class-validator `ValidationError[]` to `ErrorDetail[]`:

```typescript
export class ValidationPipeErrorHandler {
  static formatErrors(errors: ValidationError[]): ErrorDetail[] {
    // Recursively flatten nested validation errors
    // Each becomes: { field: "email", message: "must be valid email", code: "invalid_format" }
  }
}
```

### Task 3.8: Error Sanitizer

**File: `libs/errors/src/sanitizers/error-sanitizer.ts`**

```typescript
export class ErrorSanitizer {
  // Build debug info (only non-production)
  static buildDebugInfo(error: unknown, service?: string, startTime?: number): DebugInfo | undefined

  // Strip file paths, connection strings from messages
  static sanitizeMessage(message: string): string

  // Redact sensitive fields: password, token, secret, apiKey, authorization, cookie, creditCard, ssn
  static redactSensitiveFields(obj: Record<string, unknown>): Record<string, unknown>
}
```

**Phase 3 TODO Checklist:**
- [x] Create `libs/errors/src/utils/error-response.builder.ts`
- [x] Create `libs/errors/src/utils/error-message.utils.ts`
- [x] Create `libs/errors/src/utils/http-status.mapper.ts`
- [x] Create `libs/errors/src/filters/global-exception.filter.ts`
- [x] Create `libs/errors/src/filters/ws-exception.filter.ts`
- [x] Create `libs/errors/src/filters/rpc-exception.filter.ts` (skipped — BullMQ uses direct error handling, not @nestjs/microservices RPC)
- [x] Create `libs/errors/src/handlers/mongoose-error.handler.ts`
- [x] Create `libs/errors/src/handlers/throttler-error.handler.ts`
- [x] Create `libs/errors/src/handlers/validation-pipe-error.handler.ts`
- [x] Create `libs/errors/src/sanitizers/error-sanitizer.ts`
- [x] Create `libs/errors/src/sanitizers/sensitive-fields.ts`
- [x] Verify TypeScript compilation

---

## PHASE 4: Integration — Wire Everything Up

**Goal:** Connect the new error library to all 4 services, replace old error handling.

### Task 4.1: Update All main.ts Files

All 4 services (Gatekeeper, Pulse, Scheduler, Worker) need:

```typescript
import { GlobalExceptionFilter } from '@app/errors';

// Process-level handlers (BEFORE NestFactory.create)
process.on('uncaughtException', (error: Error) => {
  logger.error('Uncaught Exception', { error: error.message, stack: error.stack, service: 'SERVICE_NAME' });
  process.exit(1);
});

process.on('unhandledRejection', (reason: unknown) => {
  logger.error('Unhandled Rejection', { reason, service: 'SERVICE_NAME' });
  process.exit(1);
});

// In bootstrap():
app.useGlobalFilters(new GlobalExceptionFilter());
app.enableShutdownHooks();
```

### Task 4.2: Update ValidationPipe

Replace the custom `exceptionFactory` to use `ValidationException` from `@app/errors`:

```typescript
app.useGlobalPipes(new ValidationPipe({
  transform: true,
  whitelist: true,
  forbidNonWhitelisted: true,
  exceptionFactory: (errors) => {
    const details = ValidationPipeErrorHandler.formatErrors(errors);
    return new ValidationException(details);
  },
}));
```

### Task 4.3: Update ResponseInterceptor

The interceptor should ONLY handle success responses — remove any `catchError` or error formatting:

- Set `request.startTime = Date.now()` at the start
- Set `X-Request-ID`, `X-Response-Time` headers
- Wrap success data in `{ success: true, data, meta }` shape
- NO error handling — that's the filter's job

### Task 4.4: Register WsExceptionFilter in Pulse

```typescript
// apps/pulse/src/pulse.gateway.ts
@UseFilters(new WsExceptionFilter())
@WebSocketGateway()
export class PulseGateway { ... }
```

### Task 4.5: Register RpcExceptionFilter in Worker/Scheduler

```typescript
// In Worker and Scheduler main.ts or module-level
app.useGlobalFilters(new RpcExceptionFilter());
```

### Task 4.6: Update libs/common Re-exports

The `libs/common/src/filters/global-exception.filter.ts` currently re-exports from `@app/errors`. This is fine — keep the re-export so existing imports don't break, but the actual implementation lives in `@app/errors`.

### Task 4.7: Clean Up Old Error Code References

**NOTE — TODO:** The old error system in `libs/common/src/errors/` (with `error-codes/`, `messages/`, `platforms/`, `error-mapper.ts`, `error-resolver.ts`) needs to be deleted. All 40+ `PostEngageException` throws in `auth.service.ts` and other files need to be updated to use the new PE-xxx codes from `@app/errors`.

<!-- TODO: Delete libs/common/src/errors/ folder entirely after migration -->
<!-- TODO: Delete or refactor libs/common/src/utils/error.factory.ts — replaced by PostEngageException -->
<!-- TODO: Delete or refactor libs/common/src/utils/parse-error-code.ts — PostgreSQL codes not applicable (we use MongoDB) -->
<!-- TODO: Update all throw new PostEngageException(ERROR_CODES.AUTH_XXX.error_code) to use new PE-AUTH-xxx codes -->

**Phase 4 TODO Checklist:**
- [x] Update `apps/gatekeeper/src/main.ts` — new filter + process handlers (already configured)
- [x] Update `apps/pulse/src/main.ts` — new filter + process handlers (already configured)
- [x] Update `apps/scheduler/src/main.ts` — new filter + process handlers (already configured)
- [x] Update `apps/worker/src/main.ts` — new filter + process handlers (already configured)
- [x] Update `ValidationPipe` to use `ValidationException`
- [x] Update `ResponseInterceptor` — success only, fixed TS cast issue
- [x] Register `WsExceptionFilter` in Pulse gateway
- [x] Register `RpcExceptionFilter` in Worker/Scheduler (skipped — BullMQ uses direct error handling)
- [x] Update `libs/common/src/filters/global-exception.filter.ts` re-export
- [x] Migrate all `PostEngageException` throws to new error codes (20+ service files migrated)
- [x] Replace all hardcoded PE-xxx strings with ERROR_CODES constants
- [x] Delete `libs/common/src/errors/` folder (old system) — already deleted by user
- [x] Deprecate `libs/common/src/utils/error.factory.ts` — removed from barrel exports
- [x] Delete `libs/common/src/utils/parse-error-code.ts` — already removed
- [x] Verify TypeScript compilation — zero errors confirmed
- [ ] Manual test: trigger each error type, verify response shape

---

## PHASE 5: Observability (Sentry + Process Handlers)

**Goal:** Production error monitoring — every 5xx gets tracked.

### Task 5.1: Install & Configure Sentry

```bash
npm install @sentry/nestjs @sentry/node
```

### Task 5.2: Sentry Init in Each Service

Before `NestFactory.create()`:

```typescript
import * as Sentry from '@sentry/node';

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
  release: process.env.APP_VERSION,
  tracesSampleRate: process.env.NODE_ENV === 'production' ? 0.2 : 1.0,
  beforeSend(event) {
    // Strip PII
    // Filter out 401, 404, 429 (expected errors)
    return event;
  },
  ignoreErrors: ['UnauthorizedException', 'ThrottlerException', 'NotFoundException'],
});
```

### Task 5.3: Integrate Sentry with GlobalExceptionFilter

Only report 5xx and unexpected errors to Sentry:

```typescript
if (httpStatus >= 500) {
  Sentry.captureException(exception, {
    tags: { service, error_code: errorResponse.error.code },
    extra: { request_id: requestId, path: request.url },
  });
}
```

**Phase 5 TODO Checklist:**
- [x] Install `@sentry/nestjs`, `@sentry/node` (v10.40.0)
- [x] Create `libs/errors/src/sentry/sentry.config.ts` — centralized SentryConfig
- [x] Initialize Sentry in all 4 main.ts (BEFORE NestFactory.create)
- [x] Add Sentry reporting to GlobalExceptionFilter
- [x] Add Sentry to WsExceptionFilter
- [x] Configure `beforeSend` for PII stripping (passwords, tokens, cookies, etc.)
- [x] Configure ignored errors (401, 403, 404, 429 + NestJS exception classes)
- [x] Add `SENTRY_DSN` + `APP_VERSION` to EnvironmentService
- [ ] Test: trigger 500, verify in Sentry dashboard

---

## PHASE 6: Resilience (Circuit Breaker, Retry)

**Goal:** External API failures don't cascade.

### Task 6.1: Circuit Breaker

```typescript
class CircuitBreaker {
  // States: CLOSED → OPEN → HALF_OPEN → CLOSED
  // Tracks failure count per external service
  // Opens circuit after N failures
  // Resets after timeout
  // HALF_OPEN: allow 1 request through to test
}
```

Apply to: Instagram API, Facebook API, Razorpay, any external HTTP call.

### Task 6.2: Retry with Exponential Backoff

```typescript
async function withRetry<T>(fn: () => Promise<T>, options: RetryOptions): Promise<T> {
  // Exponential backoff: 1s, 2s, 4s, 8s...
  // Only retry on: ECONNRESET, ETIMEDOUT, 502, 503, 504
  // Never retry: 400, 401, 403, 404, 409, 422
}
```

### Task 6.3: Graceful Shutdown

```typescript
process.on('SIGTERM', async () => {
  logger.log('SIGTERM received. Graceful shutdown...');
  // 1. Stop accepting new requests
  // 2. Wait for in-flight requests (30s)
  // 3. Close DB/Redis connections
  // 4. Flush Sentry
  await Sentry.flush(5000);
  process.exit(0);
});
```

**Phase 6 TODO Checklist:**
- [x] Create `libs/errors/src/resilience/circuit-breaker.ts` — CLOSED → OPEN → HALF_OPEN state machine
- [x] Create `libs/errors/src/resilience/retry.ts` — exponential backoff + jitter + Retry-After support
- [x] Create `libs/errors/src/resilience/graceful-shutdown.ts` — SIGTERM/SIGINT + Sentry flush
- [x] Replace `app.enableShutdownHooks()` with `GracefulShutdown.register()` in all 4 services
- [x] Apply circuit breaker to Instagram/Facebook API service calls — `ResilientHttpService` wraps all 9 Instagram services
- [x] Apply retry to transient failure scenarios — Razorpay adapter wrapped with `withRetry`, Instagram via `ResilientHttpService`
- [x] Create `ResilientHttpService` in `libs/common/src/http/` — extends HttpService with CircuitBreaker + withRetry
- [x] Create `HttpModule.forResilient()` factory for easy module registration
- [ ] Test: kill MongoDB, verify circuit breaker
- [ ] Test: graceful shutdown with in-flight requests

---

## Implementation Order & Status

```
Day 1:  Phase 1 (Core library setup, types, exception classes)        ✅ DONE
Day 2:  Phase 2 (All 14 error code files + aggregator + constants)    ✅ DONE
Day 3:  Phase 3 (Filters, handlers, sanitizers)                       ✅ DONE
Day 4:  Phase 4 (Wire up to all services, migrate throws)             ✅ DONE
Day 5:  Phase 5 (Sentry integration)                                  ✅ DONE
Day 6:  Phase 6 (Circuit breaker, retry, graceful shutdown)           ✅ DONE
Day 7:  Testing & cleanup                                             ⬜ PENDING
```

### Completed on March 1, 2026:
- **Phases 1–4 fully implemented** — zero TypeScript compilation errors
- **103 error codes** across 14 domains with type-safe `ERROR_CODES` constants
- **20+ service files migrated** from hardcoded strings to `ERROR_CODES.DOMAIN.CODE` pattern
- **All imports updated** — services use `@app/errors`, `@app/common` re-exports for backward compat
- **WsExceptionFilter** applied to Pulse gateway
- **RpcExceptionFilter** skipped — BullMQ uses direct error handling, not `@nestjs/microservices`
- **Phase 5 (Sentry)** — `SentryConfig` class with PII stripping, ignored client errors, env-aware sampling
- **Phase 6 (Resilience)** — `CircuitBreaker`, `withRetry`, `GracefulShutdown` utilities
- **All 4 main.ts** updated with `SentryConfig.init()` + `GracefulShutdown.register()`
- **EnvironmentService** updated with `sentryDsn` + `appVersion` getters
- **ResilientHttpService** — wraps HttpService with circuit breaker + retry for all 9 Instagram + 1 Facebook API services
- **Razorpay adapter** — key payment calls wrapped with `withRetry` (createOrder, verifyPayment, capturePayment, getTransactionDetails)

---

## Files Summary

### New Files (35+)

| File | Phase | Purpose |
|------|-------|---------|
| `libs/errors/tsconfig.lib.json` | 1 | Library TypeScript config |
| `libs/errors/src/errors.module.ts` | 1 | NestJS module |
| `libs/errors/src/index.ts` | 1 | Barrel exports |
| `libs/errors/src/types/error-response.types.ts` | 1 | Response shape types |
| `libs/errors/src/types/error-code.types.ts` | 1 | ErrorType, ErrorDomain enums |
| `libs/errors/src/types/error-definition.types.ts` | 1 | ErrorCodeDefinition interface |
| `libs/errors/src/core/postengage.exception.ts` | 1 | Main exception class |
| `libs/errors/src/core/validation.exception.ts` | 1 | Validation exception |
| `libs/errors/src/core/platform.exception.ts` | 1 | External platform exception |
| `libs/errors/src/codes/auth.codes.ts` | 2 | 20 auth error codes |
| `libs/errors/src/codes/user.codes.ts` | 2 | 8 user error codes |
| `libs/errors/src/codes/validation.codes.ts` | 2 | 8 validation error codes |
| `libs/errors/src/codes/rate-limit.codes.ts` | 2 | 5 rate limit codes |
| `libs/errors/src/codes/payment.codes.ts` | 2 | 8 payment codes |
| `libs/errors/src/codes/social.codes.ts` | 2 | 8 social media codes |
| `libs/errors/src/codes/engagement.codes.ts` | 2 | 8 engagement codes |
| `libs/errors/src/codes/file.codes.ts` | 2 | 6 file upload codes |
| `libs/errors/src/codes/websocket.codes.ts` | 2 | 5 WebSocket codes |
| `libs/errors/src/codes/queue.codes.ts` | 2 | 6 queue job codes |
| `libs/errors/src/codes/internal.codes.ts` | 2 | 5 internal codes |
| `libs/errors/src/codes/external.codes.ts` | 2 | 5 external service codes |
| `libs/errors/src/codes/api.codes.ts` | 2 | 5 API contract codes |
| `libs/errors/src/codes/database.codes.ts` | 2 | 6 database codes |
| `libs/errors/src/codes/index.ts` | 2 | Aggregator + lookup |
| `libs/errors/src/filters/global-exception.filter.ts` | 3 | HTTP exception filter |
| `libs/errors/src/filters/ws-exception.filter.ts` | 3 | WebSocket filter |
| `libs/errors/src/filters/rpc-exception.filter.ts` | 3 | RPC/Queue filter |
| `libs/errors/src/handlers/mongoose-error.handler.ts` | 3 | Mongoose → PostEngageException |
| `libs/errors/src/handlers/throttler-error.handler.ts` | 3 | Throttler → PE-RATE-001 |
| `libs/errors/src/handlers/validation-pipe-error.handler.ts` | 3 | class-validator → ErrorDetail[] |
| `libs/errors/src/sanitizers/error-sanitizer.ts` | 3 | Stack trace stripping, PII redaction |
| `libs/errors/src/sanitizers/sensitive-fields.ts` | 3 | Sensitive field list |
| `libs/errors/src/utils/error-response.builder.ts` | 3 | Builds unified response |
| `libs/errors/src/utils/error-message.utils.ts` | 3 | getErrorMessage helpers |
| `libs/errors/src/utils/http-status.mapper.ts` | 3 | ErrorType → HTTP status |

### Files to Delete (after migration)

<!-- TODO: These files contain the old error system and should be deleted once all code is migrated -->

| File | Reason |
|------|--------|
| `libs/common/src/errors/` (entire folder) | Old error code system, messages, platform mappings |
| `libs/common/src/utils/error.factory.ts` | Replaced by PostEngageException |
| `libs/common/src/utils/parse-error-code.ts` | PostgreSQL error codes — we use MongoDB |

### Files to Modify

| File | Phase | Changes |
|------|-------|---------|
| `apps/gatekeeper/src/main.ts` | 4, 5 | New filter, Sentry, process handlers |
| `apps/pulse/src/main.ts` | 4, 5 | New filter, WsFilter, Sentry |
| `apps/scheduler/src/main.ts` | 4, 5 | New filter, RpcFilter, Sentry |
| `apps/worker/src/main.ts` | 4, 5 | New filter, RpcFilter, Sentry |
| `libs/common/src/interceptors/response.interceptor.ts` | 4 | Remove error handling, success only |
| `libs/common/src/pipes/validation.pipe.ts` | 4 | Use ValidationException |
| `libs/common/src/filters/global-exception.filter.ts` | 4 | Update re-export |
| `apps/gatekeeper/src/modules/auth/auth.service.ts` | 4 | Migrate 40+ throws to new codes |
| All service files using PostEngageException | 4 | Update import and error codes |

---

## References

- [Stripe API Error Handling](https://docs.stripe.com/error-handling)
- [GitHub REST API Error Format](https://docs.github.com/en/rest/using-the-rest-api/troubleshooting-the-rest-api)
- [Twilio Error Dictionary](https://www.twilio.com/docs/api/errors)
- [Cloudflare API Response Format](https://developers.cloudflare.com/api/)
- [NestJS Exception Filters](https://docs.nestjs.com/exception-filters)
- [NestJS Libraries](https://docs.nestjs.com/cli/libraries)
- [NestJS Microservices Exception Filters](https://docs.nestjs.com/microservices/exception-filters)
