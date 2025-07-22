# Payment AI - Database Documentation

## Database Overview

Payment AI uses a hybrid PostgreSQL + pgvector architecture that separates structured execution data from vector-based AI enhancements, ensuring safety and auditability while enabling progressive learning.

## Database Schema

### Core Tables

#### 1. Chat Sessions Management
```sql
-- ChatGPT-style multi-chat interface support
CREATE TABLE chat_sessions (
    chat_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    session_name VARCHAR(255),
    status VARCHAR(50) DEFAULT 'draft', -- draft, completed, read_only, abandoned
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    completed_at TIMESTAMP WITH TIME ZONE,
    rule_id UUID, -- Links to payment_scripts when rule is created
    
    INDEX idx_chat_user (user_id),
    INDEX idx_chat_status (status),
    INDEX idx_chat_created (created_at)
);
```

#### 2. AI Conversations
```sql
-- Message history for each chat session
CREATE TABLE ai_conversations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    chat_id UUID NOT NULL REFERENCES chat_sessions(chat_id),
    message_type VARCHAR(20) NOT NULL, -- user, assistant, system
    content TEXT NOT NULL,
    context_data JSONB, -- Additional context for AI processing
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    INDEX idx_conv_chat (chat_id),
    INDEX idx_conv_type (message_type),
    INDEX idx_conv_created (created_at)
);
```

#### 3. Payment Scripts (Core Rules)
```sql
-- Structured payment rule definitions
CREATE TABLE payment_scripts (
    script_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    chat_id UUID REFERENCES chat_sessions(chat_id),
    
    -- Rule Metadata
    rule_name VARCHAR(255) NOT NULL,
    rule_type VARCHAR(50) NOT NULL, -- contractor_payment, revenue_allocation, profit_distribution
    status VARCHAR(50) DEFAULT 'draft', -- draft, active, paused, archived
    
    -- Financial Parameters
    source_account JSONB NOT NULL, -- {address, type, name}
    recipients JSONB NOT NULL, -- [{address, name, share/amount}]
    amount_logic JSONB NOT NULL, -- {type: fixed/percentage/conditional, value, conditions}
    
    -- Execution Configuration
    execution_schedule JSONB, -- {type: one_time/recurring, frequency, start_date}
    conditions JSONB, -- Additional conditions for execution
    limits JSONB, -- Safety limits and constraints
    
    -- Approval and Safety
    requires_approval BOOLEAN DEFAULT true,
    user_approval BOOLEAN DEFAULT false,
    safety_checks JSONB, -- Validation rules and safety constraints
    
    -- Metadata
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    last_execution TIMESTAMP WITH TIME ZONE,
    
    INDEX idx_script_user (user_id),
    INDEX idx_script_type (rule_type),
    INDEX idx_script_status (status),
    INDEX idx_script_chat (chat_id)
);
```

#### 4. Payment Executions
```sql
-- Execution history and results
CREATE TABLE payment_executions (
    execution_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    script_id UUID NOT NULL REFERENCES payment_scripts(script_id),
    
    -- Execution Details
    execution_type VARCHAR(20) NOT NULL, -- mock, real
    status VARCHAR(50) NOT NULL, -- pending, completed, failed, cancelled
    
    -- Transaction Data
    source_account VARCHAR(255) NOT NULL,
    recipients JSONB NOT NULL, -- Actual recipients and amounts
    total_amount DECIMAL(20,8),
    currency VARCHAR(10),
    
    -- Blockchain/Payment Data
    transaction_hashes JSONB, -- Array of transaction hashes if applicable
    gas_fees DECIMAL(20,8),
    network VARCHAR(50),
    
    -- Execution Context
    triggered_by VARCHAR(50), -- schedule, manual, condition
    trigger_context JSONB,
    error_details TEXT,
    
    -- Timestamps
    scheduled_at TIMESTAMP WITH TIME ZONE,
    executed_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    completed_at TIMESTAMP WITH TIME ZONE,
    
    INDEX idx_exec_script (script_id),
    INDEX idx_exec_status (status),
    INDEX idx_exec_type (execution_type),
    INDEX idx_exec_scheduled (scheduled_at)
);
```

#### 5. Treasury Configuration
```sql
-- Company and user treasury settings
CREATE TABLE treasury_configuration (
    config_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    company_name VARCHAR(255),
    
    -- Account Configuration
    accounts JSONB NOT NULL, -- Array of available accounts
    default_settings JSONB, -- Default rules and preferences
    
    -- Security Settings
    approval_settings JSONB, -- Approval requirements and thresholds
    notification_settings JSONB, -- Alert and notification preferences
    
    -- Compliance
    compliance_rules JSONB, -- Regulatory and internal compliance rules
    audit_settings JSONB, -- Audit trail preferences
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    INDEX idx_config_user (user_id),
    UNIQUE(user_id) -- One config per user
);
```

### Knowledge Base Tables (Vector-Enhanced)

#### 6. Company Knowledge Base
```sql
-- Progressive learning of company entities
CREATE TABLE company_knowledge_base (
    entity_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    
    -- Entity Information
    entity_type VARCHAR(50) NOT NULL, -- account, recipient, amount, schedule, condition
    entity_name VARCHAR(255) NOT NULL,
    entity_data JSONB NOT NULL, -- Structured entity information
    
    -- Learning Metadata
    confidence_score DECIMAL(3,2) DEFAULT 0.50, -- 0.00 to 1.00
    usage_count INTEGER DEFAULT 1,
    last_used TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    -- User Validation
    user_verified BOOLEAN DEFAULT false,
    verification_date TIMESTAMP WITH TIME ZONE,
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    INDEX idx_knowledge_user (user_id),
    INDEX idx_knowledge_type (entity_type),
    INDEX idx_knowledge_verified (user_verified),
    INDEX idx_knowledge_confidence (confidence_score)
);
```

#### 7. Knowledge Embeddings (Vector Storage)
```sql
-- Vector storage for semantic search and matching
CREATE TABLE knowledge_embeddings (
    embedding_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_id UUID NOT NULL REFERENCES company_knowledge_base(entity_id),
    
    -- Vector Data
    embedding VECTOR(1536), -- OpenAI embedding dimension
    embedding_model VARCHAR(100) DEFAULT 'text-embedding-3-small',
    
    -- Search Metadata
    searchable_text TEXT NOT NULL,
    entity_context JSONB, -- Additional context for matching
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    INDEX idx_embedding_entity (entity_id)
);

-- Vector similarity search index
CREATE INDEX idx_embedding_cosine ON knowledge_embeddings 
USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);
```

### Audit and Compliance Tables

#### 8. Rule Audit Trail
```sql
-- Comprehensive audit logging
CREATE TABLE rule_audit_trail (
    audit_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    script_id UUID REFERENCES payment_scripts(script_id),
    user_id UUID NOT NULL,
    
    -- Action Details
    action_type VARCHAR(50) NOT NULL, -- create, update, delete, execute, approve
    action_details JSONB NOT NULL, -- What changed
    
    -- Context
    source_component VARCHAR(50), -- chat, rules_ui, api, system
    user_context JSONB, -- User session and context
    
    -- Change Tracking
    old_values JSONB, -- Previous state
    new_values JSONB, -- New state
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    INDEX idx_audit_script (script_id),
    INDEX idx_audit_user (user_id),
    INDEX idx_audit_action (action_type),
    INDEX idx_audit_created (created_at)
);
```

#### 9. Mock Execution History
```sql
-- Track all mock executions for validation
CREATE TABLE mock_execution_history (
    mock_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    script_id UUID NOT NULL REFERENCES payment_scripts(script_id),
    chat_id UUID REFERENCES chat_sessions(chat_id),
    
    -- Mock Details
    mock_parameters JSONB NOT NULL, -- Input parameters for mock
    mock_results JSONB NOT NULL, -- Calculated results
    validation_status VARCHAR(50) NOT NULL, -- passed, failed, warning
    validation_details JSONB, -- Validation results and issues
    
    -- User Interaction
    user_approved BOOLEAN,
    user_feedback TEXT,
    proceeded_to_real BOOLEAN DEFAULT false,
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    INDEX idx_mock_script (script_id),
    INDEX idx_mock_chat (chat_id),
    INDEX idx_mock_status (validation_status),
    INDEX idx_mock_created (created_at)
);
```

## Database Setup Script

See the complete setup script in `payment_ai_database_setup_20250720.sql` which includes:
- Table creation with proper constraints
- Index optimization for performance
- Sample data for development
- Security roles and permissions
- Triggers for audit trail automation

## Key Design Principles

### 1. Safety-First Architecture
- **Structured Execution**: Only structured data used for real payments
- **Vector Context**: Vector data used only for AI enhancement, never execution
- **Audit Trail**: Complete history of all actions and changes

### 2. Progressive Learning
- **Entity Recognition**: Automatic detection of company-specific entities
- **User Validation**: Human approval required for knowledge base updates
- **Confidence Scoring**: Machine learning confidence with human override

### 3. Multi-Chat Support
- **Isolation**: Each chat creates one rule with proper context isolation
- **State Management**: Clear progression from draft to completed to read-only
- **Audit Integration**: Full traceability from conversation to execution

### 4. Performance Optimization
- **Strategic Indexing**: Optimized for common query patterns
- **Vector Search**: Efficient similarity search using ivfflat
- **Data Partitioning**: Ready for scaling with proper partitioning strategies

---

*Last Updated: July 22, 2025*
*Database Version: Hybrid PostgreSQL + pgvector with Knowledge Base*
*Setup Script: payment_ai_database_setup_20250720.sql*communication patterns
- **Context Limitation Handling**: Smart context management for large conversations
- **Message Optimization**: Intelligent filtering of conversation history

#### Advanced Analytics
- **Performance Dashboards**: Treasury automation effectiveness metrics
- **Cost Analysis**: Payment efficiency and cost optimization
- **Compliance Reporting**: Automated regulatory reporting

#### Integration Ecosystem
- **Banking APIs**: Direct integration with traditional banking systems
- **Blockchain Networks**: Multi-chain payment execution
- **Accounting Systems**: Integration with QuickBooks, Xero, SAP
- **ERP Integration**: Enterprise resource planning connectivity

## Risk Assessment & Mitigation

### Technical Risks

#### 1. AI Hallucination in Financial Context
- **Risk**: AI generating incorrect payment parameters
- **Mitigation**: Comprehensive validation, human approval, structured data separation
- **Status**: Validation framework designed and implemented

#### 2. Database Performance at Scale
- **Risk**: PostgreSQL performance degradation with large conversation history
- **Mitigation**: Strategic indexing, data archiving, read replicas
- **Status**: Performance optimization planned for Phase 2

#### 3. N8N Workflow Complexity
- **Risk**: Complex AI workflows becoming difficult to maintain
- **Mitigation**: Modular design, comprehensive documentation, binary migration path
- **Status**: Architecture designed for maintainability

### Business Risks

#### 4. Regulatory Compliance
- **Risk**: Payment automation may trigger regulatory scrutiny
- **Mitigation**: Comprehensive audit trail, human approval requirements, compliance features
- **Status**: Compliance framework integrated into design

#### 5. User Adoption
- **Risk**: Treasury professionals may resist AI-powered automation
- **Mitigation**: Educational AI capabilities, progressive disclosure, safety-first approach
- **Status**: Educational framework implemented in AI agent

## Success Metrics

### MVP Success Criteria
- **Technical**: 95%+ rule parsing accuracy, <2 second response times
- **User Experience**: 60%+ time reduction in rule creation, 90%+ user satisfaction
- **Safety**: Zero unauthorized executions, 100% audit trail coverage
- **Adoption**: 10+ active companies, 50+ payment rules created

### Production Success Criteria
- **Scale**: 100+ companies, 1000+ payment rules, $10M+ processed
- **Performance**: 99.9% uptime, <1 second response times
- **Compliance**: Zero compliance violations, 100% audit pass rate
- **Business**: $500K+ ARR, 80%+ customer retention

## Technology Stack Justification

### Why This Stack Was Chosen

#### Frontend: Lovable Dev + ReactJS
**Decision Rationale:**
- **Rapid MVP Development**: Lovable.dev enables quick creation of sophisticated interfaces
- **Professional Quality**: Generates enterprise-grade UI suitable for CFOs and treasury managers
- **Comparison Advantage**: More stable than v0.dev or bolt.new for complex financial interfaces
- **ReactJS Foundation**: Provides solid foundation for future custom development
- **Cost-Effective**: Faster development reduces time-to-market and initial costs

#### AI Platform: N8N
**Decision Rationale:**
- **Community Support**: Vast documentation and knowledge base created by active community
- **Integration Ecosystem**: Extensive list of pre-built integrations for financial and business systems
- **Stability**: More reliable than make.com or linear.app for complex workflow automation
- **Learning Curve**: Better documentation makes development process smoother
- **Migration Path**: Can be converted to binary code for production performance when needed

#### Database: PostgreSQL + pgvector
**Decision Rationale:**
- **Hybrid Architecture**: Supports both structured data (execution) and vector data (AI enhancement)
- **Safety Separation**: Clear separation between execution data and AI context
- **Supabase Integration**: Managed PostgreSQL with built-in features
- **Vector Capabilities**: Native vector search without separate vector database complexity
- **ACID Compliance**: Critical for financial data integrity and audit requirements

#### Trade-offs Acknowledged
- **Performance vs Speed**: Some performance sacrifice for faster MVP development
- **Vendor Lock-in**: Lovable.dev dependency managed with ReactJS foundation
- **Scalability**: Architecture designed to migrate components as scale requirements grow

## Next Steps & Action Items

### Immediate Actions (This Week)
1. **Database Setup**: Deploy Supabase PostgreSQL with pgvector extension
2. **N8N Environment**: Set up N8N instance and basic workflow structure
3. **Rule Saving Logic**: Implement first iteration of AI agent rule saving
4. **Frontend Integration**: Begin connecting Lovable.dev UI to real database

### Sprint Planning (Next 2 Weeks)
1. **Complete Rule Saving**: Fix critical rule saving bug
2. **End-to-End Testing**: Test complete user flow from chat to database
3. **Knowledge Base Integration**: Connect company context learning
4. **Basic Authentication**: Implement secure user login and session management

### Monthly Milestones
- **Month 1**: MVP core functionality complete and tested
- **Month 2**: Production safety features and multi-company support
- **Month 3**: Advanced features and scaling preparation
- **Month 6**: Production deployment with enterprise customers

---

*Last Updated: July 22, 2025*
*Development Phase: Core Implementation Sprint*
*Next Review: Weekly sprint planning meetings*
*Critical Path: Rule saving implementation → End-to-end testing → MVP completion*