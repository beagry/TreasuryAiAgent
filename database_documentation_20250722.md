# Payment AI - Database Documentation

## Database Overview

Payment AI uses a hybrid PostgreSQL + pgvector architecture that separates structured execution data from vector-based AI enhancements, ensuring safety and auditability while enabling progressive learning.

## Database Schema

### Core Tables

#### 1. Chat Sessions Management
```sql
create table public.chat_sessions (
  id uuid not null default extensions.uuid_generate_v4 (),
  chat_name character varying(255) null,
  rule_type character varying(50) null,
  status character varying(50) null default 'draft'::character varying,
  script_id uuid null,
  is_read_only boolean null default false,
  created_at timestamp without time zone null default now(),
  updated_at timestamp without time zone null default now(),
  last_activity timestamp without time zone null default now(),
  constraint chat_sessions_pkey primary key (id),
  constraint fk_chat_sessions_script_id foreign KEY (script_id) references payment_scripts (id) on delete set null
) TABLESPACE pg_default;

create index IF not exists idx_chat_sessions_status on public.chat_sessions using btree (status) TABLESPACE pg_default;

create index IF not exists idx_chat_sessions_updated_at on public.chat_sessions using btree (updated_at) TABLESPACE pg_default;

create index IF not exists idx_chat_sessions_script_id on public.chat_sessions using btree (script_id) TABLESPACE pg_default;

create trigger trigger_chat_sessions_updated_at BEFORE
update on chat_sessions for EACH row
execute FUNCTION update_updated_at_column ();
```

#### 2. AI Conversations
```sql
-- Message history for each chat session
create table public.ai_conversations (
  id uuid not null default extensions.uuid_generate_v4 (),
  chat_id uuid not null,
  session_id uuid not null,
  message_type character varying(50) not null,
  content text not null,
  parameters_extracted jsonb null,
  script_id uuid null,
  created_at timestamp without time zone null default now(),
  status character varying(20) null default 'pending'::character varying,
  constraint ai_conversations_pkey primary key (id),
  constraint ai_conversations_chat_id_fkey foreign KEY (chat_id) references chat_sessions (id) on delete CASCADE
) TABLESPACE pg_default;

create index IF not exists idx_ai_conversations_chat_id on public.ai_conversations using btree (chat_id) TABLESPACE pg_default;

create index IF not exists idx_ai_conversations_session_id on public.ai_conversations using btree (session_id) TABLESPACE pg_default;

create index IF not exists idx_ai_conversations_status on public.ai_conversations using btree (status) TABLESPACE pg_default;

create trigger trigger_update_chat_activity
after INSERT on ai_conversations for EACH row
execute FUNCTION update_chat_last_activity ();
```

#### 3. Payment Rules
```sql
-- Structured payment rule definitions
create table public.payment_scripts (
  id uuid not null default extensions.uuid_generate_v4 (),
  rule_name text not null default 'Rule Name #'::text,
  status character varying(50) null default 'draft'::character varying,
  parameters jsonb not null,
  created_at timestamp with time zone null default now(),
  updated_at timestamp with time zone null default now(),
  created_by text null,
  total_amount numeric(18, 6) null,
  execution_frequency character varying(50) null,
  start_date date null,
  end_date date null,
  short_description text null,
  last_time_executed timestamp with time zone null,
  constraint payment_scripts_pkey primary key (id)
) TABLESPACE pg_default;

create index IF not exists idx_payment_scripts_status on public.payment_scripts using btree (status) TABLESPACE pg_default;

create index IF not exists idx_payment_scripts_execution_frequency on public.payment_scripts using btree (execution_frequency) TABLESPACE pg_default;

create index IF not exists idx_payment_scripts_created_at on public.payment_scripts using btree (created_at) TABLESPACE pg_default;

create trigger trigger_payment_scripts_updated_at BEFORE
update on payment_scripts for EACH row
execute FUNCTION update_updated_at_column ();
```

#### 4. Rules Executions history
```sql
-- Execution history and results
create table public.mock_execution_history (
  id uuid not null default extensions.uuid_generate_v4 (),
  rule_id uuid not null,
  execution_date timestamp with time zone not null default now(),
  status text null default 'success'::text,
  constraint mock_execution_history_pkey primary key (id),
  constraint mock_execution_history_rule_id_fkey foreign KEY (rule_id) references payment_scripts (id) on delete CASCADE
) TABLESPACE pg_default;

create index IF not exists idx_mock_execution_history_script_id on public.mock_execution_history using btree (rule_id) TABLESPACE pg_default;

create index IF not exists idx_mock_execution_history_execution_date on public.mock_execution_history using btree (execution_date) TABLESPACE pg_default;
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
create table public.company_knowledge_base (
  id uuid not null default extensions.uuid_generate_v4 (),
  entity_type character varying(100) not null,
  entity_name character varying(255) not null,
  entity_value text not null,
  confidence_score numeric(3, 2) null default 0.95,
  usage_count integer null default 1,
  verified_by_user boolean null default false,
  created_at timestamp without time zone null default now(),
  updated_at timestamp without time zone null default now(),
  last_used timestamp without time zone null default now(),
  constraint company_knowledge_base_pkey primary key (id)
) TABLESPACE pg_default;

create index IF not exists idx_company_knowledge_entity_type on public.company_knowledge_base using btree (entity_type) TABLESPACE pg_default;

create index IF not exists idx_company_knowledge_entity_name on public.company_knowledge_base using btree (entity_name) TABLESPACE pg_default;

create index IF not exists idx_company_knowledge_verified on public.company_knowledge_base using btree (verified_by_user) TABLESPACE pg_default;
```

### Audit and Compliance Tables

#### 7. Rule Audit Trail
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

*Last Updated: July 23, 2025*
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

*Last Updated: July 23, 2025*
*Development Phase: Core Implementation Sprint*
*Next Review: Weekly sprint planning meetings*
*Critical Path: Rule saving implementation → End-to-end testing → MVP completion*
