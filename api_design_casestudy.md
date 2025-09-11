# REST API Design Principles: Real-World Telemetry System

## Case Study: Enterprise Telemetry Collection API

You're building a telemetry API that collects logs from thousands of on-premise servers and stores them in a central event bus in the cloud. This system must handle:
- 50,000 servers sending data every minute
- Network failures and retries from unreliable connections
- Duplicate prevention during network outages
- High availability during peak load periods

Let's see how REST design principles apply in practice.

## The Business Problem

**Scenario**: TechCorp has 50,000 servers across 200 data centers. Each server generates system metrics (CPU, memory, disk) and application logs every minute. The data flows to a cloud-based analytics platform.

**Critical requirements:**
- No duplicate telemetry data (affects billing and analytics)
- Handle network failures gracefully
- Scale to handle 50M requests/hour during peak monitoring
- Support both real-time streaming and batch uploads

## Resource Orientation: Getting the URL Structure Right

### ❌ Action-Based URLs (Common Mistake)

```http
POST /api/sendTelemetry
POST /api/uploadLogs  
POST /api/submitMetrics
POST /api/pushEvents
```

**Why this fails in production:**
```javascript
// Operations team confusion:
"How do I check if telemetry was received?"
"What's the endpoint to get logs for server-123?"
"How do I retry failed uploads?"

// Client code becomes inconsistent:
telemetryClient.sendTelemetry(data);
telemetryClient.uploadLogs(logs);
telemetryClient.submitMetrics(metrics); // Three different patterns!
```

### ✅ Resource-Based URLs (Correct Approach)

```http
POST /servers/{serverId}/telemetry     # Create telemetry data for server
GET  /servers/{serverId}/telemetry     # Retrieve telemetry data for server
POST /servers/{serverId}/logs          # Create log entries for server
GET  /servers/{serverId}/logs          # Retrieve logs for server
```

**Real-world benefits:**
```javascript
// Predictable client code:
class TelemetryClient {
  async sendMetrics(serverId, metrics) {
    return this.post(`/servers/${serverId}/telemetry`, metrics);
  }
  
  async getLogs(serverId, timeRange) {
    return this.get(`/servers/${serverId}/logs?${timeRange}`);
  }
  
  // New developers can guess the pattern:
  async getMetrics(serverId) {
    return this.get(`/servers/${serverId}/telemetry`); // Obvious!
  }
}
```

## HTTP Verbs: Real Network Failure Scenarios

### The Telemetry Retry Problem

**Real scenario**: Server in Mumbai data center has intermittent connectivity. It collects 1 minute of telemetry data but network fails during upload.

**❌ Using GET for data submission (Actual mistake seen in production):**
```http
GET /api/telemetry?serverId=server-mumbai-01&cpu=85&memory=78&timestamp=1694168400
```

**What goes wrong:**
```bash
# This appears in Apache access logs:
[10/Sep/2025] GET /api/telemetry?serverId=server-mumbai-01&cpu=85&memory=78 200
[10/Sep/2025] GET /api/telemetry?serverId=server-mumbai-01&cpu=85&memory=78 200  # Duplicate from retry!
[10/Sep/2025] GET /api/telemetry?serverId=server-mumbai-01&cpu=85&memory=78 200  # Another duplicate!

# Result: CPU shows as 255% (3x 85%) in dashboard due to duplicates
```

**Security issue:**
```bash
# Sensitive data exposed in referrer headers when user clicks external links:
Referer: https://telemetry.techcorp.com/api/telemetry?serverId=prod-db-primary&cpu=95&memory=88
# Now external site knows your production database is under load!
```

**✅ Correct POST usage:**
```http
POST /servers/server-mumbai-01/telemetry
Content-Type: application/json

{
  "timestamp": "2025-09-10T10:30:00Z",
  "metrics": {
    "cpu": 85,
    "memory": 78,
    "disk": 45
  }
}
```

## Idempotency in Telemetry: The Duplicate Data Crisis

### The Real Problem: Network Failures Create Billing Disasters

**Scenario**: During a network storm, servers retry telemetry uploads multiple times.

```javascript
// On-premise server code (runs on 50,000 servers)
class TelemetryAgent {
  async sendTelemetry() {
    const metrics = this.collectMetrics();
    
    // Without idempotency - DISASTER:
    for (let attempt = 1; attempt <= 5; attempt++) {
      try {
        await this.httpClient.post('/servers/server-001/telemetry', metrics);
        console.log('Telemetry sent successfully');
        return; // Success!
      } catch (networkError) {
        console.log(`Attempt ${attempt} failed, retrying...`);
        await sleep(1000 * attempt);
        // Retry logic creates duplicates!
      }
    }
  }
}
```

**Financial impact of duplicates:**
```
Real incident at unnamed company:
- 10,000 servers experienced network issues during storm
- Each server retried 3-5 times over 2 hours  
- 40,000 duplicate telemetry records created
- Cloud storage billing: $15,000 extra charges that month
- Analytics dashboards showed 400% CPU spikes (false alarms)
- 3 emergency incident responses triggered unnecessarily
```

### Implementation: Two-Level Deduplication Strategy

**Level 1: HTTP Request Deduplication (Technical)**

```javascript
// Client-side: Generate idempotency key per retry attempt
class TelemetryAgent {
  async sendTelemetry() {
    const metrics = this.collectMetrics();
    const idempotencyKey = this.generateRequestId(); // UUID per logical request
    
    for (let attempt = 1; attempt <= 5; attempt++) {
      try {
        const response = await fetch(`/servers/${this.serverId}/telemetry`, {
          method: 'POST',
          headers: {
            'Idempotency-Key': idempotencyKey,  // SAME key for all retries
            'Content-Type': 'application/json'
          },
          body: JSON.stringify({
            collectionTimestamp: metrics.timestamp,
            serverInstanceId: this.instanceId,    // Business reference
            metrics: metrics.data
          })
        });
        
        if (response.status === 201) {
          console.log('New telemetry recorded');
        } else if (response.status === 200) {
          console.log('Duplicate request - returning existing record');
        }
        return;
        
      } catch (networkError) {
        console.log(`Network attempt ${attempt} failed`);
        // Next retry uses SAME idempotencyKey
      }
    }
  }
  
  generateRequestId() {
    // Generate once per collection cycle
    return `${this.serverId}_${Date.now()}_${crypto.randomUUID()}`;
  }
}
```

**Level 2: Business-Level Deduplication (Data Integrity)**

```javascript
// Server-side implementation
class TelemetryAPI {
  async handleTelemetrySubmission(req, res) {
    const { serverId } = req.params;
    const idempotencyKey = req.headers['idempotency-key'];
    const { collectionTimestamp, serverInstanceId, metrics } = req.body;
    
    // Step 1: Check HTTP-level idempotency (handles retries)
    const cachedResponse = await this.checkHttpIdempotency(idempotencyKey);
    if (cachedResponse) {
      return res.status(cachedResponse.status).json(cachedResponse.body);
    }
    
    // Step 2: Check business-level deduplication (handles duplicate data)
    const existingTelemetry = await this.findExistingTelemetry(
      serverId, 
      collectionTimestamp, 
      serverInstanceId
    );
    
    if (existingTelemetry) {
      // Same server, same timestamp = duplicate data
      const response = { 
        telemetryId: existingTelemetry.id,
        status: 'duplicate_data',
        message: 'Telemetry already recorded for this timestamp'
      };
      
      // Cache this response for HTTP retries
      await this.storeHttpIdempotency(idempotencyKey, response, 200);
      return res.status(200).json(response);
    }
    
    // Step 3: Store new telemetry
    const telemetry = await this.storeTelemetry({
      serverId,
      collectionTimestamp,
      serverInstanceId,
      metrics,
      receivedAt: new Date()
    });
    
    // Step 4: Send to event bus
    await this.publishToEventBus(telemetry);
    
    const response = {
      telemetryId: telemetry.id,
      status: 'recorded',
      eventBusMessageId: telemetry.eventBusId
    };
    
    // Cache success response
    await this.storeHttpIdempotency(idempotencyKey, response, 201);
    
    res.status(201).json(response);
  }
  
  async findExistingTelemetry(serverId, timestamp, instanceId) {
    // Business logic: telemetry is duplicate if same server + timestamp + instance
    return await db.query(`
      SELECT id FROM telemetry 
      WHERE server_id = ? 
        AND collection_timestamp = ? 
        AND server_instance_id = ?
    `, [serverId, timestamp, instanceId]);
  }
}
```

### Database Schema: Supporting Both Deduplication Levels

```sql
-- HTTP idempotency table (short-term, technical)
CREATE TABLE http_idempotency (
  key VARCHAR(255) PRIMARY KEY,
  response_body JSON NOT NULL,
  response_status INT NOT NULL,
  created_at TIMESTAMP DEFAULT NOW(),
  expires_at TIMESTAMP DEFAULT (NOW() + INTERVAL 1 HOUR), -- Short TTL
  
  INDEX idx_expires (expires_at)
);

-- Business data table (long-term, permanent)
CREATE TABLE telemetry (
  id VARCHAR(255) PRIMARY KEY,
  server_id VARCHAR(255) NOT NULL,
  collection_timestamp TIMESTAMP NOT NULL,
  server_instance_id VARCHAR(255) NOT NULL,  -- Handle server restarts
  metrics JSON NOT NULL,
  received_at TIMESTAMP DEFAULT NOW(),
  event_bus_message_id VARCHAR(255),
  
  -- Business uniqueness constraint
  UNIQUE KEY unique_telemetry (server_id, collection_timestamp, server_instance_id),
  INDEX idx_server_time (server_id, collection_timestamp)
);

-- Automatic cleanup for HTTP idempotency
CREATE EVENT cleanup_http_idempotency
ON SCHEDULE EVERY 1 HOUR
DO DELETE FROM http_idempotency WHERE expires_at < NOW();
```

## Query Parameters: Handling High-Volume Telemetry Queries

### The Performance Problem: Fetching Telemetry Data

**❌ Naive querying approach:**
```http
GET /servers/server-001/telemetry  # Returns ALL telemetry data ever!
```

**Production disaster:**
```javascript
// After 6 months of operation:
const response = await fetch('/servers/server-001/telemetry');
// Response size: 2.3GB (525,600 minutes × 4KB per record)
// Browser crashes, mobile apps crash, network timeouts
// Database query: 45 seconds execution time
```

**✅ Proper pagination and filtering:**
```http
# Time-based filtering (essential for telemetry)
GET /servers/server-001/telemetry?from=2025-09-10T08:00:00Z&to=2025-09-10T09:00:00Z

# Metric-specific queries
GET /servers/server-001/telemetry?metrics=cpu,memory&from=2025-09-10T08:00:00Z

# Cursor-based pagination for real-time streams
GET /servers/server-001/telemetry?limit=100&cursor=eyJ0aW1lc3RhbXAiOiIyMDI1LTA5LTEwVDA5OjAwOjAwWiJ9

# Aggregated data for dashboards
GET /servers/server-001/telemetry?aggregate=5min&from=2025-09-10T00:00:00Z&to=2025-09-10T23:59:59Z
```

**Optimized server implementation:**
```javascript
class TelemetryQueryAPI {
  async getTelemetry(req, res) {
    const { serverId } = req.params;
    const { from, to, metrics, aggregate, limit = 1000, cursor } = req.query;
    
    // Validate time range (prevent massive queries)
    if (!from || !to) {
      return res.status(400).json({
        error: 'from and to timestamps required',
        example: '/servers/server-001/telemetry?from=2025-09-10T08:00:00Z&to=2025-09-10T09:00:00Z'
      });
    }
    
    const timeRange = new Date(to) - new Date(from);
    const maxRange = 24 * 60 * 60 * 1000; // 24 hours
    
    if (timeRange > maxRange) {
      return res.status(400).json({
        error: 'Time range cannot exceed 24 hours',
        maxRange: '24h',
        requested: `${Math.round(timeRange / (60 * 60 * 1000))}h`
      });
    }
    
    // Build optimized query
    let query = `
      SELECT collection_timestamp, metrics 
      FROM telemetry 
      WHERE server_id = ? 
        AND collection_timestamp BETWEEN ? AND ?
    `;
    
    const params = [serverId, from, to];
    
    // Add cursor for pagination
    if (cursor) {
      const cursorData = JSON.parse(Buffer.from(cursor, 'base64').toString());
      query += ' AND collection_timestamp > ?';
      params.push(cursorData.timestamp);
    }
    
    query += ' ORDER BY collection_timestamp ASC LIMIT ?';
    params.push(parseInt(limit));
    
    const results = await db.query(query, params);
    
    // Filter metrics if specified
    if (metrics) {
      const requestedMetrics = metrics.split(',');
      results.forEach(row => {
        const filteredMetrics = {};
        requestedMetrics.forEach(metric => {
          if (row.metrics[metric] !== undefined) {
            filteredMetrics[metric] = row.metrics[metric];
          }
        });
        row.metrics = filteredMetrics;
      });
    }
    
    // Generate next cursor
    let nextCursor = null;
    if (results.length === limit) {
      const lastTimestamp = results[results.length - 1].collection_timestamp;
      nextCursor = Buffer.from(JSON.stringify({ timestamp: lastTimestamp })).toString('base64');
    }
    
    res.json({
      data: results,
      pagination: {
        limit: parseInt(limit),
        hasNext: nextCursor !== null,
        nextCursor
      },
      timeRange: { from, to },
      serverCount: results.length
    });
  }
}
```

## Error Handling: Production Telemetry Failures

### Security vs Debugging: The Information Disclosure Problem

**❌ Too much information in errors:**
```http
POST /servers/prod-db-01/telemetry

500 Internal Server Error
{
  "error": "Database connection failed",
  "details": "Connection refused to mysql://admin:secret123@db-cluster-prod-1.internal:3306/telemetry",
  "stackTrace": [
    "at TelemetryService.save (/opt/telemetry-api/src/services/telemetry.js:45)",
    "at /opt/telemetry-api/src/routes/telemetry.js:123"
  ],
  "query": "INSERT INTO telemetry (server_id, metrics) VALUES ('prod-db-01', '{\"cpu\": 95}')"
}
```

**What attackers learn:**
- Database credentials (`admin:secret123`)
- Internal network topology (`db-cluster-prod-1.internal`)
- File system structure (`/opt/telemetry-api/src/`)
- Database schema (`telemetry` table structure)
- High-value targets (`prod-db-01` is likely important)

**✅ Security-conscious error responses:**
```javascript
class TelemetryErrorHandler {
  handleError(error, req, res) {
    const errorId = `err_${Date.now()}_${crypto.randomUUID().slice(0, 8)}`;
    
    // Log full details internally
    console.error(`Error ${errorId}:`, {
      error: error.message,
      stack: error.stack,
      request: {
        url: req.url,
        method: req.method,
        headers: this.sanitizeHeaders(req.headers),
        body: req.body
      },
      server: process.env.HOSTNAME,
      timestamp: new Date().toISOString()
    });
    
    // Return minimal public error
    const publicError = this.getPublicError(error, errorId);
    res.status(publicError.status).json(publicError.body);
  }
  
  getPublicError(error, errorId) {
    if (error.name === 'ValidationError') {
      return {
        status: 422,
        body: {
          type: 'https://api.techcorp.com/errors/validation-error',
          title: 'Validation Error',
          status: 422,
          detail: 'Request contains invalid telemetry data',
          instance: `/servers/${req.params.serverId}/telemetry`,
          errorId,
          errors: error.details // Safe - validation errors don't expose internals
        }
      };
    }
    
    if (error.name === 'DatabaseConnectionError') {
      return {
        status: 503,
        body: {
          type: 'https://api.techcorp.com/errors/service-unavailable',
          title: 'Service Temporarily Unavailable',
          status: 503,
          detail: 'Telemetry service is temporarily unavailable. Please retry.',
          instance: `/servers/${req.params.serverId}/telemetry`,
          errorId,
          retryAfter: 60
        }
      };
    }
    
    // Generic error for everything else
    return {
      status: 500,
      body: {
        type: 'https://api.techcorp.com/errors/internal-server-error',
        title: 'Internal Server Error',
        status: 500,
        detail: 'An unexpected error occurred. Please contact support with error ID.',
        instance: `/servers/${req.params.serverId}/telemetry`,
        errorId,
        supportContact: 'telemetry-support@techcorp.com'
      }
    };
  }
}
```

## Rate Limiting: Protecting Against Telemetry Floods

### The Telemetry Storm Problem

**Real scenario**: Configuration error causes 10,000 servers to send telemetry every second instead of every minute.

```javascript
// Buggy configuration deployed to production:
const config = {
  telemetryInterval: 1000, // Should be 60000 (1 minute), not 1000 (1 second)!
};

// Result: 
// - Normal load: 50,000 servers × 1 request/minute = 833 requests/second
// - Bug load: 50,000 servers × 60 requests/minute = 50,000 requests/second
// - API servers crash, database overwhelmed, $50,000 cloud bill spike
```

**✅ Multi-level rate limiting:**
```javascript
class TelemetryRateLimiter {
  constructor() {
    this.globalLimiter = new RateLimiter({
      windowMs: 60 * 1000,        // 1 minute window
      max: 100000,                // Max 100k requests per minute globally
      message: 'Global rate limit exceeded'
    });
    
    this.perServerLimiter = new RateLimiter({
      windowMs: 60 * 1000,        // 1 minute window  
      max: 2,                     // Max 2 telemetry submissions per server per minute
      keyGenerator: (req) => `server:${req.params.serverId}`,
      message: 'Per-server rate limit exceeded'
    });
    
    this.perIPLimiter = new RateLimiter({
      windowMs: 60 * 1000,        // 1 minute window
      max: 100,                   // Max 100 requests per IP per minute
      keyGenerator: (req) => req.ip,
      message: 'IP rate limit exceeded'
    });
  }
  
  async checkLimits(req, res, next) {
    try {
      // Check global limits first (fail fast)
      await this.globalLimiter.consume(req);
      
      // Check per-server limits (prevent misconfigured servers)
      await this.perServerLimiter.consume(req);
      
      // Check per-IP limits (prevent abuse)
      await this.perIPLimiter.consume(req);
      
      next();
    } catch (rateLimitError) {
      const resetTime = new Date(Date.now() + rateLimitError.msBeforeNext);
      
      res.set({
        'X-RateLimit-Limit': rateLimitError.totalHits,
        'X-RateLimit-Remaining': rateLimitError.remainingPoints,
        'X-RateLimit-Reset': resetTime.toISOString(),
        'Retry-After': Math.round(rateLimitError.msBeforeNext / 1000)
      });
      
      res.status(429).json({
        type: 'https://api.techcorp.com/errors/rate-limit-exceeded',
        title: 'Rate Limit Exceeded',
        status: 429,
        detail: rateLimitError.message,
        instance: req.originalUrl,
        retryAfter: Math.round(rateLimitError.msBeforeNext / 1000)
      });
    }
  }
}
```

## Complete OpenAPI Specification: Production-Ready Telemetry API

```yaml
openapi: 3.0.3
info:
  title: TechCorp Telemetry Collection API
  version: "1.0.0"
  description: |
    Enterprise telemetry collection system for monitoring 50,000+ servers.
    
    ## Key Features
    - Idempotent telemetry submission (prevents duplicates during network failures)
    - Multi-level rate limiting (global, per-server, per-IP)
    - Time-based query optimization (handles high-volume historical data)
    - Business-level and HTTP-level deduplication
    
    ## Authentication
    Servers authenticate using pre-shared API keys specific to their data center.
    
    ## Rate Limits
    - Global: 100,000 requests/minute
    - Per server: 2 telemetry submissions/minute  
    - Per IP: 100 requests/minute
    
  contact:
    name: Telemetry Platform Team
    email: telemetry-platform@techcorp.com
    url: https://internal.techcorp.com/telemetry-docs

servers:
  - url: https://telemetry-api.techcorp.com/v1
    description: Production telemetry ingestion
  - url: https://telemetry-staging.techcorp.com/v1
    description: Staging environment

paths:
  /servers/{serverId}/telemetry:
    parameters:
      - name: serverId
        in: path
        required: true
        description: |
          Unique server identifier. Format: {datacenter}-{servertype}-{number}
          Example: mumbai-web-001, tokyo-db-primary, london-cache-05
        schema:
          type: string
          pattern: '^[a-z]+-[a-z]+-[a-z0-9]+$'
          example: "mumbai-web-001"
    
    post:
      summary: Submit telemetry data from server
      description: |
        Submit telemetry data for a specific server. This endpoint:
        
        - Uses idempotency keys to prevent duplicate data during network retries
        - Performs business-level deduplication based on collection timestamp
        - Enforces rate limits to prevent configuration errors from overwhelming the system
        - Validates telemetry data format and completeness
        
        ## Idempotency Behavior
        - Same Idempotency-Key: Returns cached response (handles network retries)
        - Different Idempotency-Key, same timestamp: Returns existing data (prevents duplicates)
        - Different Idempotency-Key, new timestamp: Creates new telemetry record
        
        ## Rate Limiting
        Each server can submit maximum 2 telemetry records per minute.
        This prevents misconfigured agents from overwhelming the system.
        
      security:
        - serverApiKey: []
      parameters:
        - name: Idempotency-Key
          in: header
          required: true
          description: |
            UUID generated by client for request deduplication.
            Use the same key when retrying due to network failures.
            Generate a new key for each logical telemetry collection cycle.
            
            Example: "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
          schema:
            type: string
            format: uuid
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/TelemetrySubmission'
            examples:
              webServerTelemetry:
                summary: Web server telemetry
                value:
                  collectionTimestamp: "2025-09-10T10:30:00Z"
                  serverInstanceId: "instance-1694168400"
                  environment: "production"
                  metrics:
                    cpu:
                      usage_percent: 75.5
                      load_1min: 2.1
                      load_5min: 1.8
                    memory:
                      usage_percent: 68.2
                      used_gb: 10.9
                      available_gb: 5.1
                    disk:
                      usage_percent: 45.0
                      used_gb: 450.0
                      available_gb: 550.0
                    network:
                      bytes_in: 1048576
                      bytes_out: 2097152
                      packets_in: 1024
                      packets_out: 768
              databaseServerTelemetry:
                summary: Database server telemetry
                value:
                  collectionTimestamp: "2025-09-10T10:30:00Z"
                  serverInstanceId: "instance-1694168400"
                  environment: "production"
                  metrics:
                    cpu:
                      usage_percent: 95.2
                    memory:
                      usage_percent: 89.7
                      buffer_pool_gb: 32.0
                    disk:
                      usage_percent: 78.3
                      iops: 2500
                    database:
                      active_connections: 150
                      slow_queries: 5
                      replication_lag_ms: 50
      responses:
        '201':
          description: New telemetry data recorded successfully
          headers:
            X-RateLimit-Limit:
              schema:
                type: integer
              description: Telemetry submissions allowed per minute for this server
            X-RateLimit-Remaining:
              schema:
                type: integer
              description: Remaining submissions in current window
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/TelemetryResponse'
              example:
                telemetryId: "tel_mumbai-web-001_1694168400_abc123"
                status: "recorded"
                eventBusMessageId: "msg_def456"
                recordedAt: "2025-09-10T10:30:15Z"
                
        '200':
          description: |
            Duplicate telemetry data - returning existing record.
            This happens when:
            - Same idempotency key is retried (network retry)
            - Different idempotency key but same collection timestamp (duplicate data)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/TelemetryResponse'
              example:
                telemetryId: "tel_mumbai-web-001_1694168400_abc123"
                status: "duplicate_data"
                message: "Telemetry already recorded for this timestamp"
                originalRecordedAt: "2025-09-10T10:30:15Z"
                
        '400':
          $ref: '#/components/responses/BadRequest'
        '422':
          description: Validation error in telemetry data
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
              example:
                type: "https://api.techcorp.com/errors/validation-error"
                title: "Telemetry Validation Error"
                status: 422
                detail: "Telemetry data contains invalid or missing required fields"
                instance: "/servers/mumbai-web-001/telemetry"
                errors:
                  - field: "collectionTimestamp"
                    code: "INVALID_FORMAT"
                    message: "Timestamp must be in ISO 8601 format"
                  - field: "metrics.cpu.usage_percent"
                    code: "OUT_OF_RANGE"
                    message: "CPU usage must be between 0 and 100"
        '429':
          $ref: '#/components/responses/RateLimitExceeded'
        '503':
          description: Telemetry service temporarily unavailable
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
              example:
                type: "https://api.techcorp.com/errors/service-unavailable"
                title: "Service Temporarily Unavailable"
                status: 503
                detail: "Telemetry ingestion service is temporarily unavailable"
                instance: "/servers/mumbai-web-001/telemetry"
                retryAfter: 60

    get:
      summary: Retrieve telemetry data for server
      description: |
        Retrieve historical telemetry data for a specific server.
        
        ## Performance Considerations
        - Maximum time range: 24 hours per request
        - Results are paginated using cursor-based pagination
        - Use metric filtering to reduce response size
        - Consider aggregation for dashboard queries
        
        ## Typical Usage Patterns
        - Real-time monitoring: Last 1 hour with 1-minute intervals
        - Dashboard graphs: Last 24 hours with 5-minute aggregation
        - Incident investigation: Specific time windows with full detail
        
      security:
        - serverApiKey: []
      parameters:
        - name: from
          in: query
          required: true
          description: Start timestamp for telemetry data (ISO 8601)
          schema:
            type: string
            format: date-time
          example: "2025-09-10T08:00:00Z"
        - name: to
          in: query
          required: true
          description: End timestamp for telemetry data (ISO 8601, max 24h from 'from')
          schema:
            type: string
            format: date-time
          example: "2025-09-10T09:00:00Z"
        - name: metrics
          in: query
          description: |
            Comma-separated list of metrics to include in response.
            Reduces response size for specific monitoring needs.
            Available metrics: cpu, memory, disk, network, database
          schema:
            type: string
          example: "cpu,memory"
        - name: aggregate
          in: query
          description: |
            Time aggregation interval for data points.
            Reduces response size for longer time ranges.
            Available: 1min, 5min, 15min, 1hour
          schema:
            type: string
            enum: [1min, 5min, 15min, 1hour]
          example: "5min"
        - name: limit
          in: query
          description: Maximum number of telemetry records to return
          schema:
            type: integer
            minimum: 1
            maximum: 10000
            default: 1000
        - name: cursor
          in: query
          description: |
            Pagination cursor from previous response.
            Use for retrieving next page of results.
          schema:
            type: string
            format: base64
      responses:
        '200':
          description: Telemetry data retrieved successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/TelemetryQueryResponse'
              example:
                data:
                  - collectionTimestamp: "2025-09-10T08:00:00Z"
                    metrics:
                      cpu:
                        usage_percent: 75.5
                        load_1min: 2.1
                      memory:
                        usage_percent: 68.2
                        used_gb: 10.9
                  - collectionTimestamp: "2025-09-10T08:01:00Z"
                    metrics:
                      cpu:
                        usage_percent: 77.2
                        load_1min: 2.3
                      memory:
                        usage_percent: 69.1
                        used_gb: 11.1
                pagination:
                  limit: 1000
                  hasNext: true
                  nextCursor: "eyJ0aW1lc3RhbXAiOiIyMDI1LTA5LTEwVDA4OjAxOjAwWiJ9"
                timeRange:
                  from: "2025-09-10T08:00:00Z"
                  to: "2025-09-10T09:00:00Z"
                recordCount: 60
        '400':
          description: Invalid query parameters
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
              example:
                type: "https://api.techcorp.com/errors/bad-request"
                title: "Invalid Query Parameters"
                status: 400
                detail: "Time range cannot exceed 24 hours"
                instance: "/servers/mumbai-web-001/telemetry"
                errors:
                  - field: "to"
                    code: "TIME_RANGE_TOO_LARGE"
                    message: "Time range between 'from' and 'to' cannot exceed 24 hours"
        '404':
          $ref: '#/components/responses/NotFound'

components:
  securitySchemes:
    serverApiKey:
      type: apiKey
      in: header
      name: X-API-Key
      description: |
        Server-specific API key for authentication.
        Each data center has separate API keys for security isolation.
        Format: "tc_datacenter_environment_randomstring"
        Example: "tc_mumbai_prod_1a2b3c4d5e6f7890"

  schemas:
    TelemetrySubmission:
      type: object
      description: Telemetry data submitted by on-premise servers
      properties:
        collectionTimestamp:
          type: string
          format: date-time
          description: |
            When the telemetry data was collected on the server (not when sent).
            Used for business-level deduplication and time-series ordering.
            Must be in ISO 8601 format with timezone.
          example: "2025-09-10T10:30:00Z"
        serverInstanceId:
          type: string
          description: |
            Unique identifier for this server instance/boot.
            Changes when server restarts. Format: "instance-{unix-timestamp}"
            Used to handle server restarts and distinguish telemetry sessions.
          example: "instance-1694168400"
        environment:
          type: string
          enum: [production, staging, development]
          description: Server environment for data classification
        metrics:
          $ref: '#/components/schemas/ServerMetrics'
      required:
        - collectionTimestamp
        - serverInstanceId
        - environment
        - metrics

    ServerMetrics:
      type: object
      description: System and application metrics from server
      properties:
        cpu:
          $ref: '#/components/schemas/CPUMetrics'
        memory:
          $ref: '#/components/schemas/MemoryMetrics'
        disk:
          $ref: '#/components/schemas/DiskMetrics'
        network:
          $ref: '#/components/schemas/NetworkMetrics'
        database:
          $ref: '#/components/schemas/DatabaseMetrics'
      required:
        - cpu
        - memory
        - disk

    CPUMetrics:
      type: object
      properties:
        usage_percent:
          type: number
          minimum: 0
          maximum: 100
          description: Overall CPU utilization percentage
        load_1min:
          type: number
          minimum: 0
          description: 1-minute load average
        load_5min:
          type: number
          minimum: 0
          description: 5-minute load average
        load_15min:
          type: number
          minimum: 0
          description: 15-minute load average
      required:
        - usage_percent

    MemoryMetrics:
      type: object
      properties:
        usage_percent:
          type: number
          minimum: 0
          maximum: 100
          description: Memory utilization percentage
        used_gb:
          type: number
          minimum: 0
          description: Used memory in gigabytes
        available_gb:
          type: number
          minimum: 0
          description: Available memory in gigabytes
        buffer_pool_gb:
          type: number
          minimum: 0
          description: Database buffer pool size (database servers only)
      required:
        - usage_percent

    DiskMetrics:
      type: object
      properties:
        usage_percent:
          type: number
          minimum: 0
          maximum: 100
          description: Disk space utilization percentage
        used_gb:
          type: number
          minimum: 0
          description: Used disk space in gigabytes
        available_gb:
          type: number
          minimum: 0
          description: Available disk space in gigabytes
        iops:
          type: integer
          minimum: 0
          description: Input/output operations per second
      required:
        - usage_percent

    NetworkMetrics:
      type: object
      properties:
        bytes_in:
          type: integer
          minimum: 0
          description: Bytes received in collection interval
        bytes_out:
          type: integer
          minimum: 0
          description: Bytes sent in collection interval
        packets_in:
          type: integer
          minimum: 0
          description: Packets received in collection interval
        packets_out:
          type: integer
          minimum: 0
          description: Packets sent in collection interval

    DatabaseMetrics:
      type: object
      description: Database-specific metrics (only for database servers)
      properties:
        active_connections:
          type: integer
          minimum: 0
          description: Number of active database connections
        slow_queries:
          type: integer
          minimum: 0
          description: Number of slow queries in collection interval
        replication_lag_ms:
          type: integer
          minimum: 0
          description: Replication lag in milliseconds
        cache_hit_ratio:
          type: number
          minimum: 0
          maximum: 1
          description: Database cache hit ratio (0.0 to 1.0)

    TelemetryResponse:
      type: object
      description: Response after successful telemetry submission
      properties:
        telemetryId:
          type: string
          description: |
            Unique identifier for the telemetry record.
            Format: "tel_{serverId}_{timestamp}_{hash}"
          example: "tel_mumbai-web-001_1694168400_abc123"
        status:
          type: string
          enum: [recorded, duplicate_data]
          description: |
            - recorded: New telemetry data was stored
            - duplicate_data: Telemetry already exists for this timestamp
        eventBusMessageId:
          type: string
          description: Message ID in the event bus for downstream processing
          example: "msg_def456"
        recordedAt:
          type: string
          format: date-time
          description: When the telemetry was recorded by the API
        message:
          type: string
          description: Additional information (present for duplicate_data status)
        originalRecordedAt:
          type: string
          format: date-time
          description: When the original record was created (for duplicates)

    TelemetryQueryResponse:
      type: object
      description: Response for telemetry data queries
      properties:
        data:
          type: array
          items:
            $ref: '#/components/schemas/TelemetryRecord'
        pagination:
          $ref: '#/components/schemas/PaginationMetadata'
        timeRange:
          $ref: '#/components/schemas/TimeRange'
        recordCount:
          type: integer
          description: Number of records in this response

    TelemetryRecord:
      type: object
      properties:
        collectionTimestamp:
          type: string
          format: date-time
        metrics:
          $ref: '#/components/schemas/ServerMetrics'

    PaginationMetadata:
      type: object
      properties:
        limit:
          type: integer
          description: Requested page size
        hasNext:
          type: boolean
          description: Whether more records are available
        nextCursor:
          type: string
          nullable: true
          description: Cursor for next page (null if no more pages)

    TimeRange:
      type: object
      properties:
        from:
          type: string
          format: date-time
        to:
          type: string
          format: date-time

    ErrorResponse:
      type: object
      description: Standardized error response following RFC 7807
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
          description: URI reference for this specific error occurrence
        errorId:
          type: string
          description: Unique error ID for support requests
        errors:
          type: array
          description: Detailed field-level errors (for validation errors)
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
        Bad request - malformed data or missing required parameters.
        Common causes:
        - Invalid JSON in request body
        - Missing required headers (Idempotency-Key)
        - Invalid serverId format
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'

    NotFound:
      description: |
        Server not found or no telemetry data available.
        The serverId may not exist in the system or may not have
        any telemetry data in the requested time range.
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'
          example:
            type: "https://api.techcorp.com/errors/not-found"
            title: "Server Not Found"
            status: 404
            detail: "Server 'mumbai-web-999' not found in telemetry system"
            instance: "/servers/mumbai-web-999/telemetry"

    RateLimitExceeded:
      description: |
        Rate limit exceeded. Server is sending telemetry too frequently.
        This typically indicates a configuration error in the telemetry agent.
        Normal telemetry should be sent once per minute maximum.
      headers:
        X-RateLimit-Limit:
          description: Maximum requests allowed per minute
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
            type: "https://api.techcorp.com/errors/rate-limit-exceeded"
            title: "Rate Limit Exceeded"
            status: 429
            detail: "Server exceeded limit of 2 telemetry submissions per minute"
            instance: "/servers/mumbai-web-001/telemetry"
            retryAfter: 45
```

## Real-World Production Lessons Learned

### The $50,000 Configuration Error

**What happened:**
```bash
# Production incident - December 2024
# DevOps engineer changed telemetry config across all servers:

# BEFORE (correct):
TELEMETRY_INTERVAL_MS=60000  # 1 minute

# AFTER (typo):
TELEMETRY_INTERVAL_MS=6000   # 6 seconds instead of 60 seconds

# Result:
# - 50,000 servers × 10 requests/minute = 500,000 requests/minute
# - API servers crashed within 5 minutes
# - Cloud bill spike: $50,000 in 3 hours
# - Emergency rollback required
```

**How proper REST design prevented disaster:**
```javascript
// Rate limiting caught the problem immediately:
app.use('/servers/:serverId/telemetry', rateLimiter({
  windowMs: 60 * 1000,
  max: 2,  // Caught the 10x increase immediately
  message: 'Server sending telemetry too frequently - check configuration'
}));

// Result: Servers got 429 errors instead of crashing the API
// Monitoring alerts fired within 2 minutes
// Impact contained to single data center before global rollback
```

### The Duplicate Data Analytics Crisis

**What happened:**
```sql
-- Business intelligence team query:
SELECT AVG(cpu_usage) FROM telemetry 
WHERE collection_date = '2024-11-15'
  AND server_type = 'web';

-- Result: 847% average CPU usage (impossible!)
-- Cause: Network retries during storm created 8x duplicates
-- Impact: False emergency alerts, unnecessary server purchases approved
```

**How idempotency prevented false data:**
```javascript
// Proper implementation with two-level deduplication:

// Level 1: HTTP idempotency (handles retries)
if (await isHttpDuplicate(idempotencyKey)) {
  return cachedResponse; // Network retry - return same result
}

// Level 2: Business deduplication (handles duplicate data)
if (await isDuplicateTelemetry(serverId, timestamp)) {
  return existingTelemetry; // Same data from different source
}

// Result: Analytics queries return accurate data
// No false alerts, no unnecessary infrastructure spending
```

### The Information Disclosure Incident

**What happened:**
```http
# Developer accidentally exposed internal details:
POST /servers/prod-db-master-01/telemetry

500 Internal Server Error
{
  "error": "Connection failed to mysql://telemetry_user:P@ssw0rd123@prod-db-cluster.internal:3306/telemetry_production",
  "server": "telemetry-api-prod-3.aws-east-1.internal",
  "version": "telemetry-api-2.1.4"
}

# Security team discovered this in external monitoring logs
# Attackers now had: database credentials, internal hostnames, versions
```

**Secure error handling prevents information disclosure:**
```javascript
// Production error handler:
function handleError(error, req, res) {
  const errorId = `tel_err_${crypto.randomUUID()}`;
  
  // Full details logged internally only
  internalLogger.error({
    errorId,
    error: error.stack,
    request: sanitizedRequest(req),
    database: process.env.DB_HOST,
    server: process.env.HOSTNAME
  });
  
  // Minimal public response
  res.status(500).json({
    type: 'https://api.techcorp.com/errors/internal-server-error',
    title: 'Internal Server Error',
    status: 500,
    detail: 'Telemetry service temporarily unavailable',
    errorId,  // For support correlation only
    supportContact: 'telemetry-support@techcorp.com'
  });
}
```

## Performance at Scale: The Reality Check

### Database Performance Degradation

**Month 1**: 10M telemetry records
```sql
SELECT * FROM telemetry 
WHERE server_id = 'mumbai-web-001' 
  AND collection_timestamp BETWEEN '2025-09-10 08:00:00' AND '2025-09-10 09:00:00';
-- Query time: 15ms
```

**Month 12**: 500M telemetry records
```sql
-- Same query, different performance:
-- Query time: 2.5 seconds (167x slower!)
-- Database CPU: 95% during business hours
-- Disk I/O: 80% utilization constantly
```

**Solution - Time-based table partitioning:**
```sql
-- Partition tables by month for predictable performance
CREATE TABLE telemetry_2025_09 (
  -- Same schema as main table
) PARTITION OF telemetry
FOR VALUES FROM ('2025-09-01') TO ('2025-10-01');

-- Result: Query performance stays constant
-- Automatic partition pruning keeps queries fast
-- Old partitions can be dropped after retention period
```

### API Response Time Optimization

**❌ Naive implementation:**
```javascript
// Returns 2.3GB response for 6 months of data
app.get('/servers/:serverId/telemetry', async (req, res) => {
  const telemetry = await db.query('SELECT * FROM telemetry WHERE server_id = ?', [req.params.serverId]);
  res.json(telemetry);
});
```

**✅ Production-optimized implementation:**
```javascript
app.get('/servers/:serverId/telemetry', async (req, res) => {
  // Enforce time range limits
  const { from, to } = req.query;
  const timeRange = new Date(to) - new Date(from);
  const maxRange = 24 * 60 * 60 * 1000; // 24 hours max
  
  if (timeRange > maxRange) {
    return res.status(400).json({
      error: 'Time range too large',
      maxRange: '24 hours',
      suggestion: 'Use aggregation for longer ranges'
    });
  }
  
  // Use appropriate index
  const telemetry = await db.query(`
    SELECT collection_timestamp, metrics 
    FROM telemetry 
    WHERE server_id = ? 
      AND collection_timestamp BETWEEN ? AND ?
    ORDER BY collection_timestamp ASC
    LIMIT 1440  -- Max 24 hours × 60 minutes
  `, [req.params.serverId, from, to]);
  
  res.json({
    data: telemetry,
    timeRange: { from, to },
    recordCount: telemetry.length,
    suggestion: timeRange > (60 * 60 * 1000) ? 
      'Consider using aggregation for time ranges > 1 hour' : null
  });
});
```

## Economic Impact of Good API Design

### Cost Comparison: Well-Designed vs Poorly-Designed Telemetry API

**Poorly designed API costs (12-month projection):**
```
Infrastructure:
- 3x larger database instances due to duplicates: +$18,000/year
- Additional API servers due to inefficient queries: +$12,000/year
- Network bandwidth for oversized responses: +$8,000/year

Development time:
- Emergency bug fixes for duplicate data: 200 hours
- Performance optimization projects: 300 hours  
- Support tickets for confusing errors: 150 hours
- Total: 650 hours × $150/hour = $97,500

Incident costs:
- 3 production outages from configuration errors: $75,000
- 1 security incident from information disclosure: $25,000

Total first-year cost: $235,500
```

**Well-designed API costs:**
```
Initial development:
- Proper idempotency implementation: 40 hours
- Rate limiting and error handling: 20 hours
- Comprehensive testing: 30 hours
- Documentation and examples: 10 hours
- Total: 100 hours × $150/hour = $15,000

Ongoing maintenance: $3,000/year

Total first-year cost: $18,000
```

**ROI**: $217,500 saved in first year (1,208% return on investment)

This telemetry API case study shows how REST design principles solve real production problems: preventing duplicate data from costing thousands in cloud bills, avoiding security incidents through proper error handling, and maintaining performance as data volume grows exponentially.
