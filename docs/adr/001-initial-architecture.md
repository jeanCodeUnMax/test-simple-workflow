# Architecture Decision Record (ADR) - Test Simple Workflow

## ADR-001: Layered Architecture for n8n Workflows

### Status
Accepted

### Context
Need to demonstrate that structured architecture applies even to simple n8n workflows. Every workflow should follow the 6-layer pattern regardless of complexity.

### Decision
Adopt strict 6-layer architecture:

1. **Layer 1 - Handlers**: HTTP entry/exit points (Webhook, Response)
2. **Layer 2 - Middleware**: Input validation and sanitization
3. **Layer 3 - Router**: Request dispatch based on action type
4. **Layer 4 - Services**: Pure business logic
5. **Layer 5 - Handlers**: HTTP response formatting
6. **Layer 6 - Utils**: Logging and observability

### Consequences

#### Positive
- Single node = single responsibility
- Easy to test each layer independently
- Correlation ID provides full traceability
- Error paths are explicit
- Adding new operations requires only Layer 4 changes

#### Negative
- More nodes than minimal approach
- Requires discipline to not mix concerns

### Example Implementation
See `workflows/simple-webhook-workflow.json` - demonstrates all 6 layers with 10 nodes.

### Date
2026-02-20

### Authors
- @jeanCodeUnMax
