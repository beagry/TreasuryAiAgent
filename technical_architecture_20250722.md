# Payment AI - Technical Architecture

## System Architecture Overview

Payment AI implements a hybrid architecture combining conversational AI, structured data execution, and progressive learning capabilities for enterprise treasury automation.

## Component Architecture

### 1. Frontend Layer (Lovable Dev + ReactJS)

#### Technology Stack
- **Platform**: Lovable Dev
- **Framework**: ReactJS
- **Backend**: NodeJS
- **Styling**: Professional treasury aesthetic with deep navy (#2C3E50) color scheme
- **UI Pattern**: ChatGPT-style multi-chat interface

#### Key Features
- **Multi-Chat System**: Each conversation creates one payment rule
- **Context Isolation**: No bleed between different rule creation conversations
- **Read-Only Audit**: Completed chats become read-only for compliance
- **Professional Interface**: Enterprise-grade design for CFOs and treasury managers

#### Interface Modes
1. **Chat Mode**: Rule creation through conversation
2. **Rules Mode**: Management of existing automations
3. **Calendar Mode**: View upcoming and completed payouts
4. **Audit Mode**: Compliance and history tracking

### 2. AI Agent Layer (N8N Platform)

#### Core Technology
- **Platform**: N8N workflow automation
- **Deployment**: Cloud or local/VM deployment
- **Future**: Binary code implementation for production

#### Agent Capabilities
- **Educational AI**: Progressive disclosure of features
- **Parameter Extraction**: Natural language to structured data
- **Validation Framework**: Comprehensive rule validation
- **Context Management**: Session and company context preservation
- **Safety First**: Mock execution only, never real payments

#### Supported Logic Types
- **Simple Recurring**: Fixed amounts on schedule
- **Conditional Logic**: IF/THEN payment rules
- **Percentage-Based**: Revenue/profit sharing calculations
- **Balance Thresholds**: Trigger-based automations
- **Multi-Step**: Complex workflow automations
- **Multi-Account**: Cross-account coordinations

### 3. Database Layer (PostgreSQL + pgvector)

#### Architecture: Hybrid Structured + Vector
```sql
-- Core Tables
chat_sessions              -- ChatGPT-style chat management
ai_conversations          -- Message history and context
payment_scripts           -- Structured rule definitions
payment_executions        -- Execution history and results
treasury_configuration    -- Company settings and accounts

-- Knowledge Base Tables
company_knowledge_base    -- Progressive learning entities
knowledge_embeddings      -- Vector storage for semantic search
rule_audit_trail         -- Comprehensive audit logging
mock_execution_history    -- Test run tracking
```

#### Data Separation Strategy
- **Structured Data**: Used for actual payment execution
- **Vector Data**: Used for AI context and learning only
- **Safety Principle**: Execution never depends on vector/AI data

#### Vector Capabilities (pgvector)
- Company entity recognition and learning
- Semantic rule conflict detection
- Intelligent auto-complete suggestions
- Context-aware entity matching

### 4. Execution Layer (Current: EigenLayer Simulation)

#### Current Implementation
- **Platform**: EigenLayer agent simulation
- **Purpose**: Mock execution and validation
- **Integration**: N8N workflow triggers

#### Production Architecture
- **Worker-Based**: 100% structured data execution
- **AI Role**: Rule creation only, not execution
- **Safety**: Multiple validation layers before execution
- **Audit**: Complete trail for every action

## Data Flow Architecture

### Rule Creation Flow
1. **User Input** → Frontend chat interface
2. **Natural Language** → AI Agent (N8N)
3. **Parameter Extraction** → Structured validation
4. **Rule Storage** → PostgreSQL structured tables
5. **Audit Trail** → Compliance logging

### Execution Flow
1. **Scheduled Trigger** → Worker process
2. **Rule Retrieval** → PostgreSQL structured data
3. **Validation** → Multi-layer safety checks
4. **Mock Execution** → Preview and approval
5. **Real Execution** → Only after human approval

### Knowledge Learning Flow
1. **Entity Detection** → AI identifies companies/accounts/recipients
2. **User Approval** → Human validates new knowledge
3. **Vector Storage** → pgvector for future reference
4. **Semantic Matching** → Enhanced auto-complete

## Integration Points

### External Integrations
- **Zep Memory**: Session memory management (being enhanced)
- **EigenLayer**: Current execution simulation
- **Blockchain Networks**: Future payment execution
- **Banking APIs**: Traditional payment rails

### Internal Integrations
- **Frontend ↔ Database**: REST API for chat and rule management
- **AI Agent ↔ Database**: Rule saving and knowledge updates
- **AI Agent ↔ Frontend**: Real-time conversation handling
- **Execution Layer ↔ Database**: Structured data retrieval

## Security Architecture

### Multi-Layer Security
1. **Authentication**: User login and session management
2. **Authorization**: Role-based access control
3. **Data Encryption**: At rest and in transit
4. **Audit Logging**: Complete action history
5. **Validation**: Multiple checkpoints before execution

### Safety Principles
- **No AI Execution**: AI only creates rules, never executes payments
- **Human Approval**: Required for all real executions
- **Mock First**: All rules tested before real deployment
- **Audit Trail**: Complete history for compliance

## Scalability Considerations

### Horizontal Scaling
- **Database**: PostgreSQL with read replicas
- **AI Agent**: Multiple N8N instances
- **Frontend**: CDN and load balancing
- **Execution**: Distributed worker processes

### Performance Optimization
- **Database Indexing**: Optimized for chat and rule queries
- **Vector Search**: Efficient similarity matching
- **Caching**: Session and rule caching
- **Async Processing**: Non-blocking operations

## Development Environment

### Local Development
- **Database**: Local PostgreSQL with pgvector
- **AI Agent**: Local N8N instance
- **Frontend**: Lovable Dev development environment
- **Testing**: Mock execution environment

### Production Deployment
- **Database**: Supabase PostgreSQL
- **AI Agent**: Cloud N8N or binary deployment
- **Frontend**: Production Lovable deployment
- **Execution**: Secure worker infrastructure

---

*Last Updated: July 22, 2025*
*Architecture Version: Hybrid PostgreSQL + Vector with Knowledge Base*
*Status: AI Agent Framework Complete - Ready for Implementation*