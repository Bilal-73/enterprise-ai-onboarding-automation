# Submission Template — Enterprise AI Onboarding Automation

## Candidate Information

**Assessment:** AI Tooling Specialist — Enterprise Onboarding Automation  
**Full Name** Bilal Imran
**Email** acc.bilalimran@gmail.com
**Linkedln** bilalimran73ai
**Submission Date:** 16/03/2026 
**Repository:** [[Your fork URL](https://github.com/Bilal-73/enterprise-ai-onboarding-automation)]

---

## Task 1 — AI-Powered Automation Solution Design

**File:** `submissions/task1_solution.md`

### Summary

Designed a 7-stage AI-driven onboarding automation architecture that covers:
- New hire intake via form/HRIS webhook trigger
- AI-powered document extraction and field validation (GPT-4o)
- Automated employee profile enrichment with role metadata
- Task generation and routing to IT, HR, Facilities, and Managers
- Personalized onboarding plan generation per role and department
- Automated communication drafting (welcome email, manager briefing)
- Milestone tracking at Day 1, 3, 14, 30, and 90

### AI Integration Points

| Stage | AI Application | Value |
|-------|---------------|-------|
| Document Extraction | Extract structured fields from uploaded PDFs | Eliminates manual data entry |
| Input Normalization | Standardize dates, names, formats | Improves downstream data quality |
| Plan Generation | Role-specific onboarding plans | Personalization at scale |
| Email Drafting | Welcome and manager briefing emails | Saves 45+ min per hire |
| Summarization | Manager briefings from raw profile data | Reduces coordination overhead |

### Prompt Engineering Approach

- All prompts enforce JSON output format for automation compatibility
- Role instructions anchor AI behavior before task specification
- Explicit null handling for missing fields prevents hallucination
- Context injection via `{{variable}}` placeholders filled at runtime
- Temperature set to 0.3 for consistent, structured extraction tasks

---

## Task 2 — Implementation Demo & Prototype Scaffold

**File:** `submissions/task2_solution.md`

### What Was Built

A complete n8n workflow scaffold with 8 nodes:
1. Webhook trigger (new hire form submission)
2. Input validation (required field check)
3. AI enrichment (OpenAI GPT-4o — profile enrichment + plan generation)
4. Airtable record creation (structured onboarding record)
5. Welcome email delivery (Gmail)
6. Manager briefing delivery (Gmail)
7. IT task creation (Jira)
8. Milestone follow-up scheduler (Day 1, 3, 14, 30, 90)

### Key Demonstration Points

- End-to-end orchestration from form trigger to task creation and emails
- AI node transforms raw form data into a complete enriched onboarding package
- All tasks, emails, and records are generated from a single AI call
- n8n workflow JSON provided for direct import
- Zapier alternative flow documented for teams not using n8n

### Technology Used

- **Automation:** n8n (primary), Zapier (alternative documented)
- **AI:** OpenAI GPT-4o with JSON response format enforced
- **Storage:** Airtable
- **Communication:** Gmail
- **Task Management:** Jira
- **Testing:** Sample payload included with expected outputs

---

## Design Decisions

**Why n8n over Zapier?**
n8n supports code nodes, parallel execution, and self-hosting — better suited for enterprise environments with data privacy requirements. Zapier alternative is documented for teams preferring no-code.

**Why a single AI call for enrichment?**
Combining extraction, enrichment, and generation into one prompt reduces latency and API cost. The structured JSON response format ensures all downstream nodes receive consistent data.

**Why Airtable for storage?**
Airtable provides a visual record of all onboarding in progress, easy filtering by status/department/start date, and simple integration with both n8n and Zapier. Can be swapped for any database.

**How is this extensible?**
The architecture supports adding HRIS integration, identity provisioning, LMS API, and compliance deadline monitoring without redesigning the core orchestration flow.

---

## Files Included

```
submissions/
  ├── task1_solution.md     ← Workflow design, AI integration, prompts, data flow
  ├── task2_solution.md     ← Prototype scaffold, n8n JSON, test data, env vars
  └── SUBMISSION_TEMPLATE.md ← This file
```
