# LedgerLoop API Specification

## Overview

This document provides detailed API specifications for the LedgerLoop system, including request/response schemas, authentication requirements, and error handling.

## Base URL

- **Production**: `https://api.ledgerloop.com/v1`
- **Staging**: `https://staging-api.ledgerloop.com/v1`
- **Development**: `http://localhost:3000/api/v1`

## Authentication

All API endpoints (except public endpoints) require authentication using JWT tokens.

### Headers

```
Authorization: Bearer <jwt_token>
Content-Type: application/json
```

### Token Structure

```json
{
  "sub": "user_id",
  "email": "user@example.com",
  "role": "user|admin|reviewer",
  "iat": 1234567890,
  "exp": 1234567890
}
```

## Error Responses

### Standard Error Format

```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Human readable error message",
    "details": {
      "field": "validation_error_detail"
    },
    "requestId": "uuid"
  }
}
```

### HTTP Status Codes

- `200` - Success
- `201` - Created
- `400` - Bad Request
- `401` - Unauthorized
- `403` - Forbidden
- `404` - Not Found
- `409` - Conflict
- `422` - Unprocessable Entity
- `429` - Too Many Requests
- `500` - Internal Server Error

## Rate Limiting

- **Default**: 100 requests per minute per IP
- **Authenticated**: 1000 requests per minute per user
- **Admin**: 10000 requests per minute

Rate limit headers:
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 99
X-RateLimit-Reset: 1234567890
```

## API Endpoints

### Authentication

#### POST /auth/register

Register a new user account.

**Request Body:**
```json
{
  "email": "user@example.com",
  "password": "securePassword123",
  "firstName": "John",
  "lastName": "Doe",
  "phoneNumber": "+1234567890",
  "dateOfBirth": "1990-01-01",
  "address": {
    "street": "123 Main St",
    "city": "Anytown",
    "state": "CA",
    "zipCode": "12345",
    "country": "US"
  }
}
```

**Response (201):**
```json
{
  "user": {
    "id": "uuid",
    "email": "user@example.com",
    "profile": {
      "firstName": "John",
      "lastName": "Doe",
      "phoneNumber": "+1234567890"
    },
    "kycStatus": "pending",
    "isActive": true,
    "createdAt": "2023-01-01T00:00:00Z"
  },
  "token": "jwt_token_here",
  "refreshToken": "refresh_token_here"
}
```

#### POST /auth/login

Authenticate user and receive JWT token.

**Request Body:**
```json
{
  "email": "user@example.com",
  "password": "securePassword123",
  "remember": true
}
```

**Response (200):**
```json
{
  "user": {
    "id": "uuid",
    "email": "user@example.com",
    "profile": {
      "firstName": "John",
      "lastName": "Doe"
    },
    "kycStatus": "verified",
    "isActive": true
  },
  "token": "jwt_token_here",
  "refreshToken": "refresh_token_here",
  "expiresIn": 3600
}
```

### User Management

#### GET /users/profile

Get current user's profile information.

**Response (200):**
```json
{
  "user": {
    "id": "uuid",
    "email": "user@example.com",
    "profile": {
      "firstName": "John",
      "lastName": "Doe",
      "phoneNumber": "+1234567890",
      "dateOfBirth": "1990-01-01",
      "address": {
        "street": "123 Main St",
        "city": "Anytown",
        "state": "CA",
        "zipCode": "12345",
        "country": "US"
      }
    },
    "xrplAddresses": ["rN7n7otQDd6FczFgLdSqtcsAUxDkw6fzRH"],
    "kycStatus": "verified",
    "creditScore": 750,
    "isActive": true,
    "createdAt": "2023-01-01T00:00:00Z",
    "updatedAt": "2023-01-01T00:00:00Z"
  }
}
```

#### PUT /users/profile

Update user profile information.

**Request Body:**
```json
{
  "profile": {
    "firstName": "John",
    "lastName": "Smith",
    "phoneNumber": "+1234567890",
    "address": {
      "street": "456 Oak Ave",
      "city": "Newtown",
      "state": "NY",
      "zipCode": "67890",
      "country": "US"
    }
  }
}
```

**Response (200):**
```json
{
  "user": {
    "id": "uuid",
    "email": "user@example.com",
    "profile": {
      "firstName": "John",
      "lastName": "Smith",
      "phoneNumber": "+1234567890",
      "address": {
        "street": "456 Oak Ave",
        "city": "Newtown",
        "state": "NY",
        "zipCode": "67890",
        "country": "US"
      }
    },
    "updatedAt": "2023-01-02T00:00:00Z"
  }
}
```

### Circle Management

#### GET /circles

List available circles with pagination and filtering.

**Query Parameters:**
- `page` (optional): Page number (default: 1)
- `limit` (optional): Items per page (default: 20, max: 100)
- `status` (optional): Filter by status (forming|active|completed)
- `maxMembers` (optional): Filter by maximum members
- `contributionAmount` (optional): Filter by contribution amount

**Response (200):**
```json
{
  "circles": [
    {
      "id": "uuid",
      "name": "Tech Workers Circle",
      "description": "A lending circle for technology professionals",
      "creatorId": "uuid",
      "parameters": {
        "maxMembers": 10,
        "contributionAmount": 1000,
        "currency": "XRP",
        "paymentFrequency": "monthly",
        "duration": 12,
        "interestRate": 0.05
      },
      "status": "forming",
      "currentMembers": 5,
      "escrowAddress": "rEscrow123...",
      "createdAt": "2023-01-01T00:00:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 1,
    "totalPages": 1,
    "hasNext": false,
    "hasPrev": false
  }
}
```

#### POST /circles

Create a new lending circle.

**Request Body:**
```json
{
  "name": "Small Business Circle",
  "description": "Supporting local entrepreneurs",
  "parameters": {
    "maxMembers": 8,
    "contributionAmount": 500,
    "currency": "XRP",
    "paymentFrequency": "monthly",
    "duration": 10,
    "interestRate": 0.04
  }
}
```

**Response (201):**
```json
{
  "circle": {
    "id": "uuid",
    "name": "Small Business Circle",
    "description": "Supporting local entrepreneurs",
    "creatorId": "uuid",
    "parameters": {
      "maxMembers": 8,
      "contributionAmount": 500,
      "currency": "XRP",
      "paymentFrequency": "monthly",
      "duration": 10,
      "interestRate": 0.04
    },
    "status": "forming",
    "members": [
      {
        "userId": "uuid",
        "joinedAt": "2023-01-01T00:00:00Z",
        "paymentOrder": 1,
        "status": "active"
      }
    ],
    "escrowAddress": "rEscrow456...",
    "multisigConfig": {
      "requiredSignatures": 5,
      "signers": ["rSigner1...", "rSigner2..."]
    },
    "createdAt": "2023-01-01T00:00:00Z"
  }
}
```

#### POST /circles/:id/join

Join an existing circle.

**Request Body:**
```json
{
  "motivation": "I want to expand my business and this circle aligns with my goals"
}
```

**Response (200):**
```json
{
  "membership": {
    "circleId": "uuid",
    "userId": "uuid",
    "joinedAt": "2023-01-01T00:00:00Z",
    "paymentOrder": 6,
    "status": "active"
  },
  "circle": {
    "id": "uuid",
    "name": "Small Business Circle",
    "currentMembers": 6,
    "status": "forming"
  }
}
```

### Loan Management

#### GET /loans

List user's loans with filtering and pagination.

**Query Parameters:**
- `page` (optional): Page number
- `limit` (optional): Items per page
- `status` (optional): Filter by status
- `circleId` (optional): Filter by circle

**Response (200):**
```json
{
  "loans": [
    {
      "id": "uuid",
      "circleId": "uuid",
      "borrowerId": "uuid",
      "amount": 5000,
      "currency": "XRP",
      "interestRate": 0.05,
      "termMonths": 12,
      "status": "active",
      "paymentSchedule": [
        {
          "dueDate": "2023-02-01",
          "amount": 450.50,
          "status": "pending"
        }
      ],
      "escrowTransactionId": "txn123",
      "createdAt": "2023-01-01T00:00:00Z",
      "updatedAt": "2023-01-01T00:00:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 1,
    "totalPages": 1
  }
}
```

#### POST /loans

Create a new loan request.

**Request Body:**
```json
{
  "circleId": "uuid",
  "amount": 5000,
  "currency": "XRP",
  "termMonths": 12,
  "purpose": "Business expansion",
  "businessPlan": {
    "description": "Expanding inventory for holiday season",
    "projectedRevenue": 15000,
    "repaymentPlan": "Monthly payments from increased sales"
  }
}
```

**Response (201):**
```json
{
  "loan": {
    "id": "uuid",
    "circleId": "uuid",
    "borrowerId": "uuid",
    "amount": 5000,
    "currency": "XRP",
    "interestRate": 0.05,
    "termMonths": 12,
    "status": "pending",
    "purpose": "Business expansion",
    "businessPlan": {
      "description": "Expanding inventory for holiday season",
      "projectedRevenue": 15000,
      "repaymentPlan": "Monthly payments from increased sales"
    },
    "createdAt": "2023-01-01T00:00:00Z"
  }
}
```

### Transaction Management

#### GET /transactions

List user's transactions with filtering.

**Query Parameters:**
- `page` (optional): Page number
- `limit` (optional): Items per page
- `type` (optional): Filter by transaction type
- `status` (optional): Filter by status
- `circleId` (optional): Filter by circle
- `startDate` (optional): Filter by date range
- `endDate` (optional): Filter by date range

**Response (200):**
```json
{
  "transactions": [
    {
      "id": "uuid",
      "type": "contribution",
      "fromUserId": "uuid",
      "toUserId": null,
      "circleId": "uuid",
      "amount": 1000,
      "currency": "XRP",
      "xrplTransactionId": "ABCD1234...",
      "status": "confirmed",
      "metadata": {
        "paymentMethod": "wallet",
        "gasUsed": 12
      },
      "createdAt": "2023-01-01T00:00:00Z",
      "confirmedAt": "2023-01-01T00:01:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 1,
    "totalPages": 1
  }
}
```

#### POST /transactions/payment

Initiate a payment transaction.

**Request Body:**
```json
{
  "type": "loan_payment",
  "toUserId": "uuid",
  "circleId": "uuid",
  "amount": 450.50,
  "currency": "XRP",
  "loanId": "uuid",
  "memo": "Monthly loan payment"
}
```

**Response (201):**
```json
{
  "transaction": {
    "id": "uuid",
    "type": "loan_payment",
    "fromUserId": "uuid",
    "toUserId": "uuid",
    "circleId": "uuid",
    "amount": 450.50,
    "currency": "XRP",
    "status": "pending",
    "metadata": {
      "loanId": "uuid",
      "memo": "Monthly loan payment"
    },
    "createdAt": "2023-01-01T00:00:00Z"
  },
  "xrplTransaction": {
    "hash": "ABCD1234...",
    "sequence": 12345,
    "fee": "12"
  }
}
```

### Grant Applications

#### GET /grants

List grant applications (user sees own, admin sees all).

**Query Parameters:**
- `page` (optional): Page number
- `limit` (optional): Items per page
- `status` (optional): Filter by status
- `amount` (optional): Filter by amount range

**Response (200):**
```json
{
  "applications": [
    {
      "id": "uuid",
      "applicantId": "uuid",
      "circleId": "uuid",
      "requestedAmount": 10000,
      "purpose": "Community garden project",
      "businessPlan": {
        "description": "Creating a sustainable community garden",
        "impact": "Feeding 50 families",
        "timeline": "6 months"
      },
      "status": "pending",
      "reviewerId": null,
      "reviewNotes": null,
      "approvedAmount": null,
      "createdAt": "2023-01-01T00:00:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 1,
    "totalPages": 1
  }
}
```

#### POST /grants

Submit a new grant application.

**Request Body:**
```json
{
  "circleId": "uuid",
  "requestedAmount": 10000,
  "purpose": "Community garden project",
  "businessPlan": {
    "description": "Creating a sustainable community garden to serve the local community",
    "impact": "Providing fresh produce to 50 low-income families",
    "timeline": "6 months implementation",
    "budget": {
      "land": 3000,
      "materials": 4000,
      "labor": 2000,
      "maintenance": 1000
    },
    "sustainability": "Self-sustaining through produce sales after year 1"
  },
  "documents": [
    {
      "type": "business_plan",
      "url": "ipfs://QmHash123...",
      "name": "Community Garden Business Plan.pdf"
    }
  ]
}
```

**Response (201):**
```json
{
  "application": {
    "id": "uuid",
    "applicantId": "uuid",
    "circleId": "uuid",
    "requestedAmount": 10000,
    "purpose": "Community garden project",
    "businessPlan": {
      "description": "Creating a sustainable community garden to serve the local community",
      "impact": "Providing fresh produce to 50 low-income families",
      "timeline": "6 months implementation"
    },
    "status": "pending",
    "documents": [
      {
        "type": "business_plan",
        "url": "ipfs://QmHash123...",
        "name": "Community Garden Business Plan.pdf"
      }
    ],
    "createdAt": "2023-01-01T00:00:00Z"
  }
}
```

#### POST /grants/:id/review

Submit a review for a grant application (admin only).

**Request Body:**
```json
{
  "decision": "approved",
  "approvedAmount": 8000,
  "reviewNotes": "Excellent project with clear community benefit. Reduced amount due to budget constraints.",
  "conditions": [
    "Quarterly progress reports required",
    "Community engagement metrics must be provided"
  ],
  "disbursementSchedule": [
    {
      "percentage": 50,
      "milestone": "Project kickoff and land preparation"
    },
    {
      "percentage": 30,
      "milestone": "Infrastructure completion"
    },
    {
      "percentage": 20,
      "milestone": "First harvest and community engagement metrics"
    }
  ]
}
```

**Response (200):**
```json
{
  "application": {
    "id": "uuid",
    "status": "approved",
    "reviewerId": "uuid",
    "reviewNotes": "Excellent project with clear community benefit.",
    "approvedAmount": 8000,
    "conditions": [
      "Quarterly progress reports required",
      "Community engagement metrics must be provided"
    ],
    "disbursementSchedule": [
      {
        "percentage": 50,
        "milestone": "Project kickoff and land preparation"
      }
    ],
    "reviewedAt": "2023-01-02T00:00:00Z"
  }
}
```

## WebSocket Events

### Real-time Updates

The system supports WebSocket connections for real-time updates.

**Connection URL:** `wss://api.ledgerloop.com/ws`

**Authentication:** Send JWT token in connection header or as first message.

### Event Types

#### Circle Updates
```json
{
  "type": "circle_update",
  "data": {
    "circleId": "uuid",
    "event": "member_joined",
    "member": {
      "userId": "uuid",
      "name": "John Doe"
    },
    "timestamp": "2023-01-01T00:00:00Z"
  }
}
```

#### Transaction Updates
```json
{
  "type": "transaction_update",
  "data": {
    "transactionId": "uuid",
    "status": "confirmed",
    "xrplTransactionId": "ABCD1234...",
    "timestamp": "2023-01-01T00:00:00Z"
  }
}
```

#### Payment Notifications
```json
{
  "type": "payment_notification",
  "data": {
    "type": "payment_due",
    "loanId": "uuid",
    "amount": 450.50,
    "dueDate": "2023-02-01",
    "timestamp": "2023-01-01T00:00:00Z"
  }
}
```

## SDK Examples

### JavaScript SDK

```javascript
import { LedgerLoopSDK } from '@ledgerloop/sdk';

const sdk = new LedgerLoopSDK({
  apiUrl: 'https://api.ledgerloop.com/v1',
  apiKey: 'your-api-key'
});

// Authenticate
await sdk.auth.login('user@example.com', 'password');

// Create a circle
const circle = await sdk.circles.create({
  name: 'My Circle',
  parameters: {
    maxMembers: 10,
    contributionAmount: 1000,
    currency: 'XRP',
    paymentFrequency: 'monthly'
  }
});

// Join a circle
await sdk.circles.join(circle.id);

// Make a payment
const payment = await sdk.transactions.createPayment({
  type: 'contribution',
  circleId: circle.id,
  amount: 1000,
  currency: 'XRP'
});
```

### Python SDK

```python
from ledgerloop import LedgerLoopSDK

sdk = LedgerLoopSDK(
    api_url='https://api.ledgerloop.com/v1',
    api_key='your-api-key'
)

# Authenticate
sdk.auth.login('user@example.com', 'password')

# List circles
circles = sdk.circles.list(status='forming')

# Apply for grant
application = sdk.grants.create({
    'circle_id': 'uuid',
    'requested_amount': 10000,
    'purpose': 'Business expansion',
    'business_plan': {
        'description': 'Expanding operations',
        'impact': 'Create 10 new jobs'
    }
})
```

## Rate Limiting Details

### Limits by Endpoint Category

| Category | Authenticated | Unauthenticated |
|----------|---------------|-----------------|
| Auth | 10/min | 5/min |
| User Profile | 60/min | N/A |
| Circles | 100/min | 10/min |
| Transactions | 200/min | N/A |
| Grants | 30/min | N/A |

### Rate Limit Response

When rate limit is exceeded:

```json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Too many requests. Please try again later.",
    "retryAfter": 60
  }
}
```

## Pagination

### Standard Pagination

Query parameters:
- `page`: Page number (1-based)
- `limit`: Items per page (max 100)

Response format:
```json
{
  "data": [...],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 100,
    "totalPages": 5,
    "hasNext": true,
    "hasPrev": false
  }
}
```

### Cursor-based Pagination

For real-time data (transactions, notifications):

Query parameters:
- `cursor`: Pagination cursor
- `limit`: Items per page

Response format:
```json
{
  "data": [...],
  "pagination": {
    "cursor": "eyJpZCI6IjEyMyJ9",
    "hasNext": true,
    "limit": 20
  }
}
```

## Testing

### Test Environment

- **Base URL**: `https://staging-api.ledgerloop.com/v1`
- **XRPL Network**: Testnet
- **Rate Limits**: Relaxed for testing

### Test Data

Test accounts and circles are available for integration testing. Contact support for test credentials.

### Postman Collection

A Postman collection with example requests is available at:
`https://api.ledgerloop.com/docs/postman-collection.json`

This API specification provides comprehensive documentation for integrating with the LedgerLoop platform, enabling developers to build applications that leverage the lending circle functionality.