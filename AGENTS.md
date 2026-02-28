# MQTT Middleware Pro - Agent Guide

**Version:** 1.0  
**Date:** 2026-02-28  
**Status:** Draft

---

## Project Overview

MQTT Middleware Pro is a modular, high-throughput integration layer designed to unify data from heterogeneous IoT gateway devices into a standardized format for real-time dashboards and historical SQL storage. The system acts as middleware between IoT devices and downstream applications, providing data normalization, caching, and API access.

### Key Characteristics

| Attribute | Description |
|-----------|-------------|
| **Purpose** | IoT gateway data integration and normalization |
| **Supported Devices** | V5008 (binary protocol), V6800 (JSON protocol) |
| **Architecture** | Event-driven, modular, configuration-based |
| **Data Flow** | RAW → SIF → SUO → UOS/Database |
| **Current Phase** | Design and architecture (no source code yet) |

---

## Technology Stack

| Component | Technology | Purpose |
|-----------|------------|---------|
| Runtime | Node.js 18+ | Application server and message processing |
| Language | TypeScript / JavaScript (ES Modules) | Implementation language |
| MQTT Client | MQTT.js | Device communication |
| Database | MySQL 8.0 | Historical data persistence |
| Cache | In-Memory / Redis | Real-time state management (UOS) |
| API | REST/HTTP | Client-facing API endpoints |
| WebSocket | ws library | Real-time data broadcasting |

### Planned but Not Implemented

- PostgreSQL database support (migration planned for future version)
- Redis backend for distributed caching (optional)
- Docker/Kubernetes deployment configurations

---

## Project Structure

```
mqtt-middleware-pro/
├── agents/                    # Agent role definitions
│   ├── orchestrator.md        # Coordination workflow
│   ├── product_agent.md       # Requirements analysis
│   ├── architect_agent.md     # System design
│   ├── coding_agent.md        # Implementation standards
│   └── qa_agent.md            # Quality assurance
│
├── skills/                    # Reusable skill definitions
│   ├── bug_hunting.md         # Bug detection patterns
│   ├── code_review.md         # Review guidelines
│   ├── mqtt_binary_handling.md # Binary parsing skills
│   ├── frontend_log_visualization.md # UI logging
│   ├── performance.md         # Optimization techniques
│   └── refactoring.md         # Code refactoring patterns
│
├── specs/                     # Technical specifications
│   ├── prd.md                 # Product Requirements Document
│   ├── V5008_Spec.md          # V5008 binary message spec
│   ├── v6800_spec.md          # V6800 JSON message spec
│   ├── SUO_UOS_DB_Spec.md     # Data layer and DB schema spec
│   └── AUDIT_REPORT.md        # Audit findings
│
├── plans/                     # Planning documents
│   ├── spec_review_analysis.md
│   ├── v6800_spec_fixes_summary.md
│   ├── v6800_spec_issues_and_fixes.md
│   └── v6800_spec_notes_plan.md
│
├── playbooks/                 # Implementation playbooks
│   └── feature_implementation.md
│
├── architecture.md            # Comprehensive architecture document
├── skills-lock.json           # Skill version tracking
└── AGENTS.md                  # This file
```

### Expected Source Structure (To Be Created)

When implementation begins, the source code should follow this structure:

```
src/
├── config/                    # Configuration management
├── core/                      # Core modules (always active)
│   ├── event-bus/             # Event bus implementation
│   ├── mqtt/                  # MQTT client (subscriber/publisher)
│   ├── parser/                # RAW → SIF parsers
│   └── normalizer/            # SIF → SUO transformers
├── modules/                   # Optional modules
│   ├── smart-hb/              # Smart heartbeat processor
│   ├── watchdog/              # Scheduled task executor
│   ├── cache/                 # UOS cache implementation
│   └── command/               # Command service
├── output/                    # Output modules
│   ├── mqtt-relay/            # MQTT message relay
│   ├── websocket/             # WebSocket broadcaster
│   ├── webhook/               # HTTP webhook sender
│   └── database/              # Database writer
├── database/                  # Database layer
├── api/                       # REST API layer
├── types/                     # TypeScript type definitions
├── utils/                     # Utility functions
└── app.ts                     # Application entry point
```

---

## Build and Development Commands

### Prerequisites

- Node.js 18 or higher
- MySQL 8.0
- MQTT Broker (ActiveMQ for development, Mosquitto/EMQX for production)

### Setup (To Be Implemented)

```bash
# Install dependencies
npm install

# Install MQTT.js specifically
npm install mqtt

# Install development dependencies
npm install --save-dev typescript @types/node nodemon
```

### Build Commands (Expected)

```bash
# Compile TypeScript
npm run build

# Development mode with hot reload
npm run dev

# Production start
npm start

# Run tests
npm test

# Run linting
npm run lint
```

### Configuration Files Needed

- `package.json` - Project dependencies and scripts
- `tsconfig.json` - TypeScript compiler configuration
- `config/default.config.json` - Default configuration
- `config/development.config.json` - Development overrides
- `config/production.config.json` - Production configuration

---

## Code Style Guidelines

### Language and Module System

- **Use JavaScript/TypeScript** as specified in `coding_agent.md`
- **Use ES Modules** (`import`/`export`) instead of CommonJS
- **No pseudo code** - all code must be production-ready
- Follow the DRY principle - extract shared logic into reusable functions/helpers

### Naming Conventions

| Element | Convention | Example |
|---------|------------|---------|
| Classes | PascalCase | `MQTTSubscriber`, `MessageParser` |
| Interfaces | PascalCase with I prefix | `IModule`, `IEventBus` |
| Functions/Methods | camelCase | `parseMessage()`, `normalizeData()` |
| Constants | UPPER_SNAKE_CASE | `DEFAULT_PORT`, `MAX_RETRIES` |
| Files | kebab-case | `mqtt-client.ts`, `smart-hb.ts` |
| Private members | _camelCase with underscore | `_config`, `_connection` |

### Code Organization

1. **One module per file** - Keep modules focused and small
2. **Index files** - Each directory should have an `index.ts` for exports
3. **Separation of concerns** - Business logic separate from infrastructure
4. **Interface-first design** - Define interfaces before implementations

### Error Handling

- Use typed errors with descriptive messages
- Always handle Promise rejections
- Log errors with context (correlation IDs)
- Never swallow exceptions silently

### Async Patterns

```typescript
// Preferred: async/await
async function processMessage(message: RawMessage): Promise<void> {
  try {
    const sif = await this.parser.parse(message);
    const suo = await this.normalizer.normalize(sif);
    await this.eventBus.emit('suo.mqtt.message', suo);
  } catch (error) {
    this.logger.error('Failed to process message', { error, messageId: message.id });
    throw new ProcessingError('Message processing failed', { cause: error });
  }
}
```

---

## Testing Instructions

### Testing Strategy

| Test Type | Coverage Target | Tools |
|-----------|-----------------|-------|
| Unit Tests | Core modules, parsers, transformers | Jest |
| Integration Tests | Database, MQTT, API endpoints | Jest + TestContainers |
| E2E Tests | Full message flow | Custom test harness |

### Critical Test Areas

1. **Message Parsing**
   - V5008 binary message parsing (all message types)
   - V6800 JSON message parsing
   - Edge cases: malformed messages, missing fields

2. **Data Transformation**
   - SIF → SUO conversion accuracy
   - Flattening rule implementation for V6800
   - Timestamp enrichment

3. **Event Flow**
   - Event bus publish/subscribe
   - Module interaction via events
   - Error propagation

4. **Database Operations**
   - SUO persistence
   - Query performance
   - Transaction handling

### Running Tests (Expected)

```bash
# Run all tests
npm test

# Run unit tests only
npm run test:unit

# Run integration tests
npm run test:integration

# Run with coverage
npm run test:coverage

# Watch mode for development
npm run test:watch
```

---

## Architecture Principles

### Core Design Patterns

| Pattern | Usage |
|---------|-------|
| **Event-Driven Architecture** | Loose coupling via event bus |
| **Plugin/Module Pattern** | Configurable enable/disable of components |
| **Factory Pattern** | Dynamic module instantiation |
| **Observer Pattern** | Event emission and subscription |
| **Repository Pattern** | Database abstraction (future) |

### Module Interface

All modules must implement the common interface:

```typescript
interface IModule {
  name: string;
  enabled: boolean;
  initialize(): Promise<void>;
  start(): Promise<void>;
  stop(): Promise<void>;
  getStatus(): ModuleStatus;
}
```

### Data Flow

```
Device → MQTT Broker → MQTTSubscriber → EventBus → MessageParser
                                                        ↓
Database ← DatabaseWriter ← EventBus ← Normalizer ← SIF
   ↓
UOS Cache ← EventBus ← SUO
   ↓
Clients (via API/WebSocket)
```

### Event Types

| Event Name | Payload | Emitters | Listeners |
|------------|---------|----------|-----------|
| `raw.mqtt.message` | RawMQTTMessage | MQTTSubscriber | MessageParser |
| `sif.message` | SIFMessage | MessageParser | Normalizer |
| `suo.mqtt.message` | SUOMessage | Normalizer | Cache, DB, Relay, WS, Webhook |
| `command.request` | CommandRequest | ApiService | CommandService |
| `command.publish` | RawCommand | CommandService | MQTTPublisher |

---

## Configuration System

### Configuration Hierarchy

1. **Default config** (`config/default.config.json`)
2. **Environment config** (`config/{env}.config.json`)
3. **Environment variables** (override file config)

### Environment Variable Mapping

```
MQTT_BROKER_HOST → mqtt.broker.host
MQTT_BROKER_PORT → mqtt.broker.port
DB_HOST → database.host
DB_PASSWORD → database.password
```

### Module Configuration

All modules can be enabled/disabled via configuration:

```json
{
  "modules": {
    "smartHB": { "enabled": true },
    "watchdog": { "enabled": true },
    "mqttRelay": { "enabled": false },
    "websocket": { "enabled": true },
    "webhook": { "enabled": false },
    "database": { "enabled": true }
  }
}
```

---

## Security Considerations

### MQTT Security

- Use TLS/SSL encryption (`mqtts` or `wss` protocol)
- Username/password authentication
- Client certificate authentication (optional)

### API Security

- JWT token-based authentication
- CORS configuration
- Rate limiting
- Input validation and sanitization

### Database Security

- Connection encryption (TLS)
- Prepared statements (parameterized queries)
- Least privilege database user

### Secrets Management

- Never commit credentials to repository
- Use environment variables for sensitive data
- Consider secret management tools (Vault, etc.)

---

## Agent Roles and Workflow

### Orchestrator Agent

Coordinates all agents in the following order:

1. **Product Agent** → Analyzes requirements, creates `tasks.md`
2. **Architect Agent** → Designs system, creates/updates `architecture.md`
3. **Coding Agent** → Implements modules based on architecture
4. **QA Agent** → Reviews code, finds bugs and issues
5. **Coding Agent** → Fixes issues identified by QA

### Coding Agent Standards

- Write clean, production-quality code
- Follow `architecture.md` strictly
- Implement one module at a time
- Avoid assumptions - ask when unclear
- Do NOT redesign architecture
- Do NOT skip required features
- Follow DRY principle

### QA Agent Responsibilities

- Find bugs and logic errors
- Identify missing features
- Suggest fixes and improvements
- Verify DRY principle compliance
- Focus on correctness and robustness
- Do NOT redesign architecture

---

## Key Specifications

### V5008 Device (Binary Protocol)

- Message format: Binary
- Topic pattern: `V5008Upload/{deviceId}/{messageType}`
- Parsing: Complex binary parsing with header detection
- Key message types: HEARTBEAT, RFID_SNAPSHOT, TEMP_HUM, NOISE_LEVEL, DOOR_STATE

### V6800 Device (JSON Protocol)

- Message format: JSON
- Topic pattern: `V6800Upload/{deviceId}/{messageType}`
- Parsing: JSON parsing with field mapping
- Key message types: DEV_MOD_INFO, HEARTBEAT, RFID_EVENT, RFID_SNAPSHOT

### Data Layers

| Layer | Description | Example |
|-------|-------------|---------|
| **RAW** | Device-specific MQTT message | Binary buffer or JSON |
| **SIF** | Standard Intermediate Format | Normalized JSON |
| **SUO** | Standard Unified Object | Enriched with metadata |
| **UOS** | Unified Object Store | In-memory cache |

---

## Development Workflow

### Before Starting Work

1. Read the relevant specification in `specs/`
2. Review `architecture.md` for module design
3. Check `agents/coding_agent.md` for standards
4. Understand the data flow for your module

### During Implementation

1. Create/update one module at a time
2. Follow the interface definitions in architecture.md
3. Add appropriate error handling
4. Write tests alongside implementation
5. Update documentation as needed

### After Implementation

1. Run all tests to ensure nothing is broken
2. Verify module can be enabled/disabled via config
3. Check for code duplication (DRY principle)
4. Review against architecture document

---

## Database Notes

### Current Implementation

- **Database**: MySQL 8.0
- **Schema**: Defined in `specs/SUO_UOS_DB_Spec.md`
- **Features**: AUTO_INCREMENT, JSON type, DATETIME(3)

### Future Migration

PostgreSQL migration is planned for a future version. When implementing database layer:

- Use parameterized queries for compatibility
- Avoid MySQL-specific features where possible
- Document any MySQL-specific implementations

---

## Deployment Considerations

### Deployment Modes

| Mode | Use Case | Configuration |
|------|----------|---------------|
| **Minimal** | Testing, development | Core modules only |
| **Standard** | Production (typical) | Core + WebSocket + Database |
| **Full** | Enterprise | All modules enabled |

### Scaling

- **Vertical**: Increase resources, optimize cache
- **Horizontal**: Multiple instances with Redis, load balancer

### Docker (Future)

Docker deployment configurations are documented in `architecture.md` but not yet implemented.

---

## Common Pitfalls

1. **Binary Parsing**: V5008 binary messages require careful byte-order handling (Big-Endian)
2. **Module Index**: V5008 uses 1-based, V6800 uses various indexing - normalize consistently
3. **Event Timing**: Async event handlers must not block the event loop
4. **Memory Management**: Cache size limits must be enforced to prevent OOM
5. **Connection Recovery**: Always implement reconnection logic for MQTT and Database

---

## References

- **Architecture**: `architecture.md` (comprehensive system design)
- **Requirements**: `specs/prd.md` (product requirements)
- **V5008 Spec**: `specs/V5008_Spec.md` (binary protocol details)
- **V6800 Spec**: `specs/v6800_spec.md` (JSON protocol details)
- **Data Spec**: `specs/SUO_UOS_DB_Spec.md` (SUO/UOS/DB schema)
- **Coding Standards**: `agents/coding_agent.md`

---

**Document End**
