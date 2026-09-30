# CRM Hygiene & Deduplication — n8n Workflow

**A production-ready workflow for detecting, scoring, and merging duplicate contacts, companies, and deals in HubSpot.**

---

## Workflow Role & Purpose

This 6-workflow suite automates the complete duplicate-management lifecycle:

| Workflow | Purpose | Trigger |
|---|---|---|
| **WF-00: Orchestrator** | Schedules dedup scans (hourly/daily/weekly) | Time-based or manual |
| **WF-10: Hygiene Engine** | Cleans contact/company data (phone, email normalization) | Called by WF-00 |
| **WF-20: Duplicate Detector** | Finds duplicate pairs via blocking-key matching, scores confidence | Called by WF-00 |
| **WF-30: Merge Executor** | Re-validates and executes irreversible HubSpot merges | Called by WF-40 (on approval) |
| **WF-40: Human Approval Gate** | Posts Slack approval cards, routes to merge or reject | Called by WF-20 |
| **WF-99: Error Handler** | Classifies failures (auth, rate limit, merge failure) and alerts on-call | Auto-triggered on any error |

---

## What It Does

### Duplicate Detection
- **Blocking-key matching**: Only compares contacts if they share ≥1 key (Email, Phone+Lastname, Name, Company, etc.)
- **Confidence scoring**: Tiered scores (100=exact email, 90=phone+name, 72=lastname+company, etc.)
- **Safety pre-checks**: Skips archived contacts, flagged do-not-merge, already-merged pairs

### Approval Workflow
- **Auto-merge tier** (score ≥95): Merges instantly, no Slack
- **Review tier** (score 72–94): Posts Slack approval card, waits for click, executes on approval
- **Log-only tier** (score 55–71): Records to audit trail, no Slack
- **Dropped** (score <55): Discarded

### Audit Trail
- Stamps `dedupe_status` field on merged contacts
- Records merge chain in `dedupe_merged_from` for traceability
- Permanent rejection blacklist in `dedupe_ignore_pairs` (prevents re-proposal)

---

## Global Config: dryRun Mode

The **⚙️ Resolve Dedup Config** node in WF-00 controls the execution mode:

### dryRun = `true` (Safe Mode — Default)
```json
"dryRun": true
```

**What happens:**
- ✅ All detection, scoring, and approval logic runs **normally**
- ✅ Slack approval cards **appear as usual**
- ❌ **NO HubSpot API write calls are made**
- ✅ Merge receipts are recorded (audit trail shows what *would* merge)
- ✅ **Safe to test** — contacts remain unchanged

**Use cases:**
- Testing the workflow before going live
- Auditing what duplicates *would* merge without risk
- Training users on the approval process
- Staging environment validation

### dryRun = `false` (Live Mode)
```json
"dryRun": false
```

**What happens:**
- ✅ All detection, scoring, and approval logic runs **normally**
- ✅ Slack approval cards **appear as usual**
- ✅ **Merges are PERMANENT in HubSpot** (irreversible)
- ✅ `dedupe_status='merged'` is stamped on contacts
- ⚠️ **PRODUCTION mode** — use with caution

**Use cases:**
- Live production dedup
- Recurring scheduled cleaning
- Post-import or migration cleanup

**⚠️ WARNING:** HubSpot contact merges **cannot be undone**. Test thoroughly with `dryRun=true` first.

---

## Configuration Details

In the **⚙️ Resolve Dedup Config** node, adjust these fields:

| Field | Default | Meaning |
|---|---|---|
| `dryRun` | `true` | Dry-run mode (no HubSpot writes) |
| `pageSize` | `100` | Records per API call (1–100) |
| `maxPagesPerRun` | `200` | Max pages scanned per execution |
| `maxRecordsForDedup` | `10,000` | Stop scanning after this count |
| `autoMergeThreshold` | `95` | Score ≥ this = auto-merge (no Slack) |
| `reviewThreshold` | `72` | Score ≥ this = Slack approval |
| `logThreshold` | `55` | Score ≥ this = log only |
| `maxMergesPerRun` | `50` | Max merges per execution (safety limit) |

---

## Prerequisites & Limitations

### HubSpot Plan
- ✅ **Starter, Professional, Enterprise**: Full merge API support
- ❌ **Free Plan**: Merge API **blocked by HubSpot**
  - Detection and approval work normally
  - Buyers must manually merge in HubSpot UI (5 sec per pair)
  - OR upgrade to Starter ($50/month)

### Required Scopes
Your n8n HubSpot credential must include:
- `crm.objects.contacts.read`
- `crm.objects.contacts.write` (for merges)
- `crm.objects.companies.read`
- `crm.objects.companies.write`
- `crm.objects.deals.read`
- `crm.objects.deals.write`

### Rate Limits
- HubSpot free/starter: 100 API calls / 10 seconds
- Workflow throttles at 429 errors, retries automatically

---

## Setup After Import

1. **Reconnect credentials**: Re-authenticate HubSpot and Slack tokens (n8n tokens don't transfer)
2. **Toggle workflows Active**: Set top-right switch to **Active** on WF-10, WF-20, WF-30, WF-40
3. **Set Error Workflow**: In Settings of WF-00, WF-10, WF-20, WF-40, set error workflow to **WF-99**
4. **Update Slack channel**: In WF-40 and WF-99, replace `#crm-hygiene-alerts` with your channel
5. **Test with dryRun=true**: Set `dryRun: true` in Global Config, upload test contacts, run WF-00
6. **Verify Slack cards appear**: Approval cards should post (no actual merges)
7. **Switch to dryRun=false**: Once satisfied, flip to live mode

---

## Scoring Examples

| Scenario | Blocking Key | Score | Routing |
|---|---|---|---|
| john.smith@techcorp.com vs johnsmith@techcorp.com | Email | 100 | ✅ Auto-merge |
| Angela Morrison (angela.morrison@harbor.com, +1-305-555-0177) vs Angie Morrison (angie@morrison.co, +1-305-555-0177) | Phone + LastName | 85 | 📢 Slack approval |
| Robert Davis vs Rob Davis, same phone+company | Phone + LastName + Company | 88 | 📢 Slack approval |
| Jennifer Miller vs Jen Miller, same company, no phone | LastName + Company | 72 | 📢 Slack approval |
| John Smith (generic first name, common email) | Name only | 56 | 📋 Log only |

---

## Troubleshooting

| Issue | Cause | Fix |
|---|---|---|
| "Workflow with ID ... does not exist" | Sub-workflow IDs not updated after import | Use "From list" dropdown to re-assign WF-10, WF-20, WF-30, WF-40 IDs |
| "Workflow is not active and cannot be executed" | WF-10/20/30/40 not toggled Active | Toggle Active (top-right switch) on all 4 sub-workflows |
| No Slack cards appear | WF-40 not running OR Slack token invalid | Check WF-20 → WF-40 execution, verify Slack credentials |
| Merges detected but not applied (dryRun=false) | HubSpot free plan OR insufficient API scope | Upgrade to Starter plan OR check credential scopes |
| "429 rate limit" in Slack alerts | Too many API calls in 10 sec window | Lower `pageSize` to 50 or reduce `maxPagesPerRun` |

---

## Licensing & Resale

This workflow is provided for:
- ✅ Personal use
- ✅ Client projects (work-for-hire)
- ✅ Resale via Fiverr, Upwork, Gumroad (as-is or modified)
- ✅ n8n Template Hub publication

No royalties or attribution required.

---

## Support & Customization

- **Modify confidence thresholds**: Edit scoring logic in WF-20 `🔍 Cluster & Score Duplicates` node
- **Add custom Slack fields**: Edit WF-40 `📤 Format Slack Card` node to include additional contact data
- **Extend to other objects**: Clone WF-20/WF-30 logic for companies/deals (templates included)
- **Change Slack channel**: Update `#crm-hygiene-alerts` in WF-40 and WF-99

---

**Version:** 1.0  
**Last Updated:** 2026-09-30  
**Compatible with:** n8n v1.5+, HubSpot CRM Starter+
