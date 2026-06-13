# SOP 04 — Interior Design Brief
**Automation level: C — Guided (agent handles production, human leads style session)**
**Timeline: 4 days from brief session**
**Deliverable: Moodboard · space plan (2 layout options) · key room visualizations · complete material + furnishing schedule · assembled PDF**

## I. OVERVIEW

Client has a property (existing or recently purchased) and needs a full design direction before making any purchase or renovation decision. Agent sends the questionnaire, receives and parses the answers, produces the moodboard, space plans, room visualizations, and material schedule, then assembles everything into one contractor-ready PDF. Human runs the style session and approves the output before delivery.

---

## II. INPUTS

| # | Input | Format | Source | Blocker? |
|---|-------|--------|--------|---------|
| 1 | Property photos (all rooms) | JPG ≥ 1920×1080 | Client | YES |
| 2 | Floor plan | PDF or JPG | Client | Preferred — derive from photos if absent |
| 3 | Style brief questionnaire responses | Text/form answers | Human-led session | YES — core of this service |
| 4 | Budget range | Entry / mid / high-end | Intake form | YES |
| 5 | Move-in / project timeline | Date or timeframe | Intake form | YES |
| 6 | Project ID | `LIOR-DB[CODE][YY][SEQ]` | Agent | Internal |

---

## III. SETUP (one-time)

```bash
# Install dependencies
brew install ffmpeg imagemagick pandoc
brew install --cask mactex   # PDF engine (LaTeX)
# Alternative PDF engine if LaTeX unavailable:
brew install wkhtmltopdf

# Verify .env keys
grep -E "IMGBB_API_KEY|VSTAGING_API_KEY|ADOBE_CLIENT_ID|ADOBE_CLIENT_SECRET|WETRANSFER_API_KEY|CALLMEBOT_PHONE|CALLMEBOT_API_KEY|NOTION_TOKEN" /Users/cashville/.env

# Keys needed:
# IMGBB_API_KEY          — image hosting for API calls
# VSTAGING_API_KEY       — Virtual Staging AI (room visualizations)
# ADOBE_CLIENT_ID        — Firefly (moodboard generation + fallback renders)
# ADOBE_CLIENT_SECRET    — Firefly
# WETRANSFER_API_KEY     — delivery
# CALLMEBOT_PHONE        — WhatsApp
# CALLMEBOT_API_KEY      — WhatsApp
# NOTION_TOKEN           — project tracking
```

---

## IV. FOLDER STRUCTURE

```bash
PROJECT_ID="LIOR-DB2601"   # replace per project
mkdir -p .tmp/${PROJECT_ID}-design-brief/{00-brief,01-client-inputs,02-moodboard,03-space-plan,04-visualizations,05-material-schedule,06-exports,07-pdf-assembly}
touch .tmp/${PROJECT_ID}-design-brief/log.txt
```

---

## V. PRODUCTION STEPS

### Step 1 — Send Questionnaire (Day 0)

Agent sends via WhatsApp:

```bash
PHONE=$(grep CALLMEBOT_PHONE /Users/cashville/.env | cut -d= -f2)
APIKEY=$(grep CALLMEBOT_API_KEY /Users/cashville/.env | cut -d= -f2)
CLIENT_NAME="[CLIENT NAME]"
QUESTIONNAIRE_URL="https://[lior-domain]/questionnaire.html"

MSG="${CLIENT_NAME} — before we start your Interior Design Brief, we need 15 minutes of your input. This shapes everything — the more precise you are, the more precisely we can design your space.

Complete here: ${QUESTIONNAIRE_URL}"

ENCODED=$(python3 -c "import urllib.parse,sys; print(urllib.parse.quote(sys.argv[1]))" "${MSG}")
curl -s "https://api.callmebot.com/whatsapp.php?phone=${PHONE}&text=${ENCODED}&apikey=${APIKEY}"
echo "Questionnaire sent."
```

**Questionnaire fields (agent parses responses into brief):**

```
1. How do you intend to use this property?
   Primary residence / Holiday home / Investment / rental / Other

2. Who will live here?
   Free text

3. Three words that describe how you want the space to feel:
   Free text

4. What do you want to avoid at all costs?
   Free text (e.g. dark rooms / cold colours / excessive furniture)

5. Reference images (attach 3–5 images you love):
   File upload or Pinterest/Instagram links

6. Preferred style:
   Modern Minimal / Contemporary Warm / Curated Luxury /
   Organic Natural / Classic Contemporary / Let LIOR decide

7. Flooring preference:
   Wood / Stone-tile / Marble / Keep existing / No preference

8. Kitchen preference:
   Open and social / Separate / No preference

9. Budget tier for furnishings:
   Entry (functional, clean) / Mid-range (quality + design) / High-end (investment pieces)

10. Anything specific about the space that bothers you currently?
    Free text
```

### Step 2 — Parse Responses into Brief (Day 1)

```bash
PROJECT_ID="LIOR-DB2601"
WORKDIR=".tmp/${PROJECT_ID}-design-brief"

cat > "${WORKDIR}/00-brief/00-design-brief.txt" << 'EOF'
PROJECT BRIEF — [PROJECT_ID]
Date: [date]
Intention: [primary / rental / holiday]
Occupants: [description]
Feel keywords: [3 words from Q3]
Avoid: [list from Q4]
Style direction: [selected or LIOR-assigned]
Flooring: [preference from Q7]
Kitchen: [preference from Q8]
Budget tier: [entry / mid / high-end]
Pain points: [list from Q10]
Reference images: [URLs or filenames from Q5]
Rooms to cover: [all rooms from property photos]
EOF
```

### Step 3 — Moodboard (Day 1)

#### 3A — Get Firefly Token

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

#### 3B — Generate Moodboard with Firefly

```bash
FIREFLY_TOKEN=$(cat /tmp/firefly_token.txt)
ADOBE_CLIENT_ID=$(grep ADOBE_CLIENT_ID /Users/cashville/.env | cut -d= -f2)
PROJECT_ID="LIOR-DB2601"
WORKDIR=".tmp/${PROJECT_ID}-design-brief"

# Adjust style direction and palette from brief
STYLE_DIRECTION="contemporary warm"
PALETTE="cream, warm oak, brushed brass, warm stone, off-white"
KEY_MATERIALS="engineered oak floor, linen sofa, travertine surfaces, brass accents"

RESPONSE=$(curl -s -X POST "https://firefly-api.adobe.io/v3/images/generate" \
  -H "Authorization: Bearer ${FIREFLY_TOKEN}" \
  -H "X-Api-Key: ${ADOBE_CLIENT_ID}" \
  -H "Content-Type: application/json" \
  -d "{
    \"prompt\": \"interior design moodboard, ${STYLE_DIRECTION} Dubai apartment, palette ${PALETTE}, materials ${KEY_MATERIALS}, flat lay composition with material swatches and furniture references, editorial lifestyle photography, warm afternoon light\",
    \"negativePrompt\": \"cheap, IKEA, cartoon, oversaturated, cold light, watermark, cluttered\",
    \"size\": { \"width\": 2048, \"height\": 1152 },
    \"numVariations\": 3,
    \"contentClass\": \"photo\",
    \"styles\": { \"presets\": [\"photo_realism\"] }
  }")

# Print output URLs to pick the best
echo $RESPONSE | python3 -c "
import sys, json
data = json.load(sys.stdin)
for i, o in enumerate(data.get('outputs', [])):
    print(f'Variant {i+1}: {o[\"image\"][\"url\"]}')
"

# Download chosen variant (replace with selected URL)
CHOSEN_URL="[selected URL]"
curl -L "${CHOSEN_URL}" -o "${WORKDIR}/02-moodboard/${PROJECT_ID}-moodboard-v1.jpg"
echo "Moodboard saved."
```

**Fallback: Replicate SDXL**
```bash
REPLICATE_KEY=$(grep REPLICATE_KEY /Users/cashville/.env | cut -d= -f2)
PRED=$(curl -s -X POST "https://api.replicate.com/v1/predictions" \
  -H "Authorization: Token ${REPLICATE_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "version":"06d6fae3b75ab68a28cd2900afa6033166910dd09fd9751047043592f8f7cc60",
    "input":{
      "prompt":"interior design moodboard, [style direction], palette [colors], [key materials], flat lay composition, material swatches and furniture samples, editorial lifestyle photography, Dubai apartment, warm light",
      "negative_prompt":"cheap, IKEA, cartoon, watermark, cold grey, generic",
      "width":1920,"height":1080,"num_outputs":3
    }
  }')
PRED_ID=$(echo $PRED | python3 -c "import sys,json; print(json.load(sys.stdin)['id'])")
while true; do
  R=$(curl -s "https://api.replicate.com/v1/predictions/${PRED_ID}" \
    -H "Authorization: Token ${REPLICATE_KEY}")
  STATUS=$(echo $R | python3 -c "import sys,json; print(json.load(sys.stdin)['status'])")
  [ "$STATUS" = "succeeded" ] && echo $R | python3 -c "import sys,json; [print(u) for u in json.load(sys.stdin)['output']]" && break
  [ "$STATUS" = "failed" ] && echo "Replicate failed" && break
  sleep 8
done
# Download chosen variant
CHOSEN_URL="[selected URL]"
curl -L "${CHOSEN_URL}" -o "${WORKDIR}/02-moodboard/${PROJECT_ID}-moodboard-v1.jpg"
```

**Moodboard must convey:**
- Color palette (5 swatches)
- Primary flooring material
- Wall treatment
- Key furniture character
- Textiles (sofa, bed linen, curtains)
- Accent material (brass / stone / ceramic)

### Step 4 — Space Plan (Day 1–2)

Create two layout options using RoomSketcher web UI (no API required).

**RoomSketcher Web UI workflow:**
```
1. Go to https://roomsketcher.com → Sign In
2. Click "New Floor Plan"
3. Enter room dimensions from floor plan (length × width in m)
4. Use "Draw Walls" tool to trace the layout
5. Add doors and windows from the left panel
6. Go to "Furnish" tab → drag and drop furniture:
   - Living: sofa facing main light source / focal point, coffee table, rug
   - Bedroom: bed with headboard against wall, 70cm circulation on both sides
   - Kitchen: ensure island or dining table does not block walkway (90cm min corridor)
   - Dining: table centered under pendant zone
7. Save as "Layout Option A"
8. Duplicate the project → modify for Option B:
   - Option A: optimizes natural light and view
   - Option B: optimizes social flow and usability
9. Go to "3D View" to verify proportions look correct
10. File → Export → Floor Plan → JPG (minimum 2000px, show dimensions ON)
```

```bash
# After export from RoomSketcher, move to project folder
mv ~/Downloads/layout-option-A.jpg "${WORKDIR}/03-space-plan/${PROJECT_ID}-layout-A-v1.jpg"
mv ~/Downloads/layout-option-B.jpg "${WORKDIR}/03-space-plan/${PROJECT_ID}-layout-B-v1.jpg"
```

**Furniture placement rules:**
- Sofa facing main light source or focal point (fireplace / TV / view)
- Dining table centered under pendant zone
- Bed: headboard against wall, 70cm circulation on both sides
- No furniture blocking walkways or doors
- Traffic flow: min 90cm corridor clearance in all rooms

For each layout, write 3–5 bullet points explaining the design rationale.

### Step 5 — Key Room Visualizations (Day 2–3)

Visualize 3 key rooms: living + master + kitchen (or client-specified).

#### 5A — Upload room photo to ImgBB

```bash
IMGBB_KEY=$(grep IMGBB_API_KEY /Users/cashville/.env | cut -d= -f2)
PROJECT_ID="LIOR-DB2601"
WORKDIR=".tmp/${PROJECT_ID}-design-brief"

PHOTO="${WORKDIR}/01-client-inputs/living-empty.jpg"
IMAGE_URL=$(curl -s -X POST "https://api.imgbb.com/1/upload" \
  -F "key=${IMGBB_KEY}" \
  -F "image=@${PHOTO}" \
  | python3 -c "import sys,json; r=json.load(sys.stdin); print(r['data']['url'])")
echo "Hosted: ${IMAGE_URL}"
```

#### 5B — Virtual Staging AI (primary — for rooms with usable photos)

```bash
VSTAGING_KEY=$(grep VSTAGING_API_KEY /Users/cashville/.env | cut -d= -f2)
ROOM="living"
STYLE="contemporary"   # modern | contemporary | minimalist | scandinavian | luxury | traditional
BUDGET_DENSITY="medium"  # low=entry, medium=mid, high=high-end

JOB=$(curl -s -X POST "https://api.virtualstaging.ai/v1/stage" \
  -H "Authorization: Bearer ${VSTAGING_KEY}" \
  -H "Content-Type: application/json" \
  -d "{
    \"image_url\": \"${IMAGE_URL}\",
    \"room_type\": \"${ROOM}_room\",
    \"style\": \"${STYLE}\",
    \"furnishing_level\": \"${BUDGET_DENSITY}\",
    \"virtual_tour\": false
  }")

JOB_ID=$(echo $JOB | python3 -c "import sys,json; print(json.load(sys.stdin)['job_id'])")
echo "Job ID: ${JOB_ID}"

# Poll for result
while true; do
  RESULT=$(curl -s -X GET "https://api.virtualstaging.ai/v1/jobs/${JOB_ID}" \
    -H "Authorization: Bearer ${VSTAGING_KEY}")
  STATUS=$(echo $RESULT | python3 -c "import sys,json; print(json.load(sys.stdin)['status'])")
  if [ "$STATUS" = "completed" ]; then
    OUTPUT_URL=$(echo $RESULT | python3 -c "import sys,json; print(json.load(sys.stdin)['output_url'])")
    curl -L "${OUTPUT_URL}" -o "${WORKDIR}/04-visualizations/${PROJECT_ID}-viz-${ROOM}-v1.jpg"
    echo "Staged: ${ROOM}"
    break
  elif [ "$STATUS" = "failed" ]; then
    echo "FAILED: ${ROOM}"
    break
  fi
  echo "Status: ${STATUS} — waiting 10s..."
  sleep 10
done
```

**Room type values:** `living_room` · `bedroom` · `kitchen` · `dining_room` · `bathroom` · `office`
**Style values:** `modern` · `contemporary` · `minimalist` · `scandinavian` · `luxury` · `traditional`

#### 5C — Fallback (no usable photo): Firefly render

```bash
FIREFLY_TOKEN=$(cat /tmp/firefly_token.txt)
ADOBE_CLIENT_ID=$(grep ADOBE_CLIENT_ID /Users/cashville/.env | cut -d= -f2)
ROOM="living"
PROMPT="living room, contemporary warm interior design, warm white walls, oak herringbone floor, linen sofa, travertine coffee table, warm afternoon light, wide angle from corner, Dubai apartment, editorial interior photography, photorealistic, 8k"

RESPONSE=$(curl -s -X POST "https://firefly-api.adobe.io/v3/images/generate" \
  -H "Authorization: Bearer ${FIREFLY_TOKEN}" \
  -H "X-Api-Key: ${ADOBE_CLIENT_ID}" \
  -H "Content-Type: application/json" \
  -d "{
    \"prompt\": \"${PROMPT}\",
    \"negativePrompt\": \"cheap, IKEA, cartoon, oversaturated, cold light, plastic, watermark\",
    \"size\": { \"width\": 2048, \"height\": 1152 },
    \"numVariations\": 3,
    \"contentClass\": \"photo\",
    \"styles\": { \"presets\": [\"photo_realism\"] }
  }")

CHOSEN_URL=$(echo $RESPONSE | python3 -c "import sys,json; print(json.load(sys.stdin)['outputs'][0]['image']['url'])")
curl -L "${CHOSEN_URL}" -o "${WORKDIR}/04-visualizations/${PROJECT_ID}-viz-${ROOM}-v1.jpg"
```

**Visualization QA per room:**
- [ ] No furniture floating or clipping
- [ ] No artifacts or unnatural shadows
- [ ] Palette consistent with moodboard direction
- [ ] Room proportions look realistic
- [ ] Lighting is warm, not cold or flat

### Step 6 — Material + Furnishing Schedule (Day 3)

Write the material schedule in Markdown. One section per room covering all rooms in scope.

```bash
PROJECT_ID="LIOR-DB2601"
WORKDIR=".tmp/${PROJECT_ID}-design-brief"

cat > "${WORKDIR}/05-material-schedule/${PROJECT_ID}-material-schedule-v1.md" << 'EOF'
---
title: "Interior Design Brief — Material Schedule"
date: "[Date]"
geometry: "margin=2cm"
fontsize: 11pt
---

# Material + Furnishing Schedule — [PROJECT_ID]
Property: [Address]
Style direction: [name]

---

## Living Room

### Floor
- Material: Natural oak, herringbone pattern, 14mm engineered
- Finish: Matte lacquer, pre-finished
- Color direction: Warm mid-honey (no grey undertone)
- Reference: Kahrs "Lapponia Ash" or equivalent

### Walls
- Finish: Matte emulsion
- Color: Farrow & Ball "Skimming Stone" No. 241 or equivalent warm off-white (LRV ~58)
- Feature wall: None at this tier

### Ceiling
- Finish: Brilliant white matte
- Height: Existing — no change

### Curtains
- Style: Linen, pinch-pleat, floor-to-ceiling
- Color: Off-white / écru (no pattern)
- Track: Ceiling-mounted, recessed

### Sofa
- Form: 3-seat modular, low-back
- Material: Performance linen or bouclé
- Color: Warm cream / stone
- Reference: [Restoration Hardware / BoConcept / Zara Home — per budget tier]

### Coffee Table
- Material: Honed travertine top
- Form: Rectangle 120cm × 60cm

### Key Lighting
- Floor lamp: Arc design, brushed brass, white linen shade
- Recessed: Warm white 2700K, wide-beam downlights
- Pendant: None in living — use floor lamp + recessed

---

## Master Bedroom

### Floor
- Material: [Same as living or carpet — per brief]

### Walls
- Color: [Slightly warmer / same base]

### Bed
- Form: Upholstered, king size
- Headboard: Tall, fabric, curved or rectangular
- Material: Boucle or performance fabric
- Color: Warm stone / muted sage / cream

### Bedside Tables
- Material: Solid oak or lacquered finish
- Style: Minimal, 1 drawer

### Lighting
- Bedside: Wall-mounted reading arms (brushed brass)
- Overhead: Recessed 2700K

---

## Kitchen

### Cabinetry
- Style: [Handleless slab / Shaker — per brief]
- Color: [Warm white / sage / mid-tone grey]

### Countertop
- Material: [Engineered quartz or honed marble]
- Color: [White vein / warm grey vein]

### Backsplash
- Material: [Subway tile / metro tile / full slab extension of countertop]

### Hardware
- Finish: Brushed brass or matte black

### Appliances
- Tier: [Entry: Bosch standard / Mid: Siemens / High: Miele / V-Zug]

---

[Continue for all rooms in scope: bathrooms, hallway, etc.]

EOF
echo "Material schedule draft saved."
```

### Step 7 — PDF Assembly (Day 3–4)

Compile all sections into a single deliverable PDF.

#### 7A — Prepare image references in Markdown

```bash
PROJECT_ID="LIOR-DB2601"
WORKDIR=".tmp/${PROJECT_ID}-design-brief"

cat > "${WORKDIR}/07-pdf-assembly/brief-content.md" << 'EOF'
---
title: "Interior Design Brief — [PROJECT_ID]"
date: "[Date]"
geometry: "margin=2cm"
fontsize: 11pt
---

# Interior Design Brief
**Property:** [Address]
**Client:** [Name]
**Prepared by:** LIOR
**Date:** [Date]

---

## Brief Summary

**Feel:** [3 keywords from questionnaire]
**Priorities:** [top 3 from brief]
**Constraints:** [avoidances from questionnaire]
**Style direction:** [name + one-line description]

---

## Moodboard

![Moodboard](.tmp/[PROJECT_ID]-design-brief/02-moodboard/[PROJECT_ID]-moodboard-v1.jpg){ width=100% }

---

## Layout Options

**Option A — [name]:** [rationale 3–5 bullets]

![Layout A](.tmp/[PROJECT_ID]-design-brief/03-space-plan/[PROJECT_ID]-layout-A-v1.jpg){ width=100% }

**Option B — [name]:** [rationale 3–5 bullets]

![Layout B](.tmp/[PROJECT_ID]-design-brief/03-space-plan/[PROJECT_ID]-layout-B-v1.jpg){ width=100% }

---

## Room Visualizations

### Living Room
![Living Room](.tmp/[PROJECT_ID]-design-brief/04-visualizations/[PROJECT_ID]-viz-living-v1.jpg){ width=100% }

### Master Bedroom
![Master Bedroom](.tmp/[PROJECT_ID]-design-brief/04-visualizations/[PROJECT_ID]-viz-master-v1.jpg){ width=100% }

### Kitchen
![Kitchen](.tmp/[PROJECT_ID]-design-brief/04-visualizations/[PROJECT_ID]-viz-kitchen-v1.jpg){ width=100% }

---

EOF
# Append material schedule inline
cat "${WORKDIR}/05-material-schedule/${PROJECT_ID}-material-schedule-v1.md" >> "${WORKDIR}/07-pdf-assembly/brief-content.md"

echo "PDF source assembled."
```

#### 7B — Generate PDF

```bash
# Primary: xelatex
pandoc "${WORKDIR}/07-pdf-assembly/brief-content.md" \
  -o "${WORKDIR}/06-exports/${PROJECT_ID}-design-brief-v1.pdf" \
  --pdf-engine=xelatex \
  -V geometry:margin=2cm \
  -V fontsize=11pt \
  --highlight-style=tango

# If xelatex fails (font not found): install fonts first
# cp [font.ttf] ~/Library/Fonts/ && fc-cache -fv
# Then retry above command.

# Fallback: wkhtmltopdf
# pandoc "${WORKDIR}/07-pdf-assembly/brief-content.md" \
#   -o "${WORKDIR}/06-exports/${PROJECT_ID}-design-brief-v1.pdf" \
#   --pdf-engine=wkhtmltopdf

echo "PDF generated: ${WORKDIR}/06-exports/${PROJECT_ID}-design-brief-v1.pdf"
ls -lh "${WORKDIR}/06-exports/"
```

---

## VI. QA CHECKLIST

Human verifies before delivery:
- [ ] Moodboard colors are cohesive — no clashing elements
- [ ] Palette reads as warm overall, consistent with brief
- [ ] Room visualizations match the moodboard direction (same palette, materials)
- [ ] Space plan furniture placement is realistic — no blocking, correct clearances
- [ ] Material schedule: all references are real and available in Dubai market
- [ ] No invented product SKUs
- [ ] PDF opens correctly — no broken image links, no placeholder text
- [ ] PDF is client-presentable (correct branding, no draft watermarks)
- [ ] Correct name, address, date on cover

---

## VII. DELIVERY

### ZIP

```bash
PROJECT_ID="LIOR-DB2601"
WORKDIR=".tmp/${PROJECT_ID}-design-brief"

cd "${WORKDIR}"
zip -r "${PROJECT_ID}-design-brief-v1.zip" "06-exports/"
echo "ZIP size: $(wc -c < "${PROJECT_ID}-design-brief-v1.zip") bytes"
unzip -l "${PROJECT_ID}-design-brief-v1.zip"
```

### WeTransfer Upload

```bash
WT_KEY=$(grep WETRANSFER_API_KEY /Users/cashville/.env | cut -d= -f2)
ZIP_FILE="${WORKDIR}/${PROJECT_ID}-design-brief-v1.zip"
ZIP_SIZE=$(wc -c < "${ZIP_FILE}")

TRANSFER=$(curl -s -X POST "https://dev.wetransfer.com/v2/transfers" \
  -H "Content-Type: application/json" -H "x-api-key: ${WT_KEY}" \
  -d "{\"message\":\"${PROJECT_ID} — Interior Design Brief LIOR\",\"files\":[{\"name\":\"${PROJECT_ID}-design-brief-v1.zip\",\"size\":${ZIP_SIZE}}]}")

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
N_ROOMS="3"

MSG="${CLIENT_NAME} — your Interior Design Brief is ready.

The document covers:
- Style direction and moodboard
- 2 layout options for your space
- Visualizations of ${N_ROOMS} key rooms
- Full material and furnishing schedule

Download (valid 7 days): ${DOWNLOAD_URL}

This document is ready to use — share it with any contractor, architect, or furniture supplier. Let us know if you would like to adjust any direction or explore specific rooms further."

ENCODED=$(python3 -c "import urllib.parse,sys; print(urllib.parse.quote(sys.argv[1]))" "${MSG}")
curl -s "https://api.callmebot.com/whatsapp.php?phone=${PHONE}&text=${ENCODED}&apikey=${APIKEY}"
echo "WhatsApp sent."
```

### Notion Update

```bash
NOTION_TOKEN=$(grep NOTION_TOKEN /Users/cashville/.env | cut -d= -f2)
PAGE_ID="[ID_NOTION — from project Notion page]"

curl -s -X PATCH "https://api.notion.com/v1/pages/${PAGE_ID}" \
  -H "Authorization: Bearer ${NOTION_TOKEN}" \
  -H "Notion-Version: 2022-06-28" \
  -H "Content-Type: application/json" \
  -d "{\"properties\":{\"Status\":{\"select\":{\"name\":\"Delivered\"}},\"Delivery Link\":{\"url\":\"${DOWNLOAD_URL}\"},\"Delivered At\":{\"date\":{\"start\":\"$(date -u +%Y-%m-%d)\"}}}}"

echo "Notion updated."
```

### Delivery Log

```bash
echo "[$(date '+%Y-%m-%d %H:%M')] DELIVERED: ${PROJECT_ID} | Service: design-brief | ZIP: ${PROJECT_ID}-design-brief-v1.zip | Link: ${DOWNLOAD_URL} | WA: sent" \
  >> "${WORKDIR}/log.txt"
```

---

## VIII. WHATSAPP TEMPLATE

```
[CLIENT NAME] — your Interior Design Brief is ready.

The document covers:
- Style direction and moodboard
- 2 layout options for your space
- Visualizations of [N] key rooms
- Full material and furnishing schedule

Download (valid 7 days): [LINK]

This document is ready to use — share it with any contractor, architect, or furniture supplier. Let us know if you would like to adjust any direction or explore specific rooms further.
```

Rules: no "AI", no "luxury", no prices, no "drone".

---

## IX. FALLBACKS

| Tool | Failure | Fix |
|---|---|---|
| Virtual Staging AI | API down or quota exhausted | Wait 30 min. If still down → use REimagineHome API: `POST https://api.reimaginehome.ai/v1/generate` with same params. If both down → use Firefly (Step 5C). |
| REimagineHome | Also unavailable | Use Firefly render (Step 5C) |
| Adobe Firefly | Token expired | Re-run token script (Step 3A). Valid 24h. |
| Firefly | API down | Use Replicate SDXL: `POST https://api.replicate.com/v1/predictions` with model hash `06d6fae3b75ab68a28cd2900afa6033166910dd09fd9751047043592f8f7cc60` |
| Pandoc PDF | Font not found (xelatex) | `cp [font.ttf] ~/Library/Fonts/ && fc-cache -fv` then retry. Or switch to `--pdf-engine=wkhtmltopdf` |
| Pandoc PDF | wkhtmltopdf not installed | `brew install wkhtmltopdf` |
| WeTransfer API | Upload fails | Manual upload at wetransfer.com |
| WeTransfer API | File >2GB | Split: PDF in one ZIP, images in another |
| WeTransfer | Completely down | Upload to Google Drive, share "anyone with link", send that link |
| CallMeBot | Phone not registered | Send "I allow callmebot to send me messages" to +34 644 29 73 73 on WhatsApp |
| CallMeBot | Rate limit | Wait 5s between messages |
| ImgBB | Upload fails | Retry after 30s. 32MB max. |

---

## X. LOG FORMAT

```
[YYYY-MM-DD HH:MM] STARTED: [PROJECT_ID] | rooms=[N] | budget=[tier]
[YYYY-MM-DD HH:MM] QUESTIONNAIRE: sent to [client] via WhatsApp
[YYYY-MM-DD HH:MM] BRIEF: parsed and saved to 00-brief/
[YYYY-MM-DD HH:MM] MOODBOARD: generated and saved
[YYYY-MM-DD HH:MM] SPACE PLAN: Option A + B exported from RoomSketcher
[YYYY-MM-DD HH:MM] VISUALIZATION: living — completed
[YYYY-MM-DD HH:MM] VISUALIZATION: master — completed
[YYYY-MM-DD HH:MM] VISUALIZATION: kitchen — completed
[YYYY-MM-DD HH:MM] FALLBACK: [tool] → [alternative] — [reason]
[YYYY-MM-DD HH:MM] PDF: generated — [filename]
[YYYY-MM-DD HH:MM] QA: APPROVED by [name]
[YYYY-MM-DD HH:MM] DELIVERED: [PROJECT_ID] | ZIP: [filename] | Link: [URL] | WA: sent
```

---

## UPSELL TRIGGER

48h after delivery, agent sends:

```
[CLIENT NAME] — glad the brief is working well for you.
If you would like to take it further — every room fully designed, project coordination,
contractor management from brief to handover — that is exactly what our full design service covers.
Worth a conversation?
```

→ Leads to SOP-05 (Full Interior Design).
