# SOP 06: Property Management
**Automation level: D: Managed (agent handles reporting and logistics; human drives owner relationship)**
**Timeline: Ongoing monthly service**
**Deliverable: Monthly performance report · pricing optimization recommendations · guest experience management · maintenance coordination**

## I. OVERVIEW

Ongoing STR management service for Dubai property owners. The workflow runs on a monthly cycle with four weekly focuses: performance review + pricing (Week 1), guest experience audit (Week 2), seasonal refresh assessment (Week 3), maintenance queue (Week 4). Report is delivered to the owner on the 5th of each month via WhatsApp. Human handles all strategic decisions; agent handles data gathering, report generation, and routine communications.

---

## II. INPUTS (onboarding: required once at contract start)

| # | Input | Format | Source | Blocker? |
|---|-------|--------|--------|---------|
| 1 | Property details | Address, type, size, furnishing level | Intake form | YES |
| 2 | Platform access | Airbnb / Booking.com / DTCM login credentials | Client | YES |
| 3 | Current property photos | All rooms, current state | Client | YES |
| 4 | Guest expectations + house rules | Text | Client | YES |
| 5 | Owner preferences | Communication frequency, escalation threshold | Intake form | YES |
| 6 | Maintenance contacts | Plumber, electrician, building management | Client | No: build as we go |
| 7 | Pricing history | Last 3 months rates | Client | No |
| 8 | Contract start date | Date | Agreement | YES |
| 9 | Property ID | `LIOR-PM[CODE][YY]` | Agent | Internal |
| 10 | Notion page ID for this property | From Leads DB | Agent | Internal |

---

## III. SETUP (one-time)

```bash
# Install dependencies
brew install pandoc
brew install --cask mactex       # LaTeX PDF engine
brew install wkhtmltopdf          # PDF fallback

# Verify .env keys
grep -E "CALLMEBOT_PHONE|CALLMEBOT_API_KEY|NOTION_TOKEN|NOTION_LEADS_DB_ID|WETRANSFER_API_KEY" /Users/cashville/.env

# Create property folder
PROPERTY_ID="LIOR-PM2601"   # replace per property
mkdir -p .tmp/${PROPERTY_ID}/{reports,maintenance,guest-log}
touch .tmp/${PROPERTY_ID}/maintenance-queue.txt
touch .tmp/${PROPERTY_ID}/log.txt
```

---

## IV. FOLDER STRUCTURE

```bash
PROPERTY_ID="LIOR-PM2601"
mkdir -p .tmp/${PROPERTY_ID}/{reports,maintenance,guest-log,exports}

# Monthly folder
MONTH="2026-06"
mkdir -p .tmp/${PROPERTY_ID}/reports/${MONTH}
```

---

## V. MONTHLY RECURRING WORKFLOW

Runs on the 1st of every month.

---

### WEEK 1: Performance Review + Pricing Optimization

#### Step 1: Pull Performance Data from Notion

```bash
PROPERTY_ID="LIOR-PM2601"
NOTION_TOKEN=$(grep NOTION_TOKEN /Users/cashville/.env | cut -d= -f2)
PAGE_ID="[ID_NOTION: property page in Leads DB]"
MONTH="$(date -u +%Y-%m)"

# Pull current property data from Notion
PROPERTY_DATA=$(curl -s "https://api.notion.com/v1/pages/${PAGE_ID}" \
  -H "Authorization: Bearer ${NOTION_TOKEN}" \
  -H "Notion-Version: 2022-06-28")

echo $PROPERTY_DATA | python3 -c "
import sys, json
data = json.load(sys.stdin)
props = data.get('properties', {})
# Print key fields
for key in ['Occupancy Rate', 'ADR', 'Gross Revenue', 'Guest Rating', 'Nights Booked']:
    val = props.get(key, {})
    print(f'{key}: {val}')
" 2>/dev/null || echo "Notion data pulled: check manually if parsing fails"
```

**Note:** Platform data (Airbnb, Booking.com) must be pulled manually from the host dashboard. Agent generates the report template; human fills in the numbers from platform exports, then agent formats and delivers.

#### Step 2: Generate Monthly Performance Report

```bash
PROPERTY_ID="LIOR-PM2601"
MONTH="$(date -u +%B %Y)"
WORKDIR=".tmp/${PROPERTY_ID}/reports/$(date -u +%Y-%m)"
mkdir -p "${WORKDIR}"

# Fill in variables from platform dashboard data before running
NIGHTS_AVAILABLE="30"
NIGHTS_BOOKED="[from Airbnb/Booking dashboard]"
OCCUPANCY_RATE="[calculated: Nights_Booked / Nights_Available × 100]"
OCCUPANCY_PRIOR="[prior month %]"
GROSS_REVENUE="[from dashboard: AED]"
ADR="[from dashboard: AED]"
REVPAR="[ADR × Occupancy_Rate / 100]"
REVENUE_PRIOR="[prior month AED]"
TOP_CHANNEL="[Airbnb / Booking / Direct]"
RATING_OVERALL="[score/5]"
RATING_CLEAN="[score/5]"
RATING_COMM="[score/5]"
RATING_LOCATION="[score/5]"
RATING_VALUE="[score/5]"
RECENT_REVIEWS="[last 3 review excerpts]"
MAINT_REPORTED="[list this month]"
MAINT_RESOLVED="[list]"
MAINT_PENDING="[list]"
UPCOMING_BOOKINGS="[count, nights, projected AED]"
GAPS="[dates with no booking in next 30 days]"

cat > "${WORKDIR}/${PROPERTY_ID}-report-$(date -u +%Y-%m).md" << EOF
---
title: "Property Report: ${PROPERTY_ID}: ${MONTH}"
geometry: "margin=2cm"
fontsize: 11pt
---

# Monthly Performance Report
**Property:** [Property address]
**Period:** ${MONTH}
**Prepared by:** LIOR Property Management

---

## Occupancy

| Metric | This month | Prior month | Change |
|--------|-----------|-------------|--------|
| Nights available | ${NIGHTS_AVAILABLE} |: |: |
| Nights booked | ${NIGHTS_BOOKED} |: |: |
| Occupancy rate | ${OCCUPANCY_RATE}% | ${OCCUPANCY_PRIOR}% | [+/-] |

## Revenue

| Metric | This month | Prior month |
|--------|-----------|-------------|
| Gross revenue | AED ${GROSS_REVENUE} | AED ${REVENUE_PRIOR} |
| Average daily rate | AED ${ADR} |: |
| RevPAR | AED ${REVPAR} |: |

Top booking channel: ${TOP_CHANNEL}

## Guest Ratings

| Category | Score |
|----------|-------|
| Overall | ${RATING_OVERALL}/5 |
| Cleanliness | ${RATING_CLEAN}/5 |
| Communication | ${RATING_COMM}/5 |
| Location | ${RATING_LOCATION}/5 |
| Value | ${RATING_VALUE}/5 |

Recent reviews:
${RECENT_REVIEWS}

## Maintenance

**Reported this month:** ${MAINT_REPORTED}
**Resolved:** ${MAINT_RESOLVED}
**Pending:** ${MAINT_PENDING}

## Upcoming (next 30 days)

Bookings: ${UPCOMING_BOOKINGS}
Gaps to fill: ${GAPS}

---

*LIOR Property Management: prepared $(date -u +%Y-%m-%d)*

EOF

echo "Report draft saved."
```

#### Step 3: Generate Report PDF

```bash
pandoc "${WORKDIR}/${PROPERTY_ID}-report-$(date -u +%Y-%m).md" \
  -o "${WORKDIR}/${PROPERTY_ID}-report-$(date -u +%Y-%m).pdf" \
  --pdf-engine=xelatex \
  -V geometry:margin=2cm \
  -V fontsize=11pt

# Fallback
# pandoc "${WORKDIR}/..." -o "${WORKDIR}/..." --pdf-engine=wkhtmltopdf

echo "Report PDF: ${WORKDIR}/${PROPERTY_ID}-report-$(date -u +%Y-%m).pdf"
```

#### Step 4: Pricing Optimization Review

Apply these rules to the upcoming 60-day calendar:

```
PRICING RULES:
- Occupancy < 70% for next 30 days  → reduce ADR by 10%
- Occupancy > 90%                   → increase ADR by 15% on next available dates
- Weekends                          → +20% base rate vs weekdays
- Peak season (Dubai Oct–Apr)       → +30–40% vs off-peak
- Last-minute (≤3 days out, vacant) → -15% to fill the gap
```

Agent generates a recommendation table: human approves before any change goes live:

```bash
cat > "${WORKDIR}/pricing-recommendations-$(date -u +%Y-%m).txt" << EOF
PRICING RECOMMENDATIONS: ${PROPERTY_ID}: $(date -u +%B %Y)
Generated: $(date -u +%Y-%m-%d)

Current occupancy next 30d: [X]%
Current ADR: AED [X]

Recommendations (AWAITING HUMAN APPROVAL before implementing):
[Date range 1]: [Action: e.g. "Reduce ADR from AED 450 to AED 405 (−10%): occupancy at 62%"]
[Date range 2]: [Action]
[Weekend uplift]: [Specific dates + proposed rate]
[Gap dates]: [Dates with no booking + suggested last-minute rate]

Status: PENDING APPROVAL
EOF

echo "Pricing recommendations ready for human review."
```

#### Step 5: Update Notion with Report Data

```bash
NOTION_TOKEN=$(grep NOTION_TOKEN /Users/cashville/.env | cut -d= -f2)
PAGE_ID="[ID_NOTION]"

curl -s -X PATCH "https://api.notion.com/v1/pages/${PAGE_ID}" \
  -H "Authorization: Bearer ${NOTION_TOKEN}" \
  -H "Notion-Version: 2022-06-28" \
  -H "Content-Type: application/json" \
  -d "{
    \"properties\": {
      \"Last Report\": {\"date\": {\"start\": \"$(date -u +%Y-%m-%d)\"}},
      \"Occupancy Rate\": {\"number\": [OCCUPANCY_VALUE]},
      \"Gross Revenue\": {\"number\": [REVENUE_VALUE]}
    }
  }"
echo "Notion updated with monthly data."
```

---

### WEEK 2: Guest Experience Audit

#### Step 1: Review New Reviews

Read all new reviews from the prior month. Tag and log recurring themes.

```bash
PROPERTY_ID="LIOR-PM2601"
WORKDIR_GUEST=".tmp/${PROPERTY_ID}/guest-log"

cat >> "${WORKDIR_GUEST}/reviews-$(date -u +%Y-%m).txt" << EOF
=== Reviews: $(date -u +%B %Y) ===
[Review 1: date, platform, score, excerpt]
[Review 2: date, platform, score, excerpt]
[Review 3: date, platform, score, excerpt]

Recurring positive themes: [list]
Recurring negative themes: [list]
Ratings below 4/5: [list: escalate to human]
Maintenance items mentioned by guests: [list: add to maintenance queue]
EOF
```

**Rule: any rating < 4/5 → human reviews and responds personally (agent drafts, human personalizes).**

Auto-response template (human personalizes before posting):
```
Thank you for staying with us, [Name]. [Acknowledge specific positive from review].
[If issue mentioned: We have addressed [specific issue] and appreciate the feedback.]
We hope to welcome you back.
```

---

### WEEK 3: Seasonal Refresh Assessment

Check if seasonal refresh is warranted:

```bash
PROPERTY_ID="LIOR-PM2601"
WORKDIR=".tmp/${PROPERTY_ID}"
CURRENT_MONTH=$(date -u +%m)

cat > "${WORKDIR}/seasonal-check-$(date -u +%Y-%m).txt" << EOF
SEASONAL REFRESH CHECK: $(date -u +%B %Y)

Trigger criteria (check each):
[ ] Property in shoulder/peak transition month (Sep=month 09, Apr=month 04)
[ ] Guest photo score trending down (from review audit)
[ ] Property photos older than 6 months
[ ] Platform dashboard showing listing quality warning

Dubai seasonal calendar:
- September (month 09): prep for peak season: refresh photos, deep clean, small decor updates
- April (month 04): post-peak: assess wear, update for summer market (longer stays, families)

Assessment: [REFRESH NEEDED / NOT NEEDED]
Recommendation if needed: [schedule Virtual Staging update / re-photograph / physical refresh]
Human approval required: YES
EOF
```

---

### WEEK 4: Maintenance Coordination

#### Step 1: Review and Prioritize Queue

```bash
PROPERTY_ID="LIOR-PM2601"
WORKDIR=".tmp/${PROPERTY_ID}"

# View current queue
cat "${WORKDIR}/maintenance-queue.txt"

# Priority system:
# P1 = affects bookings (AC, plumbing, safety, lock) → fix before next check-in
# P2 = cosmetic (scuff marks, minor damage) → batch monthly
# P3 = future planning → batch quarterly
```

#### Step 2: Contact Vendor for P1 Items

```bash
PHONE=$(grep CALLMEBOT_PHONE /Users/cashville/.env | cut -d= -f2)
APIKEY=$(grep CALLMEBOT_API_KEY /Users/cashville/.env | cut -d= -f2)
VENDOR_PHONE="[vendor WhatsApp number]"
VENDOR_NAME="[Vendor name]"
PROPERTY_ADDRESS="[Property address]"
ISSUE="[maintenance type and description]"
NEXT_CHECKIN="[next check-in date]"

MSG="Hi ${VENDOR_NAME}, this is LIOR Property Management on behalf of ${PROPERTY_ADDRESS}. We have a ${ISSUE} that needs attention before ${NEXT_CHECKIN}. Can you confirm availability? Access: [key safe code / building management contact]"

ENCODED=$(python3 -c "import urllib.parse,sys; print(urllib.parse.quote(sys.argv[1]))" "${MSG}")
curl -s "https://api.callmebot.com/whatsapp.php?phone=${VENDOR_PHONE}&text=${ENCODED}&apikey=${APIKEY}"
echo "Vendor contacted."
```

#### Step 3: Log Every Maintenance Action

```bash
PROPERTY_ID="LIOR-PM2601"
WORKDIR=".tmp/${PROPERTY_ID}"

# Log format: append one line per action:
echo "[$(date '+%Y-%m-%d')] P[1/2/3]: [issue description]. Vendor: [name]. Scheduled: [date]. Cost: AED [X]. Resolved: [date or PENDING]" \
  >> "${WORKDIR}/maintenance-queue.txt"

# Example:
# [2026-06-03] P1: AC unit in master not cooling. Vendor: CoolFix Dubai. Scheduled: 2026-06-04. Cost: AED 350. Resolved: 2026-06-04.
```

---

### MONTH-END: Send Report to Owner (5th of month)

#### Step 1: Upload Report to WeTransfer

```bash
PROPERTY_ID="LIOR-PM2601"
REPORT_FILE=".tmp/${PROPERTY_ID}/reports/$(date -u +%Y-%m)/${PROPERTY_ID}-report-$(date -u +%Y-%m).pdf"

WT_KEY=$(grep WETRANSFER_API_KEY /Users/cashville/.env | cut -d= -f2)
FILE_SIZE=$(wc -c < "${REPORT_FILE}")
FILE_NAME="${PROPERTY_ID}-report-$(date -u +%Y-%m).pdf"

TRANSFER=$(curl -s -X POST "https://dev.wetransfer.com/v2/transfers" \
  -H "Content-Type: application/json" -H "x-api-key: ${WT_KEY}" \
  -d "{\"message\":\"${PROPERTY_ID}: Monthly Report $(date -u +%B %Y)\",\"files\":[{\"name\":\"${FILE_NAME}\",\"size\":${FILE_SIZE}}]}")

TRANSFER_ID=$(echo $TRANSFER | python3 -c "import sys,json; print(json.load(sys.stdin)['id'])")
UPLOAD_URL=$(echo $TRANSFER | python3 -c "import sys,json; print(json.load(sys.stdin)['files'][0]['multipart']['url'])")

curl -s -X PUT "${UPLOAD_URL}" \
  -H "Content-Type: application/octet-stream" \
  --data-binary @"${REPORT_FILE}"

DOWNLOAD_URL=$(curl -s -X PUT "https://dev.wetransfer.com/v2/transfers/${TRANSFER_ID}/finalize" \
  -H "x-api-key: ${WT_KEY}" | python3 -c "import sys,json; print(json.load(sys.stdin)['url'])")

echo "DOWNLOAD LINK: ${DOWNLOAD_URL}"
```

#### Step 2: Send WhatsApp to Owner

```bash
PHONE=$(grep CALLMEBOT_PHONE /Users/cashville/.env | cut -d= -f2)
APIKEY=$(grep CALLMEBOT_API_KEY /Users/cashville/.env | cut -d= -f2)
OWNER_NAME="[OWNER NAME]"
PROPERTY_ADDR="[PROPERTY ADDRESS]"
MONTH_LABEL="$(date -u +%B)"
OCCUPANCY="[X]"
NIGHTS="[N]"
REVENUE="[X]"
RATING="[X]"
HIGHLIGHT="[one-line highlight or concern: e.g. occupancy strong / one guest mention of AC: resolved]"
PRICING_NOTE="[brief 2-line pricing recommendation for next month]"

MSG="${OWNER_NAME}: ${PROPERTY_ADDR}: ${MONTH_LABEL} report is ready.

Occupancy: ${OCCUPANCY}% (${NIGHTS} nights booked)
Revenue: AED ${REVENUE} gross
Guest rating: ${RATING}/5

${HIGHLIGHT}

Full report: ${DOWNLOAD_URL}
Pricing note for next month: ${PRICING_NOTE}"

ENCODED=$(python3 -c "import urllib.parse,sys; print(urllib.parse.quote(sys.argv[1]))" "${MSG}")
curl -s "https://api.callmebot.com/whatsapp.php?phone=${PHONE}&text=${ENCODED}&apikey=${APIKEY}"
echo "Report WhatsApp sent to owner."
```

#### Step 3: Update Notion

```bash
NOTION_TOKEN=$(grep NOTION_TOKEN /Users/cashville/.env | cut -d= -f2)
PAGE_ID="[ID_NOTION]"

curl -s -X PATCH "https://api.notion.com/v1/pages/${PAGE_ID}" \
  -H "Authorization: Bearer ${NOTION_TOKEN}" \
  -H "Notion-Version: 2022-06-28" \
  -H "Content-Type: application/json" \
  -d "{\"properties\":{\"Status\":{\"select\":{\"name\":\"Active\"}},\"Last Report\":{\"date\":{\"start\":\"$(date -u +%Y-%m-%d)\"}},\"Report Link\":{\"url\":\"${DOWNLOAD_URL}\"}}}"

echo "Notion updated."
```

#### Step 4: Monthly Log Entry

```bash
PROPERTY_ID="LIOR-PM2601"
echo "[$(date '+%Y-%m-%d')] MONTHLY REPORT DELIVERED: ${PROPERTY_ID} | Month: $(date -u +%Y-%m) | Occupancy: ${OCCUPANCY}% | Revenue: AED ${REVENUE} | Link: ${DOWNLOAD_URL}" \
  >> ".tmp/${PROPERTY_ID}/log.txt"
```

---

## VI. GUEST EXPERIENCE: CHECK-IN MANAGEMENT

Send welcome message to incoming guest 2h before check-in:

```bash
PHONE=$(grep CALLMEBOT_PHONE /Users/cashville/.env | cut -d= -f2)
APIKEY=$(grep CALLMEBOT_API_KEY /Users/cashville/.env | cut -d= -f2)
GUEST_PHONE="[guest WhatsApp from booking]"
GUEST_NAME="[Guest name]"
PROPERTY_ADDRESS="[Property address]"
ACCESS_INSTRUCTIONS="[e.g. key safe code: 1234 / doorman: ask for unit 1201]"
WIFI_NETWORK="[network name]"
WIFI_PASSWORD="[password]"

MSG="Welcome ${GUEST_NAME}: your apartment is ready. Address: ${PROPERTY_ADDRESS}. ${ACCESS_INSTRUCTIONS}. WiFi: ${WIFI_NETWORK} / ${WIFI_PASSWORD}. Any questions, message here. Enjoy your stay."

ENCODED=$(python3 -c "import urllib.parse,sys; print(urllib.parse.quote(sys.argv[1]))" "${MSG}")
curl -s "https://api.callmebot.com/whatsapp.php?phone=${GUEST_PHONE}&text=${ENCODED}&apikey=${APIKEY}"
echo "Guest welcome sent."
```

**Welcome setup checklist (confirm with cleaner before every check-in):**
- [ ] Property fully cleaned
- [ ] Fresh linens and towels
- [ ] Welcome note printed on kitchen counter
- [ ] Coffee, tea, hand soap, toilet paper (3+ rolls per bathroom)
- [ ] All appliances functional
- [ ] A/C pre-set to 22°C
- [ ] All lights working
- [ ] WiFi password visible

---

## VII. QA CHECKLIST

Monthly report:
- [ ] All numerical data sourced from platform dashboard (not estimated)
- [ ] Occupancy rate calculated correctly (booked nights / available nights)
- [ ] RevPAR calculated correctly (ADR × occupancy rate / 100)
- [ ] Maintenance items from guest reviews added to queue
- [ ] Pricing recommendations in the report match the rules above
- [ ] PDF opens correctly
- [ ] Report sent on or before 5th of month

---

## VIII. WHATSAPP TEMPLATE (monthly report to owner)

```
[OWNER NAME]: [PROPERTY ADDRESS]: [Month] report is ready.

Occupancy: [X]% ([N] nights booked)
Revenue: AED [X] gross
Guest rating: [X]/5

[One-line highlight or concern]

Full report: [LINK]
Pricing note for next month: [brief 2-line note]
```

Rules: no "AI", no "luxury", no prices (except actual revenue figures), no "drone".

---

## IX. FALLBACKS

| Tool | Failure | Fix |
|---|---|---|
| WeTransfer API | Upload fails | Manual upload at wetransfer.com |
| WeTransfer API | File too large | Report is typically small (PDF only): unlikely. If so, compress images. |
| WeTransfer | Down | Google Drive: share link with "anyone can view" |
| Pandoc PDF | xelatex font error | `cp [font.ttf] ~/Library/Fonts/ && fc-cache -fv`. Retry. Or use `--pdf-engine=wkhtmltopdf` |
| CallMeBot | Phone not registered | Send "I allow callmebot to send me messages" to +34 644 29 73 73 |
| Notion API | Rate limit | Wait 60s and retry |
| Platform data unavailable | Dashboard inaccessible | Ask human to pull manually from platform, paste into report template |

---

## X. ESCALATION: HUMAN MUST HANDLE

- Guest complaint rated <3/5 or requesting refund
- Maintenance issue costs above AED 2,000 (without owner pre-approval)
- Platform account warning or suspension risk
- Booking dispute
- Owner requests pricing change above ±30%
- Emergency: flood, break-in, guest incident

---

## XI. ANNUAL REVIEW (January)

Generate full-year summary from 12 monthly reports:

```bash
PROPERTY_ID="LIOR-PM2601"
YEAR="2026"

# Compile all monthly report data
cat > ".tmp/${PROPERTY_ID}/reports/${PROPERTY_ID}-annual-summary-${YEAR}.md" << EOF
---
title: "Annual Property Review: ${PROPERTY_ID}: ${YEAR}"
geometry: "margin=2cm"
fontsize: 11pt
---

# Annual Review: ${YEAR}

## Performance Summary

| Month | Occupancy | Revenue (AED) | ADR (AED) | Rating |
|-------|-----------|--------------|-----------|--------|
| January | | | | |
[... fill from monthly reports ...]
| December | | | | |
| **TOTAL / AVG** | | | | |

## Maintenance Cost Summary
Total maintenance spend: AED [X]
Key repairs: [list]

## Recommendations for [YEAR+1]
- Pricing strategy: [notes]
- Refresh budget: [AED estimate]
- Platform changes: [if any]

EOF

pandoc ".tmp/${PROPERTY_ID}/reports/${PROPERTY_ID}-annual-summary-${YEAR}.md" \
  -o ".tmp/${PROPERTY_ID}/reports/${PROPERTY_ID}-annual-summary-${YEAR}.pdf" \
  --pdf-engine=xelatex

echo "Annual summary generated."
```

---

## XII. LOG FORMAT

```
[YYYY-MM-DD HH:MM] REPORT STARTED: [PROPERTY_ID] | Month: [YYYY-MM]
[YYYY-MM-DD HH:MM] PRICING REVIEW: recommendations ready: awaiting human approval
[YYYY-MM-DD HH:MM] GUEST AUDIT: [N] reviews reviewed | [N] below 4/5: escalated
[YYYY-MM-DD HH:MM] MAINTENANCE: [N] items reviewed | [N] P1 contacted | [N] P2/P3 batched
[YYYY-MM-DD HH:MM] REPORT DELIVERED: [PROPERTY_ID] | Month: [YYYY-MM] | Link: [URL] | WA: sent
[YYYY-MM-DD HH:MM] ESCALATION: [reason] → human notified
[YYYY-MM-DD HH:MM] MAINTENANCE LOG: [date] [priority] [issue] [vendor] [cost] [resolved]
```

---

## XIII. OFFBOARDING

When contract ends:

```bash
PROPERTY_ID="LIOR-PM2601"

# 1. Generate final performance report covering full contract period
# (compile all monthly reports into annual format)

# 2. Log offboarding
echo "[$(date '+%Y-%m-%d')] OFFBOARDING: ${PROPERTY_ID} | Contract end | Final report generated | Credentials returned to owner | Maintenance log archived" \
  >> ".tmp/${PROPERTY_ID}/log.txt"

# 3. Archive to permanent storage
cp -r ".tmp/${PROPERTY_ID}/" "05-clients/lior/property-management/${PROPERTY_ID}/"
echo "Archived to permanent storage."
```

Checklist:
- [ ] Final performance report delivered to owner
- [ ] All platform login credentials returned to owner
- [ ] Maintenance contact log handed over
- [ ] Final property inspection walkthrough completed
- [ ] All reports archived to `05-clients/lior/property-management/`
