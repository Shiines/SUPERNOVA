# SOP 09 — Design Concept
**Automation level: B — Auto + human gate**
**Timeline: 4 days from brief confirmation**
**Deliverable: Moodboard · key room visualizations · complete material + finish specification · 10-page concept book PDF — contractor-ready**

## I. OVERVIEW

Full creative direction for a space, resolved before any contractor is briefed. Client brief in → moodboard + room visualizations + complete material schedule → one assembled concept book PDF that can be handed directly to any designer, builder, or fit-out team. Agent runs the full pipeline; human gates the output before delivery.

---

## II. INPUTS

| # | Input | Format | Source | Blocker? |
|---|-------|--------|--------|---------|
| 1 | Property photos (all rooms, current state) | JPG ≥ 1920×1080 | Client | YES |
| 2 | Floor plan (existing) | PDF, JPG, or sketch | Client | YES |
| 3 | Style brief + questionnaire responses | Text + reference images | Intake form | YES |
| 4 | Budget tier | Entry / Mid / High-end | Intake form | YES |
| 5 | Rooms in scope | List | Intake form | YES |
| 6 | Target outcome | Sell / rent / live / redesign | Intake form | YES |
| 7 | Project ID | `LIOR-DC[CODE][YY][SEQ]` | Agent | Internal |

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
# IMGBB_API_KEY          — image hosting for API calls
# VSTAGING_API_KEY       — Virtual Staging AI (room visualizations)
# ADOBE_CLIENT_ID        — Firefly (moodboard + fallback renders)
# ADOBE_CLIENT_SECRET    — Firefly
# WETRANSFER_API_KEY     — delivery
# CALLMEBOT_PHONE        — WhatsApp
# CALLMEBOT_API_KEY      — WhatsApp
# NOTION_TOKEN           — project tracking
```

---

## IV. FOLDER STRUCTURE

```bash
PROJECT_ID="LIOR-DC2601"   # replace per project
mkdir -p .tmp/${PROJECT_ID}-design-concept/{00-brief,01-client-inputs,02-moodboard,03-visualizations,04-materials,05-exports,06-pdf-assembly}
touch .tmp/${PROJECT_ID}-design-concept/log.txt
```

---

## V. PRODUCTION STEPS

### Step 1 — Parse Brief (Day 1)

```bash
PROJECT_ID="LIOR-DC2601"
WORKDIR=".tmp/${PROJECT_ID}-design-concept"

cat > "${WORKDIR}/00-brief/00-concept-brief.txt" << 'EOF'
DESIGN CONCEPT BRIEF — [PROJECT_ID]
─────────────────────────────────────────────────────────
Outcome goal: [sell / STR rental / long-term rental / personal use]
Rooms in scope: [list]
Budget tier: [entry / mid / high-end]
Style direction: [client-selected or LIOR-assigned]
Key preferences: [top 5 from questionnaire]
Avoidances: [top 3]
References provided: [yes/no — filenames or URLs]
Renovation openness: [cosmetic only / open to structural]
Contractor to receive documents: [yes — name / no — client manages]
─────────────────────────────────────────────────────────

Reference image analysis (if provided):
  Recurring visual themes: [describe patterns seen across references]
  3 adjectives from references: [word 1, word 2, word 3]
  Style direction refined: [update from initial based on analysis]
EOF
echo "Brief parsed."
```

### Step 2 — Moodboard (Day 1)

One comprehensive moodboard covering the full concept — one cohesive aesthetic statement for the whole property.

#### 2A — Get Firefly Token (primary tool)

```bash
ADOBE_CLIENT_ID=$(grep ADOBE_CLIENT_ID /Users/cashville/.env | cut -d= -f2)
ADOBE_CLIENT_SECRET=$(grep ADOBE_CLIENT_SECRET /Users/cashville/.env | cut -d= -f2)

FIREFLY_TOKEN=$(curl -s -X POST "https://ims-na1.adobelogin.com/ims/token/v3" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials&client_id=${ADOBE_CLIENT_ID}&client_secret=${ADOBE_CLIENT_SECRET}&scope=openid,AdobeID,firefly_enterprise,firefly_api" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

echo $FIREFLY_TOKEN > /tmp/firefly_token.txt
echo "Token obtained. Valid 24h."
```

#### 2B — Generate Moodboard with Firefly

```bash
FIREFLY_TOKEN=$(cat /tmp/firefly_token.txt)
ADOBE_CLIENT_ID=$(grep ADOBE_CLIENT_ID /Users/cashville/.env | cut -d= -f2)
PROJECT_ID="LIOR-DC2601"
WORKDIR=".tmp/${PROJECT_ID}-design-concept"

# Adjust style direction and palette from brief
STYLE_DIR="contemporary warm"
PALETTE="cream #F5F0E8, warm oak, brushed brass, warm stone #C4A882, off-white"
KEY_MATERIALS="engineered oak floor, linen sofa, travertine surfaces, brass accents, linen curtains"

RESPONSE=$(curl -s -X POST "https://firefly-api.adobe.io/v3/images/generate" \
  -H "Authorization: Bearer ${FIREFLY_TOKEN}" \
  -H "X-Api-Key: ${ADOBE_CLIENT_ID}" \
  -H "Content-Type: application/json" \
  -d "{
    \"prompt\": \"interior design concept board, ${STYLE_DIR} Dubai residential, palette ${PALETTE}, materials ${KEY_MATERIALS}, lifestyle editorial, flat lay collage composition with swatches, furniture samples, textiles, and lifestyle photography, warm light\",
    \"negativePrompt\": \"cheap, IKEA, cartoon, oversaturated, cold light, watermark, cluttered, generic\",
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

# Download the best variant
CHOSEN_URL="[selected URL from above]"
curl -L "${CHOSEN_URL}" -o "${WORKDIR}/02-moodboard/${PROJECT_ID}-moodboard-v1.jpg"
echo "Moodboard saved."
```

**Fallback: Midjourney Discord**
```
In Discord #general-X channel:
/imagine prompt: interior design concept board, [style direction], palette [colors], [key materials], lifestyle editorial, flat lay collage with swatches and furniture references, Dubai residential, warm light --ar 16:9 --v 6
Click U1/U2/U3/U4 to upscale → Save Image As → save to 02-moodboard/
```

**Moodboard QA:**
- [ ] Colors cohesive — no clashing elements
- [ ] Palette reads warm overall (not cold, not grey-dominated)
- [ ] Material mix feels intentional — not a random assortment
- [ ] Representative of a specific lifestyle, not generic
- [ ] Color palette (5 swatches) visible
- [ ] Primary flooring material visible
- [ ] Key furniture character visible
- [ ] Accent material (brass / stone / ceramic) visible

### Step 3 — Color Grade Moodboard (ImageMagick)

```bash
PROJECT_ID="LIOR-DC2601"
WORKDIR=".tmp/${PROJECT_ID}-design-concept"

magick "${WORKDIR}/02-moodboard/${PROJECT_ID}-moodboard-v1.jpg" \
  -modulate 100,95,100 \
  -color-matrix "1.03 0 0 0 0.99 0 0 0 0.91" \
  -strip \
  "${WORKDIR}/02-moodboard/${PROJECT_ID}-moodboard-v1.jpg"
echo "Color grade applied."
```

### Step 4 — Key Room Visualizations (Days 2–3)

Visualize the 3 most impactful rooms:
- Sales-focused: living + master + kitchen
- Rental optimization: living + kitchen + outdoor/balcony

The concept style from the moodboard must be applied precisely to each room: same color palette, same material direction, same furniture character.

#### 4A — Upload room photo to ImgBB

```bash
IMGBB_KEY=$(grep IMGBB_API_KEY /Users/cashville/.env | cut -d= -f2)
PROJECT_ID="LIOR-DC2601"
WORKDIR=".tmp/${PROJECT_ID}-design-concept"

ROOM="living"
PHOTO="${WORKDIR}/01-client-inputs/${ROOM}-empty.jpg"

IMAGE_URL=$(curl -s -X POST "https://api.imgbb.com/1/upload" \
  -F "key=${IMGBB_KEY}" \
  -F "image=@${PHOTO}" \
  | python3 -c "import sys,json; r=json.load(sys.stdin); print(r['data']['url'])")
echo "Hosted: ${IMAGE_URL}"
```

#### 4B — Virtual Staging AI (primary — for rooms with photos)

```bash
VSTAGING_KEY=$(grep VSTAGING_API_KEY /Users/cashville/.env | cut -d= -f2)
ROOM="living"
# Map budget tier to density: entry=low, mid=medium, high=high
DENSITY="medium"
STYLE="contemporary"

JOB=$(curl -s -X POST "https://api.virtualstaging.ai/v1/stage" \
  -H "Authorization: Bearer ${VSTAGING_KEY}" \
  -H "Content-Type: application/json" \
  -d "{
    \"image_url\": \"${IMAGE_URL}\",
    \"room_type\": \"${ROOM}_room\",
    \"style\": \"${STYLE}\",
    \"furnishing_level\": \"${DENSITY}\",
    \"virtual_tour\": false
  }")

JOB_ID=$(echo $JOB | python3 -c "import sys,json; print(json.load(sys.stdin)['job_id'])")
echo "Job submitted: ${JOB_ID}"

while true; do
  RESULT=$(curl -s "https://api.virtualstaging.ai/v1/jobs/${JOB_ID}" \
    -H "Authorization: Bearer ${VSTAGING_KEY}")
  STATUS=$(echo $RESULT | python3 -c "import sys,json; print(json.load(sys.stdin)['status'])")
  if [ "$STATUS" = "completed" ]; then
    OUTPUT_URL=$(echo $RESULT | python3 -c "import sys,json; print(json.load(sys.stdin)['output_url'])")
    curl -L "${OUTPUT_URL}" -o "${WORKDIR}/03-visualizations/${PROJECT_ID}-viz-${ROOM}-v1.jpg"
    echo "Visualization complete: ${ROOM}"
    break
  elif [ "$STATUS" = "failed" ]; then
    echo "FAILED: ${ROOM} — switching to Firefly"
    echo "[$(date '+%Y-%m-%d %H:%M')] FALLBACK vstaging → Firefly for ${ROOM}" >> "${WORKDIR}/log.txt"
    break
  fi
  echo "${ROOM}: ${STATUS} — waiting 10s..."
  sleep 10
done
```

**Fallback: Firefly render (if no usable photo or Virtual Staging fails)**

```bash
FIREFLY_TOKEN=$(cat /tmp/firefly_token.txt)
ADOBE_CLIENT_ID=$(grep ADOBE_CLIENT_ID /Users/cashville/.env | cut -d= -f2)
ROOM="living"
PALETTE="cream, warm oak, brushed brass, warm stone"
STYLE_DIR="contemporary warm"

RESPONSE=$(curl -s -X POST "https://firefly-api.adobe.io/v3/images/generate" \
  -H "Authorization: Bearer ${FIREFLY_TOKEN}" \
  -H "X-Api-Key: ${ADOBE_CLIENT_ID}" \
  -H "Content-Type: application/json" \
  -d "{
    \"prompt\": \"${ROOM} interior, ${STYLE_DIR}, palette ${PALETTE}, wide angle from corner, warm afternoon light, Dubai apartment, editorial interior photography, photorealistic, 8k\",
    \"negativePrompt\": \"cheap, IKEA, cartoon, oversaturated, cold light, watermark, floating furniture\",
    \"size\": { \"width\": 2048, \"height\": 1152 },
    \"numVariations\": 3,
    \"contentClass\": \"photo\",
    \"styles\": { \"presets\": [\"photo_realism\"] }
  }")

CHOSEN_URL=$(echo $RESPONSE | python3 -c "import sys,json; print(json.load(sys.stdin)['outputs'][0]['image']['url'])")
curl -L "${CHOSEN_URL}" -o "${WORKDIR}/03-visualizations/${PROJECT_ID}-viz-${ROOM}-v1.jpg"
echo "Firefly render saved: ${ROOM}"
```

**Run visualization steps for each room (living, master, kitchen). Verify each image before proceeding.**

**Visualization QA per room:**
- [ ] Color palette from moodboard visible in the render
- [ ] Material direction from moodboard present (floor, walls, textiles)
- [ ] Furniture character matches moodboard
- [ ] No floating or clipping furniture
- [ ] No AI artifacts or impossible shadows
- [ ] Room proportions realistic

### Step 5 — Material + Finish Specification (Day 3)

Write the complete specification. One section per room. Cover every room in scope. All references must be real and available in Dubai.

```bash
PROJECT_ID="LIOR-DC2601"
WORKDIR=".tmp/${PROJECT_ID}-design-concept"

cat > "${WORKDIR}/04-materials/${PROJECT_ID}-material-spec-v1.md" << 'EOF'
# Material + Finish Specification — [PROJECT_ID]
Style direction: [name]
Budget tier: [entry / mid / high-end]

---

## Living Room

### Floor
- Material: Engineered oak, herringbone pattern, 70mm × 350mm boards
- Finish: Matte lacquer, pre-finished
- Color direction: Warm mid-honey — no grey undertone
- Reference: Kahrs "Lapponia Ash" or equivalent

### Walls
- Finish: Matte emulsion
- Color: Farrow & Ball "Skimming Stone" No.241 or equivalent (LRV ~58, warm off-white)
- Feature wall: None recommended at this tier

### Ceiling
- Finish: Brilliant white matte
- Height: Existing — no change

### Curtains
- Style: Linen, pinch-pleat, floor-to-ceiling
- Color: Off-white / écru (no pattern)
- Track: Ceiling-mounted, recessed into cornice

### Sofa
- Form: 3-seat, low profile, track arm
- Material: Performance linen or bouclé
- Color: Warm cream / stone
- Reference: [Entry: Zara Home Nordic / Mid: BoConcept / High: Restoration Hardware]

### Coffee Table
- Material: Honed travertine top, matte black powder-coated base
- Size: 120cm × 60cm (rectangular)

### Lighting
- Pendant: None in living — use floor lamp + recessed
- Floor lamp: Arc design, brushed brass arm, white linen shade
- Recessed: Warm white 2700K, wide-beam downlights (not spot)

### Art
- Scale: One large-format horizontal work (min 120cm wide)
- Style: Abstract, warm tones, organic mark-making
- Frame: None, or thin natural oak

---

## Master Bedroom

### Floor
- Material: [Same as living / soft carpet for warmth]
- Color: [Warm neutral]

### Walls
- Color: [Slightly warmer than living / same base]

### Bed
- Form: Upholstered, king size
- Headboard: Tall, fabric — curved or rectangular
- Material: Bouclé or performance fabric
- Color: Warm stone / muted sage / cream

### Bedside Tables
- Material: Solid oak or lacquered
- Style: Minimal, 1 drawer

### Lighting
- Bedside: Wall-mounted reading arms, brushed brass
- Overhead: Recessed 2700K — no central pendant

### Curtains
- Style: Same as living — linen, pinch-pleat, blackout lining for bedroom

---

## Kitchen

### Cabinetry
- Style: [Handleless slab / Shaker — per brief]
- Color: [Warm white / sage green / mid grey]
- Hardware: [Brushed brass / matte black / integrated push-to-open]

### Countertop
- Material: [Engineered quartz / honed marble]
- Color reference: [White vein on cream base / warm grey vein]

### Backsplash
- Material: [Metro tile in warm white / full slab quartz extension / zellige in warm tone]

### Appliances
- Tier reference:
  Entry → Bosch Series 4 standard
  Mid → Siemens iQ500 or equivalent
  High-end → Miele Generation 7000 or V-Zug

---

[Continue for all rooms in scope: bathrooms, hallway, second bedrooms, etc.]

EOF
echo "Material spec draft saved."
```

### Step 6 — Concept Book PDF Assembly (Day 3–4)

10-page structure: cover + concept statement + moodboard + palette + rooms + materials.

```bash
PROJECT_ID="LIOR-DC2601"
WORKDIR=".tmp/${PROJECT_ID}-design-concept"

cat > "${WORKDIR}/06-pdf-assembly/concept-book.md" << EOF
---
title: "Design Concept — ${PROJECT_ID}"
date: "$(date -u +%B %Y)"
geometry: "margin=2cm"
fontsize: 11pt
---

# Design Concept
**Property:** [Address]
**Client:** [Name]
**Prepared by:** LIOR
**Date:** $(date -u +%B %Y)

---

## Concept Statement

[2 paragraphs:

Paragraph 1 — design intent: describe the overall vision for the space, what mood it creates, how it responds to the client's life and goals.

Paragraph 2 — spatial vision: describe how each room connects, the material logic, why these specific choices serve this specific home.]

---

## Moodboard

![Moodboard](${WORKDIR}/02-moodboard/${PROJECT_ID}-moodboard-v1.jpg){ width=100% }

---

## Color + Material Palette

[List the 5 palette colors with their names / hex values. List the 5 key materials with one-line descriptions.]

| Element | Selection | Reference |
|---------|-----------|-----------|
| Wall base | Warm off-white | Farrow & Ball Skimming Stone No.241 |
| Floor | Warm oak herringbone | Kahrs Lapponia Ash or equiv |
| Accent | Honed travertine | [Supplier] |
| Metal finish | Brushed brass | [Supplier] |
| Textile | Performance linen | Warm cream |

---

## Living Room

![Living Room](${WORKDIR}/03-visualizations/${PROJECT_ID}-viz-living-v1.jpg){ width=100% }

### Material Specification

[Living room section from material spec — paste here]

---

## Master Bedroom

![Master Bedroom](${WORKDIR}/03-visualizations/${PROJECT_ID}-viz-master-v1.jpg){ width=100% }

### Material Specification

[Master bedroom section from material spec — paste here]

---

## Kitchen

![Kitchen](${WORKDIR}/03-visualizations/${PROJECT_ID}-viz-kitchen-v1.jpg){ width=100% }

### Material Specification

[Kitchen section from material spec — paste here]

---

[Continue for additional rooms if in scope]

---

## Ready to build this?

Contact LIOR to move into full project management — every material sourced, every contractor briefed, every decision tracked.

*LIOR Studio — [contact details]*

EOF

pandoc "${WORKDIR}/06-pdf-assembly/concept-book.md" \
  -o "${WORKDIR}/05-exports/${PROJECT_ID}-design-concept-v1.pdf" \
  --pdf-engine=xelatex \
  -V geometry:margin=2cm \
  -V fontsize=11pt

# Fallback PDF engine
# pandoc "${WORKDIR}/06-pdf-assembly/concept-book.md" \
#   -o "${WORKDIR}/05-exports/${PROJECT_ID}-design-concept-v1.pdf" \
#   --pdf-engine=wkhtmltopdf

echo "Concept book PDF: ${WORKDIR}/05-exports/${PROJECT_ID}-design-concept-v1.pdf"
ls -lh "${WORKDIR}/05-exports/"
```

---

## VI. QA CHECKLIST

Human verifies before delivery:
- [ ] Concept feels coherent — moodboard, visualizations, materials speak the same visual language
- [ ] Material specs are realistic for the stated budget tier
- [ ] No invented product SKUs — all references are real and available in Dubai
- [ ] Dubai market availability: nothing requiring 4–8 week import flagged as "standard" item
- [ ] PDF opens correctly — no broken image links, no placeholder text
- [ ] PDF is client-presentable: correct branding, correct project name, date
- [ ] Concept statement is specific to this project (not generic copy)
- [ ] All rooms in scope are present in the document

---

## VII. DELIVERY

### ZIP

```bash
PROJECT_ID="LIOR-DC2601"
WORKDIR=".tmp/${PROJECT_ID}-design-concept"

cd "${WORKDIR}"
zip -r "${PROJECT_ID}-design-concept-v1.zip" "05-exports/"
echo "ZIP size: $(wc -c < "${PROJECT_ID}-design-concept-v1.zip") bytes"
unzip -l "${PROJECT_ID}-design-concept-v1.zip"
```

### WeTransfer Upload

```bash
WT_KEY=$(grep WETRANSFER_API_KEY /Users/cashville/.env | cut -d= -f2)
ZIP_FILE="${WORKDIR}/${PROJECT_ID}-design-concept-v1.zip"
ZIP_SIZE=$(wc -c < "${ZIP_FILE}")

TRANSFER=$(curl -s -X POST "https://dev.wetransfer.com/v2/transfers" \
  -H "Content-Type: application/json" -H "x-api-key: ${WT_KEY}" \
  -d "{\"message\":\"${PROJECT_ID} — Design Concept LIOR\",\"files\":[{\"name\":\"${PROJECT_ID}-design-concept-v1.zip\",\"size\":${ZIP_SIZE}}]}")

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

MSG="${CLIENT_NAME} — your Design Concept is ready.

The document covers everything a contractor needs to execute your vision:
- Full moodboard and creative direction
- Visualizations of ${N_ROOMS} key rooms
- Complete material and finish specification

This is contractor-ready. Share it directly with any designer, builder, or fit-out team.

Download (valid 7 days): ${DOWNLOAD_URL}

Let us know if you would like to adjust any direction before you move forward."

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
echo "[$(date '+%Y-%m-%d %H:%M')] DELIVERED: ${PROJECT_ID} | Service: design-concept | ZIP: ${PROJECT_ID}-design-concept-v1.zip | Link: ${DOWNLOAD_URL} | WA: sent" \
  >> "${WORKDIR}/log.txt"
```

---

## VIII. WHATSAPP TEMPLATE

```
[CLIENT NAME] — your Design Concept is ready.

The document covers everything a contractor needs to execute your vision:
- Full moodboard and creative direction
- Visualizations of [N] key rooms
- Complete material and finish specification

This is contractor-ready. Share it directly with any designer, builder, or fit-out team.

Download (valid 7 days): [LINK]

Let us know if you would like to adjust any direction before you move forward.
```

Rules: no "AI", no "luxury", no prices, no "drone".

---

## IX. FALLBACKS

| Tool | Failure | Fix |
|---|---|---|
| Adobe Firefly | Token expired | Re-run token script (Step 2A). Valid 24h. |
| Firefly | API down | Midjourney Discord `/imagine` (same prompt, add `--ar 16:9 --v 6`) |
| Midjourney | Unavailable | Replicate SDXL `POST https://api.replicate.com/v1/predictions` — model hash `06d6fae3b75ab68a28cd2900afa6033166910dd09fd9751047043592f8f7cc60` |
| Virtual Staging AI | API down or quota exhausted | Wait 30 min. Try REimagineHome: `POST https://api.reimaginehome.ai/v1/generate`. Both down → Firefly. |
| REimagineHome | Also down | Use Firefly render (Step 4, fallback section) |
| Pandoc PDF | xelatex font error | `cp [font.ttf] ~/Library/Fonts/ && fc-cache -fv`. Or `--pdf-engine=wkhtmltopdf` |
| WeTransfer API | Upload fails | Manual upload at wetransfer.com |
| WeTransfer | File >2GB | Unlikely for concept book. If so: split PDF + images separately |
| WeTransfer | Down | Google Drive — share "anyone with link can view" |
| CallMeBot | Phone not registered | Send "I allow callmebot to send me messages" to +34 644 29 73 73 |
| ImgBB | Upload fails | Retry after 30s. 32MB max. |

---

## X. LOG FORMAT

```
[YYYY-MM-DD HH:MM] STARTED: [PROJECT_ID] | rooms=[N] | budget=[tier] | outcome=[sell/rent/live]
[YYYY-MM-DD HH:MM] BRIEF: parsed — style=[direction], avoidances=[top 3]
[YYYY-MM-DD HH:MM] MOODBOARD: generated — selected variant [1/2/3]
[YYYY-MM-DD HH:MM] VISUALIZATION: [room] — completed via [vstaging / Firefly / Midjourney]
[YYYY-MM-DD HH:MM] MATERIALS: specification written — [N] rooms
[YYYY-MM-DD HH:MM] PDF: generated — [filename]
[YYYY-MM-DD HH:MM] FALLBACK: [tool] → [alternative] — [room] — [reason]
[YYYY-MM-DD HH:MM] QA: APPROVED by [name]
[YYYY-MM-DD HH:MM] DELIVERED: [PROJECT_ID] | ZIP: [filename] | Link: [URL] | WA: sent
```

---

## UPSELL TRIGGER

48h after delivery:

```
[CLIENT NAME] — if you need someone to manage the project through to completion — coordinating contractors, approving samples, making sure the concept lands as designed — that is exactly what our full interior design service includes.
It might be worth a quick conversation.
```

→ SOP-05 (Full Interior Design).
