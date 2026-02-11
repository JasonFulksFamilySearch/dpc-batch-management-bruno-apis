# DPC Batch Management Service - Bruno API Collection

Comprehensive Bruno API collection for the DPC (Digital Preservation & Conservation) Batch Management Service, providing batch processing, delivery request management, and user access control.

## Overview

This collection provides complete API access to the DPC Batch Management Service, which manages:
- **Batch Processing** - Create, process, and deliver content batches
- **Delivery Requests** - Track and manage delivery workflows
- **Access Control** - Manage user permissions and access units
- **Monitoring** - Health checks, history tracking, and auditing

## Getting Started

### Prerequisites
- [Bruno](https://www.usebruno.com/) - Open source API client
- Valid FamilySearch OAuth credentials
- Access to DPC Batch Management Service (localhost, integration, beta, or production)

### Opening the Collection
1. Open Bruno
2. Click "Open Collection"
3. Navigate to `/Users/fulksjas/dev/Misc/bruno.apis/DPC BatchManagement`
4. Select the collection

### Authentication Setup

This collection uses **OAuth 2.0 with PKCE** for authentication:

1. **Select Environment**: Choose Localhost, Integration, Beta, or Production
2. **Trigger OAuth Flow**:
   - Open any request
   - Click "Run"
   - Bruno will open your browser for FamilySearch login
3. **Complete Authorization**:
   - Log in with your FamilySearch credentials
   - Authorize the application
   - Bruno will capture the access token automatically
4. **Token Storage**: Access token is stored in the environment variable `access_token`

**OAuth Configuration:**
- Authorization URL: `https://{{authSubdomain}}.familysearch.org/service/ident/cis/cis-web/oauth2/v3/authorization`
- Token URL: `https://{{authSubdomain}}.familysearch.org/service/ident/cis/cis-web/oauth2/v3/token`
- Client ID: `fs-internal-dev-key-000136`
- PKCE: Enabled
- Auto-refresh: Disabled (tokens valid for 24 hours)

## Environment Guide

### Localhost
- **Base URL**: `http://localhost:5000`
- **Auth Domain**: `integration.familysearch.org`
- **Purpose**: Local development against running service on port 5000
- **Database**: MySQL on localhost:3306
- **AWS**: LocalStack on localhost:4566

### Integration
- **Base URL**: `http://batchmanagement.records.service.integ.us-east-1.dev.fslocal.org`
- **Auth Domain**: `integration.familysearch.org`
- **Purpose**: Development and integration testing
- **Notes**: Shared development environment

### Beta
- **Base URL**: `http://batchmanagement.records.service.beta.us-east-1.test.fslocal.org`
- **Auth Domain**: `beta.familysearch.org`
- **Purpose**: Pre-production testing and validation
- **Notes**: Production-like environment for final testing

### Production
- **Base URL**: `http://batchmanagement.records.service.prod.us-east-1.prod.fslocal.org`
- **Auth Domain**: `www.familysearch.org` (ident subdomain)
- **Purpose**: Live production environment
- **⚠️ CAUTION**: Use with care - production data!

## Collection Structure

```
DPC Batch Management Service/
├── collection.bru              # OAuth config, global headers
├── bruno.json                  # Collection metadata
├── README.md                   # This file
├── environments/               # Environment configurations
│   ├── Localhost.bru
│   ├── Integration.bru
│   ├── Beta.bru
│   └── Production.bru
│
├── Access Unit/                # Access unit CRUD (9 endpoints)
├── Admin/                      # System administration (10 endpoints)
├── Archive/                    # Batch archival (2 endpoints)
├── Audit/                      # Audit logging (4 endpoints)
├── Authorization/              # Permission checks (5 endpoints)
├── Batch/                      # Batch lifecycle (17 endpoints)
├── Batch State/                # Batch state transitions (3 endpoints)
├── CIS User/                   # User management (7 endpoints)
├── Delivery Item/              # Item operations (3 endpoints)
├── Delivery Request/           # Request management (25 endpoints)
├── Delivery Request Inline/    # Inline requests (1 endpoint)
├── Delivery Request Lock/      # Request locking (6 endpoints)
├── Delivery Status/            # Status tracking (3 endpoints)
├── DevOps/                     # Operations endpoints (3 endpoints)
├── Expiration/                 # Expiration management (10 endpoints)
├── Healthcheck/                # Health monitoring (3 endpoints)
├── History/                    # History tracking (5 endpoints)
└── User Access Unit/           # User-unit relationships (11 endpoints)
```

## Core Workflows

### 1. Create and Process a Batch

**Workflow:**
1. **Create Access Unit** (if needed)
   - `POST /accessUnit/create`
   - Captures `accessUnitId`
2. **Create Delivery Request**
   - `POST /deliveryRequest/store/info/v2`
   - Captures `requestId`
3. **Store Request Groups**
   - `POST /deliveryRequest/store/groups`
   - Associates batches with request
4. **Mark Batch Ready**
   - `POST /batch/mark/ready`
   - Queues for packaging
5. **Monitor Status**
   - `GET /deliveryRequest/summary/v2`
   - Check batch states

### 2. Manage User Access

**Workflow:**
1. **Create CIS User**
   - `POST /cisUser/create`
   - Captures `cisUserId`
2. **Grant Access Unit**
   - `POST /userAccessUnit/grantAccess`
   - Links user to access unit
3. **Set Primary Unit**
   - `PUT /cisUser/me/primaryAccessUnit`
   - User's default unit
4. **Verify Access**
   - `GET /authorize/deliveryRequest`
   - Check permissions

### 3. Monitor Batch Lifecycle

**States:**
- `QUEUED` → `VALIDATING` → `PREPARING` → `READY_FOR_DELIVERY` → `DELIVERED`
- Error states: `RECORD_ERROR`, `DOWNLOAD_ERRORED`
- Download states: `DOWNLOAD_STARTED`, `DOWNLOAD_PAUSED`, `DOWNLOAD_FINISHED`

**Monitoring:**
1. **Get Batch Details** - `GET /batch/{batchId}`
2. **Get Request Summary** - `GET /deliveryRequest/summary/v2`
3. **Get History** - `POST /history/deliveryStates`
4. **Get Statistics** - `GET /batch/page`

## Variables

### Environment Variables
- `domain` - Base URL for the service
- `authSubdomain` - OAuth authorization domain
- `authClientId` - OAuth client ID
- `userAgent` - User-Agent header value

### Captured Variables (used across requests)
- `accessUnitId` - Access unit identifier
- `batchId` - Batch identifier
- `requestId` - Delivery request identifier
- `cisUserId` - CIS user identifier
- `projectId` - Project identifier
- `deliveryId` - Delivery identifier

### How Variable Capture Works

Request files include post-response scripts that automatically capture IDs:

```javascript
script:post-response {
  if (res.status === 201 && res.body.id) {
    bru.setVar("resourceId", res.body.id);
  }
}
```

These captured variables are then used in subsequent requests:
```
{{domain}}/batch/{{batchId}}
```

## Key Features

### Request Body Documentation
All POST/PUT/PATCH requests include comprehensive inline comments:

```json
{
  # REQUIRED: Access unit name
  # Max length: 255 characters
  # Must be unique across the system
  "accessUnitName": "Archive Division {{$timestamp}}",

  # OPTIONAL: Custodian ID
  # Format: Integer
  # Get valid IDs from: GET /accessUnit
  # Default: null (no custodian)
  "custodianId": null
}
```

### Test Assertions
Every request includes validation tests:

```javascript
tests {
  test("Status is 201 Created", function() {
    expect(res.status).to.equal(201);
  });

  test("Response has ID", function() {
    expect(res.body.id).to.be.a('string');
  });
}
```

### Pagination Support
List endpoints support Spring Boot pagination:

```
?page=0&size=20&sort=createdDate,desc
```

## API Versioning

Some endpoints have multiple versions:
- **v1** (deprecated) - Uses `requestFor` (access unit name)
- **v2** (current) - Uses `requestForAccessUnitId` (numeric ID)
- **v3** (latest) - Enhanced filtering and pagination

Always prefer the latest version when available.

## Common Issues

### 401 Unauthorized
**Cause**: OAuth token expired or invalid
**Solution**: Re-run OAuth flow or check environment configuration

### 403 Forbidden
**Cause**: User lacks required permissions
**Solution**:
- Verify user has `AdministerDeliveryUsersPerm` for admin endpoints
- Check user is assigned to correct access unit
- Use `GET /userAccessUnit/me/accessUnits` to see your access

### 404 Not Found
**Cause**: Resource doesn't exist or access denied
**Solution**: Verify IDs are correct and user has access

### 409 Conflict
**Cause**: Resource already exists (e.g., duplicate access unit name)
**Solution**: Use different name or update existing resource

### 400 Bad Request with Query Params
**Cause**: Missing required parameters or invalid format
**Solution**: Check inline comments for required fields and valid values

## Permissions Required

### Admin Operations
Require `AdministerDeliveryUsersPerm` TARS permission:
- Access Unit management
- CIS User management
- User Access Unit assignment
- Admin configuration

### Regular Operations
Require valid session + access unit membership:
- Batch operations
- Delivery request operations
- History tracking
- Status monitoring

## Service Architecture

### Technology Stack
- **Framework**: Spring Boot 3.x
- **JDK**: Amazon Corretto 25
- **Database**: MySQL 8.0 (RDS)
- **Messaging**: AWS SQS
- **Storage**: AWS S3
- **Authentication**: FamilySearch OAuth 2.0

### Key Components
- **BatchController** - Batch lifecycle management
- **DeliveryMonitor** - Async status monitoring
- **BatchPackager** - Creates deliverable packages
- **MetsFileCreator** - Generates METS metadata
- **ExpirationService** - Manages batch expiration

### AWS Resources
- **SQS Queues**:
  - `sqs-package-batch-internal` - Packaging operations
  - `sqs-standard-batch` - Standard processing
- **SNS Topics**:
  - `sns-request-ready` - Request notifications
- **S3 Buckets**:
  - `delivery-files` - Batch deliverables

## Related Documentation

- **API Documentation**: `/docs` endpoint (Enunciate)
- **OpenAPI Spec**: `/v3/api-docs` (SpringDoc)
- **Health Check**: `/healthcheck/vitals`
- **Actuator**: Port 9000 (`/actuator/health`, `/actuator/info`)

## Development

### Local Setup
1. Start Docker services:
   ```bash
   docker-compose up -d
   ```
2. Run application:
   ```bash
   ./mvnw spring-boot:run -Dspring-boot.run.profiles=local
   ```
3. Set Bruno environment to "Localhost"
4. Begin testing API endpoints

### Project Repository
- Location: `/Users/fulksjas/dev/DPC/Batch Management Service/pipe-dpc-batch-management-service`
- Blueprint: `blueprint.yml` (Elastic Beanstalk deployment)
- Controllers: `webapp/src/main/java/.../controller/`

## Support

For issues or questions:
- Check endpoint documentation in `.bru` files (comprehensive inline comments)
- Review test assertions for expected behavior
- Check service logs for detailed error messages
- Verify OAuth token is valid and user has required permissions

## Collection Metadata

- **Total Endpoints**: 127+
- **Total Controllers**: 18
- **Authentication**: OAuth 2.0 with PKCE
- **Created**: February 2026
- **Version**: 1.0
- **Maintained By**: DPC Team
