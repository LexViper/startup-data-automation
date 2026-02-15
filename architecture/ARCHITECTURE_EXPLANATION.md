# System Architecture Documentation

## Startup Data Collection & Automation System

---

## 📋 Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [System Components](#system-components)
3. [Data Flow](#data-flow)
4. [Technology Stack](#technology-stack)
5. [Implementation Phases](#implementation-phases)
6. [Scalability Considerations](#scalability-considerations)
7. [Security & Privacy](#security--privacy)

---

## Architecture Overview

The Startup Data Collection System is designed as a **modular, scalable automation platform** for discovering, collecting, and managing startup data for business development and outreach purposes.

### Design Principles

1. **Modularity** - Each component can be developed and deployed independently
2. **Scalability** - Architecture supports growth from 40 to 10,000+ records
3. **Automation** - Minimize manual intervention through workflow orchestration
4. **Data Quality** - Built-in validation and filtering mechanisms
5. **Flexibility** - Easy to add new data sources and integrations

---

## System Components

### 1. DATA ACQUISITION LAYER

The data acquisition layer is responsible for collecting startup information from multiple sources.

#### Current Implementation (Phase 1)
**Manual Research**
- **Source**: Crunchbase Web Portal
  - Purpose: Comprehensive startup database
  - Data: Company info, funding, founders
  - Access: Manual web browsing and extraction

- **Source**: Y Combinator Directory
  - Purpose: YC-backed startup listings
  - Data: Batch info, company details, founders
  - Access: Public directory browsing

- **Source**: Product Hunt
  - Purpose: New product launches
  - Data: Product descriptions, categories, upvotes
  - Access: Manual platform research

**Data Collected**: 40 Indian startup records with 11 standardized fields

#### Designed for Future (Phase 2)
**API Integration**
- **Crunchbase API**
  - Endpoint: `https://api.crunchbase.com/v4/entities/organizations`
  - Authentication: API key (HTTP Header)
  - Rate Limit: Based on subscription tier
  - Data Format: JSON

- **PredictLeads API**
  - Endpoint: `https://predictleads.com/api/companies`
  - Authentication: API token
  - Rate Limit: 1000 requests/month (free tier)
  - Data Format: JSON

- **OpenCorporates API**
  - Endpoint: `https://api.opencorporates.com/companies`
  - Authentication: API key
  - Rate Limit: 500 requests/month (free tier)
  - Data Format: JSON

---

### 2. DATA PROCESSING LAYER (n8n Workflow)

The processing layer handles data transformation, cleaning, validation, and routing using n8n workflow automation.

#### Workflow Components

**A. Data Ingestion**
- **Node Type**: HTTP Request / CSV Reader
- **Purpose**: Fetch data from APIs or load from CSV
- **Configuration**:
  - Method: GET for APIs
  - Headers: Authentication tokens
  - Query Parameters: Field selection, pagination
- **Output**: Raw JSON/CSV data

**B. Data Transformation**
- **Node Type**: Code (JavaScript)
- **Purpose**: Clean, format, and structure data
- **Operations**:
  - Extract relevant fields from API responses
  - Normalize field names and formats
  - Convert data types (dates, numbers)
  - Add metadata (source, timestamp)
  - Handle missing values with defaults
- **Example Logic**:
```javascript
const transformedData = [];
for (const item of items) {
    const startup = item.json;
    transformedData.push({
        company_name: startup.properties?.name || 'N/A',
        website: startup.properties?.website || '',
        founder_name: startup.properties?.founder || 'N/A',
        // ... additional fields
        collected_at: new Date().toISOString(),
        source: 'Crunchbase API'
    });
}
return transformedData.map(data => ({ json: data }));
```

**C. Data Filtering**
- **Node Type**: Filter
- **Purpose**: Apply quality control and criteria matching
- **Conditions**:
  - Non-empty required fields (name, industry)
  - Valid email format
  - Indian location (if filtering by geography)
  - Specific industries (FinTech, EdTech, etc.)
  - Funding stage criteria
- **Output**: Only validated records proceed

**D. Data Validation**
- **Implicit**: Through filter conditions
- **Checks**:
  - Required field presence
  - Data type validation
  - Format validation (URLs, emails)
  - Duplicate detection (by company name + website)

---

### 3. DATA STORAGE LAYER

The storage layer provides persistent storage and easy access to collected data.

#### Current Implementation (Phase 1)
**CSV Format**
- **File**: `startup_data_assignment_ready.csv`
- **Location**: `/data` directory
- **Records**: 40 Indian startups
- **Encoding**: UTF-8
- **Advantages**:
  - Human-readable
  - Excel/Sheets compatible
  - Easy to share and version control
  - No infrastructure required
- **Limitations**:
  - No concurrent access control
  - Limited query capabilities
  - Manual updates required

#### Designed for Future (Phase 2)
**Scalable Storage**

**PostgreSQL Database** (Primary Storage)
- **Purpose**: Structured, scalable data storage
- **Schema**:
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
    last_updated TIMESTAMP DEFAULT NOW(),
    UNIQUE(company_name, website)
);
```
- **Indexes**:
  - `idx_industry` on industry column
  - `idx_location` on location column
  - `idx_stage` on startup_stage column
- **Advantages**:
  - ACID compliance
  - Complex querying with SQL
  - Concurrent access support
  - Data integrity constraints
  - Scalable to millions of records

**Google Sheets Integration** (Team Access)
- **Purpose**: Real-time collaboration and viewing
- **Node Type**: Google Sheets (n8n)
- **Operation**: Append rows
- **Configuration**:
  - Document ID: Linked spreadsheet
  - Sheet Name: "Startups"
  - Column Mapping: Automated field mapping
- **Advantages**:
  - No technical skills required for viewing
  - Real-time updates
  - Easy filtering and sorting
  - Share with non-technical team members
  - Export to various formats

---

### 4. MONITORING & ALERTS LAYER

The monitoring layer provides visibility into system operations and alerts stakeholders.

#### Designed Components

**Email Notifications**
- **Node Type**: Email Send (SMTP)
- **Trigger**: Workflow completion or failure
- **Configuration**:
  - SMTP Server: Gmail, SendGrid, or custom
  - Recipients: Team distribution list
  - Subject: Status indicator (✅ Success / ❌ Failure)
- **Content**:
  - Summary statistics (records processed)
  - Execution timestamp
  - Links to data (Google Sheets, dashboard)
  - Error details (if failure)

**Example Notification**:
```
Subject: ✅ Daily Startup Data Collection Complete

Hi Team,

The daily startup data collection has completed successfully!

📊 Summary:
- Total startups processed: 47
- New startups added: 5
- Duplicates filtered: 2
- Data saved to database ✓
- Google Sheets updated ✓
- Collection timestamp: 2026-02-15 09:00:00

View the data: [Link to Google Sheet]

Best regards,
Automation System
```

---

## Data Flow

### End-to-End Data Journey

```
1. DATA SOURCES
   ├─ Crunchbase API
   ├─ Y Combinator Directory
   └─ Product Hunt
          ↓
2. INGESTION (n8n HTTP Request Node)
   ├─ Fetch data via REST APIs
   ├─ Handle pagination
   └─ Manage rate limits
          ↓
3. TRANSFORMATION (n8n Code Node)
   ├─ Extract relevant fields
   ├─ Normalize formats
   ├─ Add metadata
   └─ Handle missing data
          ↓
4. FILTERING (n8n Filter Node)
   ├─ Validate required fields
   ├─ Check data quality
   ├─ Apply business criteria
   └─ Remove duplicates
          ↓
5. STORAGE
   ├─ PostgreSQL (structured storage)
   │   ├─ Insert new records
   │   ├─ Update existing records
   │   └─ Maintain data integrity
   │
   └─ Google Sheets (team access)
       ├─ Append new rows
       └─ Real-time collaboration
          ↓
6. NOTIFICATION (n8n Email Node)
   ├─ Send success confirmation
   ├─ Include summary stats
   └─ Provide data access links
```

### Data Flow Diagram

*[Your architecture diagram image would go here]*

---

## Technology Stack

### Core Technologies

| Component | Technology | Purpose | Version |
|-----------|-----------|---------|---------|
| Workflow Engine | n8n | Automation & orchestration | 0.200.0+ |
| Data Format | CSV | Current data storage | UTF-8 |
| Database | PostgreSQL | Scalable storage (designed) | 14.x |
| Cloud Storage | Google Sheets | Team collaboration (designed) | API v4 |
| Programming | JavaScript | Data transformation logic | ES6+ |
| Runtime | Node.js | n8n execution environment | 14.x+ |

### APIs & Services

| Service | Purpose | Integration Status |
|---------|---------|-------------------|
| Crunchbase API | Startup data collection | Designed |
| PredictLeads API | Technology & funding data | Designed |
| OpenCorporates API | Company registration data | Designed |
| Google Sheets API | Data export & collaboration | Designed |
| SMTP Service | Email notifications | Designed |

### Development Tools

- **Version Control**: Git
- **Code Editor**: VS Code (recommended)
- **Database Client**: pgAdmin / DBeaver
- **API Testing**: Postman / Insomnia
- **Container Platform**: Docker (optional)

---

## Implementation Phases

### Phase 1: Manual Data Collection ✅ COMPLETE

**Timeline**: Week 1
**Status**: ✅ Delivered

**Activities**:
- ✅ Identified data sources (Crunchbase, YC, Product Hunt)
- ✅ Defined data schema (11 columns)
- ✅ Manually researched and collected 40 Indian startups
- ✅ Structured data in CSV format
- ✅ Validated data quality

**Deliverables**:
- `startup_data_assignment_ready.csv` with 40 records
- Data covering multiple industries (FinTech, EdTech, E-commerce, etc.)
- Geographic focus on major Indian startup hubs

**Outcome**: Demonstrated understanding of data requirements and manual research methodology.

---

### Phase 2: Workflow Design ✅ COMPLETE

**Timeline**: Week 1-2
**Status**: ✅ Delivered

**Activities**:
- ✅ Designed n8n workflow architecture
- ✅ Created workflow with 7 nodes:
  - Schedule Trigger
  - HTTP Request (API)
  - Code Transformation
  - Filter Validation
  - PostgreSQL Storage
  - Google Sheets Export
  - Email Notification
- ✅ Added documentation sticky notes
- ✅ Configured node connections and data flow

**Deliverables**:
- `startup_data_workflow.json` - Importable n8n workflow
- Workflow screenshot showing architecture
- Node configuration details

**Outcome**: Demonstrated ability to design automated data pipelines and understand workflow orchestration.

---

### Phase 3: API Integration ⏳ PLANNED

**Timeline**: Week 3-4
**Status**: ⏳ Future Implementation

**Activities**:
- Set up Crunchbase API credentials
- Configure PredictLeads API access
- Implement OpenCorporates integration
- Add authentication and rate limiting
- Test API endpoints and data quality
- Handle pagination and error scenarios

**Deliverables**:
- Live API connections
- Automated data fetching (daily/weekly schedule)
- Error handling and retry logic

**Outcome**: Transition from manual to automated data collection.

---

### Phase 4: Database & Storage ⏳ PLANNED

**Timeline**: Week 4-5
**Status**: ⏳ Future Implementation

**Activities**:
- Set up PostgreSQL database
- Create tables and indexes
- Migrate CSV data to database
- Configure Google Sheets API
- Implement data synchronization
- Set up backup strategies

**Deliverables**:
- PostgreSQL database with startup data
- Google Sheets integration for team access
- Automated backups

**Outcome**: Scalable, persistent data storage with team collaboration features.

---

### Phase 5: Monitoring & Optimization ⏳ PLANNED

**Timeline**: Week 5-6
**Status**: ⏳ Future Implementation

**Activities**:
- Configure email notifications
- Implement logging and error tracking
- Add data quality metrics
- Optimize workflow performance
- Create dashboard for monitoring
- Set up alerting for failures

**Deliverables**:
- Email notification system
- Monitoring dashboard
- Performance metrics

**Outcome**: Reliable, observable system with proactive error detection.

---

## Scalability Considerations

### Current Capacity
- **Records**: 40 startups
- **Storage**: CSV file (~50 KB)
- **Processing**: Manual updates
- **Limitations**: Single user, no automation

### Designed Capacity
- **Records**: 10,000+ startups
- **Storage**: PostgreSQL database (scalable to millions)
- **Processing**: Automated daily collection
- **Concurrent Users**: Multiple team members via Google Sheets

### Scaling Strategies

#### Horizontal Scaling
- **Multiple API Sources**: Add AngelList, LinkedIn, Twitter APIs
- **Geographic Expansion**: Support multiple countries
- **Industry Verticals**: Expand beyond current industries

#### Vertical Scaling
- **Database Optimization**:
  - Proper indexing for fast queries
  - Partitioning by date or industry
  - Regular VACUUM and ANALYZE operations
- **Workflow Optimization**:
  - Parallel processing for multiple APIs
  - Batch operations for database inserts
  - Caching for frequently accessed data

#### Performance Targets
- **Data Collection**: 1000 startups/hour
- **Database Queries**: < 100ms average response time
- **API Rate Limits**: Respect provider limits (implement backoff)
- **Workflow Execution**: < 5 minutes for full cycle

---

## Security & Privacy

### Data Protection

**Current State**:
- CSV stored locally (not sensitive data)
- No PII beyond publicly available information
- Manual access control

**Production Requirements**:

#### Authentication & Authorization
- API keys stored in environment variables (never in code)
- n8n credentials encrypted at rest
- Database access via role-based permissions
- Google Sheets access via OAuth 2.0

#### Data Privacy
- Comply with GDPR / local data protection laws
- Only collect publicly available information
- Provide data deletion mechanism
- Implement data retention policies

#### Security Best Practices
```
✓ Use HTTPS for all API calls
✓ Encrypt database connections (SSL/TLS)
✓ Regular security updates for n8n and dependencies
✓ Audit logs for data access
✓ Rate limiting to prevent abuse
✓ Input validation to prevent injection attacks
```

### Compliance Considerations

**Data Sources**:
- Respect Terms of Service for each platform
- Honor robots.txt for web scraping
- Use official APIs when available
- Implement proper rate limiting

**Data Usage**:
- Business development and outreach purposes
- No selling or redistribution of data
- Clear opt-out mechanism for contacted companies

---

## Future Enhancements

### Phase 6+: Advanced Features

**AI-Powered Enrichment**
- Use Claude/GPT API to generate:
  - Personalized outreach emails
  - WhatsApp message templates
  - Competitive analysis
  - Investment recommendations

**CRM Integration**
- HubSpot connector
- Salesforce integration
- Custom outreach tracking
- Response management

**Analytics Dashboard**
- Industry trends visualization
- Funding stage distribution
- Geographic heat maps
- Time-series analysis

**Machine Learning**
- Lead scoring (predict conversion likelihood)
- Company categorization
- Founder pattern analysis
- Success prediction models

---

## Conclusion

The Startup Data Collection System demonstrates a **phased approach** to building scalable automation:

1. ✅ **Phase 1**: Manual data collection establishes baseline
2. ✅ **Phase 2**: Workflow design shows technical capability
3. ✅ **Phase 3**: Clear roadmap for production implementation

This architecture balances **immediate deliverables** (CSV data, workflow design) with **future scalability** (API integration, database storage, monitoring), demonstrating both **practical execution** and **strategic thinking**.

---


