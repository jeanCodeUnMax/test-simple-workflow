# Test Simple Workflow - Architecture Example

## Description
Workflow n8n minimal qui démontre l'**architecture stricte en couches**.

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    LAYER ARCHITECTURE                        │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  LAYER 1: INPUT (Handler)                                     │
│  ┌─────────────────────────────────────────┐                  │
│  │  Webhook Trigger                      │  ← HTTP entry    │
│  │  - Path: /test-webhook                │                  │
│  │  - Method: POST                     │                  │
│  └─────────────────────────────────────────┘                  │
│                         │                                     │
│                         ▼                                     │
│  LAYER 2: VALIDATION (Middleware)                             │
│  ┌─────────────────────────────────────────┐                  │
│  │  Validate Input (Code Node)           │  ← Sanitize    │
│  │  - Check required fields              │                  │
│  │  - Sanitize data                      │                  │
│  │  - Generate correlationId             │                  │
│  └─────────────────────────────────────────┘                  │
│                         │                                     │
│                         ▼                                     │
│  LAYER 3: ROUTING (Switch)                                    │
│  ┌─────────────────────────────────────────┐                  │
│  │  Action Router                        │  ← Dispatch    │
│  │  - Route by action type               │                  │
│  │  - 4 outputs: create/update/delete/error│                │
│  └─────────────────────────────────────────┘                  │
│        │        │        │                                   │
│        ▼        ▼        ▼                                   │
│  LAYER 4: BUSINESS LOGIC (Services)                           │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐                         │
│  │ Create  │ │ Update  │ │ Delete  │  ← Operations           │
│  │ Service │ │ Service │ │ Service │                         │
│  └─────────┘ └─────────┘ └─────────┘                         │
│        │        │        │                                   │
│        └────────┼────────┘                                   │
│                 ▼                                             │
│  LAYER 5: RESPONSE (Handler)                                  │
│  ┌─────────────────────────────────────────┐                  │
│  │  Success Response                     │  ← HTTP exit     │
│  │  - JSON formatted                     │                  │
│  │  - Include correlationId              │                  │
│  └─────────────────────────────────────────┘                  │
│                 │                                             │
│                 ▼                                             │
│  LAYER 6: OBSERVABILITY (Utils)                               │
│  ┌─────────────────────────────────────────┐                  │
│  │  Log Operation                        │  ← Audit trail   │
│  │  - Structured logging                 │                  │
│  │  - Correlation tracking                 │                │
│  └─────────────────────────────────────────┘                  │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

## Nodes Structure

| **Node** | **Layer** | **Purpose** | **Type** |
|----------|-----------|-------------|----------|
| Webhook Trigger | Handler | HTTP entry point | webhook |
| Validate Input | Middleware | Input sanitization | code |
| Action Router | Router | Request dispatch | switch |
| Create Service | Service | Business logic | code |
| Update Service | Service | Business logic | code |
| Delete Service | Service | Business logic | code |
| Error Handler | Service | Error processing | code |
| Success Response | Handler | HTTP response | respondToWebhook |
| Log Operation | Utils | Audit logging | code |

## Code Patterns

### 1. Validation Layer Pattern
```javascript
// Check required fields
const required = ['name', 'action'];
const missing = required.filter(field => !input[field]);

// Sanitize input
const sanitized = {
  name: String(input.name).trim().substring(0, 100),
  action: ['create', 'update', 'delete'].includes(input.action) ? input.action : 'unknown'
};

// Generate tracking ID
const correlationId = headers?.['x-correlation-id'] || Math.random().toString(36).substring(7);
```

### 2. Service Layer Pattern
```javascript
// Pure business logic, no HTTP concerns
const result = {
  id: generateId(),
  name: data.name,
  action: 'create',
  status: 'created',
  createdAt: new Date().toISOString(),
  correlationId: correlationId  // Pass through for tracing
};

// Return structured data
return {
  json: {
    success: true,
    operation: 'create',
    result: result,
    correlationId: correlationId
  }
};
```

### 3. Switch Router Pattern
```javascript
// Route based on action type
rules: [
  { output: 0, condition: "action === 'create'" },
  { output: 1, condition: "action === 'update'" },
  { output: 2, condition: "action === 'delete'" },
  { output: 3, condition: "fallback" }  // Error
]
```

## Connections Flow

```
Webhook Trigger ──▶ Validate Input ──▶ Action Router
                                          │
          ┌───────────┬───────────┬───────┼───────┐
          │           │           │       │       │
          ▼           ▼           ▼       ▼       ▼
     Create Svc   Update Svc   Delete Svc  │   Error Svc
          │           │           │       │       │
          └───────────┴───────────┴───────┼───────┘
                                          │
                                          ▼
                                Success Response ──▶ Log Operation
```

## Testing

### Manual Test via curl
```bash
# Create operation
curl -X POST http://localhost:5678/webhook/test-webhook \
  -H "Content-Type: application/json" \
  -H "x-correlation-id: test-123" \
  -d '{"name": "Test Item", "action": "create"}'

# Expected response:
# {
#   "success": true,
#   "data": {
#     "id": "abc123",
#     "name": "Test Item",
#     "action": "create",
#     "status": "created"
#   },
#   "correlationId": "test-123"
# }

# Validation error
curl -X POST http://localhost:5678/webhook/test-webhook \
  -H "Content-Type: application/json" \
  -d '{"name": "Test"}'

# Expected response:
# {
#   "success": false,
#   "error": "Missing required fields: action",
#   "stage": "validation"
# }
```

## Architecture Principles Demonstrated

1. **Single Responsibility**: Each node does one thing
2. **Separation of Concerns**: HTTP, validation, business logic separated
3. **Traceability**: correlationId flows through all layers
4. **Error Handling**: Dedicated error path with fallback
5. **Structured Logging**: All operations logged with context

## Validation

Run structure validation:
```bash
node ../scripts/validate-structure.js .
```

Expected output:
```
✅ STRUCTURE VALID (score: 95%)
✅ Root files present
✅ workflows/ directory present
✅ docs/ directory present
⚠️ tests/ directory missing (optional for example)
```
