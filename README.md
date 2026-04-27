# Authentication Service - Complete Technical Specification

 
**Service Type:** Core Security Microservice (IAM Layer)  
**Domain:** Identity & Access Management  
**Port:** 3001

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Architecture Overview](#2-architecture-overview)
3. [API Endpoints](#3-api-endpoints)
4. [Inter-Service Communication](#4-inter-service-communication)
5. [Database Design](#5-database-design)
6. [Authentication Flows](#6-authentication-flows)
7. [Security Design](#7-security-design)
8. [Event System](#8-event-system)
9. [Caching Strategy](#9-caching-strategy)
10. [Rate Limiting](#10-rate-limiting)
11. [Deployment Architecture](#11-deployment-architecture)
12. [Monitoring & Observability](#12-monitoring--observability)
13. [Environment Variables](#13-environment-variables)
14. [Error Codes](#14-error-codes)
15. [API Response Format](#15-api-response-format)

---

## 1. Executive Summary

The Authentication Service is a stateless microservice responsible for all user identity and access management operations. It provides secure user registration, login/logout functionality, JWT-based token management, OTP verification, and inter-service authentication.

### 1.1 Service Characteristics

| Attribute | Value |
|-----------|-------|
| Architecture Style | Stateless Microservice |
| Authentication Type | Token-based (JWT) |
| Event Integration | Kafka |
| API Gateway | Kong (protected) |
| Database | PostgreSQL with Redis for sessions |
| Communication | REST (External) + gRPC (Internal) |

---

## 2. Architecture Overview

### 2.1 System Architecture

```
                           ┌─────────────────────────────────────┐
                           │         External Clients            │
                           │   (Web, Mobile, Partner Services)    │
                           └─────────────────┬───────────────────┘
                                             │
                           ┌─────────────────▼───────────────────┐
                           │          Kong API Gateway           │
                           │     (Port 80/443 - Public)          │
                           └─────────────────┬───────────────────┘
                                             │
                           ┌─────────────────▼───────────────────┐
                           │    Authentication Service           │
                           │         (Port 3001)               │
                           └──────┬───────────┬────────────┬───┘
                                  │           │            │
              ┌───────────────────┼───────────┼────────────┼───────────────────┐
              │                   │           │            │                   │
              ▼                   ▼           ▼            ▼                   ▼
        ┌──────────┐       ┌─────────┐  ┌─────────┐  ┌──────────┐      ┌──────────┐
        │  PostgreSQL   │       │  Redis  │  │  Kafka  │  │   SMTP   │      │  gRPC    │
        │ Primary  │       │Session │  │ Events │  │ Server  │      │ Services │
        └──────────┘       └────────┘  └─────────┘  └──────────┘      └──────────┘
```

### 2.2 Service Boundaries

**Inside Auth Service (Responsibilities):**
- Email, Phone, and Password management
- Role assignment: AFFILIATE, BUSINESS, ADMIN, SUPER_ADMIN
- JWT and Refresh Token generation
- Session control and tracking
- OTP generation and verification
- Login security and rate limiting
- Inter-service token validation
- User identity lookup

**Outside Auth Service (Not Responsibilities):**
- Affiliate profile data (Affiliate Service)
- Business profile data (Business Service)
- KYC documents (KYC Service)
- Wallet and payments (Payment Service)
- Permissions (Authorization Service)

### 2.3 Inter-Service Communication Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Inter-Service Communication                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌──────────────┐       gRPC (Internal)       ┌──────────────┐       │
│  │   Consumer  │◄────────────────────────────►│     Auth     │       │
│  │   Service   │                              │    Service   │       │
│  └──────────────┘                             └──────────────┘       │
│        │                                                │              │
│        │            Kafka Events (Async)                │              │
│        └────────────────────────────────────────────────┘              │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 3. API Endpoints

### 3.1 External API Endpoints (Public)

All endpoints are prefixed with `/auth` and accessible via API Gateway.

#### 3.1.1 User Registration

| Attribute | Value |
|-----------|-------|
| Endpoint | `POST /auth/register` |
| Rate Limit | 3 requests/minute |
| Auth Required | No |

**Request:**
```json
{
  "full_name": "John Doe",
  "email": "john@email.com",
  "phone": "+8801xxxxxxx",
  "password": "SecurePass123",
  "role": "AFFILIATE"
}
```

**Response (Success - 201):**
```json
{
  "success": true,
  "message": "User registered successfully",
  "data": {
    "user_id": "550e8400-e29b-41d4-a716-446655440000",
    "role": "AFFILIATE",
    "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
  }
}
```

**Response (Error - 400):**
```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid email format",
    "details": [
      {
        "field": "email",
        "message": "Invalid email format"
      }
    ]
  }
}
```

---

#### 3.1.2 User Login

| Attribute | Value |
|-----------|-------|
| Endpoint | `POST /auth/login` |
| Rate Limit | 5 requests/minute |
| Auth Required | No |

**Request:**
```json
{
  "email_or_phone": "john@email.com",
  "password": "SecurePass123",
  "device_id": "device_abc_123"
}
```

**Response (Success - 200):**
```json
{
  "success": true,
  "data": {
    "user_id": "550e8400-e29b-41d4-a716-446655440000",
    "role": "AFFILIATE",
    "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "session_id": "550e8400-e29b-41d4-a716-446655440001",
    "requires_otp": false
  }
}
```

**Response (Requires OTP - 200):**
```json
{
  "success": true,
  "data": {
    "requires_otp": true,
    "session_id": "550e8400-e29b-41d4-a716-446655440001",
    "message": "OTP sent to your email"
  }
}
```

---

#### 3.1.3 Token Refresh

| Attribute | Value |
|-----------|-------|
| Endpoint | `POST /auth/refresh` |
| Rate Limit | 10 requests/minute |
| Auth Required | No |

**Request:**
```json
{
  "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

**Response (Success - 200):**
```json
{
  "success": true,
  "data": {
    "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
  }
}
```

---

#### 3.1.4 User Logout

| Attribute | Value |
|-----------|-------|
| Endpoint | `POST /auth/logout` |
| Rate Limit | 30 requests/minute |
| Auth Required | Yes (JWT) |

**Headers:**
```
Authorization: Bearer <access_token>
```

**Response (Success - 200):**
```json
{
  "success": true,
  "message": "Logged out successfully"
}
```

---

#### 3.1.5 Request OTP

| Attribute | Value |
|-----------|-------|
| Endpoint | `POST /auth/request-otp` |
| Rate Limit | 5 requests/10 minutes |
| Auth Required | No |

**Request:**
```json
{
  "email_or_phone": "john@email.com",
  "type": "LOGIN"
}
```

**Response (Success - 200):**
```json
{
  "success": true,
  "message": "OTP sent to your email"
}
```

---

#### 3.1.6 Verify OTP

| Attribute | Value |
|-----------|-------|
| Endpoint | `POST /auth/verify-otp` |
| Rate Limit | 5 requests/10 minutes |
| Auth Required | No |

**Request:**
```json
{
  "otp_code": "123456",
  "session_id": "550e8400-e29b-41d4-a716-446655440001"
}
```

**Response (Success - 200):**
```json
{
  "success": true,
  "message": "OTP verified successfully",
  "data": {
    "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
  }
}
```

---

#### 3.1.7 Change Password

| Attribute | Value |
|-----------|-------|
| Endpoint | `POST /auth/change-password` |
| Rate Limit | 5 requests/minute |
| Auth Required | Yes (JWT) |

**Request:**
```json
{
  "current_password": "OldPassword123",
  "new_password": "NewPassword456"
}
```

**Response (Success - 200):**
```json
{
  "success": true,
  "message": "Password changed successfully"
}
```

---

#### 3.1.8 Get Current User

| Attribute | Value |
|-----------|-------|
| Endpoint | `GET /auth/me` |
| Rate Limit | 60 requests/minute |
| Auth Required | Yes (JWT) |

**Response (Success - 200):**
```json
{
  "success": true,
  "data": {
    "user_id": "550e8400-e29b-41d4-a716-446655440000",
    "full_name": "John Doe",
    "email": "john@email.com",
    "phone": "+8801xxxxxxx",
    "role": "AFFILIATE",
    "status": "ACTIVE",
    "email_verified": true,
    "phone_verified": false,
    "created_at": "2026-01-01T00:00:00Z"
  }
}
```

---

### 3.2 Internal API Endpoints (gRPC)

Internal endpoints are used for inter-service communication. These are not exposed via API Gateway.

#### 3.2.1 ValidateToken

Validates a JWT token and returns user information.

| Attribute | Value |
|-----------|-------|
| Service | AuthService |
| Method | ValidateToken |
| Port | 3001 |

**Request:**
```protobuf
message ValidateTokenRequest {
  string token = 1;
}
```

**Response:**
```protobuf
message ValidateTokenResponse {
  bool valid = 1;
  string user_id = 2;
  string role = 3;
  string status = 4;
  int64 expires_at = 5;
}
```

---

#### 3.2.2 GetUserById

Retrieves user information by public ID.

| Attribute | Value |
|-----------|-------|
| Service | AuthService |
| Method | GetUserById |
| Port | 3001 |

**Request:**
```protobuf
message GetUserByIdRequest {
  string user_id = 1;
}
```

**Response:**
```protobuf
message GetUserByIdResponse {
  string user_id = 1;
  string email = 2;
  string phone = 3;
  string role = 4;
  string status = 5;
  bool email_verified = 6;
  bool phone_verified = 7;
}
```

---

#### 3.2.3 GetUserByEmail

Retrieves user information by email.

| Attribute | Value |
|-----------|-------|
| Service | AuthService |
| Method | GetUserByEmail |
| Port | 3001 |

**Request:**
```protobuf
message GetUserByEmailRequest {
  string email = 1;
}
```

**Response:** Same as GetUserByIdResponse

---

#### 3.2.4 VerifyCredentials

Verifies user credentials (used by other services for authentication).

| Attribute | Value |
|-----------|-------|
| Service | AuthService |
| Method | VerifyCredentials |
| Port | 3001 |

**Request:**
```protobuf
message VerifyCredentialsRequest {
  string email_or_phone = 1;
  string password = 2;
}
```

**Response:**
```protobuf
message VerifyCredentialsResponse {
  bool valid = 1;
  string user_id = 2;
  string role = 3;
  string status = 4;
}
```

---

#### 3.2.5 CheckPermission

Checks if user has specific permission.

| Attribute | Value |
|-----------|-------|
| Service | AuthService |
| Method | CheckPermission |
| Port | 3001 |

**Request:**
```protobuf
message CheckPermissionRequest {
  string user_id = 1;
  string permission = 2;
}
```

**Response:**
```protobuf
message CheckPermissionResponse {
  bool allowed = 1;
}
```

---

### 3.3 Health Check Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/health` | GET | Liveness probe |
| `/health/ready` | GET | Readiness probe |
| `/metrics` | GET | Prometheus metrics |

**Health Response:**
```json
{
  "status": "healthy",
  "service": "auth-service",
  "version": "1.0.0",
  "timestamp": "2026-04-27T00:00:00Z"
}
```

---

## 4. Inter-Service Communication

### 4.1 Communication Patterns

#### 4.1.1 Synchronous Communication (gRPC)

Used for:
- Token validation
- User lookup
- Permission checking

```
┌─────────────┐                          ┌─────────────┐
│  Service A  │───── gRPC Call ──────►│    Auth    │
│             │◄────── Response ────── │  Service   │
└─────────────┘                          └─────────────┘
```

#### 4.1.2 Asynchronous Communication (Kafka)

Used for:
- User registration events
- Login events
- Password change events
- Security events

```
┌─────────────┐                          ┌─────────────┐
│  Auth      │───── Publish Event ──────►│    Kafka   │
│  Service   │                          │  Broker    │
└─────────────┘                          └──────┬────┘
                                                   │
                              ┌────────────────────┼────────────────────┐
                              │                    │                    │
                              ▼                    ▼                    ▼
                        ┌──────────┐          ┌──────────┐          ┌──────────┐
                        │Affiliate│          │Business │          │Analytics│
                        │Service │          │Service  │          │Service │
                        └────────┘          └────────┘          └────────┘
```

### 4.2 gRPC Service Definition

```protobuf
syntax = "proto3";

package auth;

service AuthService {
  rpc ValidateToken (ValidateTokenRequest) returns (ValidateTokenResponse);
  rpc GetUserById (GetUserByIdRequest) returns (GetUserByIdResponse);
  rpc GetUserByEmail (GetUserByEmailRequest) returns (GetUserByIdResponse);
  rpc VerifyCredentials (VerifyCredentialsRequest) returns (VerifyCredentialsResponse);
  rpc CheckPermission (CheckPermissionRequest) returns (CheckPermissionResponse);
  rpc HealthCheck (HealthCheckRequest) returns (HealthCheckResponse);
}

message ValidateTokenRequest {
  string token = 1;
}

message ValidateTokenResponse {
  bool valid = 1;
  string user_id = 2;
  string role = 3;
  string status = 4;
  int64 expires_at = 5;
}

message GetUserByIdRequest {
  string user_id = 1;
}

message GetUserByIdResponse {
  string user_id = 1;
  string email = 2;
  string phone = 3;
  string role = 4;
  string status = 5;
  bool email_verified = 6;
  bool phone_verified = 7;
}

message GetUserByEmailRequest {
  string email = 1;
}

message VerifyCredentialsRequest {
  string email_or_phone = 1;
  string password = 2;
}

message VerifyCredentialsResponse {
  bool valid = 1;
  string user_id = 2;
  string role = 3;
  string status = 4;
}

message CheckPermissionRequest {
  string user_id = 1;
  string permission = 2;
}

message CheckPermissionResponse {
  bool allowed = 1;
}

message HealthCheckRequest {}

message HealthCheckResponse {
  bool healthy = 1;
  string version = 2;
}
```

### 4.3 HTTP-to-gRPC Mapping

For services that don't support gRPC:

| gRPC Method | HTTP Endpoint | Method |
|------------|-------------|--------|
| ValidateToken | `/auth/internal/validate` | POST |
| GetUserById | `/auth/internal/user/:id` | GET |
| GetUserByEmail | `/auth/internal/user/email` | GET |
| VerifyCredentials | `/auth/internal/verify` | POST |
| CheckPermission | `/auth/internal/permission` | POST |

---

## 5. Database Design

### 5.1 Users Table

```sql
CREATE TABLE users (
  id BIGINT PRIMARY KEY DEFAULT,
  public_id UUID UNIQUE NOT NULL,
  full_name VARCHAR(255) NOT NULL,
  email VARCHAR(255) UNIQUE,
  phone VARCHAR(30) UNIQUE,
  password_hash TEXT NOT NULL,
  role VARCHAR(50) DEFAULT 'AFFILIATE',
  status VARCHAR(50) DEFAULT 'ACTIVE',
  email_verified BOOLEAN DEFAULT FALSE,
  phone_verified BOOLEAN DEFAULT FALSE,
  failed_login_attempts INT DEFAULT 0,
  locked_until TIMESTAMP NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_public_id ON users(public_id);
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_phone ON users(phone);
CREATE INDEX idx_users_status ON users(status);
```

### 5.2 Sessions Table (Redis Primary)

```sql
CREATE TABLE sessions (
  id BIGINT PRIMARY KEY,
  session_id UUID UNIQUE NOT NULL,
  user_public_id UUID NOT NULL,
  refresh_token TEXT,
  device_id VARCHAR(255),
  ip_address VARCHAR(45),
  user_agent TEXT,
  expires_at TIMESTAMP NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_sessions_user_public_id ON sessions(user_public_id);
CREATE INDEX idx_sessions_session_id ON sessions(session_id);
CREATE INDEX idx_sessions_expires_at ON sessions(expires_at);
```

### 5.3 OTP Logs Table

```sql
CREATE TABLE otp_logs (
  id BIGINT PRIMARY KEY,
  otp_id UUID UNIQUE NOT NULL,
  user_public_id UUID NOT NULL,
  otp_code VARCHAR(10) NOT NULL,
  type VARCHAR(50) NOT NULL,
  expires_at TIMESTAMP NOT NULL,
  used BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_otp_logs_user_public_id ON otp_logs(user_public_id);
CREATE INDEX idx_otp_logs_otp_code ON otp_logs(otp_code);
CREATE INDEX idx_otp_logs_expires_at ON otp_logs(expires_at);
```

### 5.4 Refresh Tokens Table

```sql
CREATE TABLE refresh_tokens (
  id BIGINT PRIMARY KEY,
  token_id UUID UNIQUE NOT NULL,
  user_public_id UUID NOT NULL,
  token_hash TEXT NOT NULL,
  revoked BOOLEAN DEFAULT FALSE,
  expires_at TIMESTAMP NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_refresh_tokens_user_public_id ON refresh_tokens(user_public_id);
CREATE INDEX idx_refresh_tokens_token_id ON refresh_tokens(token_id);
CREATE INDEX idx_refresh_tokens_expires_at ON refresh_tokens(expires_at);
```

### 5.5 Rate Limit Table

```sql
CREATE TABLE rate_limits (
  id BIGINT PRIMARY KEY,
  identifier VARCHAR(255) NOT NULL,
  endpoint VARCHAR(100) NOT NULL,
  count INT DEFAULT 0,
  window_start TIMESTAMP NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_rate_limits_identifier_endpoint ON rate_limits(identifier, endpoint);
```

---

## 6. Authentication Flows

### 6.1 Registration Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     User Registration Flow                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Client              Auth Service              Kafka      Database            │
│     │                    │                      │            │              │
│     │── POST /auth/register ──►│                      │            │              │
│     │                    │                      │            │              │
│     │                    │── Validate input ──►│            │              │
│     │                    │◄── Validation ────│            │              │
│     │                    │                      │            │              │
│     │                    │── Check exists ─────────────────►   │          │
│     │                    │◄── Not exists ─────────────────  │              │
│     │                    │                      │            │              │
│     │                    │── Hash password ──►                     │              │
│     │                    │                      │            │              │
│     │                    │── Insert user ─────────────────►            │          │
│     │                    │◄── Created ───────────────── │              │
│     │                    │                      │            │              │
│     │                    │── Generate JWT ──►                                  │
│     │                    │── Generate refresh token ──►                            │
│     │                    │                      │            │              │
│     │                    │── Publish Kafka ────────────────────►             │
│     │                    │   (user_registered)  │            │              │
│     │                    │                      │            │              │
│     │◄─ 201 Created ───│                      │            │              │
│     │                    │                      │            │              │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.2 Login Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Login Flow                                    │
├───────────��─��───────────────────────────────────────────────────────────┤
│                                                                         │
│   Client              Auth Service              Redis      Database            │
│     │                    │                      │            │              │
│     │── POST /auth/login ───►│                      │            │              │
│     │                    │                      │            │              │
│     │                    │── Validate input ──►│            │              │
│     │                    │◄── Valid ──────│            │              │
│     │                    │                      │            │              │
│     │                    │── Get user ─────────────────►            │          │
│     │                    │◄── User data ────────────────│              │
│     │                    │                      │            │              │
│     │                    │── Check status ──►│            │              │
│     │                    │◄── Active ──────│            │              │
│     │                    │                      │            │              │
│     │                    │── Verify password ──►                    │          │
│     │                    │◄── Valid ──────│            │              │
│     │                    │                      │            │              │
│     │                    │── Check failed attempts ──►│            │              │
│     │                    │◄── Count ─────│            │              │
│     │                    │                      │            │              │
│     │                    │── Check rate limit ──►                    │          │
│     │                    │◄── Allowed ────│            │              │
│     │                    │                      │            │              │
│     │                    │                      │            │              │
│     │                    │── Generate JWT ──►                                  │
│     │                    │── Generate refresh token ──►                            │
│     │                    │                      │            │              │
│     │                    │── Store session ──►   │            │              │
│     │                    │                      │            │              │
│     │                    │── Publish Kafka ──────►   │            │              │
│     │                    │   (user_logged_in)  │            │              │
│     │                    │                      │            │              │
│     │◄─ 200 OK ─────────│                      │            │              │
│     │                    │                      │            │              │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.3 Token Refresh Flow

```
┌──────────────────────────────────────────────────────────��─��────────────┐
│                     Token Refresh Flow                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Client              Auth Service              Redis      Database   │
│     │                    │                      │            │         │
│     │── POST /auth/refresh ─►│                      │            │         │
│     │                    │                      │            │         │
│     │                    │── Get token ──►                     │         │
│     │                    │◄── Token data ────────│            │         │
│     │                    │                      │            │         │
│     │                    │── Verify not revoked ──►│          │         │
│     │                    │◄── Not revoked ───────│            │         │
│     │                    │                      │            │         │
│     │                    │── Check expiry ──►                     │         │
│     │                    │◄── Not expired ──────│            │         │
│     │                    │                      │            │         │
│     │                    │── Generate new JWT ──►                             │
│     │                    │                      │            │         │
│     │                    │── Update refresh token ──►│            │         │
│     │                    │                      │            │         │
│     │                    │── Publish Kafka ──────►  │            │         │
│     │                    │   (token_refreshed) │            │         │
│     │                    │                      │            │         │
│     │◄─ 200 OK ─────────│                      │            │         │
│     │                    │                      │            │         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Security Design

### 7.1 Password Security

- **Algorithm:** bcrypt with cost factor 12
- **Minimum Length:** 8 characters
- **Requirements:** Mixed case, numbers, special characters
- **Storage:** Salted hash in password_hash field

### 7.2 Token Security

| Token Type | Expiry | Length | Algorithm |
|-----------|--------|--------|----------|
| Access Token (JWT) | 15 minutes | 2048 bits | RS256 |
| Refresh Token | 30 days | 256 bits | HS256 |

### 7.3 JWT Payload Structure

```json
{
  "sub": "550e8400-e29b-41d4-a716-446655440000",
  "email": "john@email.com",
  "role": "AFFILIATE",
  "iat": 1700000000,
  "exp": 1700009000,
  "iss": "auth-service"
}
```

### 7.4 Security Headers

```
Strict-Transport-Security: max-age=31536000; includeSubDomains
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 1; mode=block
Content-Security-Policy: default-src 'self'
```

---

## 8. Event System

### 8.1 Kafka Events

#### 8.1.1 Event: user_registered

```json
{
  "event_type": "user_registered",
  "event_id": "550e8400-e29b-41d4-a716-446655440000",
  "timestamp": "2026-04-27T00:00:00Z",
  "data": {
    "user_id": "550e8400-e29b-41d4-a716-446655440000",
    "email": "john@email.com",
    "role": "AFFILIATE",
    "ip_address": "192.168.1.1"
  }
}
```

#### 8.1.2 Event: user_logged_in

```json
{
  "event_type": "user_logged_in",
  "event_id": "550e8400-e29b-41d4-a716-446655440001",
  "timestamp": "2026-04-27T00:00:00Z",
  "data": {
    "user_id": "550e8400-e29b-41d4-a716-446655440000",
    "session_id": "550e8400-e29b-41d4-a716-446655440001",
    "ip_address": "192.168.1.1",
    "device_id": "device_abc_123"
  }
}
```

#### 8.1.3 Event: user_logged_out

```json
{
  "event_type": "user_logged_out",
  "event_id": "550e8400-e29b-41d4-a716-446655440002",
  "timestamp": "2026-04-27T00:00:00Z",
  "data": {
    "user_id": "550e8400-e29b-41d4-a716-446655440000",
    "session_id": "550e8400-e29b-41d4-a716-446655440001"
  }
}
```

#### 8.1.4 Event: password_changed

```json
{
  "event_type": "password_changed",
  "event_id": "550e8400-e29b-41d4-a716-446655440003",
  "timestamp": "2026-04-27T00:00:00Z",
  "data": {
    "user_id": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

#### 8.1.5 Event: login_failed

```json
{
  "event_type": "login_failed",
  "event_id": "550e8400-e29b-41d4-a716-446655440004",
  "timestamp": "2026-04-27T00:00:00Z",
  "data": {
    "email_or_phone": "john@email.com",
    "ip_address": "192.168.1.1",
    "reason": "invalid_password"
  }
}
```

### 8.2 Kafka Configuration

```javascript
{
  kafka: {
    brokers: ['localhost:9092'],
    topic: 'auth-events',
    partitions: 6,
    replicationFactor: 3,
    acks: 'all',
    retries: 3
  }
}
```

---

## 9. Caching Strategy

### 9.1 Redis Key Patterns

| Key Pattern | Data Type | TTL | Purpose |
|------------|----------|-----|---------|
| `session:{session_id}` | Hash | 30 days | Session data |
| `otp:{otp_code}` | String | 5 minutes | OTP cache |
| `refresh:{token_id}` | String | 30 days | Refresh token |
| `rate:{ip}:{endpoint}` | Counter | 1 minute | Rate limit |
| `failed:{email}` | Counter | 15 minutes | Failed login |
| `user:{user_id}` | Hash | 1 hour | User cache |

### 9.2 Cache TTL Configuration

| Data Type | Default TTL | Configurable |
|----------|-------------|-------------|
| Session | 30 days | Yes (1-90 days) |
| OTP | 5 minutes | No |
| Refresh Token | 30 days | Yes (1-90 days) |
| Rate Limit | 1 minute | Yes (1-10 minutes) |
| User Cache | 1 hour | Yes (15min-24hr) |

---

## 10. Rate Limiting

### 10.1 Global Rate Limits

| Endpoint | Limit | Window | Auth Required |
|----------|-------|--------|-------------|
| `/auth/register` | 3 | 1 minute | No |
| `/auth/login` | 5 | 1 minute | No |
| `/auth/request-otp` | 5 | 10 minutes | No |
| `/auth/verify-otp` | 5 | 10 minutes | No |
| `/auth/refresh` | 10 | 1 minute | No |
| `/auth/logout` | 30 | 1 minute | Yes |
| `/auth/change-password` | 5 | 1 minute | Yes |

### 10.2 IP-Based Throttling

- **Limit:** 100 requests per minute per IP
- **Block Duration:** 15 minutes after exceeding
- **Sliding Window Algorithm**

### 10.3 Account Lockout

- **Failed Attempts:** 5 consecutive failures
- **Lockout Duration:** 15 minutes
- **Cooldown Period:** 30 minutes between lockouts

---

## 11. Deployment Architecture

### 11.1 Docker Configuration

```dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY . .

EXPOSE 3001

USER node

CMD ["node", "src/index.js"]
```

### 11.2 Docker Compose

```yaml
version: '3.8'

services:
  auth-service:
    build: .
    ports:
      - "3001:3001"
    environment:
      - PORT=3001
      - DB_HOST=mysql
      - REDIS_HOST=redis
      - KAFKA_BROKERS=kafka:9092
    depends_on:
      - mysql
      - redis
      - kafka
    networks:
      - auth-network

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    networks:
      - auth-network

  mysql:
    image: postgres:15-alpine
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_ROOT_PASSWORD=password
      - POSTGRES_DATABASE=auth_db
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - auth-network

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    ports:
      - "9092:9092"
    environment:
      - KAFKA_NODE_ID=1
      - KAFKA_LISTENER_SECURITY_PROTOCOL_MAP=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
      - KAFKA_CONTROLLER_QUORUM_VOTERS=1@kafka:9093
      - KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
    networks:
      - auth-network

networks:
  auth-network:
    driver: bridge

volumes:
  postgres-data:
```

### 11.3 Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-service
  labels:
    app: auth-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: auth-service
  template:
    metadata:
      labels:
        app: auth-service
    spec:
      containers:
      - name: auth-service
        image: auth-service:latest
        ports:
        - containerPort: 3001
        env:
        - name: PORT
          value: "3001"
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: auth-config
              key: db.host
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3001
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 3001
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: auth-service
spec:
  selector:
    app: auth-service
  ports:
  - port: 3001
    targetPort: 3001
  type: ClusterIP
```

---

## 12. Monitoring & Observability

### 12.1 Metrics (Prometheus)

| Metric | Type | Description |
|--------|------|-------------|
| `auth_login_success_total` | Counter | Successful logins |
| `auth_login_failure_total` | Counter | Failed logins |
| `auth_registration_total` | Counter | New registrations |
| `auth_token_refresh_total` | Counter | Token refreshes |
| `auth_active_sessions` | Gauge | Active sessions |
| `auth_request_duration_seconds` | Histogram | Request latency |
| `auth_request_total` | Counter | Total requests |
| `auth_otp_sent_total` | Counter | OTP codes sent |

### 12.2 Logs

- **Format:** JSONstructured
- **Level:** ERROR, WARN, INFO, DEBUG
- **Correlation ID:** `x-correlation-id` header

**Log Entry Example:**
```json
{
  "timestamp": "2026-04-27T00:00:00.000Z",
  "level": "info",
  "message": "User logged in",
  "correlation_id": "550e8400-e29b-41d4-a716-446655440000",
  "user_id": "550e8400-e29b-41d4-a716-446655440000",
  "ip_address": "192.168.1.1",
  "duration_ms": 45
}
```

### 12.3 Distributed Tracing

- **Header:** `x-correlation-id`
- **Format:** UUID
- **Propagation:** All services

---

## 13. Environment Variables

### 13.1 Required Variables

| Variable | Description | Type | Default |
|----------|-------------|------|--------|
| PORT | Service port | number | 3001 |
| NODE_ENV | Environment | string | development |
| DB_HOST | PostgreSQL host | string | localhost |
| DB_PORT | PostgreSQL port | number | 5432 |
| DB_USER | PostgreSQL user | string | postgres |
| DB_PASSWORD | PostgreSQL password | string | - |
| DB_NAME | Database name | string | auth_db |
| REDIS_HOST | Redis host | string | localhost |
| REDIS_PORT | Redis port | number | 6379 |
| JWT_SECRET | JWT signing secret | string | - |
| JWT_REFRESH_SECRET | Refresh token secret | string | - |
| KAFKA_BROKERS | Kafka brokers | string | localhost:9092 |

### 13.2 Optional Variables

| Variable | Description | Type | Default |
|----------|-------------|------|--------|
| JWT_EXPIRES_IN | JWT expiry time | string | 15m |
| JWT_REFRESH_EXPIRES_IN | Refresh token expiry | string | 30d |
| BCRYPT_ROUNDS | bcrypt cost factor | number | 12 |
| OTP_LENGTH | OTP code length | number | 6 |
| OTP_EXPIRE_MINUTES | OTP expiry minutes | number | 5 |
| RATE_LIMIT_WINDOW_MS | Rate limit window | number | 60000 |
| RATE_LIMIT_MAX_REQUESTS | Max requests | number | 100 |
| SESSION_EXPIRE_DAYS | Session expiry days | number | 30 |

---

## 14. Error Codes

### 14.1 HTTP Error Codes

| Code | Error Code | Description |
|------|-----------|-------------|
| 400 | VALIDATION_ERROR | Invalid request data |
| 401 | UNAUTHORIZED | Invalid or missing token |
| 403 | FORBIDDEN | Insufficient permissions |
| 404 | NOT_FOUND | Resource not found |
| 409 | CONFLICT | Resource already exists |
| 422 | UNPROCESSABLE_ENTITY | Validation failed |
| 429 | RATE_LIMIT_EXCEEDED | Too many requests |
| 500 | INTERNAL_ERROR | Server error |
| 503 | SERVICE_UNAVAILABLE | Service unavailable |

### 14.2 Application Error Codes

| Code | Description |
|------|-------------|
| INVALID_EMAIL | Invalid email format |
| INVALID_PASSWORD | Password too weak |
| USER_EXISTS | User already exists |
| USER_NOT_FOUND | User not found |
| INVALID_CREDENTIALS | Wrong password |
| ACCOUNT_SUSPENDED | Account is suspended |
| ACCOUNT_LOCKED | Account is locked |
| TOKEN_EXPIRED | Token has expired |
| TOKEN_INVALID | Token is invalid |
| SESSION_EXPIRED | Session has expired |
| OTP_EXPIRED | OTP has expired |
| OTP_INVALID | OTP is invalid |
| RATE_LIMITED | Too many attempts |

---

## 15. API Response Format

### 15.1 Success Response

```json
{
  "success": true,
  "message": "Operation completed successfully",
  "data": {
    // Response data
  },
  "timestamp": "2026-04-27T00:00:00Z"
}
```

### 15.2 Error Response

```json
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "Error message",
    "details": [
      {
        "field": "field_name",
        "message": "Field-specific error"
      }
    ]
  },
  "timestamp": "2026-04-27T00:00:00Z"
}
```

### 15.3 Pagination Response

```json
{
  "success": true,
  "data": [
    // Array of items
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 100,
    "total_pages": 5
  },
  "timestamp": "2026-04-27T00:00:00Z"
}
```

---

## 16. Acceptance Criteria

1. ✅ User can register with email/phone and password
2. ✅ User can login and receive JWT access token
3. ✅ User can refresh token before expiry
4. ✅ User can logout and invalidate session
5. ✅ OTP can be sent and verified
6. ✅ Rate limiting prevents abuse
7. ✅ Failed login attempts are tracked
8. ✅ All events are published to Kafka
9. ✅ Sessions are stored in Redis
10. ✅ All API responses follow standard format
11. ✅ Internal services can validate tokens via gRPC
12. ✅ User lookup available for other services

---

