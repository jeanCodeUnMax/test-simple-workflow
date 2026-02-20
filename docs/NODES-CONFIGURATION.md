# Node Configuration Details

## Layer 1: Webhook Trigger (Handler)

### Configuration
```json
{
  "type": "n8n-nodes-base.webhook",
  "typeVersion": 2,
  "parameters": {
    "path": "test-webhook",
    "responseMode": "responseNode",
    "options": {}
  }
}
```

### Purpose
- HTTP entry point
- Receives POST requests
- Passes to validation layer

---

## Layer 2: Validate Input (Middleware)

### Configuration
```json
{
  "type": "n8n-nodes-base.code",
  "typeVersion": 2,
  "parameters": {
    "jsCode": "// Input validation and sanitization"
  }
}
```

### Code Logic
```javascript
// 1. Check required fields
const required = ['name', 'action'];
const missing = required.filter(field => !input[field]);

// 2. Return early if invalid
if (missing.length > 0) {
  return {
    json: {
      valid: false,
      error: `Missing required fields: ${missing.join(', ')}`
    }
  };
}

// 3. Sanitize input
const sanitized = {
  name: String(input.name).trim().substring(0, 100),
  action: ['create', 'update', 'delete'].includes(input.action) 
    ? input.action 
    : 'unknown',
  metadata: input.metadata || {},
  timestamp: new Date().toISOString()
};

// 4. Generate correlation ID for tracing
const correlationId = headers?.['x-correlation-id'] 
  || Math.random().toString(36).substring(7);
```

### Expressions Used
- `{{ $input.first().json.body }}` - Access POST body
- `{{ $input.first().json.headers }}` - Access headers
- `{{ $json.valid }}` - Check validation result

---

## Layer 3: Action Router (Router)

### Configuration
```json
{
  "type": "n8n-nodes-base.switch",
  "typeVersion": 3,
  "parameters": {
    "rules": {
      "values": [
        {
          "output": 0,
          "conditions": {
            "conditions": [
              {
                "value1": "={{ $json.data.action }}",
                "operation": "equals",
                "value2": "create"
              }
            ]
          }
        },
        {
          "output": 1,
          "conditions": {
            "conditions": [
              {
                "value1": "={{ $json.data.action }}",
                "operation": "equals",
                "value2": "update"
              }
            ]
          }
        },
        {
          "output": 2,
          "conditions": {
            "conditions": [
              {
                "value1": "={{ $json.data.action }}",
                "operation": "equals",
                "value2": "delete"
              }
            ]
          }
        }
      ]
    },
    "fallbackOutput": 3
  }
}
```

### Routing Logic
| **Output** | **Condition** | **Destination** |
|------------|---------------|-----------------|
| 0 | action == "create" | Create Service |
| 1 | action == "update" | Update Service |
| 2 | action == "delete" | Delete Service |
| 3 | fallback | Error Handler |

---

## Layer 4: Service Nodes (Business Logic)

### Create Service
```javascript
const input = $input.first().json;
const correlationId = input.correlationId;
const data = input.data;

// Pure business logic - no HTTP
const result = {
  id: Math.random().toString(36).substring(7),
  name: data.name,
  action: 'create',
  status: 'created',
  createdAt: new Date().toISOString(),
  correlationId: correlationId
};

return {
  json: {
    success: true,
    operation: 'create',
    result: result,
    correlationId: correlationId
  }
};
```

### Update Service
```javascript
const result = {
  id: data.metadata?.id || 'unknown',
  name: data.name,
  action: 'update',
  status: 'updated',
  updatedAt: new Date().toISOString(),
  correlationId: correlationId
};
```

### Delete Service
```javascript
const result = {
  id: data.metadata?.id || 'unknown',
  name: data.name,
  action: 'delete',
  status: 'deleted',
  deletedAt: new Date().toISOString(),
  correlationId: correlationId
};
```

### Error Handler
```javascript
return {
  json: {
    success: false,
    operation: 'unknown',
    error: `Unknown action: ${input.data?.action}`,
    correlationId: input.correlationId,
    supportedActions: ['create', 'update', 'delete']
  }
};
```

---

## Layer 5: Success Response (Handler)

### Configuration
```json
{
  "type": "n8n-nodes-base.respondToWebhook",
  "typeVersion": 1.1,
  "parameters": {
    "respondWith": "json",
    "responseBody": "={{ { success: true, data: $json.result, correlationId: $json.correlationId } }}"
  }
}
```

### Response Format
```json
{
  "success": true,
  "data": {
    "id": "abc123",
    "name": "Test Item",
    "action": "create",
    "status": "created",
    "createdAt": "2026-02-20T..."
  },
  "correlationId": "test-123"
}
```

---

## Layer 6: Log Operation (Utils)

### Configuration
```json
{
  "type": "n8n-nodes-base.code",
  "typeVersion": 2,
  "parameters": {
    "jsCode": "// Structured logging"
  }
}
```

### Logging Logic
```javascript
const logEntry = {
  timestamp: new Date().toISOString(),
  level: result.success ? 'info' : 'error',
  correlationId: result.correlationId,
  operation: result.operation,
  success: result.success,
  error: result.error || null
};

console.log('[LOG]', JSON.stringify(logEntry));
```

---

## Error Handling Node (Validation Error)

### Configuration
```json
{
  "type": "n8n-nodes-base.respondToWebhook",
  "typeVersion": 1.1,
  "parameters": {
    "respondWith": "json",
    "responseBody": "={{ { success: false, error: $json.error, stage: 'validation' } }}"
  }
}
```

### Connection
- From: Validate Input (when valid: false)
- To: Error Response

---

## Connections Summary

```
Webhook Trigger
    ↓
Validate Input ──┬──▶ [valid: true] ──▶ Action Router
                  │
                  └──▶ [valid: false] ──▶ Validation Error

Action Router
    ├──▶ create ──▶ Create Service ──┐
    ├──▶ update ──▶ Update Service ──┼──▶ Success Response ──▶ Log Operation
    ├──▶ delete ──▶ Delete Service ──┤
    └──▶ fallback ──▶ Error Handler ──┘
```
