# System Architecture

**Startup Data Collection & Automation System**

---

## Overview

A modular, scalable data collection and processing system designed for automated startup intelligence gathering. The architecture follows a layered approach with clear separation between data acquisition, processing, storage, and monitoring.

**Design Goals:**
- Modular components that can scale independently
- Support for both manual and automated data collection
- Quality-first approach with validation at each stage
- Extensible to multiple data sources and outputs

---

## Architecture Diagram

![System Architecture](../architecture/architecture_diagram.png)

---

## System Layers

### 1. Data Acquisition Layer

**Purpose:** Collect startup data from multiple sources

#### Current Implementation
**Manual Research**
- Crunchbase Web Portal
- Y Combinator Directory  
- Product Hunt Listings

**Output:** 40 validated Indian startup records in standardized format

#### Designed Capability
**API Integration**

| API | Endpoint | Authentication | Rate Limit |
|-----|----------|----------------|------------|
| Crunchbase | `/v4/entities/organizations` | API Key (Header) | Tier-based |
| PredictLeads | `/api/companies` | Bearer Token | 1000/month |
| OpenCorporates | `/companies` | API Key | 500/month |

**Data Format:** JSON responses mapped to standardized schema

---

### 2. Data Processing Layer

**Technology:** n8n workflow automation

#### Processing Pipeline

```
┌─────────────────┐
│ Data Ingestion  │  → Fetch from API or load CSV
└────────┬────────┘
         ↓
┌─────────────────┐
│ Transformation  │  → Clean, normalize, enrich
└────────┬────────┘
         ↓
┌─────────────────┐
│ Quality Filter  │  → Validate required fields
└────────┬────────┘
         ↓
┌─────────────────┐
│ Validation      │  → Check formats, remove duplicates
└─────────────────┘
```

#### Transformation Logic

**Operations:**
- Extract relevant fields from API responses
- Normalize data types (dates, enums, text)
- Add metadata (source, timestamp)
- Handle null/missing values with defaults

**Implementation:**
```javascript
// JavaScript code node in n8n
const transformedData = items.map(item => ({
    company_name: item.properties?.name || 'N/A',
    website: item.properties?.website || '',
    industry: item.properties?.categories?.join(', ') || 'N/A',
    collected_at: new Date().toISOString(),
    source: 'Crunchbase API'
}));
```

#### Quality Filtering

**Validation Rules:**
- Required fields: company_name, industry
- Format checks: email format, URL format
- Duplicate detection: (company_name + website) uniqueness
- Data completeness: minimum 60% field population

---

### 3. Data Storage Layer

#### Current Implementation

**CSV Storage**
- Format: UTF-8 encoded CSV
- Records: 40 Indian startups
- Fields: 11 standardized columns
- Location: `/data/startup_data_assignment_ready.csv`

**Advantages:**
- Simple, portable format
- No infrastructure required
- Version control friendly
- Human-readable

**Limitations:**
- No concurrent write support
- Limited query capabilities
- Manual synchronization required

#### Designed Capability

**PostgreSQL Database**

Schema design:
```sql
CREATE TABLE startups (
    id SERIAL PRIMARY KEY,
    company_name VARCHAR(255) NOT NULL,
    website VARCHAR(500),
    founder_name VARCHAR(255),
    contact_email VARCHAR(255),
    industry VARCHAR(100),
    sub_sector VARCHAR(100),
    product_description TEXT,
    target_customers TEXT,
    location VARCHAR(200),
    startup_stage VARCHAR(50),
    source VARCHAR(100),
    collected_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(company_name, website)
);

CREATE INDEX idx_industry ON startups(industry);
CREATE INDEX idx_location ON startups(location);
CREATE INDEX idx_stage ON startups(startup_stage);
```

**Google Sheets Integration**
- Real-time collaboration
- No-code data access for non-technical users
- Auto-sync on workflow execution
- Export-friendly format

---

### 4. Monitoring & Alerts Layer

**Email Notifications**
- Trigger: Workflow completion or failure
- Content: Summary statistics, timestamps, error details
- Recipients: Team distribution list
- Protocol: SMTP

**Notification Template:**
```
Subject: ✅ Startup Data Collection Complete

Summary:
- Records processed: 47
- New records: 5
- Duplicates filtered: 2
- Execution time: 45s
- Timestamp: 2026-02-15 09:00:00

[Link to Google Sheet]
```

---

## Data Flow

### End-to-End Process

```
┌──────────────┐
│ Data Sources │  Crunchbase, YC, Product Hunt
└──────┬───────┘
       │
       ↓ HTTP GET / Manual Research
┌──────────────┐
│ n8n Workflow │
│              │
│ ┌──────────┐ │
│ │ Ingest   │ │  Fetch/Load data
│ └────┬─────┘ │
│      ↓       │
│ ┌──────────┐ │
│ │Transform │ │  Clean & normalize
│ └────┬─────┘ │
│      ↓       │
│ ┌──────────┐ │
│ │ Filter   │ │  Validate quality
│ └────┬─────┘ │
│      ↓       │
│ ┌──────────┐ │
│ │ Validate │ │  Check duplicates
│ └────┬─────┘ │
└──────┼───────┘
       │
       ├──────────────┬──────────────┐
       ↓              ↓              ↓
┌──────────┐   ┌──────────┐   ┌──────────┐
│   CSV    │   │PostgreSQL│   │  Sheets  │
│ (Current)│   │(Designed)│   │(Designed)│
└──────────┘   └──────────┘   └────┬─────┘
                                    │
                                    ↓
                              ┌──────────┐
                              │  Email   │
                              │  Alert   │
                              └──────────┘
```

---

## Technology Stack

### Core Components

| Layer | Technology | Version | Purpose |
|-------|-----------|---------|---------|
| Orchestration | n8n | 0.200.0+ | Workflow automation |
| Storage (Current) | CSV | UTF-8 | Data persistence |
| Storage (Designed) | PostgreSQL | 14.x | Relational database |
| Collaboration | Google Sheets | API v4 | Team access |
| Processing | JavaScript | ES6+ | Data transformation |
| Runtime | Node.js | 14.x+ | Execution environment |

### External APIs

| Service | Purpose | Status |
|---------|---------|--------|
| Crunchbase API | Company data | Designed |
| PredictLeads API | Tech intelligence | Designed |
| OpenCorporates API | Registration data | Designed |
| Google Sheets API | Data export | Designed |
| SMTP | Notifications | Designed |

---

## Scalability

### Current Capacity
- **Records:** 40 startups
- **Storage:** ~50 KB CSV file
- **Processing:** Manual workflow execution
- **Users:** Single operator

### Design Capacity
- **Records:** 10,000+ startups
- **Storage:** PostgreSQL (millions of records)
- **Processing:** Automated daily execution
- **Users:** Multiple concurrent via Google Sheets

### Scaling Strategy

**Horizontal Scaling:**
- Add new API sources (AngelList, LinkedIn)
- Geographic expansion (multiple countries)
- Industry verticals (sector-specific data)

**Vertical Scaling:**
- Database indexing and partitioning
- Parallel API requests (batch processing)
- Caching frequently accessed data
- Query optimization

**Performance Targets:**
- API data collection: 1000 records/hour
- Database queries: <100ms response time
- Workflow execution: <5 minutes full cycle

---

## Data Schema

### Standard Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| company_name | String | Yes | Official company name |
| website | URL | Yes | Company website |
| founder_name | String | No | Primary founder |
| contact_email | Email | No | Business contact |
| industry | String | Yes | Primary industry |
| sub_sector | String | No | Specific focus area |
| product_description | Text | No | What the company does |
| target_customers | Text | No | Target market |
| location | String | No | City, State format |
| startup_stage | Enum | No | Funding stage |
| source | String | Yes | Data source identifier |

### Data Types

**Enums:**
- `startup_stage`: ['Seed', 'Series A', 'Series B', 'Series C', 'Series D+', 'Public', 'Bootstrapped']
- `industry`: ['FinTech', 'EdTech', 'HealthTech', 'E-commerce', 'Mobility', 'SaaS', 'Other']

---

## Security & Privacy

### Data Protection

**Current:**
- Publicly available data only
- No PII beyond business contacts
- Local storage (CSV)

**Production Requirements:**
- API credentials in environment variables
- Encrypted database connections (SSL/TLS)
- Role-based access control (RBAC)
- Audit logging for data access

### Compliance

**Data Collection:**
- Respect API Terms of Service
- Honor rate limits
- Use official APIs (no scraping)

**Data Usage:**
- Business development purposes only
- No data redistribution
- Opt-out mechanism for contacted companies

---

## Deployment

### Current Setup
- Local n8n instance
- Manual workflow execution
- CSV file storage
- No automated scheduling

### Production Setup (Designed)

**Infrastructure:**
```
┌─────────────────────────────────────┐
│ Docker Container (n8n)              │
│ ├─ Workflow engine                  │
│ ├─ Scheduled triggers               │
│ └─ Environment variables            │
└─────────────┬───────────────────────┘
              │
              ├─────────────────┬──────────────────┐
              ↓                 ↓                  ↓
┌──────────────────┐  ┌──────────────────┐  ┌────────────────┐
│ PostgreSQL       │  │ Google Sheets    │  │ SMTP Server    │
│ (Data storage)   │  │ (Team access)    │  │ (Alerts)       │
└──────────────────┘  └──────────────────┘  └────────────────┘
```

**Deployment Steps:**
1. Set up PostgreSQL database
2. Configure n8n with environment variables
3. Import workflow JSON
4. Add API credentials
5. Set up Google Sheets OAuth
6. Configure SMTP settings
7. Enable workflow schedule
8. Test end-to-end execution

---

## Error Handling

### Strategy

**API Failures:**
- Exponential backoff retry (3 attempts)
- Fallback to cached data
- Log errors with context
- Email notification on repeated failures

**Data Quality Issues:**
- Skip invalid records (don't halt workflow)
- Log validation failures
- Maintain error count metrics
- Alert on high error rate (>10%)

**System Failures:**
- Workflow state persistence
- Automatic restart on crash
- Health check monitoring
- Backup notification channels

---

## Monitoring

### Metrics

**Operational:**
- Workflow execution time
- API response times
- Error rate per source
- Data quality score

**Business:**
- New startups discovered per day
- Industry distribution
- Geographic coverage
- Data completeness percentage

### Observability

**Logging:**
- Structured logs (JSON format)
- Log levels: DEBUG, INFO, WARN, ERROR
- Correlation IDs for tracing
- Retention: 30 days

**Alerting:**
- Workflow failures (immediate)
- High error rate (>10%)
- API quota exhaustion
- Data quality degradation

---

## Current State vs Designed State

| Component | Current | Designed |
|-----------|---------|----------|
| **Data Collection** | Manual research | Automated API calls |
| **Storage** | CSV file | PostgreSQL + Sheets |
| **Workflow** | Design only | Scheduled execution |
| **Monitoring** | None | Email alerts + metrics |
| **Access** | Single user | Multi-user (Sheets) |
| **Updates** | Manual | Automated daily |

---

## Conclusion

The system architecture balances **immediate delivery** (manual data collection, CSV storage) with **future scalability** (API integration, database storage, automation). The modular design allows incremental enhancement without requiring complete rebuild.

**Key Architectural Decisions:**
- n8n for workflow orchestration (visual, maintainable)
- CSV for current storage (simple, portable)
- PostgreSQL for future storage (scalable, queryable)
- Phased approach (manual → automated)

This architecture demonstrates both **practical execution** (delivering working components) and **strategic thinking** (designing for scale).

