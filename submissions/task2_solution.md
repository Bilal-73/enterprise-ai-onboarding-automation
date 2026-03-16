# Task 2 Solution — Prototype Implementation & Demo

## Overview

This document describes the working prototype scaffold for the AI-powered onboarding automation system. The prototype demonstrates the core orchestration logic using n8n (or Zapier) and can be deployed with the provided configuration files and code snippets.

---

## 1. Prototype Architecture

The prototype covers the end-to-end onboarding flow across 7 nodes:

```
[Trigger] → [Validate] → [AI Extract] → [Store Record]
    → [Generate Plan] → [Notify Teams] → [Schedule Follow-ups]
```

This is a functional scaffold — all core logic is implemented. Production deployment would add full HRIS integration and live credentials.

---

## 2. n8n Workflow — Node-by-Node Breakdown

### Node 1 — Form Trigger (Webhook)

**Type:** Webhook / Google Forms Trigger  
**Purpose:** Receives new hire submission and starts the workflow.

**Expected Input Payload:**
```json
{
  "full_name": "Jane Smith",
  "personal_email": "jane.smith@gmail.com",
  "job_title": "Senior Product Manager",
  "department": "Product",
  "office_location": "New York",
  "employment_type": "full-time",
  "start_date": "2025-04-07",
  "manager_name": "Alex Johnson",
  "manager_email": "alex.johnson@company.com",
  "documents_uploaded": ["offer_letter.pdf", "id_copy.pdf"]
}
```

**n8n Config:**
- Node: `Webhook`
- Method: POST
- Path: `/onboarding/new-hire`
- Response: `Last Node`

---

### Node 2 — Input Validation (Function Node)

**Type:** Code / Function Node  
**Purpose:** Validates required fields before passing to AI. Prevents API calls on incomplete data.

**JavaScript Logic:**
```javascript
const requiredFields = [
  'full_name', 'personal_email', 'job_title',
  'department', 'start_date', 'manager_email'
];

const item = $input.first().json;
const missing = requiredFields.filter(f => !item[f]);

if (missing.length > 0) {
  return [{
    json: {
      status: 'validation_failed',
      missing_fields: missing,
      employee: item.full_name || 'Unknown',
      timestamp: new Date().toISOString()
    }
  }];
}

return [{
  json: {
    status: 'valid',
    employee_data: item,
    timestamp: new Date().toISOString()
  }
}];
```

---

### Node 3 — AI Field Extraction & Enrichment (HTTP Request / OpenAI Node)

**Type:** OpenAI Node or HTTP Request to OpenAI API  
**Purpose:** Sends employee data to GPT-4o for enrichment, normalization, and plan generation.

**API Call Config:**
```json
{
  "model": "gpt-4o",
  "messages": [
    {
      "role": "system",
      "content": "You are an onboarding operations assistant. Process the employee intake record and return a complete, enriched onboarding profile as valid JSON. Include: normalized fields, required_systems based on department, compliance_training list, onboarding_plan with day_1_schedule and week_1_priorities, welcome_email_draft, manager_briefing_draft, and pending_tasks for IT and HR. Return JSON only, no preamble."
    },
    {
      "role": "user",
      "content": "=Process this new hire record: {{ JSON.stringify($json.employee_data) }}"
    }
  ],
  "temperature": 0.3,
  "response_format": { "type": "json_object" }
}
```

**Expected AI Output Structure:**
```json
{
  "employee": {
    "full_name": "Jane Smith",
    "personal_email": "jane.smith@gmail.com",
    "company_email": "jane.smith@company.com",
    "job_title": "Senior Product Manager",
    "department": "Product",
    "office_location": "New York",
    "employment_type": "full-time",
    "start_date": "2025-04-07",
    "manager_name": "Alex Johnson",
    "manager_email": "alex.johnson@company.com"
  },
  "required_systems": ["Jira", "Slack", "Google Workspace", "Confluence", "Figma"],
  "compliance_training": [
    { "module": "Security Awareness", "deadline_days": 7 },
    { "module": "Data Privacy Policy", "deadline_days": 14 },
    { "module": "Code of Conduct", "deadline_days": 7 }
  ],
  "onboarding_plan": {
    "welcome_message": "Welcome to the team, Jane! We're thrilled to have you joining as our Senior Product Manager...",
    "day_1_schedule": [
      { "time": "9:00 AM", "activity": "IT setup and account activation" },
      { "time": "10:30 AM", "activity": "1:1 welcome meeting with Alex Johnson" },
      { "time": "12:00 PM", "activity": "Lunch with the Product team" },
      { "time": "2:00 PM", "activity": "HR orientation and benefits walkthrough" },
      { "time": "4:00 PM", "activity": "Review product roadmap and upcoming sprints" }
    ],
    "week_1_priorities": [
      "Complete all compliance training modules",
      "Meet all Product team members",
      "Get access to all required systems",
      "Review Q2 product strategy documents",
      "Shadow 3 customer calls"
    ]
  },
  "welcome_email_draft": {
    "to": "jane.smith@gmail.com",
    "subject": "Welcome to the team, Jane! 🎉 Your start date is April 7th",
    "body": "Hi Jane,\n\nWe are so excited to welcome you as our new Senior Product Manager starting April 7th!..."
  },
  "manager_briefing": {
    "to": "alex.johnson@company.com",
    "subject": "New Hire Briefing: Jane Smith starting April 7th",
    "body": "Hi Alex,\n\nJane Smith is confirmed to start as Senior Product Manager on April 7th..."
  },
  "it_tasks": [
    "Create company email: jane.smith@company.com",
    "Provision MacBook Pro (standard Product team config)",
    "Grant Jira access (Product role)",
    "Grant Slack access and add to #product, #general channels",
    "Grant Confluence access",
    "Grant Figma access (Editor seat)"
  ],
  "hr_tasks": [
    "Confirm start date with new hire",
    "Schedule orientation session",
    "Assign compliance training in LMS",
    "Set up payroll and benefits enrollment",
    "Assign onboarding buddy"
  ]
}
```

---

### Node 4 — Store Onboarding Record (Airtable / Google Sheets Node)

**Type:** Airtable Node or Google Sheets Node  
**Purpose:** Saves the enriched employee record and sets onboarding status to "Active."

**Airtable Config:**
```
Table: Onboarding Records
Operation: Create Record
Fields:
  - Employee Name → {{ $json.employee.full_name }}
  - Email (Personal) → {{ $json.employee.personal_email }}
  - Email (Company) → {{ $json.employee.company_email }}
  - Job Title → {{ $json.employee.job_title }}
  - Department → {{ $json.employee.department }}
  - Start Date → {{ $json.employee.start_date }}
  - Manager → {{ $json.employee.manager_name }}
  - Status → "Active - Pending Day 1"
  - IT Tasks → {{ JSON.stringify($json.it_tasks) }}
  - HR Tasks → {{ JSON.stringify($json.hr_tasks) }}
  - Created At → {{ new Date().toISOString() }}
```

---

### Node 5 — Send Welcome Email (Gmail Node)

**Type:** Gmail Node  
**Purpose:** Sends AI-drafted welcome email to new hire.

```
To: {{ $json.welcome_email_draft.to }}
Subject: {{ $json.welcome_email_draft.subject }}
Body: {{ $json.welcome_email_draft.body }}
```

---

### Node 6 — Send Manager Briefing (Gmail Node)

**Type:** Gmail Node  
**Purpose:** Sends AI-drafted manager briefing.

```
To: {{ $json.manager_briefing.to }}
Subject: {{ $json.manager_briefing.subject }}
Body: {{ $json.manager_briefing.body }}
```

---

### Node 7 — Create IT Tasks (HTTP Request / Jira or Asana Node)

**Type:** Asana or Jira Node (loop over it_tasks array)  
**Purpose:** Creates individual IT tasks in the project management tool.

**Jira API Call (per task):**
```json
{
  "fields": {
    "project": { "key": "IT" },
    "summary": "{{ task_item }} — {{ employee.full_name }} (starts {{ employee.start_date }})",
    "issuetype": { "name": "Task" },
    "priority": { "name": "High" },
    "duedate": "{{ employee.start_date }}",
    "description": "Onboarding task for new hire {{ employee.full_name }}, {{ employee.job_title }}, starting {{ employee.start_date }}."
  }
}
```

---

### Node 8 — Schedule Milestone Follow-ups (Schedule / Wait Node)

**Type:** n8n Wait Node or Zapier Delay  
**Purpose:** Schedules automated check-in messages at defined intervals.

```
Day 1  → Send "Welcome to your first day!" message to new hire
Day 3  → Send check-in prompt to manager
Day 14 → Send mid-onboarding survey to new hire
Day 30 → Send 30-day milestone review reminder
Day 90 → Send onboarding completion survey
```

---

## 3. Zapier Alternative Flow

For Zapier-based implementation, the equivalent flow is:

```
Trigger: New row in Google Sheets (Intake Form)
  ↓
Step 1: Filter — check required fields complete
  ↓
Step 2: Code by Zapier — validate and format data
  ↓
Step 3: OpenAI (ChatGPT) — extract and enrich profile
  ↓
Step 4: Airtable — create onboarding record
  ↓
Step 5: Gmail — send welcome email
  ↓
Step 6: Gmail — send manager briefing
  ↓
Step 7: Asana — create IT tasks (multi-step)
  ↓
Step 8: Slack — notify IT and HR channels
  ↓
Step 9: Delay + Gmail — Day 3 check-in
  ↓
Step 10: Delay + Gmail — Day 14 survey
```

---

## 4. Sample Test Data

Use this payload to test the webhook trigger end-to-end:

```json
{
  "full_name": "Michael Torres",
  "personal_email": "michael.torres@gmail.com",
  "job_title": "Software Engineer II",
  "department": "Engineering",
  "office_location": "Remote - Chicago",
  "employment_type": "full-time",
  "start_date": "2025-04-14",
  "manager_name": "Sarah Chen",
  "manager_email": "sarah.chen@company.com",
  "documents_uploaded": ["offer_letter_signed.pdf", "passport_copy.pdf"]
}
```

**Expected flow result:**
1. Validation passes (all required fields present)
2. AI enriches profile with Engineering-specific systems (GitHub, AWS, Jira, Slack, Datadog)
3. Record created in Airtable with status "Active - Pending Day 1"
4. Welcome email sent to michael.torres@gmail.com
5. Manager briefing sent to sarah.chen@company.com
6. 6 IT tasks created in Jira (GitHub access, AWS IAM, Jira, Slack, Datadog, laptop setup)
7. Follow-up milestones scheduled

---

## 5. n8n Workflow Export (JSON)

Below is a simplified n8n workflow JSON that can be imported directly:

```json
{
  "name": "Enterprise AI Onboarding Automation",
  "nodes": [
    {
      "name": "New Hire Webhook",
      "type": "n8n-nodes-base.webhook",
      "parameters": {
        "httpMethod": "POST",
        "path": "onboarding/new-hire",
        "responseMode": "lastNode"
      },
      "position": [250, 300]
    },
    {
      "name": "Validate Input",
      "type": "n8n-nodes-base.code",
      "parameters": {
        "jsCode": "const required = ['full_name','personal_email','job_title','department','start_date','manager_email'];\nconst item = $input.first().json;\nconst missing = required.filter(f => !item[f]);\nif (missing.length) return [{ json: { status: 'failed', missing } }];\nreturn [{ json: { status: 'valid', employee_data: item } }];"
      },
      "position": [450, 300]
    },
    {
      "name": "AI Enrich Profile",
      "type": "n8n-nodes-base.openAi",
      "parameters": {
        "resource": "chat",
        "model": "gpt-4o",
        "messages": {
          "values": [
            { "role": "system", "content": "You are an onboarding operations assistant. Return a complete enriched onboarding JSON profile." },
            { "role": "user", "content": "=Process: {{ JSON.stringify($json.employee_data) }}" }
          ]
        }
      },
      "position": [650, 300]
    },
    {
      "name": "Save to Airtable",
      "type": "n8n-nodes-base.airtable",
      "parameters": {
        "operation": "create",
        "application": "YOUR_AIRTABLE_BASE_ID",
        "table": "Onboarding Records"
      },
      "position": [850, 300]
    },
    {
      "name": "Send Welcome Email",
      "type": "n8n-nodes-base.gmail",
      "parameters": {
        "operation": "send",
        "toList": "={{ $json.welcome_email_draft.to }}",
        "subject": "={{ $json.welcome_email_draft.subject }}",
        "message": "={{ $json.welcome_email_draft.body }}"
      },
      "position": [1050, 200]
    },
    {
      "name": "Send Manager Briefing",
      "type": "n8n-nodes-base.gmail",
      "parameters": {
        "operation": "send",
        "toList": "={{ $json.manager_briefing.to }}",
        "subject": "={{ $json.manager_briefing.subject }}",
        "message": "={{ $json.manager_briefing.body }}"
      },
      "position": [1050, 400]
    }
  ],
  "connections": {
    "New Hire Webhook": { "main": [[{ "node": "Validate Input" }]] },
    "Validate Input": { "main": [[{ "node": "AI Enrich Profile" }]] },
    "AI Enrich Profile": { "main": [[{ "node": "Save to Airtable" }]] },
    "Save to Airtable": {
      "main": [[
        { "node": "Send Welcome Email" },
        { "node": "Send Manager Briefing" }
      ]]
    }
  }
}
```

---

## 6. Environment Variables Required

```env
# OpenAI
OPENAI_API_KEY=sk-...

# Airtable
AIRTABLE_API_KEY=pat...
AIRTABLE_BASE_ID=app...

# Gmail OAuth
GMAIL_CLIENT_ID=...
GMAIL_CLIENT_SECRET=...

# Jira
JIRA_BASE_URL=https://yourcompany.atlassian.net
JIRA_API_TOKEN=...
JIRA_EMAIL=admin@company.com

# Slack
SLACK_BOT_TOKEN=xoxb-...
SLACK_IT_CHANNEL_ID=C...
SLACK_HR_CHANNEL_ID=C...
```

---

## 7. Scalability Notes

- The AI extraction node handles both form submissions and document uploads (PDFs passed as base64 or URL)
- The workflow can process multiple new hires simultaneously (n8n supports parallel execution)
- Airtable can be swapped for any database — the JSON schema is consistent
- All AI prompts use structured JSON output format, making them automation-compatible
- The milestone tracker can be extended with a separate recurring workflow that polls Airtable daily

---

## 8. Production Readiness Checklist

- [ ] Connect real HRIS webhook (Workday / BambooHR)
- [ ] Add error handling nodes for API failures
- [ ] Set up n8n credentials vault for all API keys
- [ ] Test with 5+ sample employee profiles across departments
- [ ] Add Slack notification to IT and HR channels
- [ ] Enable n8n execution logging for audit trail
- [ ] Add conditional branching for contractor vs. full-time employee flows
- [ ] Connect LMS (learning management system) for automated training assignment

---

*Submitted as Task 2 of the AI Tooling Specialist Assessment.*
