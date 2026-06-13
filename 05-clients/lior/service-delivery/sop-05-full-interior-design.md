# SOP 05 — Full Interior Design
**Automation level: C — Guided (agent handles production and coordination; human leads all client sessions)**
**Timeline: 2–4 weeks depending on property size**
**Deliverable: Complete multi-room design · interior architecture plans · full visualization · material + furniture specification · contractor-ready documentation · project coordination**

## I. OVERVIEW

The highest-value service. Client gets a single point of contact from first concept to finished space. Five phases with human-gated checkpoints between each. Agent manages the entire production pipeline — questionnaire to final handover package. Human manages the client relationship, presents concept, and confirms direction before each phase advances.

---

## II. INPUTS

| # | Input | Format | Source | Blocker? |
|---|-------|--------|--------|---------|
| 1 | Property photos (every room, every angle) | JPG ≥ 1920×1080 | Client | YES |
| 2 | Floor plan + dimensions | PDF, CAD, or measured sketch | Client / surveyor | YES |
| 3 | Extended style brief (45-min call + questionnaire) | Call notes + form | Human-led | YES |
| 4 | Renovation scope | What can be changed vs fixed | Intake + call | YES |
| 5 | Budget range | AED or USD range | Intake | YES |
| 6 | Project timeline | Target completion date | Intake | YES |
| 7 | Contractor list | Names/contacts if existing relationships | Intake | No |
| 8 | Project ID | `LIOR-FD[CODE][YY][SEQ]` | Agent | Internal |

---

## III. SETUP (one-time)

```bash
# Install dependencies
brew install ffmpeg imagemagick pandoc
brew install --cask mactex       # LaTeX PDF engine
brew install wkhtmltopdf          # PDF engine fallback

# Verify .env keys
grep -E "IMGBB_API_KEY|VSTAGING_API_KEY|ADOBE_CLIENT_ID|ADOBE_CLIENT_SECRET|WETRANSFER_API_KEY|CALLMEBOT_PHONE|CALLMEBOT_API_KEY|NOTION_TOKEN" /Users/cashville/.env

# Keys needed:
# IMGBB_API_KEY          — image hosting
# VSTAGING_API_KEY       — Virtual Staging AI (room visualizations)
# ADOBE_CLIENT_ID        — Firefly (concept visualizations + fallback)
# ADOBE_CLIENT_SECRET    — Firefly
# WETRANSFER_API_KEY     — delivery
# CALLMEBOT_PHONE        — WhatsApp
# CALLMEBOT_API_KEY      — WhatsApp
# NOTION_TOKEN           — project tracking
```

---

## IV. FOLDER STRUCTURE

```bash
PROJECT_ID="LIOR-FD2601"   # replace per project
mkdir -p .tmp/${PROJECT_ID}-full-design/{00-brief,01-client-inputs,02-concept/{moodboards,space-plans,concept-viz},03-design/{visualizations,space-plans},04-documentation,05-handover,06-exports}
touch .tmp/${PROJECT_ID}-full-design/log.txt
```

---

## V. PROJECT PHASES

```
Phase 1: Discovery + Brief        (Days 1–3)   → human checkpoint: brief confirmed in writing
Phase 2: Concept + Direction      (Days 4–7)   → human checkpoint: one direction selected
Phase 3: Full Design Development  (Days 8–14)  → human checkpoint: full design approved / revision
Phase 4: Specification + Docs     (Days 15–18) → human checkpoint: documents approved for contractor
Phase 5: Coordination + Handover  (Days 19–28) → final delivery
```

**Rule: a phase does not start until the previous checkpoint is confirmed.**

---

## VI. PHASE 1 — DISCOVERY + BRIEF (Days 1–3)

### Step 1 — Send Extended Questionnaire

```bash
PHONE=$(grep CALLMEBOT_PHONE /Users/cashville/.env | cut -d= -f2)
APIKEY=$(grep CALLMEBOT_API_KEY /Users/cashville/.env | cut -d= -f2)
CLIENT_NAME="[CLIENT NAME]"
QUESTIONNAIRE_URL="https://[lior-domain]/questionnaire.html"

MSG="${CLIENT_NAME} — to begin your full design project, we need your input on a few things before our call. This takes about 15 minutes and shapes everything that follows.

Complete here: ${QUESTIONNAIRE_URL}"

ENCODED=$(python3 -c "import urllib.parse,sys; print(urllib.parse.quote(sys.argv[1]))" "${MSG}")
curl -s "https://api.callmebot.com/whatsapp.php?phone=${PHONE}&text=${ENCODED}&apikey=${APIKEY}"
echo "Questionnaire sent."
```

**Full questionnaire (SOP-04 fields 1–10, plus these additions for Full Design):**

```
11. Which rooms are highest priority for transformation?
    Rank: Living / Kitchen / Master / Other bedrooms / Bathrooms

12. What structural changes are you open to?
    Walls / partitions / Kitchen layout / Bathroom position / None — cosmetic only

13. Do you have existing furniture you want to keep?
    Free text + photos if yes

14. Any specific suppliers or brands you want to work with?
    Free text

15. Is there a phasing requirement? (e.g. do bedrooms first, then living)
    Free text

16. Who is the main decision-maker?
    Free text (one person / couple / family)
```

### Step 2 — Parse Brief

```bash
PROJECT_ID="LIOR-FD2601"
WORKDIR=".tmp/${PROJECT_ID}-full-design"

cat > "${WORKDIR}/00-brief/00-full-design-brief.txt" << 'EOF'
FULL DESIGN BRIEF — [PROJECT_ID]
Date: [date]
Rooms: [list with priority ranking]
Constraints: [fixed elements — load-bearing walls, existing plumbing, electrical panel, window positions]
Renovation scope: [what is possible vs fixed]
Budget tier: [entry / mid / high] — [estimated AED range per room]
Timeline: [target completion, milestones working backward]
Decision-maker: [name(s)]
Key preferences: [top 5 from questionnaire]
Key avoidances: [top 3]
Existing furniture to keep: [list or none]
Preferred suppliers: [list or none]
Phasing: [if applicable]
Style direction hypothesis: [agent's suggested direction — for human to confirm in Phase 2]
EOF
```

**Human confirms brief in writing → Phase 2 starts.**

WhatsApp checkpoint to client:

```bash
MSG="${CLIENT_NAME} — your project brief is confirmed. Here is what we are working from:

Style direction: [hypothesis]
Priority rooms: [top 3]
Renovation scope: [cosmetic / partial / structural]
Target completion: [date]

We will share the first concept presentation within [N] days. Any corrections before we start?"

ENCODED=$(python3 -c "import urllib.parse,sys; print(urllib.parse.quote(sys.argv[1]))" "${MSG}")
curl -s "https://api.callmebot.com/whatsapp.php?phone=${PHONE}&text=${ENCODED}&apikey=${APIKEY}"
```

---

## VII. PHASE 2 — CONCEPT + DIRECTION (Days 4–7)

### Step 1 — Get Firefly Token

```bash
ADOBE_CLIENT_ID=$(grep ADOBE_CLIENT_ID /Users/cashville/.env | cut -d= -f2)
ADOBE_CLIENT_SECRET=$(grep ADOBE_CLIENT_SECRET /Users/cashville/.env | cut -d= -f2)

FIREFLY_TOKEN=$(curl -s -X POST "https://ims-na1.adobelogin.com/ims/token/v3" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials&client_id=${ADOBE_CLIENT_ID}&client_secret=${ADOBE_CLIENT_SECRET}&scope=openid,AdobeID,firefly_enterprise,firefly_api" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

echo $FIREFLY_TOKEN > /tmp/firefly_token.txt
echo "Token obtained."
```

### Step 2 — Generate Two Direction Moodboards

Use Firefly to generate one moodboard per direction (see SOP-04 Step 3 for full moodboard generation script — same process, run twice with different direction prompts).

```bash
FIREFLY_TOKEN=$(cat /tmp/firefly_token.txt)
ADOBE_CLIENT_ID=$(grep ADOBE_CLIENT_ID /Users/cashville/.env | cut -d= -f2)
PROJECT_ID="LIOR-FD2601"
WORKDIR=".tmp/${PROJECT_ID}-full-design"

for DIRECTION in A B; do
  if [ "$DIRECTION" = "A" ]; then
    PROMPT="interior design moodboard, contemporary warm Dubai apartment, palette cream and warm oak and brushed brass, linen textures, flat lay composition with material swatches and furniture references, editorial lifestyle photography, warm afternoon light"
  else
    PROMPT="interior design moodboard, modern minimal Dubai apartment, palette white and charcoal and matte black, clean lines, flat lay composition with material swatches and furniture references, architectural editorial photography, cool daylight"
  fi

  RESPONSE=$(curl -s -X POST "https://firefly-api.adobe.io/v3/images/generate" \
    -H "Authorization: Bearer ${FIREFLY_TOKEN}" \
    -H "X-Api-Key: ${ADOBE_CLIENT_ID}" \
    -H "Content-Type: application/json" \
    -d "{
      \"prompt\": \"${PROMPT}\",
      \"negativePrompt\": \"cheap, IKEA, cartoon, oversaturated, watermark\",
      \"size\": { \"width\": 2048, \"height\": 1152 },
      \"numVariations\": 3,
      \"contentClass\": \"photo\",
      \"styles\": { \"presets\": [\"photo_realism\"] }
    }")

  CHOSEN_URL=$(echo $RESPONSE | python3 -c "import sys,json; print(json.load(sys.stdin)['outputs'][0]['image']['url'])")
  curl -L "${CHOSEN_URL}" -o "${WORKDIR}/02-concept/moodboards/${PROJECT_ID}-direction-${DIRECTION}-moodboard-v1.jpg"
  echo "Moodboard Direction ${DIRECTION} saved."
done
```

### Step 3 — Concept Visualization (1 key room per direction)

Use Virtual Staging AI if photos exist, Firefly if not (see SOP-04 Steps 5B/5C for full scripts).

```bash
# Save outputs to:
# .tmp/${PROJECT_ID}-full-design/02-concept/concept-viz/${PROJECT_ID}-direction-A-living-viz-v1.jpg
# .tmp/${PROJECT_ID}-full-design/02-concept/concept-viz/${PROJECT_ID}-direction-B-living-viz-v1.jpg
```

### Step 4 — Space Plan for Priority Room (per direction)

Use RoomSketcher web UI (see SOP-04 Step 4 for full RoomSketcher workflow).

```bash
# Save outputs to:
# .tmp/${PROJECT_ID}-full-design/02-concept/space-plans/${PROJECT_ID}-direction-A-spaceplan-v1.jpg
# .tmp/${PROJECT_ID}-full-design/02-concept/space-plans/${PROJECT_ID}-direction-B-spaceplan-v1.jpg
```

### Step 5 — Human Presents to Client

Human presents concept via call or in-person meeting. Client selects one direction.

If client wants elements from both → human resolves into a hybrid → documented in brief:

```bash
echo "DIRECTION SELECTED: [A / B / Hybrid — description]" >> "${WORKDIR}/00-brief/00-full-design-brief.txt"
echo "[$(date '+%Y-%m-%d %H:%M')] CHECKPOINT PHASE 2: direction selected — [direction]" >> "${WORKDIR}/log.txt"
```

**WhatsApp checkpoint after direction confirmed:**

```bash
MSG="${CLIENT_NAME} — direction confirmed. We are now moving into full design development for all rooms.

This phase covers: [list all rooms].

We will check back with you before finalizing. Estimated timeline: [N] days."

ENCODED=$(python3 -c "import urllib.parse,sys; print(urllib.parse.quote(sys.argv[1]))" "${MSG}")
curl -s "https://api.callmebot.com/whatsapp.php?phone=${PHONE}&text=${ENCODED}&apikey=${APIKEY}"
```

---

## VIII. PHASE 3 — FULL DESIGN DEVELOPMENT (Days 8–14)

Execute full production across all rooms in the approved direction.

### For each room, produce three deliverables:

#### 3A — Visualization

Upload room photo to ImgBB, then run Virtual Staging AI:

```bash
IMGBB_KEY=$(grep IMGBB_API_KEY /Users/cashville/.env | cut -d= -f2)
VSTAGING_KEY=$(grep VSTAGING_API_KEY /Users/cashville/.env | cut -d= -f2)
ADOBE_CLIENT_ID=$(grep ADOBE_CLIENT_ID /Users/cashville/.env | cut -d= -f2)
FIREFLY_TOKEN=$(cat /tmp/firefly_token.txt)
PROJECT_ID="LIOR-FD2601"
WORKDIR=".tmp/${PROJECT_ID}-full-design"

ROOMS=("living" "master" "kitchen" "bedroom2" "bathroom" "hallway")  # adjust per project

for ROOM in "${ROOMS[@]}"; do
  PHOTO="${WORKDIR}/01-client-inputs/${ROOM}-empty.jpg"

  if [ -f "${PHOTO}" ]; then
    # Upload to ImgBB
    IMAGE_URL=$(curl -s -X POST "https://api.imgbb.com/1/upload" \
      -F "key=${IMGBB_KEY}" -F "image=@${PHOTO}" \
      | python3 -c "import sys,json; r=json.load(sys.stdin); print(r['data']['url'])")

    # Virtual Staging AI
    JOB=$(curl -s -X POST "https://api.virtualstaging.ai/v1/stage" \
      -H "Authorization: Bearer ${VSTAGING_KEY}" \
      -H "Content-Type: application/json" \
      -d "{\"image_url\":\"${IMAGE_URL}\",\"room_type\":\"${ROOM}_room\",\"style\":\"contemporary\",\"furnishing_level\":\"medium\",\"virtual_tour\":false}")

    JOB_ID=$(echo $JOB | python3 -c "import sys,json; print(json.load(sys.stdin)['job_id'])")

    while true; do
      RESULT=$(curl -s "https://api.virtualstaging.ai/v1/jobs/${JOB_ID}" \
        -H "Authorization: Bearer ${VSTAGING_KEY}")
      STATUS=$(echo $RESULT | python3 -c "import sys,json; print(json.load(sys.stdin)['status'])")
      if [ "$STATUS" = "completed" ]; then
        OUTPUT_URL=$(echo $RESULT | python3 -c "import sys,json; print(json.load(sys.stdin)['output_url'])")
        curl -L "${OUTPUT_URL}" -o "${WORKDIR}/03-design/visualizations/${PROJECT_ID}-viz-${ROOM}-v1.jpg"
        echo "Staged: ${ROOM}"
        break
      elif [ "$STATUS" = "failed" ]; then
        echo "FAILED: ${ROOM} — switching to Firefly"
        # Firefly fallback
        RESPONSE=$(curl -s -X POST "https://firefly-api.adobe.io/v3/images/generate" \
          -H "Authorization: Bearer ${FIREFLY_TOKEN}" \
          -H "X-Api-Key: ${ADOBE_CLIENT_ID}" \
          -H "Content-Type: application/json" \
          -d "{\"prompt\":\"${ROOM} interior design, contemporary warm Dubai apartment, photorealistic, editorial photography\",\"negativePrompt\":\"cheap, watermark, cartoon\",\"size\":{\"width\":2048,\"height\":1152},\"numVariations\":1,\"contentClass\":\"photo\",\"styles\":{\"presets\":[\"photo_realism\"]}}")
        URL=$(echo $RESPONSE | python3 -c "import sys,json; print(json.load(sys.stdin)['outputs'][0]['image']['url'])")
        curl -L "${URL}" -o "${WORKDIR}/03-design/visualizations/${PROJECT_ID}-viz-${ROOM}-v1.jpg"
        echo "[$(date '+%Y-%m-%d %H:%M')] FALLBACK vstaging → Firefly for ${ROOM}" >> "${WORKDIR}/log.txt"
        break
      fi
      echo "${ROOM}: ${STATUS} — waiting 10s..."
      sleep 10
    done
  else
    echo "No photo for ${ROOM} — using Firefly render"
    RESPONSE=$(curl -s -X POST "https://firefly-api.adobe.io/v3/images/generate" \
      -H "Authorization: Bearer ${FIREFLY_TOKEN}" \
      -H "X-Api-Key: ${ADOBE_CLIENT_ID}" \
      -H "Content-Type: application/json" \
      -d "{\"prompt\":\"${ROOM} contemporary warm Dubai apartment interior, photorealistic, editorial photography, 8k\",\"negativePrompt\":\"cheap, watermark, cartoon\",\"size\":{\"width\":2048,\"height\":1152},\"numVariations\":1,\"contentClass\":\"photo\",\"styles\":{\"presets\":[\"photo_realism\"]}}")
    URL=$(echo $RESPONSE | python3 -c "import sys,json; print(json.load(sys.stdin)['outputs'][0]['image']['url'])")
    curl -L "${URL}" -o "${WORKDIR}/03-design/visualizations/${PROJECT_ID}-viz-${ROOM}-v1.jpg"
  fi
done
echo "All room visualizations complete."
```

**Visualization QA per room:**
- [ ] No floating or clipping furniture
- [ ] No artifacts or impossible shadows
- [ ] Palette consistent with selected direction
- [ ] Room proportions realistic
- [ ] Natural light enters from correct side per floor plan

#### 3B — Space Plan (one per room)

Use RoomSketcher for each room (see SOP-04 Step 4 for workflow). One layout per room — direction already selected.

```bash
# Save each export to:
# .tmp/${PROJECT_ID}-full-design/03-design/space-plans/${PROJECT_ID}-plan-[room]-v1.jpg
```

#### 3C — Material Specification Draft

Write room-by-room specification (follow SOP-04 Step 6 format). For high-budget tier: include specific brand + model references.

```bash
cat > "${WORKDIR}/03-design/draft-material-schedule.md" << 'EOF'
# Draft Material Schedule — [PROJECT_ID]
[Follow SOP-04 Step 6 format for each room]
[For high budget tier: specify exact brand + product line per item]

## Interior Architecture Notes (structural changes):
# For any room with structural changes, note:
# - Proposed wall removal / partition addition (design intent only — not structural calc)
# - Proposed kitchen reconfiguration (island position, appliance layout)
# - Bathroom layout changes (WC / shower / bath positions)
# IMPORTANT: all structural items flagged for architect / structural engineer sign-off
EOF
```

**Human reviews Phase 3 output with client. One revision round included.**

WhatsApp checkpoint:

```bash
MSG="${CLIENT_NAME} — the full design development for all [N] rooms is complete. I will share it with you today for review.

One revision round is included if you would like to adjust anything before we move to documentation."

ENCODED=$(python3 -c "import urllib.parse,sys; print(urllib.parse.quote(sys.argv[1]))" "${MSG}")
curl -s "https://api.callmebot.com/whatsapp.php?phone=${PHONE}&text=${ENCODED}&apikey=${APIKEY}"
```

---

## IX. PHASE 4 — SPECIFICATION + DOCUMENTATION (Days 15–18)

Assemble three contractor-ready documents.

### Document 1 — Design Book (PDF)

```bash
PROJECT_ID="LIOR-FD2601"
WORKDIR=".tmp/${PROJECT_ID}-full-design"

# Build the Markdown source
cat > "${WORKDIR}/04-documentation/design-book.md" << 'EOF'
---
title: "Full Interior Design — [PROJECT_ID]"
date: "[Date]"
geometry: "margin=2cm"
fontsize: 11pt
---

# Full Interior Design
**Property:** [Address]
**Client:** [Name]
**Prepared by:** LIOR
**Date:** [Date]

---

## Design Narrative

[2 paragraphs: concept statement + spatial intent. Write from the approved direction brief.]

---

## Overall Floor Plan

![Overall Floor Plan](.tmp/[PROJECT_ID]-full-design/03-design/space-plans/[PROJECT_ID]-plan-overall-v1.jpg){ width=100% }

---

[Per room block — repeat for each room:]

## Living Room

![Visualization](.tmp/[PROJECT_ID]-full-design/03-design/visualizations/[PROJECT_ID]-viz-living-v1.jpg){ width=100% }

![Space Plan](.tmp/[PROJECT_ID]-full-design/03-design/space-plans/[PROJECT_ID]-plan-living-v1.jpg){ width=100% }

[Material specification for living room — from draft-material-schedule.md]

---

[Repeat block for master, kitchen, all rooms]

---

*Questions or adjustments: contact LIOR*

EOF

# Compile PDF
pandoc "${WORKDIR}/04-documentation/design-book.md" \
  -o "${WORKDIR}/04-documentation/${PROJECT_ID}-design-book-v1.pdf" \
  --pdf-engine=xelatex \
  -V geometry:margin=2cm \
  -V fontsize=11pt

echo "Design book PDF generated."
```

### Document 2 — Material Schedule (structured PDF)

```bash
cat > "${WORKDIR}/04-documentation/material-schedule-table.md" << 'EOF'
---
title: "Material Schedule — [PROJECT_ID]"
geometry: "margin=1.5cm"
fontsize: 10pt
---

# Material Schedule — [PROJECT_ID]

| Room | Element | Specification | Supplier | Qty estimate | Notes |
|------|---------|--------------|---------|-------------|-------|
| Living | Flooring | Natural oak 14mm engineered, matte, herringbone | [Supplier] | ~40m² | Underfloor heating compatible |
| Living | Walls | Farrow & Ball Skimming Stone No.241 | [Local paint supplier] | ~60m² | 2 coats |
| Master | Flooring | [same as living / carpet] | [Supplier] | ~20m² | |
| Kitchen | Countertop | Honed engineered quartz, white vein | [Supplier] | ~8m² | |
[Continue for all rooms and elements]

EOF

pandoc "${WORKDIR}/04-documentation/material-schedule-table.md" \
  -o "${WORKDIR}/04-documentation/${PROJECT_ID}-material-schedule-v1.pdf" \
  --pdf-engine=xelatex \
  -V geometry:margin=1.5cm

echo "Material schedule PDF generated."
```

### Document 3 — Scope of Works

```bash
cat > "${WORKDIR}/04-documentation/scope-of-works.md" << 'EOF'
---
title: "Scope of Works — [PROJECT_ID]"
geometry: "margin=2cm"
fontsize: 11pt
---

# Scope of Works — [PROJECT_ID]
Property: [Address]
Budget envelope: AED [X]
Target completion: [Date]

---

## Living Room
- Strip existing flooring (type: [parquet / tile / carpet]) — full room
- Install engineered oak herringbone ([spec])
  Approx area: [X]m²
- Repaint all walls — color: [ref] — 2 coats minimum
- Ceiling-mounted curtain track — [X]m run
- No structural changes required

---

## Kitchen
- Strip existing cabinetry (full kitchen)
- Install new [style] kitchen units — [color]
- Install countertop: [material, color ref]
- Install backsplash: [material, dimensions]
- Appliances: [list brand tier]

[STRUCTURAL NOTE: Removal of wall between kitchen and living requires structural assessment.
LIOR cannot confirm load-bearing status — consult structural engineer before work begins.]

---

[Continue for all rooms]

---

## Total Scope Summary

| Room | Scope level | Estimated allocation | Priority |
|------|------------|---------------------|---------|
| Living | Cosmetic | AED X,XXX–X,XXX | High |
| Kitchen | Full | AED X,XXX–X,XXX | High |
| Master | Partial | AED X,XXX–X,XXX | Medium |
| **TOTAL** | | **AED XX,XXX–XX,XXX** | |

*All allocations are LIOR estimates. Final costs depend on contractor quotes, material sourcing, and site conditions.*

EOF

pandoc "${WORKDIR}/04-documentation/scope-of-works.md" \
  -o "${WORKDIR}/04-documentation/${PROJECT_ID}-scope-v1.pdf" \
  --pdf-engine=xelatex

echo "Scope of works PDF generated."
```

**Human approves documents → Phase 5 starts.**

WhatsApp checkpoint:

```bash
MSG="${CLIENT_NAME} — your full documentation package is ready for review. Three documents:

1. Design Book (all rooms, visualizations, and material specs)
2. Material Schedule (purchasing reference)
3. Scope of Works (contractor instructions)

Ready to send to your contractor as soon as you approve. Sending now for your review."

ENCODED=$(python3 -c "import urllib.parse,sys; print(urllib.parse.quote(sys.argv[1]))" "${MSG}")
curl -s "https://api.callmebot.com/whatsapp.php?phone=${PHONE}&text=${ENCODED}&apikey=${APIKEY}"
```

---

## X. PHASE 5 — COORDINATION + HANDOVER (Days 19–28)

### Step 1 — Assemble Handover Package

```bash
PROJECT_ID="LIOR-FD2601"
WORKDIR=".tmp/${PROJECT_ID}-full-design"

mkdir -p "${WORKDIR}/05-handover"

# Copy all finalized documents
cp "${WORKDIR}/04-documentation/${PROJECT_ID}-design-book-v1.pdf" "${WORKDIR}/05-handover/"
cp "${WORKDIR}/04-documentation/${PROJECT_ID}-material-schedule-v1.pdf" "${WORKDIR}/05-handover/"
cp "${WORKDIR}/04-documentation/${PROJECT_ID}-scope-v1.pdf" "${WORKDIR}/05-handover/"

# Create visualizations ZIP
zip -r "${WORKDIR}/05-handover/${PROJECT_ID}-all-visualizations.zip" \
  "${WORKDIR}/03-design/visualizations/"

# Create floor plans ZIP
zip -r "${WORKDIR}/05-handover/${PROJECT_ID}-floor-plans.zip" \
  "${WORKDIR}/03-design/space-plans/"

ls -lh "${WORKDIR}/05-handover/"
```

### Step 2 — Send to Contractors (if email access configured)

```bash
# For each contractor, personalize a message
CONTRACTOR_NAME="[Contractor name]"
CONTRACTOR_SCOPE="[their specific rooms / scope]"

# Agent drafts, human approves before sending
cat << EOF
Subject: Design Package — [Property Address] — [PROJECT_ID]

Hi ${CONTRACTOR_NAME},

Please find attached the full design brief for [property address].

Your scope: ${CONTRACTOR_SCOPE}

Documents included:
- Design Book (reference visualizations + material intent)
- Scope of Works (your instruction document)
- Material Schedule (purchasing guide)

Budget allocation for your scope: AED [X]
Target start: [date]

Please review and share your quote. Questions: contact LIOR directly.

LIOR Studio
EOF
```

### Step 3 — Check-in Reminders in Notion

```bash
NOTION_TOKEN=$(grep NOTION_TOKEN /Users/cashville/.env | cut -d= -f2)
PAGE_ID="[ID_NOTION]"

# Update project status + set check-in milestones
curl -s -X PATCH "https://api.notion.com/v1/pages/${PAGE_ID}" \
  -H "Authorization: Bearer ${NOTION_TOKEN}" \
  -H "Notion-Version: 2022-06-28" \
  -H "Content-Type: application/json" \
  -d "{
    \"properties\": {
      \"Status\": {\"select\": {\"name\": \"In Construction\"}},
      \"Week 2 Check-in\": {\"date\": {\"start\": \"$(date -u -v+14d +%Y-%m-%d)\"}},
      \"Week 4 Check-in\": {\"date\": {\"start\": \"$(date -u -v+28d +%Y-%m-%d)\"}}
    }
  }"
echo "Notion milestones set."
```

---

## XI. QA CHECKLIST

**Phase 2 — Concept:**
- [ ] Both moodboards feel clearly distinct
- [ ] Concept visualization matches the moodboard it represents
- [ ] Space plan is physically plausible (correct clearances)

**Phase 3 — Full Design:**
- [ ] Every room has a visualization + space plan
- [ ] Palette consistent across all rooms in the selected direction
- [ ] Material schedule covers every room in scope
- [ ] Structural changes clearly flagged — not specified by LIOR

**Phase 4 — Documents:**
- [ ] Design book has no placeholder text, no broken image links
- [ ] Material schedule: all references real and available in Dubai
- [ ] Scope of works: realistic for stated budget
- [ ] Structural items flagged — contractor to get engineer sign-off
- [ ] All three PDFs open correctly

---

## XII. DELIVERY

### ZIP

```bash
PROJECT_ID="LIOR-FD2601"
WORKDIR=".tmp/${PROJECT_ID}-full-design"

cd "${WORKDIR}"
zip -r "${PROJECT_ID}-full-design-v1.zip" "05-handover/"
echo "ZIP size: $(wc -c < "${PROJECT_ID}-full-design-v1.zip") bytes"
unzip -l "${PROJECT_ID}-full-design-v1.zip"
```

### WeTransfer Upload

```bash
WT_KEY=$(grep WETRANSFER_API_KEY /Users/cashville/.env | cut -d= -f2)
ZIP_FILE="${WORKDIR}/${PROJECT_ID}-full-design-v1.zip"
ZIP_SIZE=$(wc -c < "${ZIP_FILE}")

TRANSFER=$(curl -s -X POST "https://dev.wetransfer.com/v2/transfers" \
  -H "Content-Type: application/json" -H "x-api-key: ${WT_KEY}" \
  -d "{\"message\":\"${PROJECT_ID} — Full Interior Design LIOR\",\"files\":[{\"name\":\"${PROJECT_ID}-full-design-v1.zip\",\"size\":${ZIP_SIZE}}]}")

TRANSFER_ID=$(echo $TRANSFER | python3 -c "import sys,json; print(json.load(sys.stdin)['id'])")
UPLOAD_URL=$(echo $TRANSFER | python3 -c "import sys,json; print(json.load(sys.stdin)['files'][0]['multipart']['url'])")

curl -s -X PUT "${UPLOAD_URL}" \
  -H "Content-Type: application/octet-stream" \
  --data-binary @"${ZIP_FILE}"

DOWNLOAD_URL=$(curl -s -X PUT "https://dev.wetransfer.com/v2/transfers/${TRANSFER_ID}/finalize" \
  -H "x-api-key: ${WT_KEY}" | python3 -c "import sys,json; print(json.load(sys.stdin)['url'])")

echo "DOWNLOAD LINK: ${DOWNLOAD_URL}"
```

### WhatsApp Notification

```bash
PHONE=$(grep CALLMEBOT_PHONE /Users/cashville/.env | cut -d= -f2)
APIKEY=$(grep CALLMEBOT_API_KEY /Users/cashville/.env | cut -d= -f2)
CLIENT_NAME="[CLIENT NAME]"
N_ROOMS="[N]"

MSG="${CLIENT_NAME} — your full design package is ready.

Everything your team needs to bring the project to life:
- Design visualization for all ${N_ROOMS} rooms
- Scale floor plans with furniture placement
- Complete material and furnishing specification
- Scope of works for your contractor

Download: ${DOWNLOAD_URL}

We are available throughout the build for any clarification or adjustments. This is your single reference document — share it directly with your contractor."

ENCODED=$(python3 -c "import urllib.parse,sys; print(urllib.parse.quote(sys.argv[1]))" "${MSG}")
curl -s "https://api.callmebot.com/whatsapp.php?phone=${PHONE}&text=${ENCODED}&apikey=${APIKEY}"
echo "WhatsApp sent."
```

### Notion Update

```bash
NOTION_TOKEN=$(grep NOTION_TOKEN /Users/cashville/.env | cut -d= -f2)
PAGE_ID="[ID_NOTION]"

curl -s -X PATCH "https://api.notion.com/v1/pages/${PAGE_ID}" \
  -H "Authorization: Bearer ${NOTION_TOKEN}" \
  -H "Notion-Version: 2022-06-28" \
  -H "Content-Type: application/json" \
  -d "{\"properties\":{\"Status\":{\"select\":{\"name\":\"Delivered\"}},\"Delivery Link\":{\"url\":\"${DOWNLOAD_URL}\"},\"Delivered At\":{\"date\":{\"start\":\"$(date -u +%Y-%m-%d)\"}}}}"

echo "Notion updated."
```

### Delivery Log

```bash
echo "[$(date '+%Y-%m-%d %H:%M')] DELIVERED: ${PROJECT_ID} | Service: full-interior-design | ZIP: ${PROJECT_ID}-full-design-v1.zip | Link: ${DOWNLOAD_URL} | WA: sent" \
  >> "${WORKDIR}/log.txt"
```

---

## XIII. WHATSAPP TEMPLATE

```
[CLIENT NAME] — your full design package is ready.

Everything your team needs to bring the project to life:
- Design visualization for all [N] rooms
- Scale floor plans with furniture placement
- Complete material and furnishing specification
- Scope of works for your contractor

Download: [LINK]

We are available throughout the build for any clarification or adjustments. This is your single reference document — share it directly with your contractor.
```

Rules: no "AI", no "luxury", no prices, no "drone".

---

## XIV. FALLBACKS

| Tool | Failure | Fix |
|---|---|---|
| Virtual Staging AI | API down or quota exhausted | Wait 30 min. Try REimagineHome: `POST https://api.reimaginehome.ai/v1/generate`. If both down → Firefly render. |
| Adobe Firefly | Token expired | Re-run token script (Phase 2 Step 1). Valid 24h. |
| Firefly | API down | Midjourney Discord. If Discord unavailable → Replicate SDXL. |
| Pandoc PDF | xelatex font error | `cp [font.ttf] ~/Library/Fonts/ && fc-cache -fv`. Retry. Or switch to `--pdf-engine=wkhtmltopdf`. |
| WeTransfer API | Upload fails | Manual upload at wetransfer.com |
| WeTransfer API | File >2GB | Split: Design Book ZIP + Materials/Plans ZIP separately |
| CallMeBot | Phone not registered | Send "I allow callmebot to send me messages" to +34 644 29 73 73 |
| CallMeBot | Message too long | Truncate to 300 characters max |

---

## XV. REVISION POLICY

- Phase 2 (concept direction): 1 revision round included
- Phase 3 (design development): 1 revision round included
- Phase 4 (documents): 1 correction pass included
- Additional revision rounds: at LIOR's discretion (billable)
- Scope changes (new rooms added mid-project): new brief + addendum required

**Escalate to human immediately when:**
- Client expresses dissatisfaction after Phase 2 checkpoint
- Structural feasibility question arises (agent does not advise on structural matters)
- Client requests contractor recommendation
- Budget constraint conflicts with design intent
- Client delays input and timeline is at risk

---

## XVI. LOG FORMAT

```
[YYYY-MM-DD HH:MM] STARTED: [PROJECT_ID] | rooms=[N] | budget=[tier] | scope=[cosmetic/partial/structural]
[YYYY-MM-DD HH:MM] PHASE 1 CHECKPOINT: brief confirmed by human
[YYYY-MM-DD HH:MM] PHASE 2 COMPLETE: concept presented — [N] moodboards + [N] viz
[YYYY-MM-DD HH:MM] PHASE 2 CHECKPOINT: Direction [A/B/Hybrid] selected by client
[YYYY-MM-DD HH:MM] PHASE 3 PROGRESS: [room] — visualization + space plan complete
[YYYY-MM-DD HH:MM] PHASE 3 CHECKPOINT: full design approved by client
[YYYY-MM-DD HH:MM] PHASE 4 COMPLETE: design book + material schedule + scope of works — PDFs generated
[YYYY-MM-DD HH:MM] PHASE 4 CHECKPOINT: documents approved by human
[YYYY-MM-DD HH:MM] FALLBACK: [tool] → [alternative] — [room] — [reason]
[YYYY-MM-DD HH:MM] DELIVERED: [PROJECT_ID] | ZIP: [filename] | Link: [URL] | WA: sent
```
