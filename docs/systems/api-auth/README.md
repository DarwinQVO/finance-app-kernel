# API/Auth System

**Status**: Universal / Domain-Agnostic
**Purpose**: Provide secure, authenticated, and authorized API access with observability

---

## Overview

The API/Auth System is an independent infrastructure layer that provides:
- **API Gateway**: Request routing, rate limiting, response caching
- **Authentication**: OAuth2 flows, API key validation
- **Authorization**: Role-based access control (RBAC)
- **Observability**: Metrics collection, dashboards, schema management

## Architecture

```
HTTP Request → API Gateway → Authentication → Authorization → Route to System → Response
                    ↓                                              ↓
              Rate Limiting                                  Metrics Collection
              Response Caching                               ProvenanceLedger Logging
```

---

## Primitives

### Gateway & Routing
- [APIGateway](primitives/APIGateway.md) - Request routing, rate limiting, response caching
- [APIRouter](primitives/APIRouter.md) - Route matching and handler dispatch

### Authentication
- [OAuth2Provider](primitives/OAuth2Provider.md) - OAuth2 authorization code flow
- [APIKeyValidator](primitives/APIKeyValidator.md) - API key management and validation

### Authorization
- [AccessControl](primitives/AccessControl.md) - Role-based access control (RBAC)

### Observability
- [MetricsCollector](primitives/MetricsCollector.md) - Prometheus-compatible metrics
- [DashboardEngine](primitives/DashboardEngine.md) - Query metrics and generate dashboards
- [SchemaRegistry](primitives/SchemaRegistry.md) - JSON schema validation and versioning

---

## Multi-Domain Applicability

The API/Auth System works identically across domains:

| Domain | Protected Resources | Use Case |
|--------|---------------------|----------|
| **Finance** | GET /transactions, POST /accounts | Secure access to financial data |
| **Healthcare** | GET /lab-results, POST /patients | HIPAA-compliant API access |
| **Legal** | GET /contracts, POST /clauses | Secure document management |
| **Research (RSRCH)** | GET /facts, POST /sources | Authenticated fact retrieval |
| **E-commerce** | GET /products, POST /orders | Customer and admin API access |
| **Manufacturing** | GET /sensors, POST /readings | IoT device authentication |

---

## Key Features

### 1. OAuth2 Flows
- **Authorization Code Flow**: Web applications
- **Client Credentials Flow**: Service-to-service
- **Refresh Tokens**: Long-lived sessions

### 2. Role-Based Access Control (RBAC)
- Define roles (admin, user, read-only)
- Assign permissions to roles (read:transactions, write:accounts)
- Enforce authorization checks before every operation

### 3. Rate Limiting
- Per-user limits: 100 requests/minute
- Per-IP limits: 1000 requests/hour
- Custom limits for premium users

### 4. Response Caching
- Cache GET responses for configurable TTL
- Cache invalidation on POST/PUT/DELETE
- Reduce load on backend systems

### 5. Observability
- **Metrics**: Request count, latency, error rate
- **Dashboards**: Real-time performance monitoring
- **Schema Validation**: Catch breaking changes early

---

## Integration with Other Systems

### Truth Construction System
- Protect endpoints: GET /transactions, POST /upload, GET /export
- Enforce authorization: Only users with "read:transactions" can query
- Log API calls → ProvenanceLedger

### Audit System
- Protect endpoints: GET /provenance, POST /retroactive-correction
- Enforce authorization: Only admins can perform corrections
- Log audit queries → ProvenanceLedger

### Finance App
- Protect domain-specific endpoints: GET /accounts, POST /counterparties
- Enforce multi-tenancy: Users can only see their own data
- Log business operations → ProvenanceLedger

---

## Security Best Practices

1. **HTTPS Only**: All API traffic encrypted in transit
2. **Token Expiration**: Short-lived access tokens (15 minutes), long-lived refresh tokens (30 days)
3. **Audit Logging**: All authentication/authorization events → ProvenanceLedger
4. **Rate Limiting**: Prevent abuse and DDoS attacks
5. **Schema Validation**: Reject malformed requests early

---

## Performance Features

- **Response Caching**: 10x faster for repeated GET requests
- **Connection Pooling**: Reuse HTTP connections for efficiency
- **Async I/O**: Non-blocking request handling
- **Streaming Responses**: Support large result sets without OOM

---

## Key Principles

1. **Independent**: Can be deployed separately from business logic systems
2. **Domain-Agnostic**: No finance-specific code
3. **Security-First**: Authentication and authorization enforced at gateway
4. **Observable**: Metrics and dashboards for production monitoring

---

## Related Documentation

- [Truth Construction System](../truth-construction/README.md) - Primary protected system
- [Audit System](../audit/README.md) - Logs all authentication/authorization events
- [Finance App Example](../../verticals/README.md) - Domain-specific API endpoints

---

**Primitives Count**: 8
**Status**: Production-ready (all primitives include security hardening)
**Last Updated**: 2025-10-27
