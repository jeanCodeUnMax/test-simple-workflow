# Architecture

## Layered Architecture

This workflow demonstrates the 6-layer architecture pattern:

1. **Handler Layer**: HTTP entry/exit
2. **Middleware**: Validation
3. **Router**: Request dispatch
4. **Service**: Business logic
5. **Handler**: Response formatting
6. **Utils**: Logging

## Flow

```
Webhook → Validate → Route → Service → Response → Log
```

See docs/NODES-CONFIGURATION.md for details.
