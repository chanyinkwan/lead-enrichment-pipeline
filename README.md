# AI Lead Enrichment Pipeline

> Automated lead scraping, enrichment, and AI-powered qualification scoring -- turning raw LinkedIn profiles into prioritised outreach lists.

## Client

**Spark Growth Agency** (anonymised) -- A B2B growth marketing agency helping SaaS startups build outbound sales pipelines.

| Detail | Value |
|--------|-------|
| Industry | B2B Marketing Agency |
| Size | 15 staff (6 SDRs, 4 marketers, 5 ops) |
| Location | London, UK |
| Monthly lead target | 500 qualified leads for clients |

## The Problem

Spark's SDRs spent 60% of their time on manual lead research instead of actually selling. Their workflow:

1. Search LinkedIn for target personas (manually)
2. Copy profile data into Google Sheets (manually)
3. Find email addresses using Hunter.io (one by one)
4. Validate emails (one by one)
5. Score leads based on gut feeling
6. Write personalised outreach (from scratch each time)

**Pain:** Each SDR processed ~20 leads/day manually. Target was 500/month across 6 SDRs. They were hitting 350-400 and burning out. Lead quality was inconsistent -- "gut feeling" scoring meant some SDRs chased unqualified leads for weeks.

## The Solution

An n8n pipeline that takes a target persona description via form input, scrapes matching LinkedIn profiles, enriches them with verified emails, scores them using AI, and outputs a prioritised Google Sheet ready for outreach.

### How It Works

```
[Form Input: Target Persona]
  "VP of Engineering at Series B SaaS, 50-200 employees, London"
       |
       v
[LinkedIn Profile Scraping]
  |-- Name, title, company, headline
  |-- Company size, industry
       |
       v
[Google Sheets: Raw Leads]
       |
       v
[Email Enrichment: Apollo.io API]
  |-- Find business email
  |-- Validate deliverability
       |
       v
[AI Lead Scoring: OpenAI]
  |-- Score 1-10 based on:
  |   - Title match
  |   - Company fit
  |   - Engagement signals
  |-- Generate personalised opener
       |
       v
[Google Sheets: Scored & Enriched]
  |-- Sorted by score (highest first)
  |-- Ready for outreach
```

### Architecture

| Component | Technology |
|-----------|-----------|
| Orchestration | n8n (self-hosted) |
| Data Source | LinkedIn profile scraping |
| Email Enrichment | Apollo.io API |
| Email Validation | Apollo.io built-in verification |
| AI Scoring | OpenAI GPT-4o |
| Data Store | Google Sheets |
| Trigger | n8n Form (web-based input) |

## Key Design Decisions

- **Why AI scoring over rule-based?** Rules miss nuance. A "Head of Digital" at a 50-person startup is more valuable than a "VP of IT" at a 10,000-person enterprise -- but they both match "senior tech leader." The AI considers context, not just keywords.
- **Why Google Sheets output?** SDRs already live in Sheets. Building a custom CRM would add friction. The workflow enhances their existing process instead of replacing it.
- **Why form-based trigger?** Different clients need different personas. A form lets SDRs define targets without touching the workflow configuration.
- **Why email validation?** Bounce rates above 5% damage sender reputation. Validating before outreach protects the agency's domain health.

## Results

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| Leads processed per SDR/day | 20 | 60-80 (depends on batch quality) | 3-4x throughput |
| Monthly qualified leads | 350-400 | 550-650 (fluctuates by month) | ~50-60% increase |
| SDR time on research | 60% | ~20% (some manual verification still needed) | ~65-70% reduction |
| Lead-to-meeting conversion | 3.2% | 5.1-5.8% (varies by client vertical) | Noticeable improvement |
| Email bounce rate | 12% | 2.5-4% (depends on data freshness) | Significant reduction |

> Results measured across 3 client accounts over a 6-month period. SaaS-focused campaigns performed better than general tech.

## Setup

### Prerequisites
- Docker & Docker Compose
- OpenAI API key
- Apollo.io API key (for email enrichment)
- Google Sheets API credentials

### Quick Start
```bash
docker compose up -d
# Import workflow/workflow.json into n8n
# Set Apollo.io API credentials
# Set OpenAI API credentials
# Connect Google Sheets
# Access the form at the n8n form URL
```

### Environment Variables
| Variable | Description |
|----------|-------------|
| `OPENAI_API_KEY` | OpenAI API key |
| `APOLLO_API_KEY` | Apollo.io API key for email enrichment |
| `GOOGLE_SHEETS_CREDENTIALS` | Google Sheets service account JSON |

## Example Run

### Input: Form Submission

```
Target persona: "VP of Engineering at Series B SaaS companies, 50-200 employees, London"
```

### Raw Scrape (10 leads)

| Name | Title | Company | Company_Size | Location |
|------|-------|---------|-------------|----------|
| Sarah Chen | VP of Engineering | DataFlow Systems | 120 | London |
| James Morrison | Technical Consultant | Accenture | 500,000 | London |
| Priya Patel | Head of Platform Engineering | CloudScale | 85 | London |
| ... | ... | ... | ... | ... |

Full sample: [`sample-data/raw_leads_input.csv`](sample-data/raw_leads_input.csv)

### After Enrichment + AI Scoring

**Lead 1: Sarah Chen** -- VP of Engineering at DataFlow Systems (Series B, 120 employees)
- **Score: 9/10** | Email: `sarah.chen@dataflowsystems.com` (verified)
- **AI Reasoning:** Perfect ICP match -- VP-level engineering leader at a Series B SaaS company. Active hiring signals suggest growth phase and strong buying intent.
- **Opener:** "Hi Sarah -- saw DataFlow is scaling the engineering team. When teams grow past 100, infrastructure decisions get harder to reverse..."

**Lead 2: James Morrison** -- Technical Consultant at Accenture
- **Score: 3/10** | Email: `james.morrison@accenture.com` (verified)
- **AI Reasoning:** Consultant at a large enterprise -- not a buyer. Advisory role means no purchasing authority. Company size (500K) well outside target range.
- **Opener:** N/A -- below score threshold

**Lead 3: Priya Patel** -- Head of Platform Engineering at CloudScale (Series B, 85 employees)
- **Score: 8/10** | Email: not found
- **AI Reasoning:** Strong fit -- senior engineering leadership at Series B SaaS within size range. "Head of Platform" indicates infrastructure decision-maker. Recommend LinkedIn InMail outreach.
- **Opener:** "Hi Priya -- your move from AWS to building CloudScale's platform is impressive. We work with Series B teams navigating that exact transition..."

Full sample: [`sample-data/enriched_leads_output.csv`](sample-data/enriched_leads_output.csv)

### Slack Notification (hot leads)

When a lead scores 8+, the pipeline sends a Slack alert:

```
:fire: Hot Lead Alert

Sarah Chen -- VP of Engineering at DataFlow Systems
Score: 9/10 | Email verified

AI Reasoning: Perfect ICP match. VP-level engineering leader at a
Series B SaaS company with 120 employees. Active hiring signals
suggest growth phase -- strong buying intent for developer tooling.

Personalised Opener:
"Hi Sarah -- saw DataFlow is scaling the engineering team. When teams
grow past 100, infrastructure decisions get harder to reverse. Worth
a quick chat about how we've helped similar Series B teams avoid
common scaling pitfalls?"

-> View in Google Sheet: [link]
```

## Challenges & Iteration

- **LinkedIn rate limiting killed V1.** Scraping more than ~100 profiles/hour triggered temporary blocks. Fix: added request throttling (random 3-8 second delays), rotating user agents, and daily batch limits of 200 profiles. Switched from real-time scraping to overnight batch processing.
- **Apollo.io email accuracy was ~70%, not the 95%+ they advertise.** Many returned emails bounced. Fix: added a secondary verification step using an SMTP validation check. Combined accuracy: ~89%. Still not perfect but acceptable.
- **AI lead scoring was overconfident initially.** It scored 80% of leads as 7+/10. Not useful. Fix: recalibrated the scoring prompt with specific negative signals ("consultant" in title = likely not a buyer, company <10 employees = too small for our client's product). After recalibration, score distribution became more useful: ~15% scored 8+, ~40% scored 5-7, ~45% scored below 5.
- **Google Sheets hit the 10-million cell limit** on one client's master sheet after 4 months. Fix: monthly archival workflow that moves processed leads to a separate archive sheet. Active sheet stays under 50K rows.

## Constraints & Trade-offs

- **Why Google Sheets over a CRM:** Spark's clients are early-stage startups. Most don't have a CRM yet. Sheets is free, familiar, and sharable. Once a client graduates to HubSpot, the enriched sheet is imported as a one-time migration.
- **Why overnight batch vs real-time:** LinkedIn rate limiting makes real-time impossible at scale. Overnight batch means SDRs get a fresh enriched list every morning. They initially wanted "instant results" but adapted within a week.
- **Why Apollo over Hunter.io:** Tested both on 500 leads. Apollo found 68% of emails, Hunter found 52%. Apollo also returned job titles and company data, saving a second lookup. Price was comparable.
- **Why form-based input:** Considered a Slack command but SDRs wanted to paste target criteria while reviewing LinkedIn. An n8n form embedded in their internal wiki was the simplest option.

## Edge Cases & Error Handling

| Scenario | Handling | Outcome |
|----------|----------|---------|
| LinkedIn profile is private/limited | Captures name and headline only, marks as "partial" in sheet | SDR decides whether to pursue manually |
| Apollo returns no email | Row marked "no email found", company domain captured for manual outreach | SDR can still use LinkedIn InMail |
| Duplicate lead (already in sheet) | Deduplication check on LinkedIn URL before insert | Skipped with log entry |
| Company no longer exists | Apollo returns stale data, AI scoring catches "company dissolved" signals | Scored 0, auto-filtered from active list |
| Rate limit hit (LinkedIn or Apollo) | Workflow pauses for 30 minutes, retries remaining batch | Continues from where it stopped, no data loss |
| Form submitted with vague criteria ("tech people in London") | AI scoring runs but results are too broad | SDR refines criteria and re-runs |

## Monitoring & Maintenance

- **Daily:** Morning Slack message: "Your enriched leads are ready. X new leads, Y scored 8+, Z emails verified."
- **Weekly:** Bounce rate tracking. If verified emails start bouncing above 3%, flag Apollo data quality issue.
- **Monthly:** Review AI scoring calibration. Compare scores against actual meeting-booked rate. Recalibrate prompt if correlation weakens.
- **Quarterly:** Client retention check. Are the leads converting? If not, the problem isn't the pipeline -- it's the targeting criteria.

## Customisation

- **Add Slack notifications:** Alert SDRs when a lead scores 8+ ("hot lead")
- **Add CRM sync:** Push enriched leads to HubSpot, Salesforce, or Pipedrive
- **Add outreach drafting:** Use OpenAI to generate personalised email drafts per lead
- **Add scheduling:** Run the pipeline daily at 6 AM with fresh search results

## Tech Stack

`n8n` `OpenAI GPT-4o` `Apollo.io` `LinkedIn Scraping` `Google Sheets` `Lead Scoring` `Docker`

## Lessons Learned

1. AI scoring improved conversion rates because it caught "looks good on paper but wrong fit" leads that humans missed under time pressure
2. The biggest ROI wasn't the automation -- it was that SDRs could now spend 85% of their time on actual conversations instead of copy-pasting data
3. Form-based input was key to adoption: SDRs felt in control of what they were searching for

---

*Built by [Kessog Chan](https://linkedin.com/in/kessogchan) -- AI Solutions | Workflow Automation | Presales & Growth*
