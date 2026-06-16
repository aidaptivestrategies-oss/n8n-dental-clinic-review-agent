# Dental Clinic Review Request Agent

An n8n workflow that automates patient review collection for dental clinics. When an appointment is marked complete, the agent waits, generates a personalized review request email via the Claude API, routes responses by sentiment, and sends a 72-hour follow-up to non-responders — all while enforcing opt-out compliance.

---

## How It Works

```
Appointment Complete Trigger (Webhook)
  └── Validate & Structure Data
        └── Lookup Opt-Out Sheet
              └── Dedup & Opt-Out Check
                    ├── OPTED OUT → (stop)
                    └── PROCEED
                          └── Wait 4 Hours Post-Appointment
                                └── Claude: Generate Review Email
                                      └── Parse & Inject Review Link
                                            └── Sentiment Router
                                                  ├── NEGATIVE → Alert Clinic Manager
                                                  └── POSITIVE/NEUTRAL
                                                        └── Send Review Request Email
                                                              └── Log Send to Opt-Out Sheet
                                                                    └── Wait 72 Hours for Response
                                                                          └── Review Completion Check
                                                                                ├── REVIEWED → (stop)
                                                                                └── NO RESPONSE → Send Follow-Up (Day 3)
```

1. **Webhook trigger** fires when a patient appointment is marked complete in your practice management system.
2. **Validate & Structure Data** normalizes the incoming payload (patient name, phone, email, appointment type).
3. **Opt-out check** looks up the patient in a Google Sheet blocklist — opted-out patients exit immediately.
4. **4-hour wait** gives patients time to settle before receiving any outreach.
5. **Claude API** generates a personalized review request email based on the patient's name and appointment type.
6. **Sentiment Router** splits the flow: negative-sentiment cases alert the clinic manager internally; positive/neutral cases send the review request to the patient.
7. **Opt-out logging** records the send in Google Sheets so the patient isn't contacted again.
8. **72-hour wait** then checks whether the patient has submitted a review.
9. **Follow-up email** goes out on Day 3 if no review was detected.

---

## Tools Connected

| Tool | Purpose |
|---|---|
| n8n | Workflow orchestration and scheduling |
| Claude API (Anthropic) | Personalized review request and follow-up email generation |
| Webhook | Receives appointment-complete events from your practice system |
| Email (SMTP / Gmail) | Sends review requests, follow-ups, and negative-sentiment alerts |
| Google Sheets | Opt-out list lookup and send logging |

---

## Required n8n Credentials

Set these up in **Settings → Credentials** before importing the workflow:

| Credential Name | Type | Used By |
|---|---|---|
| Anthropic account | Anthropic API | Claude: Generate Review Email |
| Gmail account (or SMTP) | Gmail OAuth2 / SMTP | Send Review Request Email, Send Follow-Up, Alert Clinic Manager |
| Google Sheets account | Google Sheets OAuth2 | Lookup Opt-Out Sheet, Log Send to Opt-Out Sheet |

---

## Configuration Placeholders

After importing, search for and replace each `{{PLACEHOLDER}}` with your real values.

| Placeholder | Description | Example |
|---|---|---|
| `{{CLINIC_NAME}}` | Your dental clinic's name | `Smile Dental Centre` |
| `{{REVIEW_LINK}}` | Your Google review URL | `https://g.page/r/your-clinic/review` |
| `{{MANAGER_EMAIL}}` | Email address for negative-sentiment alerts | `manager@smiledentalcentre.com` |
| `{{OPT_OUT_SHEET_ID}}` | Google Sheets document ID for the opt-out list | `1aBcD...` |
| `{{OPT_OUT_TAB_NAME}}` | Tab name within the opt-out sheet | `Opt-Outs` |

---

## Setup Instructions

1. In n8n, go to **Workflows → Import from file** and select `dental-clinic-review-agent.json`.
2. Configure the three credentials (Anthropic API, Gmail/SMTP, Google Sheets OAuth2).
3. Replace all `{{PLACEHOLDER}}` values as described above.
4. In your practice management system (or CRM), configure a webhook to POST to the n8n webhook URL when an appointment is marked complete. The payload should include at minimum:
   - `patientName` — full name
   - `patientEmail` — email address
   - `appointmentType` — type of visit (e.g. `cleaning`, `consultation`)
5. Test with a sample webhook payload (use n8n's built-in test webhook) before activating.
6. Activate the workflow and monitor the first few live executions.
