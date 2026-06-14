# SOP: Studio Partner (Agency Retainer)
**Automation level: B: Auto + monthly human review**
**Timeline: Ongoing monthly retainer**
**Deliverable: 5 Full Listing Packs/month · priority 48h turnaround · 1 cinematic tour/month · co-branded visuals · 10% referral revenue tracking · monthly performance report**

## I. OVERVIEW

For real estate agencies with active listing inventory. LIOR acts as the agency's in-house visual studio: always on, always on-brand, always 48h. Agent manages the production pipeline and ledger; human manages the agency relationship and approves the monthly report. Each listing pack runs SOP-02 at priority 48h turnaround. One cinematic tour is included per month: the agency designates which listing.

---

## II. ONBOARDING INPUTS

| # | Input | Format | Source | Blocker? |
|---|-------|--------|--------|---------|
| 1 | Agreement signed | PDF | Agent | YES |
| 2 | Agency name + logo + brand colors | PNG/SVG + hex | Agency | YES |
| 3 | Primary contact name + WhatsApp | Text | Agency | YES |
| 4 | Billing contact + method | Text + payment | Agency | YES |
| 5 | Co-branding preference | "LIOR + Agency" or "White-label" | Agency | YES |
| 6 | Default style if no brief provided | Warm / Minimal / Standard | Agreement | YES |
| 7 | Contract start date | Date | Agreement | YES |
| 8 | Agency code | 3-letter code for project IDs | Agent | Internal |

---

## III. SETUP (one-time)

```bash
# Install dependencies (same as SOP-02)
brew install ffmpeg imagemagick pandoc
brew install --cask mactex
brew install wkhtmltopdf

# Verify .env keys
grep -E "IMGBB_API_KEY|VSTAGING_API_KEY|RUNWAY_API_KEY|ADOBE_CLIENT_ID|ADOBE_CLIENT_SECRET|WETRANSFER_API_KEY|CALLMEBOT_PHONE|CALLMEBOT_API_KEY|NOTION_TOKEN" /Users/cashville/.env

# Create agency folder
AGENCY_CODE="GGP"   # replace per agency (3-letter code)
mkdir -p .tmp/studio-partner-${AGENCY_CODE}/{partner-profile,ledger,reports,referrals}
touch .tmp/studio-partner-${AGENCY_CODE}/ledger/partner-ledger-$(date -u +%Y-%m).txt
touch .tmp/studio-partner-${AGENCY_CODE}/referrals/referral-log.txt
touch .tmp/studio-partner-${AGENCY_CODE}/log.txt
```

---

## IV. FOLDER STRUCTURE

```bash
AGENCY_CODE="GGP"
mkdir -p .tmp/studio-partner-${AGENCY_CODE}/{partner-profile,ledger,reports,referrals}
# Per listing: follows SOP-02 folder structure under a different project ID prefix
# Project IDs: [AGENCY_CODE]-[YYYYMM]-[SEQ] e.g. GGP-202606-01
```

---

## V. STEP 1: ONBOARDING: CREATE PARTNER PROFILE

```bash
AGENCY_CODE="GGP"
WORKDIR=".tmp/studio-partner-${AGENCY_CODE}"

cat > "${WORKDIR}/partner-profile/partner-profile.txt" << 'EOF'
PARTNER PROFILE: [AGENCY NAME]
─────────────────────────────────────────────────────────
Agency: [name]
Agency code: [3-letter code]
Primary contact: [name] / [WhatsApp number]
Billing contact: [name] / [email]
Brand: [logo filename] / [primary color hex] / [font if specified]
Co-branding: [LIOR + Agency / White-label]
Style default: [agreed default style if no brief: Contemporary Warm / Modern Minimal / Standard]
Retainer tier: Standard (5 listing packs/month)
Cinematic tour: 1 per month (agency designates which listing)
Rollover terms: Unused listings roll to next month: max 1 month carry
Contract start: [date]
Contract renewal: [date]
─────────────────────────────────────────────────────────
EOF
echo "Partner profile saved."
```

### Setup Simplified Brief Template

Send to agency contact:

```bash
PHONE=$(grep CALLMEBOT_PHONE /Users/cashville/.env | cut -d= -f2)
APIKEY=$(grep CALLMEBOT_API_KEY /Users/cashville/.env | cut -d= -f2)
CONTACT_NAME="[CONTACT NAME]"

MSG="${CONTACT_NAME}: welcome to Studio Partner. To submit listings, use this format in each message:

LIOR LISTING BRIEF
Agency: [Agency name]
Listing ref: [your internal ref]
Property: [address]
Rooms to stage: [N]
Style: [warm / minimal / standard] or leave blank for default
Special instructions: [any notes]
Turnaround: [48h standard / 24h urgent]
Photos attached: yes/no
Video attached: yes/no

We will confirm receipt and deliver within 48h. Any questions, message here."

ENCODED=$(python3 -c "import urllib.parse,sys; print(urllib.parse.quote(sys.argv[1]))" "${MSG}")
curl -s "https://api.callmebot.com/whatsapp.php?phone=${PHONE}&text=${ENCODED}&apikey=${APIKEY}"
echo "Brief template sent to agency."
```

---

## VI. MONTHLY LISTING FLOW

### When a Listing Brief Arrives

```
LISTING RECEIVED
     │
     ├─ Validate inputs (photos ≥ 1920×1080, video if cinematic tour)
     │    Missing item → request via WhatsApp → wait 24h → begin on receipt
     │
     ├─ Assign Project ID: [AGENCY_CODE]-[YYYYMM]-[SEQ]
     │    Example: GGP-202606-01
     │
     ├─ Execute SOP-02 (Full Listing Pack): 48h priority turnaround
     │    Staging: up to 8 rooms
     │    Cinematic tour: YES for designated tour listing / NO for others
     │
     └─ Deliver → update partner ledger → WhatsApp confirmation
```

**SOP-02 reference: use this for every listing pack:**
- Virtual Staging AI: 8 rooms staged
- Runway Gen-3: animated renders for cinematic tour
- FFmpeg: social cuts (30s reel 9:16, 15s teaser 9:16, 30s landscape)
- WeTransfer: delivery
- WhatsApp: notify agency contact on delivery

**48h clock starts when all inputs are received (not when brief is sent).**

### Validate Inputs

```bash
AGENCY_CODE="GGP"
LISTING_REF="GGP-202606-01"
WORKDIR=".tmp/studio-partner-${AGENCY_CODE}"

# Check photos minimum quality
echo "Photo validation for ${LISTING_REF}:"
for PHOTO in .tmp/${LISTING_REF}-listing-pack/01-client-inputs/photos/*.jpg; do
  DIMS=$(magick identify -format "%wx%h" "${PHOTO}" 2>/dev/null)
  WIDTH=$(echo $DIMS | cut -dx -f1)
  if [ "${WIDTH}" -lt "1920" ] 2>/dev/null; then
    echo "  FAIL: $(basename ${PHOTO}): ${DIMS} (min 1920px wide)"
  else
    echo "  OK:   $(basename ${PHOTO}): ${DIMS}"
  fi
done
```

If photo quality fails → request reshoot:

```bash
PHONE=$(grep CALLMEBOT_PHONE /Users/cashville/.env | cut -d= -f2)
APIKEY=$(grep CALLMEBOT_API_KEY /Users/cashville/.env | cut -d= -f2)
CONTACT_NAME="[CONTACT NAME]"
LISTING_REF="[listing ref]"
ROOM="[room name]"
ISSUE="too dark / blurry / angle insufficient"

MSG="${CONTACT_NAME}: for ${LISTING_REF}, the photo for ${ROOM} is ${ISSUE}. Could you reshoot from the opposite corner of the room, in daylight, phone horizontal at chest height? We will start immediately on receipt."

ENCODED=$(python3 -c "import urllib.parse,sys; print(urllib.parse.quote(sys.argv[1]))" "${MSG}")
curl -s "https://api.callmebot.com/whatsapp.php?phone=${PHONE}&text=${ENCODED}&apikey=${APIKEY}"
```

### Update Partner Ledger

Update after every delivery:

```bash
AGENCY_CODE="GGP"
MONTH="$(date -u +%B %Y)"
WORKDIR=".tmp/studio-partner-${AGENCY_CODE}"

# Ledger file for this month
LEDGER="${WORKDIR}/ledger/partner-ledger-$(date -u +%Y-%m).txt"

# Create or append
cat >> "${LEDGER}" << EOF
| $(date -u +%Y-%m-%d) | GGP-202606-01 | [address] | 8 | NO | Delivered | [download URL] |
EOF

# View current ledger status
cat "${LEDGER}"
echo ""
DELIVERED=$(grep -c "Delivered" "${LEDGER}" 2>/dev/null || echo 0)
echo "Listings delivered this month: ${DELIVERED} / 5"
REMAINING=$((5 - DELIVERED))
echo "Listings remaining: ${REMAINING}"
```

---

## VII. CO-BRANDING

### Option A: Co-branded (LIOR + Agency)

Add watermark to all delivered still images:

```bash
AGENCY_CODE="GGP"
AGENCY_NAME="[Agency Name]"
YEAR="$(date -u +%Y)"
PROJECT_ID="GGP-202606-01"
EXPORT_DIR=".tmp/${PROJECT_ID}-listing-pack/08-exports"

for IMG in "${EXPORT_DIR}/staged-rooms/"*.jpg; do
  magick "${IMG}" \
    -font Inter-Light \
    -pointsize 18 \
    -fill "rgba(255,255,255,0.3)" \
    -gravity SouthEast \
    -annotate +20+20 "LIOR × ${AGENCY_NAME} · ${YEAR}" \
    "${IMG}"
done
echo "Watermark applied to all staged images."
```

Add credit to cinematic tour (last 3 seconds text overlay: handled in FFmpeg final export in SOP-02):
```
Text: "Visual direction: LIOR × [Agency Name]"
Font: Cormorant Garamond Light, centered, 2s fade in → fade out with music
```

### Option B: White-label (Agency brand only)

Remove all LIOR references from deliverables. If agency requests their logo, add it instead:

```bash
# No LIOR watermark: deliver files as-is
# If agency logo provided, add theirs instead:
AGENCY_LOGO="${WORKDIR}/partner-profile/agency-logo.png"
for IMG in "${EXPORT_DIR}/staged-rooms/"*.jpg; do
  magick "${IMG}" \
    \( "${AGENCY_LOGO}" -resize "200x50>" \) \
    -gravity SouthEast \
    -geometry +20+20 \
    -composite \
    "${IMG}"
done
```

---

## VIII. MONTHLY CINEMATIC TOUR

One cinematic tour is included per month. The agency designates which listing gets it.

**If agency has not designated by the 25th of the prior month: agent sends:**

```bash
PHONE=$(grep CALLMEBOT_PHONE /Users/cashville/.env | cut -d= -f2)
APIKEY=$(grep CALLMEBOT_API_KEY /Users/cashville/.env | cut -d= -f2)
CONTACT_NAME="[CONTACT NAME]"
NEXT_MONTH="[Month name]"
DESIGNATION_DEADLINE="$(date -v+5d -u +%d %B)"

MSG="${CONTACT_NAME}: which listing should we schedule for the cinematic tour in ${NEXT_MONTH}? If we do not hear back by ${DESIGNATION_DEADLINE}, we will allocate it to the highest-value listing in the queue."

ENCODED=$(python3 -c "import urllib.parse,sys; print(urllib.parse.quote(sys.argv[1]))" "${MSG}")
curl -s "https://api.callmebot.com/whatsapp.php?phone=${PHONE}&text=${ENCODED}&apikey=${APIKEY}"
```

**Cinematic tour production:** run SOP-02 in full for the designated listing, including Runway Gen-3 animation.

---

## IX. REFERRAL REVENUE TRACKING (10% on design projects)

When an agency client contacts LIOR for any design service (SOP-04 through SOP-10) via agency referral:

```bash
AGENCY_CODE="GGP"
WORKDIR=".tmp/studio-partner-${AGENCY_CODE}"
PROJECT_ID="LIOR-DC2601"
SERVICE="Design Concept"
VALUE="8500"  # AED
CREDIT=$(python3 -c "print(int(${VALUE} * 0.10))")

# Log the referral
echo "| $(date -u +%Y-%m-%d) | ${PROJECT_ID} | ${SERVICE} | AED ${VALUE} | AED ${CREDIT} | Pending |" \
  >> "${WORKDIR}/referrals/referral-log.txt"

# Update Notion: tag the project as referral source = agency
NOTION_TOKEN=$(grep NOTION_TOKEN /Users/cashville/.env | cut -d= -f2)
NOTION_PAGE_ID="[ID of the referred project's Notion page]"

curl -s -X PATCH "https://api.notion.com/v1/pages/${NOTION_PAGE_ID}" \
  -H "Authorization: Bearer ${NOTION_TOKEN}" \
  -H "Notion-Version: 2022-06-28" \
  -H "Content-Type: application/json" \
  -d "{\"properties\":{\"Referral Source\":{\"rich_text\":[{\"text\":{\"content\":\"${AGENCY_CODE}\"}}]},\"Referral Credit\":{\"number\":${CREDIT}}}}"

echo "Referral logged: ${PROJECT_ID} | Credit: AED ${CREDIT}"
cat "${WORKDIR}/referrals/referral-log.txt"
```

---

## X. MONTHLY PERFORMANCE REPORT (sent 1st of each month)

### Step 1: Generate Report

```bash
AGENCY_CODE="GGP"
AGENCY_NAME="[Agency Name]"
WORKDIR=".tmp/studio-partner-${AGENCY_CODE}"
MONTH_LABEL="$(date -u +%B %Y)"
LEDGER="${WORKDIR}/ledger/partner-ledger-$(date -u +%Y-%m).txt"

# Calculate stats from ledger
DELIVERED=$(grep -c "Delivered" "${LEDGER}" 2>/dev/null || echo 0)
ROLLOVER="[N from prior month: check prior ledger]"
AVG_TURNAROUND="[X]"
ON_TIME_RATE="[%]"
REVISIONS="[N]"
REFERRAL_THIS_MONTH="[N projects]"
REFERRAL_CREDIT="[AED X]"
REFERRAL_CUMULATIVE="[AED X]"
IN_PROGRESS="[N]"
NEXT_TOUR="[property name or TBD]"

cat > "${WORKDIR}/reports/${AGENCY_CODE}-report-$(date -u +%Y-%m).md" << EOF
---
title: "Studio Partner Report: ${AGENCY_NAME}: ${MONTH_LABEL}"
geometry: "margin=2cm"
fontsize: 11pt
---

# Studio Partner Report
**Agency:** ${AGENCY_NAME}
**Period:** ${MONTH_LABEL}
**Prepared by:** LIOR

---

## Listings

| Metric | Value |
|--------|-------|
| Delivered this month | ${DELIVERED} / 5 |
| Rollover to next month | $((5 - DELIVERED)) |
| Cinematic tour | [Property name]: delivered [date] |

---

## Turnaround

| Metric | Value | Target |
|--------|-------|--------|
| Average delivery time | ${AVG_TURNAROUND}h | 48h |
| On-time rate | ${ON_TIME_RATE}% | 100% |
| Revision requests | ${REVISIONS} |: |

---

## Referral Revenue

| Metric | Value |
|--------|-------|
| Projects this month | ${REFERRAL_THIS_MONTH} |
| Referral credit | AED ${REFERRAL_CREDIT} |
| Cumulative credit | AED ${REFERRAL_CUMULATIVE} |

---

## Upcoming

| Item | Status |
|------|--------|
| Listings in progress | ${IN_PROGRESS} |
| Next cinematic tour | ${NEXT_TOUR} |

---

*LIOR Studio: ${MONTH_LABEL}*

EOF

pandoc "${WORKDIR}/reports/${AGENCY_CODE}-report-$(date -u +%Y-%m).md" \
  -o "${WORKDIR}/reports/${AGENCY_CODE}-report-$(date -u +%Y-%m).pdf" \
  --pdf-engine=xelatex \
  -V geometry:margin=2cm

# Fallback
# pandoc ... --pdf-engine=wkhtmltopdf

echo "Monthly report generated."
```

### Step 2: Upload Report to WeTransfer

```bash
WT_KEY=$(grep WETRANSFER_API_KEY /Users/cashville/.env | cut -d= -f2)
AGENCY_CODE="GGP"
REPORT_FILE=".tmp/studio-partner-${AGENCY_CODE}/reports/${AGENCY_CODE}-report-$(date -u +%Y-%m).pdf"
FILE_SIZE=$(wc -c < "${REPORT_FILE}")
FILE_NAME="${AGENCY_CODE}-report-$(date -u +%Y-%m).pdf"
MONTH_LABEL="$(date -u +%B %Y)"

TRANSFER=$(curl -s -X POST "https://dev.wetransfer.com/v2/transfers" \
  -H "Content-Type: application/json" -H "x-api-key: ${WT_KEY}" \
  -d "{\"message\":\"${AGENCY_CODE} Studio Partner Report: ${MONTH_LABEL}\",\"files\":[{\"name\":\"${FILE_NAME}\",\"size\":${FILE_SIZE}}]}")

TRANSFER_ID=$(echo $TRANSFER | python3 -c "import sys,json; print(json.load(sys.stdin)['id'])")
UPLOAD_URL=$(echo $TRANSFER | python3 -c "import sys,json; print(json.load(sys.stdin)['files'][0]['multipart']['url'])")

curl -s -X PUT "${UPLOAD_URL}" \
  -H "Content-Type: application/octet-stream" \
  --data-binary @"${REPORT_FILE}"

REPORT_DOWNLOAD_URL=$(curl -s -X PUT "https://dev.wetransfer.com/v2/transfers/${TRANSFER_ID}/finalize" \
  -H "x-api-key: ${WT_KEY}" | python3 -c "import sys,json; print(json.load(sys.stdin)['url'])")

echo "DOWNLOAD LINK: ${REPORT_DOWNLOAD_URL}"
```

### Step 3: Send WhatsApp to Agency Contact

```bash
PHONE=$(grep CALLMEBOT_PHONE /Users/cashville/.env | cut -d= -f2)
APIKEY=$(grep CALLMEBOT_API_KEY /Users/cashville/.env | cut -d= -f2)
CONTACT_NAME="[CONTACT NAME]"

MSG="${CONTACT_NAME}: ${MONTH_LABEL} partner report ready.

${DELIVERED} listings delivered · ${AVG_TURNAROUND}h avg turnaround · AED ${REFERRAL_CREDIT} referral credit.

Full report: ${REPORT_DOWNLOAD_URL}"

ENCODED=$(python3 -c "import urllib.parse,sys; print(urllib.parse.quote(sys.argv[1]))" "${MSG}")
curl -s "https://api.callmebot.com/whatsapp.php?phone=${PHONE}&text=${ENCODED}&apikey=${APIKEY}"
echo "Monthly report WhatsApp sent."
```

### Step 4: Update Notion

```bash
NOTION_TOKEN=$(grep NOTION_TOKEN /Users/cashville/.env | cut -d= -f2)
AGENCY_PAGE_ID="[ID of agency page in Notion Leads DB]"

curl -s -X PATCH "https://api.notion.com/v1/pages/${AGENCY_PAGE_ID}" \
  -H "Authorization: Bearer ${NOTION_TOKEN}" \
  -H "Notion-Version: 2022-06-28" \
  -H "Content-Type: application/json" \
  -d "{\"properties\":{\"Last Report\":{\"date\":{\"start\":\"$(date -u +%Y-%m-%d)\"}},\"Report Link\":{\"url\":\"${REPORT_DOWNLOAD_URL}\"}}}"

echo "Notion updated."
```

### Step 5: Log

```bash
echo "[$(date '+%Y-%m-%d %H:%M')] MONTHLY REPORT DELIVERED: ${AGENCY_CODE} | Month: $(date -u +%Y-%m) | Delivered: ${DELIVERED}/5 | Referral: AED ${REFERRAL_CREDIT} | Link: ${REPORT_DOWNLOAD_URL}" \
  >> ".tmp/studio-partner-${AGENCY_CODE}/log.txt"
```

---

## XI. ROLLOVER TRACKING

Rules:
- Unused listings in a month roll to the next month (max 1-month carry)
- Rollover does not accumulate beyond 1 extra month
- Example: 3 listings used in June → 2 roll to July → July total = 7 (5 monthly + 2 rollover)

```bash
AGENCY_CODE="GGP"
CURRENT_MONTH="$(date -u +%Y-%m)"
PRIOR_MONTH="$(date -u -v-1m +%Y-%m)"
WORKDIR=".tmp/studio-partner-${AGENCY_CODE}"

# Calculate rollover
PRIOR_LEDGER="${WORKDIR}/ledger/partner-ledger-${PRIOR_MONTH}.txt"
PRIOR_DELIVERED=$(grep -c "Delivered" "${PRIOR_LEDGER}" 2>/dev/null || echo 0)
ROLLOVER=$((5 - PRIOR_DELIVERED))
if [ $ROLLOVER -lt 0 ]; then ROLLOVER=0; fi

CURRENT_MONTH_TOTAL=$((5 + ROLLOVER))

echo "Prior month delivered: ${PRIOR_DELIVERED}/5"
echo "Rollover to this month: ${ROLLOVER}"
echo "Total available this month: ${CURRENT_MONTH_TOTAL}"
```

Send unused slots reminder if rollover > 0:

```bash
if [ $ROLLOVER -gt 0 ]; then
  PHONE=$(grep CALLMEBOT_PHONE /Users/cashville/.env | cut -d= -f2)
  APIKEY=$(grep CALLMEBOT_API_KEY /Users/cashville/.env | cut -d= -f2)
  CONTACT_NAME="[CONTACT NAME]"

  MSG="${CONTACT_NAME}: you have ${ROLLOVER} listing slot(s) carried over from last month. You now have ${CURRENT_MONTH_TOTAL} slots available this month. Send listings when ready."

  ENCODED=$(python3 -c "import urllib.parse,sys; print(urllib.parse.quote(sys.argv[1]))" "${MSG}")
  curl -s "https://api.callmebot.com/whatsapp.php?phone=${PHONE}&text=${ENCODED}&apikey=${APIKEY}"
  echo "Rollover notification sent."
fi
```

---

## XII. WHATSAPP TEMPLATES

### Monthly Report

```
[CONTACT NAME]: [Month] partner report ready.

[N] listings delivered · [X]h avg turnaround · AED [X] referral credit.

Full report: [LINK]
```

### Unused Slots Reminder

```
[CONTACT NAME]: you have [N] listing slot(s) carried over from last month. You now have [N] slots available this month. Send listings when ready.
```

### Renewal (30 days before)

```
[CONTACT NAME]: your Studio Partner agreement renews on [date]. Everything continues as normal. Let us know if you would like to discuss your plan or adjust scope.
```

Rules for all templates: no "AI", no "luxury", no prices, no "drone".

---

## XIII. QA CHECKLIST

Per listing delivery:
- [ ] All rooms staged (up to 8)
- [ ] Co-branding applied correctly (watermark for co-branded, none or agency logo for white-label)
- [ ] Cinematic tour delivered for designated listing only
- [ ] Delivered within 48h of receiving complete inputs
- [ ] Partner ledger updated on delivery

Monthly report:
- [ ] Stats pulled from ledger: not estimated
- [ ] Referral credits calculated correctly (10% of invoiced project value)
- [ ] Rollover calculated correctly
- [ ] Report sent on 1st of month (or within 3 days)
- [ ] Notion updated

---

## XIV. FALLBACKS

| Tool | Failure | Fix |
|---|---|---|
| WeTransfer API | Upload fails | Manual upload at wetransfer.com |
| WeTransfer | Report >2GB | Unlikely (PDF only). If so, compress: `ghostscript -dPDFSETTINGS=/ebook -dNOPAUSE -dQUIET -dBATCH -sDEVICE=pdfwrite -sOutputFile=out.pdf in.pdf` |
| WeTransfer | Down | Google Drive: share "anyone with link can view" |
| Pandoc PDF | xelatex font error | `cp [font.ttf] ~/Library/Fonts/ && fc-cache -fv`. Or `--pdf-engine=wkhtmltopdf` |
| CallMeBot | Phone not registered | Send "I allow callmebot to send me messages" to +34 644 29 73 73 |
| Notion API | Rate limit | Wait 60s and retry |
| ImageMagick watermark | Font not found | `brew install imagemagick` with extra fonts or use fallback font name `Helvetica` |

---

## XV. ESCALATION: HUMAN MUST HANDLE

- Agency exceeds 5 listings in a month → discuss upgrade or ad-hoc billing
- Turnaround target missed (>48h) → immediate alert + explanation to agency contact
- Referral project dispute
- Agency contact changes
- Agency signals intent to cancel
- Any revision dispute

---

## XVI. UPGRADE PATH

If agency consistently uses all 5 slots + rolls over for 2+ consecutive months:

```bash
PHONE=$(grep CALLMEBOT_PHONE /Users/cashville/.env | cut -d= -f2)
APIKEY=$(grep CALLMEBOT_API_KEY /Users/cashville/.env | cut -d= -f2)
CONTACT_NAME="[CONTACT NAME]"
AGENCY_CODE="GGP"
MONTHS_AT_CAPACITY="[N]"

MSG="${CONTACT_NAME}: you have been at full capacity for ${MONTHS_AT_CAPACITY} months. We could expand your monthly allocation to 10 listings with a priority slot structure. Worth a quick call this week?"

ENCODED=$(python3 -c "import urllib.parse,sys; print(urllib.parse.quote(sys.argv[1]))" "${MSG}")
curl -s "https://api.callmebot.com/whatsapp.php?phone=${PHONE}&text=${ENCODED}&apikey=${APIKEY}"
echo "Upgrade message sent."
```

---

## XVII. OFFBOARDING

If agency cancels (honour 30-day notice):

```bash
AGENCY_CODE="GGP"
WORKDIR=".tmp/studio-partner-${AGENCY_CODE}"

# 1. Deliver any queued work within notice period
# 2. Generate final summary report (all months)
# 3. Apply any outstanding referral credits
# 4. Archive ledger

echo "[$(date '+%Y-%m-%d')] OFFBOARDED: ${AGENCY_CODE} | Final work delivered | Ledger archived | Credits: AED [X] applied" \
  >> "${WORKDIR}/log.txt"

# Archive
cp -r "${WORKDIR}/" "05-clients/lior/studio-partners/${AGENCY_CODE}/"
echo "Archived."
```

---

## XVIII. LOG FORMAT

```
[YYYY-MM-DD HH:MM] LISTING RECEIVED: [AGENCY_CODE]-[YYYYMM]-[SEQ] | ref=[agency ref] | rooms=[N] | tour=[yes/no]
[YYYY-MM-DD HH:MM] INPUTS VALIDATED: [listing ID]: all OK / [item] rejected → reshoot requested
[YYYY-MM-DD HH:MM] PRODUCTION STARTED: [listing ID]
[YYYY-MM-DD HH:MM] LISTING DELIVERED: [listing ID] | turnaround=[Xh] | Link: [URL] | Ledger updated
[YYYY-MM-DD HH:MM] ROLLOVER: [N] slots carried to [YYYY-MM]
[YYYY-MM-DD HH:MM] REFERRAL LOGGED: [project ID] | credit: AED [X]
[YYYY-MM-DD HH:MM] MONTHLY REPORT DELIVERED: [AGENCY_CODE] | Month: [YYYY-MM] | [N]/5 listings | Link: [URL]
[YYYY-MM-DD HH:MM] ESCALATION: [reason] → human notified
```
