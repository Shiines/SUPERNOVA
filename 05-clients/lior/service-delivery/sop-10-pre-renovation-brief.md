# SOP 10 — Pre-Renovation Brief
**Automation level: B — Auto + human gate**
**Timeline: 4 days from confirmed inputs**
**Deliverable: Before/after visualizations · annotated scope of works · material + finish schedule · contractor-ready brief PDF**

## I. OVERVIEW

A complete visual dossier for the client's architect or contractor. Current state photos in → "after renovation" visualizations + room-by-room scope of works + full material specification → one assembled PDF that eliminates guesswork for any builder. Agent produces; human gates the output before delivery.

---

## II. INPUTS

| # | Input | Format | Source | Blocker? |
|---|-------|--------|--------|---------|
| 1 | Current state photos (all rooms) | JPG ≥ 1920×1080, every wall per room | Client | YES |
| 2 | Existing floor plan | PDF, JPG, or measured sketch | Client | YES |
| 3 | Renovation goals per room | What client wants to change | Intake form | YES |
| 4 | Budget range | Total renovation budget AED or USD | Intake form | YES |
| 5 | Style direction | Text, references, or questionnaire | Intake form | YES |
| 6 | Renovation constraints | Structural limits, building rules, phasing | Intake form | No |
| 7 | Target completion date | Date or timeframe | Intake form | YES |
| 8 | Project ID | `LIOR-RB[CODE][YY][SEQ]` | Agent | Internal |

---

## III. SETUP (one-time)

```bash
# Install dependencies
brew install imagemagick pandoc
brew install --cask mactex        # LaTeX PDF engine
brew install wkhtmltopdf           # PDF fallback

# Verify .env keys
grep -E "IMGBB_API_KEY|VSTAGING_API_KEY|ADOBE_CLIENT_ID|ADOBE_CLIENT_SECRET|WETRANSFER_API_KEY|CALLMEBOT_PHONE|CALLMEBOT_API_KEY|NOTION_TOKEN" /Users/cashville/.env

# Keys needed:
# IMGBB_API_KEY          — image hosting
# VSTAGING_API_KEY       — Virtual Staging AI (primary "after" visualization)
# ADOBE_CLIENT_ID        — Firefly (rooms with structural changes — no existing photo to stage)
# ADOBE_CLIENT_SECRET    — Firefly
# WETRANSFER_API_KEY     — delivery
# CALLMEBOT_PHONE        — WhatsApp
# CALLMEBOT_API_KEY      — WhatsApp
# NOTION_TOKEN           — project tracking
```

---

## IV. FOLDER STRUCTURE

```bash
PROJECT_ID="LIOR-RB2601"   # replace per project
mkdir -p .tmp/${PROJECT_ID}-reno-brief/{00-brief,01-client-inputs,02-visualizations/{before,after,before-after},03-scope,04-materials,05-exports,06-pdf-assembly}
touch .tmp/${PROJECT_ID}-reno-brief/log.txt
```

---

## V. PRODUCTION STEPS

### Step 1 — Parse Renovation Goals (Day 1)

```bash
PROJECT_ID="LIOR-RB2601"
WORKDIR=".tmp/${PROJECT_ID}-reno-brief"

cat > "${WORKDIR}/00-brief/00-renovation-brief.txt" << 'EOF'
RENOVATION BRIEF — [PROJECT_ID]
─────────────────────────────────────────────────────────
LIVING ROOM
  Current issues: [dark / dated finishes / poor layout / no storage]
  Goal: [open to kitchen? / new flooring? / repaint? / reconfigure?]
  Scope level: [cosmetic / partial / full]
  Structural change: [yes/no — specify if yes]

KITCHEN
  Current issues: [cramped / no island / outdated / poor storage]
  Goal: [new layout? / new cabinetry? / new appliances? / island addition?]
  Scope level: [cosmetic / partial / full]
  Structural change: [yes/no]

MASTER BEDROOM
  Current issues: [...]
  Goal: [...]
  Scope level: [cosmetic / partial / full]

BATHROOMS
  Current issues: [...]
  Goal: [...]
  Scope level: [cosmetic / partial / full]

[Add all rooms]

BUILDING CONSTRAINTS: [Any restrictions from building management, strata, permits]
TOTAL BUDGET: AED [X,XXX–X,XXX]
STYLE DIRECTION: [from brief]
TARGET COMPLETION: [date]

BUDGET SPLIT (LIOR draft allocation — confirm with contractor):
  Flooring:      ~[X]%
  Paint/finishes: ~[X]%
  Kitchen:       ~[X]%
  Bathrooms:     ~[X]%
  Other:         ~[X]%

Dubai market budget guidance:
  Cosmetic room (paint + flooring + fixtures): AED 8,000–20,000
  Partial room (above + some furniture/fit-out): AED 20,000–50,000
  Full kitchen (strip + new cabs + countertop + appliances): AED 40,000–120,000
  Bathroom wet room (full): AED 25,000–60,000
  (All figures approximate — actual quotes may vary significantly by contractor)
─────────────────────────────────────────────────────────
EOF
echo "Renovation brief parsed."
```

### Step 2 — Before/After Visualizations (Days 1–2)

For each room being renovated: produce the "after" state. For rooms with structural changes (wall removed, kitchen reconfigured), use Firefly since the existing photo no longer represents the post-change space.

#### 2A — Copy before photos

```bash
PROJECT_ID="LIOR-RB2601"
WORKDIR=".tmp/${PROJECT_ID}-reno-brief"

# Copy client input photos to before folder
cp "${WORKDIR}/01-client-inputs/living-current.jpg" "${WORKDIR}/02-visualizations/before/${PROJECT_ID}-before-living.jpg"
cp "${WORKDIR}/01-client-inputs/kitchen-current.jpg" "${WORKDIR}/02-visualizations/before/${PROJECT_ID}-before-kitchen.jpg"
cp "${WORKDIR}/01-client-inputs/master-current.jpg" "${WORKDIR}/02-visualizations/before/${PROJECT_ID}-before-master.jpg"
# ... copy all rooms
echo "Before photos organized."
```

#### 2B — Virtual Staging AI for cosmetic/partial rooms (no structural change)

```bash
IMGBB_KEY=$(grep IMGBB_API_KEY /Users/cashville/.env | cut -d= -f2)
VSTAGING_KEY=$(grep VSTAGING_API_KEY /Users/cashville/.env | cut -d= -f2)
PROJECT_ID="LIOR-RB2601"
WORKDIR=".tmp/${PROJECT_ID}-reno-brief"

ROOM="living"
PHOTO="${WORKDIR}/01-client-inputs/${ROOM}-current.jpg"

# Upload to ImgBB
IMAGE_URL=$(curl -s -X POST "https://api.imgbb.com/1/upload" \
  -F "key=${IMGBB_KEY}" -F "image=@${PHOTO}" \
  | python3 -c "import sys,json; r=json.load(sys.stdin); print(r['data']['url'])")
echo "Hosted: ${IMAGE_URL}"

# Stage "after" state
JOB=$(curl -s -X POST "https://api.virtualstaging.ai/v1/stage" \
  -H "Authorization: Bearer ${VSTAGING_KEY}" \
  -H "Content-Type: application/json" \
  -d "{
    \"image_url\": \"${IMAGE_URL}\",
    \"room_type\": \"${ROOM}_room\",
    \"style\": \"contemporary\",
    \"furnishing_level\": \"medium\",
    \"virtual_tour\": false
  }")

JOB_ID=$(echo $JOB | python3 -c "import sys,json; print(json.load(sys.stdin)['job_id'])")

while true; do
  RESULT=$(curl -s "https://api.virtualstaging.ai/v1/jobs/${JOB_ID}" \
    -H "Authorization: Bearer ${VSTAGING_KEY}")
  STATUS=$(echo $RESULT | python3 -c "import sys,json; print(json.load(sys.stdin)['status'])")
  if [ "$STATUS" = "completed" ]; then
    OUTPUT_URL=$(echo $RESULT | python3 -c "import sys,json; print(json.load(sys.stdin)['output_url'])")
    curl -L "${OUTPUT_URL}" -o "${WORKDIR}/02-visualizations/after/${PROJECT_ID}-after-${ROOM}-v1.jpg"
    echo "After visualization: ${ROOM}"
    break
  elif [ "$STATUS" = "failed" ]; then
    echo "FAILED: ${ROOM} — use Firefly (Step 2C)"
    echo "[$(date '+%Y-%m-%d %H:%M')] FALLBACK vstaging → Firefly for ${ROOM}" >> "${WORKDIR}/log.txt"
    break
  fi
  echo "${ROOM}: ${STATUS} — waiting 10s..."
  sleep 10
done
```

#### 2C — Firefly for rooms with structural changes (or as fallback)

```bash
ADOBE_CLIENT_ID=$(grep ADOBE_CLIENT_ID /Users/cashville/.env | cut -d= -f2)
ADOBE_CLIENT_SECRET=$(grep ADOBE_CLIENT_SECRET /Users/cashville/.env | cut -d= -f2)

FIREFLY_TOKEN=$(curl -s -X POST "https://ims-na1.adobelogin.com/ims/token/v3" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials&client_id=${ADOBE_CLIENT_ID}&client_secret=${ADOBE_CLIENT_SECRET}&scope=openid,AdobeID,firefly_enterprise,firefly_api" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")
echo $FIREFLY_TOKEN > /tmp/firefly_token.txt

# Example: open-plan living-kitchen after wall removal
PROJECT_ID="LIOR-RB2601"
WORKDIR=".tmp/${PROJECT_ID}-reno-brief"
ROOM="kitchen"
# Describe the POST-renovation state precisely in the prompt
PROMPT="open plan living-kitchen, post-renovation interior, new herringbone oak floor, white painted walls, new handleless kitchen cabinetry in warm white, quartz countertop, island with pendant lights, contemporary warm style, photorealistic, editorial architecture photography, Dubai residential, warm afternoon light, wide angle"

RESPONSE=$(curl -s -X POST "https://firefly-api.adobe.io/v3/images/generate" \
  -H "Authorization: Bearer ${FIREFLY_TOKEN}" \
  -H "X-Api-Key: ${ADOBE_CLIENT_ID}" \
  -H "Content-Type: application/json" \
  -d "{
    \"prompt\": \"${PROMPT}\",
    \"negativePrompt\": \"cheap, watermark, cartoon, oversaturated, cold light, old kitchen, before state\",
    \"size\": { \"width\": 2048, \"height\": 1152 },
    \"numVariations\": 3,
    \"contentClass\": \"photo\",
    \"styles\": { \"presets\": [\"photo_realism\"] }
  }")

echo $RESPONSE | python3 -c "
import sys, json
data = json.load(sys.stdin)
for i, o in enumerate(data.get('outputs', [])):
    print(f'Variant {i+1}: {o[\"image\"][\"url\"]}')
"

CHOSEN_URL="[selected URL]"
curl -L "${CHOSEN_URL}" -o "${WORKDIR}/02-visualizations/after/${PROJECT_ID}-after-${ROOM}-v1.jpg"
echo "Firefly after render saved: ${ROOM}"
```

**Firefly fallback: Replicate SDXL**
```bash
REPLICATE_KEY=$(grep REPLICATE_KEY /Users/cashville/.env | cut -d= -f2)
PRED=$(curl -s -X POST "https://api.replicate.com/v1/predictions" \
  -H "Authorization: Token ${REPLICATE_KEY}" \
  -H "Content-Type: application/json" \
  -d "{
    \"version\":\"06d6fae3b75ab68a28cd2900afa6033166910dd09fd9751047043592f8f7cc60\",
    \"input\":{
      \"prompt\":\"${ROOM} room, post-renovation interior, ${CHANGES}, ${STYLE_DIRECTION}, photorealistic, editorial architecture photography, Dubai residential, warm afternoon light\",
      \"negative_prompt\":\"cheap, IKEA, cartoon, watermark, cold grey\",
      \"width\":1920,\"height\":1080,\"num_outputs\":3
    }
  }")
PRED_ID=$(echo $PRED | python3 -c "import sys,json; print(json.load(sys.stdin)['id'])")
while true; do
  R=$(curl -s "https://api.replicate.com/v1/predictions/${PRED_ID}" \
    -H "Authorization: Token ${REPLICATE_KEY}")
  STATUS=$(echo $R | python3 -c "import sys,json; print(json.load(sys.stdin)['status'])")
  [ "$STATUS" = "succeeded" ] && echo $R | python3 -c "import sys,json; [print(u) for u in json.load(sys.stdin)['output']]" && break
  [ "$STATUS" = "failed" ] && echo "Replicate failed" && break
  sleep 8
done
CHOSEN_URL="[selected URL]"
curl -L "${CHOSEN_URL}" -o "${WORKDIR}/02-visualizations/after/${PROJECT_ID}-after-${ROOM}-v1.jpg"
```

#### 2D — Create Before/After Side-by-Side (ImageMagick)

```bash
PROJECT_ID="LIOR-RB2601"
WORKDIR=".tmp/${PROJECT_ID}-reno-brief"

for ROOM in living kitchen master bathroom; do
  BEFORE="${WORKDIR}/02-visualizations/before/${PROJECT_ID}-before-${ROOM}.jpg"
  AFTER="${WORKDIR}/02-visualizations/after/${PROJECT_ID}-after-${ROOM}-v1.jpg"

  if [ -f "${BEFORE}" ] && [ -f "${AFTER}" ]; then
    # Resize both to same height before appending
    magick "${BEFORE}" -resize "x1080" /tmp/before-resized.jpg
    magick "${AFTER}" -resize "x1080" /tmp/after-resized.jpg

    # Side-by-side
    magick +append /tmp/before-resized.jpg /tmp/after-resized.jpg \
      -strip \
      "${WORKDIR}/02-visualizations/before-after/${PROJECT_ID}-before-after-${ROOM}.jpg"

    echo "Before/after created: ${ROOM}"
  else
    echo "Missing before or after for ${ROOM} — skipping before/after composite"
  fi
done
```

**Visualization QA per room:**
- [ ] "After" visualization is consistent with the stated renovation scope — do not show structural changes if scope is cosmetic only
- [ ] Materials in visualization match the brief palette
- [ ] Side-by-side images are at the same height and proportion
- [ ] No artifacts in "after" images
- [ ] "After" looks achievable, not exaggerated

### Step 3 — Scope of Works (Day 2)

A structured, room-by-room works list. This is what a contractor quotes from.

```bash
PROJECT_ID="LIOR-RB2601"
WORKDIR=".tmp/${PROJECT_ID}-reno-brief"

cat > "${WORKDIR}/03-scope/${PROJECT_ID}-scope-of-works-v1.md" << 'EOF'
---
title: "Scope of Works — [PROJECT_ID]"
geometry: "margin=2cm"
fontsize: 11pt
---

# Scope of Works
**Property:** [Address]
**Prepared by:** LIOR
**Date:** [Date]
**Budget envelope:** AED [X,XXX – X,XXX]
**Target completion:** [Date]

---

## Living Room

### To demolish / remove:
- Existing flooring (type: [parquet / tile / carpet]) — full room ~[X]m²
- Existing ceiling light fixture
- [Other removals]

### To supply and install:
- Engineered oak flooring, herringbone, [spec] — approx [X]m²
- Repaint all walls — color: [ref] — 2 coats minimum
- Ceiling-mounted curtain track — [X]m run × [N] curtains
- [Other items]

### Not changing:
- Existing plasterwork (in good condition)
- Window frames
- Electrical sockets (no relocation)

### Notes / flags for contractor:
- Confirm floor substrate before ordering herringbone — may require levelling
- Check north-facing wall for moisture before painting

**Estimated budget allocation: AED [X,XXX – X,XXX]**

---

## Kitchen

### To demolish / remove:
- Existing cabinetry (full kitchen strip)
- Existing tile backsplash
- Existing countertop

### To supply and install:
- New kitchen units — [style: handleless / shaker / slab] — [color]
- Countertop: [material: engineered quartz / marble] — [color ref]
- Backsplash: [material] — [dimensions]
- Island: [dimensions] — [worktop material]
- Appliances: [list: oven / hob / hood / fridge — brand tier]

### Structural note:
⚠ If wall between kitchen and living is to be removed: REQUIRES STRUCTURAL ASSESSMENT before work begins. LIOR cannot confirm load-bearing status — consult structural engineer.

**Estimated budget allocation: AED [X,XXX – X,XXX]**

---

## Master Bedroom

### To demolish / remove:
- [Existing items to remove]

### To supply and install:
- [Items to install]

### Not changing:
- [Items staying]

**Estimated budget allocation: AED [X,XXX – X,XXX]**

---

## Bathroom(s)

### To demolish / remove:
- [Full wet room strip if applicable]
- [Or partial: tiles only / fixtures only]

### To supply and install:
- [Tiles: material, size, format]
- [Fixtures: basin, WC, shower / bath — brand tier]
- [Accessories: towel rails, mirrors, lighting]

**Estimated budget allocation: AED [X,XXX – X,XXX]**

---

[Continue for all rooms in scope]

---

## Total Scope Summary

| Room | Scope level | Estimated allocation | Priority |
|------|------------|---------------------|---------|
| Living room | Cosmetic | AED X,XXX–X,XXX | High |
| Kitchen | Full | AED X,XXX–X,XXX | High |
| Master bedroom | Partial | AED X,XXX–X,XXX | Medium |
| Bathrooms | Partial | AED X,XXX–X,XXX | Medium |
| **TOTAL** | | **AED XX,XXX–XX,XXX** | |

*All allocations are LIOR estimates only. Final costs depend on contractor quotes, material sourcing, and site conditions. Budget envelope must be confirmed with contractor before commitment.*

EOF
echo "Scope of works saved."
```

### Step 4 — Material + Finish Schedule (Day 3)

Full specification of every material decision. Follows SOP-09 Step 5 format. Cover every room in renovation scope. Include Dubai market availability tier for each item.

```bash
PROJECT_ID="LIOR-RB2601"
WORKDIR=".tmp/${PROJECT_ID}-reno-brief"

cat > "${WORKDIR}/04-materials/${PROJECT_ID}-material-schedule-v1.md" << 'EOF'
---
title: "Material Schedule — [PROJECT_ID]"
geometry: "margin=1.5cm"
fontsize: 10pt
---

# Material Schedule — [PROJECT_ID]
Property: [Address]
Style direction: [name]
Budget tier: [entry / mid / high-end]

Dubai market availability tiers:
- Available locally (same week): common tiles, standard paint, basic flooring
- Available Dubai (1–2 week lead): mid-range European brands at Marina Mall / Design District
- Import required (4–8 weeks): specific European/UK brands not stocked locally — flag these

---

## Living Room

### Flooring
| Item | Specification | Reference | Qty | Availability | Notes |
|------|--------------|---------|-----|-------------|-------|
| Engineered oak | Herringbone, 14mm, matte lacquer, warm mid-honey | Kahrs Lapponia Ash or equiv | ~[X]m² | Available Dubai | Confirm underfloor heating compatibility |

### Walls
| Item | Specification | Reference | Coverage | Availability |
|------|--------------|---------|---------|-------------|
| Matte emulsion | Warm off-white, LRV ~58 | F&B Skimming Stone No.241 or equiv | ~[X]m² | Available locally |

### Curtains
| Item | Specification | Color | Track |
|------|--------------|-------|-------|
| Linen pinch-pleat | Floor-to-ceiling | Off-white / écru | Ceiling-mounted recessed |

### Furniture
| Item | Form | Material | Color | Budget ref |
|------|------|---------|-------|-----------|
| Sofa | 3-seat modular, low-back | Performance linen | Warm cream | [Entry: Zara Home / Mid: BoConcept / High: RH] |
| Coffee table | Rectangle 120×60 | Honed travertine top | — | [Supplier] |

---

## Kitchen

| Item | Specification | Reference | Qty | Availability |
|------|--------------|---------|-----|-------------|
| Cabinetry | Handleless slab, warm white | [Brand per budget tier] | [N] units | Available Dubai |
| Countertop | Honed engineered quartz, white vein | [Supplier] | ~[X]m² | Available locally |
| Backsplash | [Tile / slab extension] | [Spec] | ~[X]m² | Available locally |
| Appliances | [Brand tier: Bosch / Siemens / Miele] | [Models] | [List] | Available Dubai |

---

[Continue for all rooms]

EOF
echo "Material schedule saved."
```

### Step 5 — Contractor Brief PDF Assembly (Day 3–4)

Single PDF combining all elements. This is what gets sent to contractors for quotes.

```bash
PROJECT_ID="LIOR-RB2601"
WORKDIR=".tmp/${PROJECT_ID}-reno-brief"

cat > "${WORKDIR}/06-pdf-assembly/contractor-brief.md" << EOF
---
title: "Pre-Renovation Brief — ${PROJECT_ID}"
date: "$(date -u +%B %Y)"
geometry: "margin=2cm"
fontsize: 11pt
---

# Pre-Renovation Brief
**Property:** [Address]
**Client ref:** [Name or ref]
**Prepared by:** LIOR
**Date:** $(date -u +%B %Y)
**Budget envelope:** AED [X]
**Target completion:** [Date]

---

## Project Overview

[Goals summary — 3–5 bullets from renovation brief]

---

## Current State

![Current — Living](${WORKDIR}/02-visualizations/before/${PROJECT_ID}-before-living.jpg){ width=100% }

![Current — Kitchen](${WORKDIR}/02-visualizations/before/${PROJECT_ID}-before-kitchen.jpg){ width=100% }

[Include current state photos for key rooms]

---

## Before / After — Living Room

![Before/After Living](${WORKDIR}/02-visualizations/before-after/${PROJECT_ID}-before-after-living.jpg){ width=100% }

[Scope of works for living room]

---

## Before / After — Kitchen

![Before/After Kitchen](${WORKDIR}/02-visualizations/before-after/${PROJECT_ID}-before-after-kitchen.jpg){ width=100% }

[Scope of works for kitchen]

---

## Before / After — Master Bedroom

![Before/After Master](${WORKDIR}/02-visualizations/before-after/${PROJECT_ID}-before-after-master.jpg){ width=100% }

[Scope of works for master]

---

[Continue for each room]

---

[Insert total scope summary table from scope-of-works.md]

---

[Insert full material schedule from material-schedule.md]

---

*Questions? Contact LIOR — [contact details]*

EOF

pandoc "${WORKDIR}/06-pdf-assembly/contractor-brief.md" \
  -o "${WORKDIR}/05-exports/${PROJECT_ID}-renovation-brief-v1.pdf" \
  --pdf-engine=xelatex \
  -V geometry:margin=2cm \
  -V fontsize=11pt

# Fallback
# pandoc "${WORKDIR}/06-pdf-assembly/contractor-brief.md" \
#   -o "${WORKDIR}/05-exports/${PROJECT_ID}-renovation-brief-v1.pdf" \
#   --pdf-engine=wkhtmltopdf

echo "Contractor brief PDF: ${WORKDIR}/05-exports/${PROJECT_ID}-renovation-brief-v1.pdf"
ls -lh "${WORKDIR}/05-exports/"
```

---

## VI. QA CHECKLIST

Human reviews before delivery:
- [ ] Scope of works is realistic — nothing impossible or contradictory
- [ ] "After" visualizations match the stated scope exactly — no structural changes shown if scope is cosmetic
- [ ] Budget allocations are defensible for Dubai market
- [ ] No structural items specified by LIOR — all structural items flagged for engineer sign-off
- [ ] Material specs: all references real and available in Dubai market
- [ ] No items flagged as "import required (4–8 weeks)" presented as quick-win items
- [ ] Before/after side-by-side images are at matching height/proportion
- [ ] PDF opens correctly — no broken image links, no placeholder text
- [ ] Correct project name, address, date on cover

---

## VII. DELIVERY

### ZIP

```bash
PROJECT_ID="LIOR-RB2601"
WORKDIR=".tmp/${PROJECT_ID}-reno-brief"

cd "${WORKDIR}"
zip -r "${PROJECT_ID}-renovation-brief-v1.zip" "05-exports/"
echo "ZIP size: $(wc -c < "${PROJECT_ID}-renovation-brief-v1.zip") bytes"
unzip -l "${PROJECT_ID}-renovation-brief-v1.zip"
```

### WeTransfer Upload

```bash
WT_KEY=$(grep WETRANSFER_API_KEY /Users/cashville/.env | cut -d= -f2)
ZIP_FILE="${WORKDIR}/${PROJECT_ID}-renovation-brief-v1.zip"
ZIP_SIZE=$(wc -c < "${ZIP_FILE}")

TRANSFER=$(curl -s -X POST "https://dev.wetransfer.com/v2/transfers" \
  -H "Content-Type: application/json" -H "x-api-key: ${WT_KEY}" \
  -d "{\"message\":\"${PROJECT_ID} — Pre-Renovation Brief LIOR\",\"files\":[{\"name\":\"${PROJECT_ID}-renovation-brief-v1.zip\",\"size\":${ZIP_SIZE}}]}")

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

MSG="${CLIENT_NAME} — your Pre-Renovation Brief is ready.

The document includes:
- Before/after visualizations for all ${N_ROOMS} rooms
- Room-by-room scope of works
- Full material and finish specification
- Estimated budget allocation per room

Share this directly with your contractor or architect to get accurate quotes. Everything they need is in one document.

Download (valid 7 days): ${DOWNLOAD_URL}

Let us know if any scope item needs adjustment before you send it out."

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
echo "[$(date '+%Y-%m-%d %H:%M')] DELIVERED: ${PROJECT_ID} | Service: renovation-brief | ZIP: ${PROJECT_ID}-renovation-brief-v1.zip | Link: ${DOWNLOAD_URL} | WA: sent" \
  >> "${WORKDIR}/log.txt"
```

---

## VIII. WHATSAPP TEMPLATE

```
[CLIENT NAME] — your Pre-Renovation Brief is ready.

The document includes:
- Before/after visualizations for all [N] rooms
- Room-by-room scope of works
- Full material and finish specification
- Estimated budget allocation per room

Share this directly with your contractor or architect to get accurate quotes. Everything they need is in one document.

Download (valid 7 days): [LINK]

Let us know if any scope item needs adjustment before you send it out.
```

Rules: no "AI", no "luxury", no prices (except actual budget allocations), no "drone".

---

## IX. FALLBACKS

| Tool | Failure | Fix |
|---|---|---|
| Virtual Staging AI | API down or quota exhausted | Wait 30 min. Try REimagineHome: `POST https://api.reimaginehome.ai/v1/generate`. Both down → Firefly (Step 2C). |
| REimagineHome | Also down | Use Firefly render |
| Adobe Firefly | Token expired | Re-run token script in Step 2C. Valid 24h. |
| Firefly | API down | Replicate SDXL: `POST https://api.replicate.com/v1/predictions` model `06d6fae3b75ab68a28cd2900afa6033166910dd09fd9751047043592f8f7cc60` |
| ImageMagick | Not installed | `brew install imagemagick` |
| Pandoc PDF | xelatex font error | `cp [font.ttf] ~/Library/Fonts/ && fc-cache -fv`. Or `--pdf-engine=wkhtmltopdf` |
| WeTransfer API | Upload fails | Manual upload at wetransfer.com |
| WeTransfer | File >2GB | Split: PDF + before/after images separately |
| WeTransfer | Down | Google Drive — share "anyone with link can view" |
| CallMeBot | Phone not registered | Send "I allow callmebot to send me messages" to +34 644 29 73 73 |
| ImgBB | Upload fails | Retry after 30s. 32MB max. |

---

## X. OPTIONAL CONTRACTOR FOLLOW-UP

If client asks LIOR to send brief directly to contractors (human approves list first):

```
Subject: Renovation Brief — [Property Address] — Quote Request

Hi [Contractor name],

Contacting you on behalf of [client name] regarding a residential renovation at [property address], Dubai.

Attached is a full renovation brief prepared by LIOR, covering scope of works, material specifications, and design intent for all rooms.

Scope summary: [2 lines from brief]
Budget envelope: AED [X]
Target start: [date]

Please review and share your quote. [Client name] is available to discuss: [client WhatsApp or email]

LIOR Studio
```

---

## XI. LOG FORMAT

```
[YYYY-MM-DD HH:MM] STARTED: [PROJECT_ID] | rooms=[N] | budget=AED [X] | scope=[cosmetic/partial/full]
[YYYY-MM-DD HH:MM] BEFORE PHOTOS: organized — [N] rooms
[YYYY-MM-DD HH:MM] VISUALIZATION: [room] — after via [vstaging / Firefly / Replicate]
[YYYY-MM-DD HH:MM] BEFORE/AFTER: side-by-side created for [room]
[YYYY-MM-DD HH:MM] SCOPE: written — [N] rooms
[YYYY-MM-DD HH:MM] MATERIALS: schedule written — [N] rooms
[YYYY-MM-DD HH:MM] PDF: generated — [filename]
[YYYY-MM-DD HH:MM] FALLBACK: [tool] → [alternative] — [room] — [reason]
[YYYY-MM-DD HH:MM] QA: APPROVED by [name]
[YYYY-MM-DD HH:MM] DELIVERED: [PROJECT_ID] | ZIP: [filename] | Link: [URL] | WA: sent
```
