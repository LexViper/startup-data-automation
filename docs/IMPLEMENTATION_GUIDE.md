# Implementation Guide

## Startup Data Collection System - n8n Workflow

This guide will help you understand and use the startup data collection system components.

---

## 📋 Table of Contents

1. [Prerequisites](#prerequisites)
2. [File Structure](#file-structure)
3. [Setting Up n8n](#setting-up-n8n)
4. [Importing the Workflow](#importing-the-workflow)
5. [Understanding the Data](#understanding-the-data)
6. [Running the Workflow](#running-the-workflow)
7. [Customization Options](#customization-options)
8. [Troubleshooting](#troubleshooting)

---

## Prerequisites

### Required Software
- **n8n** - Workflow automation tool
  - Installation: `npm install n8n -g` or use Docker
  - Version: 0.200.0 or higher
- **Node.js** (if using npm installation)
  - Version: 14.x or higher
- **Docker** (optional, for containerized deployment)

### Optional Requirements
- **PostgreSQL** (for database storage in production)
- **Google Account** (for Google Sheets integration)
- **SMTP Server** (for email notifications)

---

## File Structure

```
startup-data-collection/
│
├── README.md                           # Main documentation
├── IMPLEMENTATION_GUIDE.md             # This file
│
├── data/
│   └── startup_data_assignment_ready.csv  # 40 Indian startup records
│
├── workflows/
│   └── startup_data_workflow.json      # n8n workflow configuration
│
├── architecture/
│   ├── architecture_diagram.png        # System architecture visual
│   └── architecture_explanation.md     # Detailed architecture docs
│
└── screenshots/
    └── n8n_workflow_screenshot.png     # Workflow implementation view
```

---

## Setting Up n8n

### Option 1: Using npm (Local Installation)

```bash
# Install n8n globally
npm install n8n -g

# Start n8n
n8n start

# Access at: http://localhost:5678
```

### Option 2: Using Docker (Recommended)

```bash
# Pull and run n8n container
docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -v ~/.n8n:/home/node/.n8n \
  n8nio/n8n

# Access at: http://localhost:5678
```

### Option 3: Using npx (No Installation Required)

```bash
# Run n8n without installing
npx n8n

# Access at: http://localhost:5678
```

---

## Importing the Workflow

### Step 1: Access n8n Interface

1. Open your browser
2. Navigate to `http://localhost:5678`
3. Create an account (first-time setup)

### Step 2: Import Workflow

1. Click on **"Workflows"** in the left sidebar
2. Click the **"+"** button to create new workflow
3. Click the **three-dot menu** (⋮) in the top right corner
4. Select **"Import from File"**
5. Choose `workflows/startup_data_workflow.json`
6. Click **"Import"**

### Step 3: Verify Import

You should see the workflow with these nodes:
- Schedule Trigger
- Fetch Startup Data
- Transform Data
- Filter Valid Startups
- Save to Database
- Export to Google Sheets
- Send Success Email

---

## Understanding the Data

### CSV Data Structure

The `startup_data_assignment_ready.csv` file contains **40 Indian startups** with the following columns:

| Column | Description | Example |
|--------|-------------|---------|
| company_name | Startup name | Razorpay |
| website | Company website URL | https://razorpay.com |
| founder_name | Founder's name | Harshil Mathur |
| contact_email | Contact email address | contact@razorpay.com |
| industry | Primary industry | FinTech |
| sub_sector | Specific sub-sector | Payment Gateway |
| product_description | What the startup does | Complete payment solutions... |
| target_customers | Who they serve | SMEs and startups... |
| location | City, State | Bangalore, Karnataka |
| startup_stage | Funding stage | Series F |
| source | Data source | Crunchbase |

### Data Coverage

- **Industries**: FinTech (14), EdTech (3), E-commerce (10), Mobility (3), HealthTech (5), Social Media (2), Services (1), Travel (2)
- **Locations**: Bangalore, Mumbai, Gurugram, Noida, and other major Indian cities
- **Stages**: Seed to Public companies
- **Total Records**: 40 validated startups

---

## Running the Workflow

### Current State (Design Mode)

The workflow is currently in **design mode** and shows the intended architecture for automated data collection.

**Important**: The workflow has red warning icons (❌) on some nodes because:
- No API credentials configured
- No database connection established
- No email SMTP settings added

**This is expected and correct for the assignment demonstration.**

### To Make It Functional (Optional)

If you want to actually execute the workflow, you'll need to:

#### 1. Configure API Credentials

**For Crunchbase API:**
1. Sign up at [Crunchbase API](https://data.crunchbase.com/docs)
2. Get your API key
3. In n8n, go to **Credentials** → Add **HTTP Header Auth**
4. Add header: `X-cb-user-key: YOUR_API_KEY`

#### 2. Set Up Database Connection

**For PostgreSQL:**
1. Install PostgreSQL locally or use cloud service
2. Create database: `CREATE DATABASE startups;`
3. Create table:
```sql
CREATE TABLE startups (
    id SERIAL PRIMARY KEY,
    company_name VARCHAR(255),
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
    collected_at TIMESTAMP DEFAULT NOW()
);
```
4. In n8n, add PostgreSQL credentials

#### 3. Configure Google Sheets

1. Go to [Google Cloud Console](https://console.cloud.google.com/)
2. Create a new project
3. Enable Google Sheets API
4. Create OAuth 2.0 credentials
5. In n8n, add Google Sheets OAuth2 credentials
6. Create a spreadsheet and get its ID

#### 4. Set Up Email Notifications

1. Use your SMTP server or Gmail
2. In n8n, add SMTP credentials:
   - Host: smtp.gmail.com (for Gmail)
   - Port: 587
   - Username: your-email@gmail.com
   - Password: App-specific password (for Gmail)

---

## Customization Options

### Modifying the Schedule

The workflow is set to run daily at 9 AM. To change:

1. Click on **"Schedule Trigger"** node
2. Modify the cron expression:
   - Every hour: `0 * * * *`
   - Every day at 6 PM: `0 18 * * *`
   - Every Monday at 9 AM: `0 9 * * 1`

### Adjusting Data Filters

In the **"Filter Valid Startups"** node, you can add conditions:

```javascript
// Example: Filter only FinTech companies
{{ $json.industry === "FinTech" }}

// Example: Filter only Series A and beyond
{{ ["Series A", "Series B", "Series C"].includes($json.startup_stage) }}

// Example: Filter only Bangalore-based startups
{{ $json.location.includes("Bangalore") }}
```

### Modifying Data Transformation

In the **"Transform Data"** node (Code node), you can customize the JavaScript:

```javascript
// Example: Add additional fields
transformedData.push({
    ...startup,
    data_quality_score: calculateQualityScore(startup),
    priority: determinePriority(startup),
    last_updated: new Date().toISOString()
});
```

---

## Troubleshooting

### Issue: Workflow won't import

**Solution:**
- Ensure you have n8n version 0.200.0 or higher
- Check if the JSON file is not corrupted
- Try creating a blank workflow and importing again

### Issue: Nodes show error icons

**Solution:**
- This is expected in design mode
- Add credentials to remove errors
- Or ignore if just demonstrating architecture

### Issue: Can't access n8n interface

**Solution:**
```bash
# Check if n8n is running
ps aux | grep n8n

# Check port 5678 availability
lsof -i :5678

# Restart n8n
n8n start
```

### Issue: Docker container not starting

**Solution:**
```bash
# Check Docker is running
docker ps

# Remove existing container
docker rm -f n8n

# Start fresh
docker run -it --rm --name n8n -p 5678:5678 n8nio/n8n
```

### Issue: CSV data not loading

**Solution:**
- Ensure CSV file is in UTF-8 encoding
- Check for special characters in data
- Verify column headers match exactly

---

## Working with the CSV Data

### Loading Data into Spreadsheet

```bash
# Import into Google Sheets (manual)
1. Open Google Sheets
2. File → Import → Upload
3. Select startup_data_assignment_ready.csv

# Import into Excel
1. Open Excel
2. Data → From Text/CSV
3. Select the CSV file
```

### Querying Data with Python

```python
import pandas as pd

# Load the data
df = pd.read_csv('data/startup_data_assignment_ready.csv')

# Filter FinTech companies
fintech = df[df['industry'] == 'FinTech']

# Get Bangalore startups
bangalore = df[df['location'].str.contains('Bangalore')]

# Count by industry
industry_counts = df['industry'].value_counts()
```

### Querying Data with SQL (if imported to database)

```sql
-- Get all FinTech startups
SELECT * FROM startups WHERE industry = 'FinTech';

-- Count startups by city
SELECT 
    SPLIT_PART(location, ',', 1) as city,
    COUNT(*) as startup_count
FROM startups
GROUP BY city
ORDER BY startup_count DESC;

-- Get startups by funding stage
SELECT startup_stage, COUNT(*) as count
FROM startups
GROUP BY startup_stage
ORDER BY count DESC;
```

---

## Next Steps

### For Assignment Submission

1. ✅ Use the CSV data as-is
2. ✅ Include workflow JSON file
3. ✅ Add architecture diagram you create
4. ✅ Reference this implementation guide
5. ✅ Include workflow screenshot

### For Production Deployment

1. ⏳ Set up API credentials
2. ⏳ Configure database (PostgreSQL recommended)
3. ⏳ Add error handling and logging
4. ⏳ Implement data deduplication
5. ⏳ Set up monitoring and alerts
6. ⏳ Create backup strategies
7. ⏳ Add authentication and security

### For Enhancement

1. 🚀 Add more data sources (PredictLeads, AngelList)
2. 🚀 Implement AI-powered data enrichment
3. 🚀 Build a web dashboard for data viewing
4. 🚀 Create automated outreach emails
5. 🚀 Add CRM integration (HubSpot, Salesforce)
6. 🚀 Implement real-time data updates

---

## Useful Resources

### n8n Documentation
- Official Docs: https://docs.n8n.io/
- Node Reference: https://docs.n8n.io/integrations/builtin/
- Community Forum: https://community.n8n.io/

### API Documentation
- Crunchbase API: https://data.crunchbase.com/docs
- PredictLeads API: https://predictleads.com/api-documentation/
- OpenCorporates API: https://api.opencorporates.com/

### Additional Tools
- n8n Desktop App: https://n8n.io/desktop
- n8n Cloud: https://n8n.io/cloud
- PostgreSQL Docs: https://www.postgresql.org/docs/

---

## Support

For questions or issues:

1. **Check the documentation** first (README.md and architecture_explanation.md)
2. **Review workflow screenshot** to understand node configuration
3. **Consult n8n community** for workflow-specific questions
4. **Verify CSV data structure** matches expected format

---

## Version History

- **v1.0** - Initial implementation with 40 Indian startup records
- Architecture designed for API integration
- n8n workflow created with data processing pipeline
- CSV-based data collection completed

---

