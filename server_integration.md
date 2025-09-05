# Partner Server API Integration Guide (v1)

This guide describes the REST API endpoints available for partner server integrations. The API provides access to experiences, user data, and session information for your organization.

## Overview

Our server API enables partners to:
- Query available experiences assigned to your organization
- Retrieve user information and session history
- Access detailed session transcripts and analytics

All API endpoints are served from `https://partnerapi.persona-ai.ai/v1` and require JWT-based authentication.

## Authentication

### Token Format

All requests must include a Bearer token in the Authorization header:

```http
Authorization: Bearer <JWT>
```

### JWT Structure

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

### Required Claims

| Claim | Type | Description | Example |
|-------|------|-------------|---------|
| `iss` | string | Issuer - Your partner identifier | `"partner_acme_corp"` |
| `sub` | string | Subject - API client identifier | `"api_client_12345"` |
| `aud` | string | Audience - Must be `"api.persona-ai.ai"` | `"api.persona-ai.ai"` |
| `iat` | number | Issued at - Unix timestamp | `1704067200` |
| `exp` | number | Expiration - Unix timestamp | `1704070800` |
| `scope` | array | API scopes granted | `["experiences:read", "users:read", "sessions:read"]` |

## Base URL

```
https://api.persona-ai.ai/v1
```

## API Endpoints

### 1. List Available Experiences

Retrieve all experiences available to your organization.

#### Request

```http
GET /experiences
```

#### Query Parameters

| Parameter | Type | Required | Description | Default |
|-----------|------|----------|-------------|---------|
| `page` | integer | No | Page number for pagination | `1` |
| `limit` | integer | No | Number of results per page (max 100) | `20` |
| `status` | string | No | Filter by status (`active`, `inactive`, `beta`) | - |

#### Response

```json
{
  "success": true,
  "data": {
    "experiences": [
      {
        "id": "exp_abc123def456",
        "name": "Customer Service Assistant",
        "description": "AI-powered customer service agent with natural conversation capabilities",
        "status": "active",
        "version": "2.1.0",
        "capabilities": ["chat", "voice", "sentiment_analysis"],
        "languages": ["en", "es", "fr"],
        "created_at": "2024-01-01T00:00:00Z",
        "updated_at": "2024-01-15T12:00:00Z"
      }
    ],
    "pagination": {
      "page": 1,
      "limit": 20,
      "total_pages": 3,
      "total_items": 45
    }
  },
  "timestamp": "2024-01-25T10:00:00Z"
}
```

### 2. List Users

Retrieve a list of all user IDs associated with your organization.

#### Request

```http
GET /users
```

#### Query Parameters

| Parameter | Type | Required | Description | Default |
|-----------|------|----------|-------------|---------|
| `page` | integer | No | Page number for pagination | `1` |
| `limit` | integer | No | Number of results per page (max 1000) | `100` |
| `created_after` | string | No | ISO 8601 date to filter users created after | - |
| `created_before` | string | No | ISO 8601 date to filter users created before | - |

#### Response

```json
{
  "success": true,
  "data": {
    "user_ids": [
      "user_001a2b3c4d5e",
      "user_002f6g7h8i9j",
      "user_003k0l1m2n3o",
      "user_004p4q5r6s7t",
      "user_005u8v9w0x1y"
    ],
    "pagination": {
      "page": 1,
      "limit": 100,
      "total_pages": 15,
      "total_items": 1485
    }
  },
  "timestamp": "2024-01-25T10:00:00Z"
}
```

### 3. Get User Details

Retrieve detailed information about a specific user.

#### Request

```http
GET /users/{user_id}
```

#### Path Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `user_id` | string | Yes | The unique identifier of the user |

#### Response

```json
{
  "success": true,
  "data": {
    "user_id": "user_001a2b3c4d5e",
    "name": "John Doe",
    "email": "john.doe@example.com",
    "image_url": "https://cdn.persona-ai.ai/avatars/user_001a2b3c4d5e.jpg",
    "status": "active",
    "session_ids": [
      "sess_2024012418300000",
      "sess_2024012415450000",
      "sess_2024012409150000",
      "sess_2024012314300000",
      "sess_2024012310200000"
    ],
    "stats": {
      "total_sessions": 145,
      "last_session_at": "2024-01-24T18:30:00Z"
    },
    "created_at": "2023-06-15T08:00:00Z",
    "updated_at": "2024-01-24T18:30:00Z"
  },
  "timestamp": "2024-01-25T10:00:00Z"
}
```

### 4. Get Session Information

Retrieve detailed information about a specific session, including transcript and analytics.

#### Request

```http
GET /sessions/{session_id}
```

#### Path Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `session_id` | string | Yes | The unique identifier of the session |

#### Response

```json
{
  "success": true,
  "data": {
    "session_id": "sess_2024012418300000",
    "user_id": "user_001a2b3c4d5e",
    "experience": {
      "id": "exp_abc123def456",
      "name": "Customer Service Assistant",
      "version": "2.1.0"
    },
    "start_time": "2024-01-24T18:30:00.000Z",
    "end_time": "2024-01-24T18:38:23.000Z",
    "duration_seconds": 503,
    "status": "completed",
    "metrics": {
      "turns": 12,
      "user_messages": 6,
      "ai_messages": 6
    },
    "transcript": [
      {
        "turn_id": 1,
        "timestamp": "2024-01-24T18:30:05.123Z",
        "speaker": "ai",
        "message": "Hello! I'm your Customer Service Assistant. How can I help you today?"
      },
      {
        "turn_id": 2,
        "timestamp": "2024-01-24T18:30:18.456Z",
        "speaker": "user",
        "message": "I need help with my recent order #12345"
      },
      {
        "turn_id": 3,
        "timestamp": "2024-01-24T18:30:19.789Z",
        "speaker": "ai",
        "message": "I'd be happy to help you with order #12345. Let me look that up for you."
      }
    ]
  },
  "timestamp": "2024-01-25T10:00:00Z"
}
```

## Error Handling

### HTTP Status Codes

| Status Code | Description |
|------------|-------------|
| `200 OK` | Request successful |
| `400 Bad Request` | Invalid request parameters |
| `401 Unauthorized` | Invalid or missing authentication token |
| `403 Forbidden` | Valid token but insufficient permissions |
| `404 Not Found` | Resource not found |
| `500 Internal Server Error` | Server error occurred |
| `503 Service Unavailable` | Service temporarily unavailable |

### Error Response Format

```json
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable error message",
    "details": {
      "field": "Additional context about the error"
    },
    "request_id": "req_abc123def456"
  },
  "timestamp": "2024-01-25T10:00:00Z"
}
```

### Common Error Codes

| Error Code | Description |
|------------|-------------|
| `AUTH_INVALID_TOKEN` | JWT token is invalid or malformed |
| `AUTH_TOKEN_EXPIRED` | JWT token has expired |
| `AUTH_INSUFFICIENT_SCOPE` | Token lacks required scope |
| `RESOURCE_NOT_FOUND` | Requested resource doesn't exist |
| `VALIDATION_ERROR` | Request parameters are invalid |
| `INTERNAL_ERROR` | Server-side error |

## Versioning

### API Version

Current API version: **v1**

The API version is included in the base URL path:

```
https://partnerapi.persona-ai.ai/v1
```

### Semantic Versioning

We follow semantic versioning (semver) for our API:

- **Major version** (1.x.x): Breaking changes
- **Minor version** (x.1.x): New features, backward compatible
- **Patch version** (x.x.1): Bug fixes, backward compatible

---

*Last Updated: January 2025*  
*Version: 1.0.0*
