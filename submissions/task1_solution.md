# Task 1 Solution — AI-Powered Onboarding Automation Design

## Overview

This document presents the complete architecture design for an AI-driven enterprise onboarding automation system. The solution covers workflow design, AI integration points, prompt engineering, and data flow across all onboarding stages.

---

## 1. Workflow Architecture

### Stage 1 — Intake & Trigger

**What happens:**
A new hire record is created when HR finalizes an offer. This triggers the automation via one of:
- A Google Form / Typeform submission by HR
- A webhook from the HRIS (e.g., Workday, BambooHR) on new employee creation
- Manual entry into an Airtable or Google Sheets intake table

**Data collected at intake:**
- Full name, personal email, phone
- Job title, department, team
- Office location / remote status
- Employment type (full-time, contractor, part-time)
- Start date
- Hiring manager name and email
- Uploaded documents (ID, signed offer letter, policy acknowledgements)

---

### Stage 2 — AI Extraction & Validation

**What happens:**
The automation sends uploaded documents and form data to an LLM API. The AI:
1. Extracts structured fields from unstructured documents
2. Validates completeness (flags missing docs or fields)
3. Normalizes inconsistent inputs (e.g., date formats, name casing)
4. Returns a clean JSON onboarding record

**AI is used here because:**
- Documents vary in format (PDFs, images, scanned forms)
- Manual review is slow and error-prone
- Field extraction at scale requires consistent logic

**Output:** A validated, structured employee profile JSON record stored in Airtable or Google Sheets.

---

### Stage 3 — Employee Profile Enrichment

**What happens:**
The workflow merges the extracted data with role metadata from an internal lookup table or HRIS:
- Required systems access (based on department/role)
- Mandatory compliance training modules
- Hardware requirements
- Office location specifics (building access, parking, badge)
- Manager contact information

**Output:** An enriched onboarding profile that drives all downstream automation steps.

---

### Stage 4 — Task Generation & Routing

**What happens:**
Based on the enriched profile, the workflow automatically creates tasks in the project management tool (Jira, Asana, ClickUp, or Trello):

| Team | Example Tasks |
|------|--------------|
| IT | Provision laptop, create company email, grant system access |
| HR | Send offer confirmation, schedule orientation, assign compliance training |
| Hiring Manager | Intro call scheduling, 30/60/90 day plan, team introductions |
| Facilities | Badge creation, desk assignment, parking pass |
| Security | NDA confirmation, background check status, security training |

Each task is tagged with the employee name, start date, and priority level. Slack or email notifications are sent to responsible teams.

---

### Stage 5 — Personalized Onboarding Plan Generation

**What happens:**
An AI step generates a role-specific onboarding plan using the enriched profile. The plan includes:
- Personalized welcome message
- First-day and first-week schedule
- Key contacts list (manager, HR, IT, buddy)
- Required training with deadlines
- Recommended resources (wikis, tool guides, org charts)
- 30/60/90 day success milestones

**AI is used here because:**
- Every role and department has different onboarding needs
- Writing personalized plans at scale is impractical manually
- AI can tailor tone and content based on seniority, location, and team

---

### Stage 6 — Communication Drafting & Delivery

**What happens:**
AI drafts and the automation sends:
- **Welcome email** to the new hire (personalized, warm, role-specific)
- **Manager briefing note** (new hire summary, their tasks, suggested talking points for day 1)
- **IT provisioning request** (structured, with all required access details)
- **HR completion checklist** (what's done, what's pending)

All emails are reviewed by the automation for completeness before sending.

---

### Stage 7 — Milestone Tracking & Feedback

**What happens:**
The workflow sets automated reminders and check-ins:
- Day 1: Welcome email auto-delivered
- Day 3: Check-in prompt sent to new hire and manager
- Day 14: Mid-onboarding feedback survey triggered
- Day 30: First milestone review reminder
- Day 90: Full onboarding completion survey

Responses are stored and summarized by AI for HR review dashboards.

---

## 2. AI Integration Points — Detailed

### 2.1 Document Understanding Prompt

**Use case:** Extract structured fields from uploaded onboarding documents.

**Prompt:**
```
You are an onboarding operations assistant for an enterprise HR team.

Your task is to extract structured information from the provided employee intake documents.

Extract the following fields:
- full_name
- personal_email
- company_email (if available)
- job_title
- department
- office_location
- employment_type (full-time / part-time / contractor)
- start_date (ISO 8601 format)
- manager_name
- manager_email
- required_systems (list of tools/systems mentioned)
- missing_documents (list any required documents not present)
- flags (any issues requiring manual HR review)

Return your response as valid JSON only. Do not include any explanation or preamble.
If a field cannot be determined from the document, set its value to null.
```

---

### 2.2 Onboarding Plan Generation Prompt

**Use case:** Generate a personalized first-week onboarding plan.

**Prompt:**
```
You are an onboarding specialist creating a personalized onboarding plan for a new employee.

Employee details:
- Name: {{full_name}}
- Role: {{job_title}}
- Department: {{department}}
- Location: {{office_location}}
- Employment type: {{employment_type}}
- Start date: {{start_date}}
- Manager: {{manager_name}}

Generate a structured onboarding plan in JSON format with the following sections:
1. welcome_message (2-3 sentences, warm and personalized to role)
2. day_1_schedule (array of time-blocked activities)
3. week_1_priorities (array of 5 key focus areas)
4. required_training (array of modules with deadlines)
5. key_contacts (array of name, role, email, purpose)
6. recommended_resources (array of links and descriptions)
7. success_milestones (30, 60, and 90 day goals for this specific role)

Be specific to the role and department. Avoid generic advice.
Return valid JSON only.
```

---

### 2.3 Welcome Email Draft Prompt

**Use case:** Draft a personalized welcome email for the new hire.

**Prompt:**
```
You are writing a welcome email on behalf of the HR team for a new employee joining the company.

New hire details:
- Name: {{full_name}}
- Role: {{job_title}}
- Department: {{department}}
- Start date: {{start_date}}
- Manager: {{manager_name}}

Write a warm, professional welcome email that:
- Addresses the employee by first name
- Expresses genuine excitement about them joining
- Briefly mentions their role and team
- Lists 3 practical things they should know before day 1 (what to bring, where to go, who to ask)
- Ends with a clear call to action (confirm start date, any documents to complete)
- Is signed from "The HR Team"

Keep the tone friendly and concise. Maximum 200 words.
```

---

### 2.4 Manager Briefing Prompt

**Use case:** Generate a handoff summary for the hiring manager.

**Prompt:**
```
You are preparing a manager briefing for a new hire starting soon.

New hire profile:
{{employee_profile_json}}

Write a concise manager briefing that includes:
1. New hire summary (name, role, start date, background highlights)
2. Action items for the manager before day 1 (3-5 items)
3. Suggested agenda for the first 1:1 meeting
4. Onboarding checklist status (what is complete, what is pending)
5. Key dates and deadlines

Keep it structured, scannable, and under 300 words.
```

---

## 3. Data Flow Diagram

```
[HRIS / Google Form Submission]
            │
            ▼
[Automation Trigger — n8n / Zapier]
            │
            ▼
[AI Node — Document Extraction & Validation]
            │
            ▼
[Structured Employee Record → Airtable / Google Sheets]
            │
     ┌──────┴──────┐
     ▼             ▼
[Task Router]   [AI Onboarding Plan Generator]
     │               │
  ┌──┴──┐            ▼
  │     │     [Welcome Email Draft]
  ▼     ▼     [Manager Briefing Draft]
[IT   [HR]         │
Tasks] Tasks]      ▼
  │          [Email / Slack Delivery]
  └──────┬──────┘
         ▼
[Milestone Tracker — Day 1, 3, 14, 30, 90]
         │
         ▼
[Feedback Collection & HR Dashboard Update]
```

---

## 4. Integration Stack

| Layer | Tools |
|-------|-------|
| Automation Orchestration | n8n (preferred), Zapier |
| AI / LLM | OpenAI GPT-4o, Claude API |
| Intake | Google Forms, Typeform, HRIS webhook |
| Data Storage | Airtable, Google Sheets |
| Communication | Gmail, Slack |
| Task Management | Asana, Jira, ClickUp |
| Scheduling | Google Calendar |
| Documentation | Notion, Confluence |
| Identity / Access | Okta, Azure AD (via IT ticketing) |

---

## 5. Prompt Design Principles

1. **Role instruction first** — Every prompt starts with a clear role ("You are an onboarding operations assistant") to anchor the AI's behavior.
2. **Structured output mandate** — All prompts explicitly require JSON output to ensure downstream automation compatibility.
3. **Field-level specificity** — Prompts list exact field names to extract or generate, reducing hallucination and ambiguity.
4. **Null handling** — Prompts instruct the model to return `null` for missing fields rather than guessing, preventing data quality issues.
5. **Scope bounding** — Word and section limits prevent over-generation and keep outputs usable in automated pipelines.
6. **Context injection** — Dynamic employee data is injected via `{{variable}}` placeholders filled by the automation tool before the API call.

---

## 6. Business Impact Summary

| Metric | Before Automation | After Automation |
|--------|------------------|-----------------|
| Average onboarding setup time | 3–5 business days | Same day / next day |
| Manual tasks per new hire | 20–40 steps | 3–5 human touch points |
| Document processing errors | Common | Near zero (AI validated) |
| Onboarding plan consistency | Varies by manager | Standardized, role-specific |
| HR administrative hours per hire | 8–12 hours | < 2 hours |

---

*Submitted as Task 1 of the AI Tooling Specialist Assessment.*
