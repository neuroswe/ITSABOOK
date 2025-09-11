# 7.2 REST API Design Principles

## Core Concepts

### Resource Orientation: Think Nouns, Not Verbs

REST treats everything as a resource identified by URLs. Resources should represent business entities or concepts, not actions.

**❌ Action-Based URLs (What NOT to do):**

```
/getUser/123
/createOrder
/updateUserProfile
/deleteProduct/456
/calculateTax
```

**Why this is problematic:**

* **Violates REST principles**: REST is about manipulating resources, not calling remote procedures
* **Reduces cacheability**: Browsers and proxies can't effectively cache action-based URLs
* **Creates URL explosion**: You end up with dozens of action endpoints instead of a few resource endpoints
* **Breaks HTTP semantics**: HTTP verbs lose their meaning when everything is a noun
* **Harder to understand**: Clients must memorize many different endpoint patterns

**✅ Resource-Based URLs (Correct approach):**

```
GET /users/123                    # Retrieve user
POST /orders                      # Create order  
PUT /users/123/profile            # Update profile
DELETE /products/456              # Delete product
GET /orders/789/tax-calculation   # Get calculated tax (as a resource)
```

**Why this works better:**

* **Predictable patterns**: Once you know the resource, you know all operations
* **Leverages HTTP caching**: GET requests can be cached effectively
* **Self-documenting**: The URL structure tells you what you're working with
* **Reduces cognitive load**: Developers can guess endpoints based on patterns

### HTTP Verbs: Use Them Correctly or Pay the Price

#### GET - Retrieve Resources

**✅ Correct usage:**

```http
GET /users/123
GET /orders?status=pending&limit=10
GET /users/123/permissions
```

**❌ Common mistakes:**

```http
GET /users/delete/123           # Using GET for state changes
GET /orders/create?amount=100   # Using GET for creation
GET /search?query=sensitive     # Exposing sensitive data in logs
```

**Why these mistakes hurt you:**

* **Security risks**: GET parameters appear in server logs, browser history, and referrer headers
* **Caching issues**: Proxies may cache destructive operations
* **Idempotency violations**: Users hitting refresh might accidentally trigger operations
* **SEO problems**: Search engines might crawl and execute these URLs

#### POST - Create Resources (Not a Garbage Can)

**✅ Appropriate POST usage:**

```http
POST /users                      # Create new user
POST /orders/123/payments        # Process payment (non-idempotent)
POST /search                     # Complex search with sensitive criteria
```

**❌ POST abuse:**

```http
POST /users/get/123              # Should be GET
POST /orders/update/456          # Should be PUT/PATCH
POST /products/delete/789        # Should be DELETE
```

**Why POST abuse is costly:**

* **Performance impact**: POST requests are never cached by default
* **Browser behavior**: Back button doesn't work intuitively with POST
* **Monitoring difficulty**: Hard to distinguish between real creates and misused POSTs
* **Developer confusion**: Team members can't predict what operations do

#### PUT vs PATCH: Choose Wisely or Confuse Everyone

**PUT - Complete Replacement:**

```http
PUT /users/123
{
  "name": "John Smith",
  "email": "john.smith@example.com", 
  "role": "admin",
  "status": "active"
}
```

**❌ Common PUT mistakes:**

```http
PUT /users/123
{
  "status": "inactive"  # Missing required fields!
}
```

**Why incomplete PUT requests cause problems:**

* **Data loss**: Missing fields might be set to null or default values
* **Inconsistent behavior**: Some implementations ignore missing fields, others don't
* **Race conditions**: Concurrent updates can overwrite each other's changes
* **Client complexity**: Clients must always fetch full resource before updating

**PATCH - Partial Updates:**

```http
PATCH /users/123
{
  "status": "inactive",
  "lastLoginAt": "2025-09-08T10:30:00Z"
}
```

**❌ PATCH pitfalls:**

```http
PATCH /users/123
{
  "email": ""  # Empty string vs null vs undefined - what does this mean?
}
```

**Why ambiguous PATCH operations fail:**

* **Semantic confusion**: Does empty string mean "clear the field" or "ignore this field"?
* **Implementation inconsistency**: Different servers handle edge cases differently
* **Testing complexity**: More edge cases to test than simple replacement
* **Rollback difficulty**: Partial updates are harder to reverse

## Why URL Structure Matters More Than You Think

### The Plural vs Singular Debate

**✅ Use plural nouns consistently:**

```
GET /users              # List all users
GET /users/123          # Get specific user
POST /users             # Create new user
```

**❌ Mixing singular and plural:**

```
GET /user               # Confusing - is this one user or all users?
GET /user/123           # Inconsistent with collection endpoint
POST /users             # Now POST uses plural but GET uses singular?
```

**Why consistency matters:**

* **Developer productivity**: Consistent patterns reduce mental overhead
* **API discoverability**: Developers can guess correct URLs
* **Code generation**: Tools can automatically generate client code from patterns
* **Documentation clarity**: Fewer special cases to explain

### Resource Relationships: Nested vs Independent

#### When to Use Nested Resources

**✅ Appropriate nesting (tight coupling):**

```http
GET /users/123/orders           # Orders belong to user
GET /posts/456/comments         # Comments belong to post  
GET /accounts/789/transactions  # Transactions belong to account
```

**❌ Over-nesting (loose coupling):**

```http
GET /users/123/orders/456/products/789/reviews/101  # Too deep!
GET /companies/123/users/456/orders/789             # Multiple ownership
```

**Why over-nesting breaks down:**

* **URL explosion**: Exponential growth in endpoint combinations
* **Authorization complexity**: Who can access deeply nested resources?
* **Performance impact**: Multiple database joins for simple operations
* **Coupling increases**: Changes to parent resources break child endpoints

#### When to Use Independent Resources

**✅ Independent resources with references:**

```http
GET /products/456    # Product exists independently
GET /orders/789      # Order references products
{
  "orderId": 789,
  "customerId": 123,
  "products": [
    {"productId": 456, "quantity": 2},
    {"productId": 457, "quantity": 1}
  ]
}
```

**Why this approach scales:**

* **Flexible access patterns**: Can access products directly or through orders
* **Reduced coupling**: Products can be updated without affecting orders
* **Better caching**: Independent resources can be cached separately
* **Simpler authorization**: Clear ownership boundaries

## Query Parameters: Power and Pitfalls

### Filtering: The Right Way and Wrong Way

**✅ Clear, predictable filtering:**

```http
GET /users?role=admin&status=active
GET /orders?createdAfter=2025-01-01&amount[gte]=100  
GET /products?category=electronics&inStock=true
```

**❌ Ambiguous or complex filtering:**

```http
GET /users?filter=role:admin,status:active           # Custom syntax
GET /orders?q=(amount>100)AND(status='pending')      # SQL-like syntax
GET /products?search=electronics+available           # Unclear operators
```

**Why complex filtering syntax fails:**

* **Client complexity**: Every client must implement custom parsing
* **URL encoding issues**: Special characters cause encoding problems
* **Security risks**: SQL-like syntax invites injection attacks
* **Limited tooling**: Standard HTTP tools can't understand custom formats

### Pagination: Offset vs Cursor-Based

**❌ Offset-based pagination problems:**

```http
GET /users?limit=20&offset=1000  # Gets slower as offset increases
```

**Issues with large offsets:**

* **Performance degradation**: Database must count and skip many records
* **Consistency problems**: New records can cause items to appear twice
* **Memory usage**: Large offsets consume more database resources

**✅ Cursor-based pagination:**

```http
GET /users?limit=20&cursor=eyJpZCI6MTIzfQ==
{
  "data": [...],
  "pagination": {
    "nextCursor": "eyJpZCI6MTQzfQ==",
    "hasNext": true
  }
}
```

**Why cursors scale better:**

* **Consistent performance**: Always fast regardless of position
* **Stable results**: No duplicates when data changes during pagination
* **Efficient queries**: Uses indexed fields for direct seeking

## Status Codes: Precision Matters

### Common Status Code Mistakes

**❌ Generic 200 for everything:**

```http
POST /users
200 OK
{
  "success": true,
  "message": "User created"
}
```

**Why this hurts clients:**

* **Caching confusion**: Clients don't know if POST succeeded or not
* **Error handling**: Can't distinguish between creation and updates
* **HTTP semantics lost**: Status codes become meaningless

**✅ Precise status codes:**

```http
POST /users
201 Created
Location: /users/124
{
  "id": 124,
  "name": "John Doe",
  ...
}
```

### Error Status Code Precision

**❌ Generic 500 for all errors:**

```http
POST /users
{
  "email": "invalid-email"
}

500 Internal Server Error
{
  "error": "Something went wrong"
}
```

**Problems with generic errors:**

* **Client confusion**: Can't distinguish between server bugs and client mistakes
* **Poor user experience**: Users don't know how to fix the problem
* **Debugging difficulty**: No distinction between different error types
* **Retry logic broken**: Clients don't know if retrying will help

**✅ Specific error codes:**

```http
POST /users
{
  "email": "invalid-email"
}

422 Unprocessable Entity
{
  "type": "https://api.example.com/errors/validation-error",
  "title": "Validation Error", 
  "status": 422,
  "detail": "The request contains invalid data",
  "errors": [
    {
      "field": "email",
      "code": "INVALID_FORMAT", 
      "message": "Email address must be valid"
    }
  ]
}
```

## Versioning: Plan for Change or Plan for Pain

### URL Versioning vs Header Versioning

**✅ URL versioning (recommended for major changes):**

```http
GET /api/v1/users/123
GET /api/v2/users/123
```

**Advantages:**

* **Visible in URLs**: Easy to see which version is being used
* **Cacheable**: Each version can be cached independently
* **Simple routing**: Web servers can route based on URL path
* **Browser-friendly**: Works with all HTTP tools and browsers

**❌ Header versioning pitfalls:**

```http
GET /api/users/123
Accept: application/vnd.api+json;version=2
```

**Why this can be problematic:**

* **Invisible versioning**: Hard to debug which version is being used
* **Caching complexity**: Proxies must understand version headers
* **Tool limitations**: Many HTTP tools ignore custom headers
* **Support overhead**: Harder to maintain multiple versions simultaneously

### Breaking Changes: What Breaks Clients

**❌ Changes that break existing clients:**

```json
// Version 1
{
  "id": 123,
  "name": "John Doe"
}

// Version 2 (BREAKING!)
{
  "userId": 123,         // Field renamed
  "fullName": "John Doe" // Field renamed
}
```

**❌ More subtle breaking changes:**

```json
// Version 1
{
  "status": "active"    // String value
}

// Version 2 (BREAKING!)
{
  "status": {           // Changed to object
    "value": "active",
    "updatedAt": "2025-09-08T10:30:00Z"
  }
}
```

**✅ Non-breaking evolution:**

```json
// Version 1
{
  "id": 123,
  "name": "John Doe"
}

// Version 1.1 (NON-BREAKING)
{
  "id": 123,
  "name": "John Doe",
  "email": "john@example.com",           // New optional field
  "createdAt": "2025-09-08T10:30:00Z"    // New optional field
}
```

## Security: What Goes Wrong and How

### Authentication Token Exposure

**❌ Tokens in URLs:**

```http
GET /users/123?token=sk_live_1234567890
```

**Why this is dangerous:**

* **Server logs**: Tokens appear in access logs
* **Browser history**: Tokens stored in browser history
* **Referrer headers**: Tokens leaked to external sites
* **Sharing risks**: URLs with tokens accidentally shared

**✅ Tokens in headers:**

```http
GET /users/123
Authorization: Bearer sk_live_1234567890
```

### Rate Limiting: Why and How

**❌ No rate limiting:**

```
// Unlimited requests allowed
```

**Consequences of missing rate limits:**

* **DoS vulnerability**: Attackers can overwhelm your servers
* **Cost explosion**: Cloud bills spike from abuse
* **Performance degradation**: Legitimate users affected by abuse
* **Resource exhaustion**: Database connections and memory consumed

**✅ Proper rate limiting:**

```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 995  
X-RateLimit-Reset: 1694168400
Retry-After: 60
```

### Idempotency: Preventing Duplicate Operations

**❌ No idempotency protection:**

```http
POST /payments
{
  "amount": 100.00,
  "customerId": 123
}
// User hits refresh -> duplicate payment!
```

**Real-world impact:**

* **Duplicate charges**: Customers charged multiple times
* **Race conditions**: Concurrent requests create duplicates
* **Poor user experience**: Users afraid to retry failed requests
* **Support overhead**: Manual cleanup of duplicate operations

**✅ Idempotency keys:**

```http
POST /payments
Idempotency-Key: a1b2c3d4-e5f6-7890-abcd-ef1234567890
{
  "amount": 100.00,
  "customerId": 123
}
// Duplicate requests return same result
```

## Performance: What Kills API Performance

### The N+1 Query Problem

**❌ Naive resource loading:**

```http
GET /orders/123
{
  "id": 123,
  "customerId": 456,                 // Requires another query
  "productIds": [789, 790, 791]      // Requires 3 more queries
}
// Total: 4 database queries for one API call
```

**Why this destroys performance:**

* **Database overload**: Each related resource requires separate query
* **Latency multiplication**: Network round-trips multiply response time
* **Resource exhaustion**: Connection pools depleted quickly
* **Scalability limits**: Performance degrades linearly with data size

**✅ Efficient resource inclusion:**

```http
GET /orders/123?include=customer,products
{
  "id": 123,
  "customer": {
    "id": 456,
    "name": "John Doe"
  },
  "products": [
    {"id": 789, "name": "Widget A"},
    {"id": 790, "name": "Widget B"}
  ]
}
// Total: 1 database query with joins
```

### Caching: What Not to Cache

**❌ Caching everything blindly:**

```http
GET /users/123/current-location
Cache-Control: max-age=3600  # DON'T cache location for 1 hour!
```

**❌ Not caching stable data:**

```http
GET /products/456/specifications  
# No caching headers for rarely-changing product specs
```

**Why caching mistakes hurt:**

* **Stale data**: Users see outdated information
* **Security issues**: Sensitive data cached inappropriately
* **Storage waste**: Caching volatile data wastes memory
* **Performance loss**: Not caching stable data increases load

## Complete OpenAPI Example with Explanations

```yaml
openapi: 3.0.3
info:
  title: E-commerce API
  version: "1.0.0"
  description: |
    Comprehensive e-commerce platform API following REST principles.
    
    ## Design Principles
    - Resource-oriented URLs (nouns, not verbs)
    - Proper HTTP verb usage
    - Consistent error handling with RFC 7807
    - Comprehensive input validation
    - Rate limiting and security headers
    
  contact:
    name: API Support
    url: https://example.com/support
    email: api-support@example.com

servers:
  - url: https://api.example.com/v1
    description: Production server
  - url: https://staging-api.example.com/v1  
    description: Staging server

paths:
  /users:
    get:
      summary: List users with filtering and pagination
      description: |
        Retrieve a paginated list of users. Demonstrates:
        - Query parameter filtering (avoid custom filter syntax)
        - Cursor-based pagination (better than offset for large datasets)  
        - Consistent response format with metadata
      parameters:
        - name: role
          in: query
          description: Filter by user role
          schema:
            type: string
            enum: [admin, customer, vendor]
          example: admin
        - name: status
          in: query  
          description: Filter by account status
          schema:
            type: string
            enum: [active, inactive, suspended]
        - name: limit
          in: query
          description: Number of results per page (max 100 to prevent abuse)
          schema:
            type: integer
            minimum: 1
            maximum: 100  
            default: 20
        - name: cursor
          in: query
          description: |
            Pagination cursor from previous response.
            Use cursor-based pagination instead of offset for:
            - Consistent performance regardless of page position
            - Stable results when data changes during pagination
          schema:
            type: string
            format: base64
      responses:
        '200':
          description: Users retrieved successfully
          headers:
            X-RateLimit-Limit:
              description: Request limit per hour
              schema:
                type: integer
            X-RateLimit-Remaining:
              description: Remaining requests in current window
              schema:
                type: integer
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    type: array
                    items:
                      $ref: '#/components/schemas/User'
                  pagination:
                    $ref: '#/components/schemas/CursorPagination'
        '400':
          $ref: '#/components/responses/BadRequest'
        '429':
          $ref: '#/components/responses/RateLimitExceeded'

    post:
      summary: Create new user
      description: |
        Create a new user account. Demonstrates:
        - Idempotency-Key header to prevent duplicate creation
        - Comprehensive input validation with detailed error responses
        - 201 Created with Location header (not generic 200 OK)
        - Proper authentication requirement
      security:
        - bearerAuth: []
      parameters:
        - name: Idempotency-Key
          in: header
          required: true
          description: |
            UUID to ensure idempotent creation. If you retry with same key:
            - Same user already exists: returns existing user  
            - Creation in progress: waits and returns result
            This prevents duplicate users from network retries or impatient users.
          schema:
            type: string
            format: uuid
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateUserRequest'
      responses:
        '201':
          description: User created successfully
          headers:
            Location:
              schema:
                type: string
              description: URL of the newly created user
              example: /users/1234
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        '400':
          $ref: '#/components/responses/BadRequest'
        '409':
          description: |
            Conflict - user already exists with this email.
            Returns 409 instead of 400 because the request is valid,
            but conflicts with existing data state.
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
              example:
                type: "https://api.example.com/errors/conflict"
                title: "Resource Conflict"
                status: 409
                detail: "User with this email already exists"
                instance: "/users"
        '422':
          $ref: '#/components/responses/ValidationError'

  /users/{userId}:
    parameters:
      - name: userId
        in: path
        required: true
        description: Unique identifier for the user
        schema:
          type: integer
          format: int64
          minimum: 1
    
    get:
      summary: Get user by ID
      description: |
        Retrieve a single user by ID. Demonstrates:
        - Simple resource retrieval
        - ETag header for conditional requests
        - 404 for non-existent resources (not 200 with error message)
      responses:
        '200':
          description: User retrieved successfully
          headers:
            ETag:
              schema:
                type: string
              description: Entity tag for conditional requests
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        '404':
          $ref: '#/components/responses/NotFound'

    put:
      summary: Replace entire user resource
      description: |
        Replace the entire user resource. Demonstrates:
        - PUT for complete replacement (all fields required)
        - If-Match header for optimistic concurrency control
        - Clear distinction from PATCH partial updates
      security:
        - bearerAuth: []
      parameters:
        - name: If-Match
          in: header
          description: |
            ETag value from previous GET request.
            Prevents lost update problem when multiple clients
            modify the same resource concurrently.
          schema:
            type: string
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/UpdateUserRequest'
      responses:
        '200':
          description: User updated successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        '404':
          $ref: '#/components/responses/NotFound'
        '412':
          description: |
            Precondition failed - ETag doesn't match.
            The resource was modified by another client.
            Client should GET fresh data and retry.
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '422':
          $ref: '#/components/responses/ValidationError'

    patch:
      summary: Partially update user
      description: |  
        Update specific user fields. Demonstrates:
        - PATCH for partial updates (only specified fields changed)
        - JSON Merge Patch format (simpler than JSON Patch)
        - Null values explicitly clear fields
      security:
        - bearerAuth: []
      requestBody:
        required: true
        content:
          application/merge-patch+json:
            schema:
              $ref: '#/components/schemas/PatchUserRequest'
            example:
              status: "inactive"
              lastLoginAt: null  # Explicitly clear this field
      responses:
        '200':
          description: User updated successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        '404':
          $ref: '#/components/responses/NotFound'

    delete:
      summary: Delete user account
      description: |
        Delete a user account. Demonstrates:
        - 204 No Content for successful deletion
        - 409 Conflict when deletion isn't allowed
        - Idempotent operation (deleting deleted user returns 204)
      security:
        - bearerAuth: []
      responses:
        '204':
          description: User deleted successfully (or was already deleted)
        '404':
          $ref: '#/components/responses/NotFound'
        '409':
          description: |
            Cannot delete user - has active orders or other dependencies.
            Returns specific error instead of generic "deletion failed".
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
              example:
                type: "https://api.example.com/errors/delete-constraint"
                title: "Cannot Delete Resource"
                status: 409
                detail: "User has 3 active orders and cannot be deleted"
                instance: "/users/123"

components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
      description: |
        JWT token in Authorization header.
        Never put authentication tokens in URL parameters - 
        they appear in logs, browser history, and referrer headers.

  schemas:
    User:
      type: object
      description: User resource with all public fields
      properties:
        id:
          type: integer
          format: int64
          example: 123
          description: Unique user identifier
        name:
          type: string
          example: "John Doe"
          description: User's full name
        email:
          type: string
          format: email
          example: "john@example.com"
          description: User's email address (unique)
        role:
          type: string
          enum: [admin, customer, vendor]
          description: User's role in the system
        status:
          type: string
          enum: [active, inactive, suspended]
          description: Current account status
        createdAt:
          type: string
          format: date-time
          description: Account creation timestamp
        updatedAt:
          type: string
          format: date-time
          description: Last modification timestamp
      required:
        - id
        - name  
        - email
        - role
        - status
        - createdAt
        - updatedAt

    CreateUserRequest:
      type: object
      description: Request body for creating new user
      properties:
        name:
          type: string
          minLength: 1
          maxLength: 100
          example: "John Doe"
          description: User's full name
        email:
          type: string
          format: email
          example: "john@example.com"  
          description: Unique email address
        role:
          type: string
          enum: [customer, vendor]
          default: customer
          description: |
            Initial user role. Admin role can only be assigned
            by existing admins via PATCH endpoint.
        password:
          type: string
          minLength: 8
          format: password
          description: |
            User's password. Must be at least 8 characters.
            Will be hashed before storage.
      required:
        - name
        - email
        - password

    UpdateUserRequest:
      type: object
      description: |
        Complete user data for PUT replacement.
        All fields are required because PUT replaces entire resource.
        Use PATCH for partial updates.
      properties:
        name:
          type: string
          minLength: 1
          maxLength: 100
        email:
          type: string
          format: email
        role:
          type: string
          enum: [admin, customer, vendor]
        status:
          type: string
          enum: [active, inactive, suspended]
      required:
        - name
        - email  
        - role
        - status

    PatchUserRequest:
      type: object
      description: |
        Partial user data for PATCH updates.
        Only include fields you want to change.
        Set fields to null to explicitly clear them.
      properties:
        name:
          type: string
          minLength: 1
          maxLength: 100
          nullable: true
        email:
          type: string
          format: email
          nullable: true
        role:
          type: string
          enum: [admin, customer, vendor]
          nullable: true
        status:
          type: string
          enum: [active, inactive, suspended]
          nullable: true

    CursorPagination:
      type: object
      description: |
        Cursor-based pagination metadata.
        Use cursors instead of offset/limit for:
        - Consistent performance regardless of page position
        - Stable results when data changes during pagination
      properties:
        limit:
          type: integer
          description: Requested page size
        hasNext:
          type: boolean
          description: Whether more results are available
        hasPrev:
          type: boolean
          description: Whether previous results are available
        nextCursor:
          type: string
          nullable: true
          description: Cursor for next page (null if no next page)
        prevCursor:
          type: string
          nullable: true
          description: Cursor for previous page (null if no previous page)

    ErrorResponse:
      type: object
      description: |
        Standardized error response following RFC 7807.
        Provides machine-readable error details that clients
        can use for proper error handling and user messaging.
      properties:
        type:
          type: string
          format: uri
          description: URI identifying the error type
        title:
          type: string
          description: Human-readable error summary
        status:
          type: integer
          description: HTTP status code
        detail:
          type: string
          description: Human-readable error explanation
        instance:
          type: string
          description: URI reference identifying specific error occurrence
        errors:
          type: array
          description: Detailed validation errors (for 422 responses)
          items:
            type: object
            properties:
              field:
                type: string
                description: Field that caused the error
              code:
                type: string
                description: Machine-readable error code
              message:
                type: string
                description: Human-readable error message

  responses:
    BadRequest:
      description: |
        Bad request - client sent invalid data.
        Use 400 for malformed JSON, invalid parameters, etc.
        Use 422 for valid JSON with semantic validation errors.
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'
          example:
            type: "https://api.example.com/errors/bad-request"
            title: "Bad Request"
            status: 400
            detail: "Request body contains malformed JSON"
            instance: "/users"

    NotFound:
      description: |
        Resource not found.
        Use 404 for non-existent resources, not for authorization failures.
        Don't return 200 with error message - use proper status codes.
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'
          example:
            type: "https://api.example.com/errors/not-found"
            title: "Resource Not Found" 
            status: 404
            detail: "User with ID 123 does not exist"
            instance: "/users/123"

    ValidationError:
      description: |
        Validation error - request is well-formed but contains invalid data.
        Use 422 instead of 400 for semantic validation failures.
        Provide detailed field-level error information.
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'
          example:
            type: "https://api.example.com/errors/validation-error"
            title: "Validation Error"
            status: 422
            detail: "Request contains invalid field values"
            instance: "/users"
            errors:
              - field: "email"
                code: "INVALID_FORMAT"
                message: "Email address must be valid"
              - field: "name"  
                code: "TOO_SHORT"
                message: "Name must be at least 1 character"

    RateLimitExceeded:
      description: |
        Too many requests - client exceeded rate limit.
        Include headers showing limit details and retry timing.
        Use 429 specifically for rate limiting, not generic 400.
      headers:
        X-RateLimit-Limit:
          description: Requests allowed per hour
          schema:
            type: integer
        X-RateLimit-Remaining:
          description: Requests remaining in current window
          schema:
            type: integer
        X-RateLimit-Reset:
          description: Unix timestamp when limit resets
          schema:
            type: integer
        Retry-After:
          description: Seconds to wait before retrying
          schema:
            type: integer
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'
          example:
            type: "https://api.example.com/errors/rate-limit-exceeded"
            title: "Rate Limit Exceeded"
            status: 429
            detail: "Request limit of 1000 per hour exceeded"
            instance: "/users"
```

## Real-World Implementation Pitfalls and Solutions

### The Temptation of Generic Endpoints

**❌ The "do everything" endpoint:**

```http
POST /api/execute
{
  "action": "get_user", 
  "params": {"id": 123}
}

POST /api/execute  
{
  "action": "create_order",
  "params": {"userId": 123, "amount": 99.99}
}
```

**Why this approach always fails:**

* **Loss of HTTP semantics**: Can't use caching, proper status codes, or HTTP methods
* **Monitoring nightmare**: Can't differentiate between different operations in metrics
* **Security complexity**: Can't apply different rate limits or permissions per operation
* **Documentation chaos**: Single endpoint documentation becomes massive and confusing
* **Client complexity**: Clients must build custom abstraction layers

**✅ Proper resource-based design:**

```http
GET /users/123
POST /orders
```

### The False Economy of Fewer Endpoints

**❌ Cramming multiple operations into one endpoint:**

```http
PATCH /users/123
{
  "operation": "update_profile",
  "data": {"name": "John Doe"}
}

PATCH /users/123
{
  "operation": "change_password", 
  "data": {"newPassword": "secret123"}
}
```

**Why this penny-wise, pound-foolish approach costs more:**

* **Mixed security requirements**: Password changes need different validation than profile updates
* **Different rate limits needed**: Password changes should be more restricted
* **Audit trail confusion**: Can't easily track what type of changes occurred
* **Testing complexity**: Single endpoint must handle multiple unrelated scenarios
* **Client error handling**: Different operations need different error recovery strategies

**✅ Separate endpoints for separate concerns:**

```http
PATCH /users/123/profile
{
  "name": "John Doe"
}

POST /users/123/password-reset
{
  "currentPassword": "old123",
  "newPassword": "secret123"
}
```

### Performance Anti-Patterns That Kill APIs

#### The Chatty API Problem

**❌ Requiring multiple requests for common operations:**

```http
# Client needs order with customer info and products
GET /orders/123                    # Request 1: Get order
GET /users/456                     # Request 2: Get customer  
GET /products/789                  # Request 3: Get product 1
GET /products/790                  # Request 4: Get product 2
GET /products/791                  # Request 5: Get product 3
```

**Real-world impact:**

* **Mobile performance**: Each request adds network latency
* **Server load**: 5x more requests than necessary
* **Race conditions**: Data might change between requests
* **Complexity explosion**: Client must manage multiple async requests

**✅ Compound documents with includes:**

```http
GET /orders/123?include=customer,products
{
  "id": 123,
  "status": "shipped",
  "customer": {
    "id": 456,
    "name": "John Doe",
    "email": "john@example.com"
  },
  "products": [
    {"id": 789, "name": "Widget A", "price": 29.99},
    {"id": 790, "name": "Widget B", "price": 39.99}
  ]
}
```

#### The Overfetching Problem

**❌ Always returning everything:**

```http
GET /users
[
  {
    "id": 123,
    "name": "John Doe",
    "email": "john@example.com",
    "address": {...},           # Not needed for user list
    "paymentMethods": [...],    # Sensitive data not needed
    "orderHistory": [...],      # Expensive to compute
    "preferences": {...},       # Not displayed in UI
    "auditLog": [...]           # Should never be in public API
  }
]
```

**Why this destroys performance:**

* **Bandwidth waste**: Mobile users pay for unnecessary data
* **Database load**: Expensive joins for unused data
* **Memory usage**: Servers use more RAM for large responses
* **Security risk**: Accidental exposure of sensitive data
* **Caching inefficiency**: Large responses harder to cache effectively

**✅ Field selection and appropriate defaults:**

```http
# Default response with essential fields only
GET /users
[
  {
    "id": 123,
    "name": "John Doe", 
    "email": "john@example.com",
    "status": "active"
  }
]

# Explicit field selection for specific needs
GET /users/123?fields=id,name,email,preferences,address
```

### Security Mistakes That Get You Hacked

#### Authentication Token Leakage

**❌ Tokens everywhere except where they belong:**

```http
# In URL (appears in logs, browser history, referrer headers)
GET /users/123?token=sk_live_1234567890

# In response body (might be cached or logged)  
{
  "user": {...},
  "authToken": "sk_live_1234567890"
}

# In cookies without proper flags
Set-Cookie: token=sk_live_1234567890
```

**Real consequences:**

* **Server log exposure**: Tokens in access logs accessible to ops team
* **Browser history**: Users' tokens stored in browser history indefinitely
* **Referrer leakage**: Tokens sent to external sites in referrer headers
* **XSS vulnerability**: JavaScript can access tokens in responses or insecure cookies
* **MITM attacks**: Tokens sent over HTTP or without secure flags

**✅ Secure token handling:**

```http
# In Authorization header only
GET /users/123
Authorization: Bearer sk_live_1234567890

# Secure cookie flags when using cookies
Set-Cookie: token=sk_live_1234567890; HttpOnly; Secure; SameSite=Strict
```

#### The Authorization Bypass

**❌ Trusting client-provided data for authorization:**

```http
# Client says which user they are
GET /users/123/orders
X-User-ID: 123

# Server trusts this without verification
```

**❌ Inconsistent authorization checking:**

```http
# Authorization checked on GET but not PUT
GET /users/123    # ✓ Checks if user can view profile
PUT /users/123    # ✗ Forgets to check if user can edit profile
```

**Why these patterns lead to breaches:**

* **Horizontal privilege escalation**: Users access other users' data
* **Vertical privilege escalation**: Regular users perform admin operations
* **Data manipulation**: Unauthorized modification of sensitive information
* **Compliance violations**: GDPR, HIPAA, and other regulations violated

**✅ Consistent server-side authorization:**

```http
# Server extracts user ID from validated JWT token
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
# JWT contains: {"userId": 456, "role": "customer"}

# Server checks: Does user 456 have permission to access user 123's data?
GET /users/123/orders
```

### Error Handling: Information vs Security

#### The TMI (Too Much Information) Problem

**❌ Helpful error messages that help attackers:**

```http
POST /login
{
  "email": "hacker@evil.com",
  "password": "wrong"
}

400 Bad Request
{
  "error": "User hacker@evil.com does not exist"  
}

POST /login 
{
  "email": "john@example.com",
  "password": "wrong"  
}

400 Bad Request
{
  "error": "Invalid password for user john@example.com"
}
```

**Security implications:**

* **User enumeration**: Attackers can discover valid email addresses
* **Information disclosure**: Different errors reveal system internals
* **Brute force facilitation**: Attackers know when they have valid usernames
* **Social engineering**: Attackers can target known user accounts

**✅ Consistent, non-revealing error messages:**

```http
POST /login
{
  "email": "anyone@anywhere.com",
  "password": "anything"
}

401 Unauthorized  
{
  "type": "https://api.example.com/errors/authentication-failed",
  "title": "Authentication Failed",
  "status": 401,
  "detail": "Invalid email or password",
  "instance": "/login"
}
```

#### The Debugging Information Leak

**❌ Development error details in production:**

```http
500 Internal Server Error
{
  "error": "SQL Error",
  "message": "Table 'users' doesn't exist",
  "stackTrace": [
    "at DatabaseConnection.query (/app/db.js:45:12)",
    "at UserService.findById (/app/users.js:23:8)"
  ],
  "sql": "SELECT * FROM users WHERE id = 123"
}
```

**What attackers learn:**

* **Database schema**: Table names and structure
* **File structure**: Application file paths and organization
* **Technology stack**: Database type, framework, dependencies
* **Potential vulnerabilities**: SQL injection points, file paths

**✅ Generic errors with internal logging:**

```http
500 Internal Server Error
{
  "type": "https://api.example.com/errors/internal-server-error",
  "title": "Internal Server Error", 
  "status": 500,
  "detail": "An unexpected error occurred. Please try again later.",
  "instance": "/users/123",
  "errorId": "err_1234567890"  # For support team lookup
}
```

## Testing Your API Design Decisions

### Load Testing Reveals Design Flaws

**Test scenario: 1000 concurrent users listing orders**

❌ **Poor design fails under load:**

```http
GET /orders  # Returns all orders for all users
# Database query: SELECT * FROM orders ORDER BY created_at DESC
# Result: Full table scan, 50MB response, server crashes
```

✅ **Good design scales:**

```http
GET /orders?limit=20&cursor=xyz  # Paginated, filtered results
# Database query: SELECT * FROM orders WHERE user_id = ? AND id > ? LIMIT 20
# Result: Index-optimized query, 50KB response, handles load
```

### API Versioning Under Real-World Pressure

**Scenario: Need to change user status from string to object**

❌ **Breaking change forces all clients to update:**

```json
// Version 1
{"status": "active"}

// Version 2 (BREAKING!)  
{"status": {"value": "active", "reason": "email_verified"}}
```

**Result**: Mobile apps break, third-party integrations fail, emergency rollback required.

✅ **Backward-compatible evolution:**

```json
// Version 1 (unchanged)
{"status": "active"}

// Version 1.1 (additive)
{
  "status": "active",
  "statusDetails": {"value": "active", "reason": "email_verified"}
}
```

**Result**: Existing clients continue working, new clients can use enhanced data.

### Security Testing Exposes Real Vulnerabilities

**Test: Can user A access user B's data?**

❌ **Insecure by default:**

```http
GET /users/456/orders
Authorization: Bearer <user-123-token>

200 OK  # Should be 403 Forbidden!
[
  {"id": 789, "amount": 99.99, "userId": 456}
]
```

✅ **Secure by default:**

```http
GET /users/456/orders  
Authorization: Bearer <user-123-token>

403 Forbidden
{
  "type": "https://api.example.com/errors/forbidden",
  "title": "Forbidden",
  "status": 403,
  "detail": "You do not have permission to access this resource"
}
```

## Monitoring and Observability: What You Can't See Will Hurt You

### Metrics That Matter

**❌ Vanity metrics that hide problems:**

* Total API calls (growing is good, right?)
* Average response time (hides outliers)
* HTTP 200 rate (ignores business logic failures)

**✅ Actionable metrics that prevent outages:**

* P95/P99 response times (catches performance degradation)
* Error rate by endpoint and status code (identifies problem areas)
* Rate limit violations by user (spots abuse patterns)
* Database query time per endpoint (finds N+1 problems)

### Logging for Debuggability vs Privacy

**❌ Logging everything:**

```
INFO: GET /users/123 - Response: {"name":"John Doe","email":"john@example.com","ssn":"123-45-6789"}
```

**❌ Logging nothing:**

```
INFO: Request processed
```

**✅ Strategic logging:**

```
INFO: GET /users/123 - User: 123, Duration: 45ms, Status: 200
ERROR: GET /users/456 - User: 123, Error: NotFound, Duration: 12ms, ErrorId: err_abc123
```

## The Economics of Good API Design

### Short-Term Costs vs Long-Term Benefits

**Poor API design appears cheaper initially:**

* Faster initial development
* Fewer upfront decisions needed
* Less documentation required
* Simpler implementation

**But costs compound:**

* **Support overhead**: More tickets from confused developers
* **Breaking changes**: Expensive migrations and client updates
* **Performance issues**: Emergency optimizations and infrastructure costs
* **Security incidents**: Breach response and compliance violations
* **Developer churn**: Team members frustrated by technical debt

**Good API design investment pays dividends:**

* **Reduced support burden**: Self-documenting, predictable APIs
* **Easier evolution**: Non-breaking changes and smooth version transitions
* **Better performance**: Efficient queries and caching from day one
* **Security by default**: Consistent authorization and input validation
* **Developer productivity**: Teams move faster with clear patterns

### The True Cost of "Just Fix It Later"

**"We'll optimize the performance later"**

* Later: Database replication needed (\$2000/month)
* Later: CDN required for large responses (\$500/month)
* Later: Caching layer to fix N+1 queries (\$1000/month)
* **Cost of fixing later: \$3500/month ongoing**
* **Cost of designing correctly upfront: 2 extra development days**

**"We'll add proper error handling later"**

* Later: Customer support overwhelmed by "something went wrong" tickets
* Later: Debugging time increases 300% due to generic errors
* Later: Security audit fails due to information disclosure
* **Cost of fixing later: 6 months of reduced team productivity**
* **Cost of designing correctly upfront: 1 week of development time**

---

This comprehensive approach to REST API design provides the practical knowledge and cautionary tales needed to build production-ready APIs that scale. The key is understanding not just what to do, but why alternatives fail and what the real-world consequences are.
