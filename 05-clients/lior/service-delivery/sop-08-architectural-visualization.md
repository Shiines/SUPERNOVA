# SOP 08: Architectural Visualization
**Automation level: C: Guided (agent handles rendering pipeline; human confirms design intent)**
**Timeline: 5 days from confirmed render brief**
**Deliverable: Exterior + interior renders · presentation-ready PDF · all renders as individual high-resolution JPGs**

## I. OVERVIEW

3D-quality architectural visualization from plans and drawings: no physical space required. Agent confirms the render brief before starting, then runs the production pipeline: exterior renders via Replicate SDXL + ControlNet (geometry-accurate), interior renders via Firefly (off-plan) or Virtual Staging AI (if physical space exists), post-processing via ImageMagick, and final assembly into a presentation PDF via Pandoc. Human confirms the brief before production and approves renders before delivery.

---

## II. INPUTS

| # | Input | Format | Source | Blocker? |
|---|-------|--------|--------|---------|
| 1 | Floor plan | PDF, CAD (DWG/DXF), or JPG | Client / architect | YES |
| 2 | Elevation drawings (min 2) | PDF or CAD | Client / architect | YES: if exterior renders required |
| 3 | Property brief | Building type, context, target audience | Intake form | YES |
| 4 | Render brief | Which views required (exterior angles, interior rooms) | Intake form | YES |
| 5 | Material palette | Text or reference images | Client or LIOR default | No: LIOR applies default |
| 6 | Style reference | Comparable project images | Client | No |
| 7 | Project ID | `LIOR-AV[CODE][YY][SEQ]` | Agent | Internal |

**If elevation drawings are missing:** exterior renders are not possible. Offer interior renders only. Note exterior as pending input.
**If CAD not available:** accept JPG + floor plan photos. Note accuracy limitations in brief.

---

## III. SETUP (one-time)

```bash
# Install dependencies
brew install imagemagick pandoc
brew install --cask mactex        # LaTeX PDF engine
brew install wkhtmltopdf           # PDF engine fallback

# Verify .env keys
grep -E "REPLICATE_API_KEY|IMGBB_API_KEY|VSTAGING_API_KEY|ADOBE_CLIENT_ID|ADOBE_CLIENT_SECRET|WETRANSFER_API_KEY|CALLMEBOT_PHONE|CALLMEBOT_API_KEY|NOTION_TOKEN" /Users/cashville/.env

# Keys needed:
# REPLICATE_API_KEY     : SDXL + ControlNet exterior renders
# IMGBB_API_KEY         : image hosting (Replicate + Runway need public URLs)
# VSTAGING_API_KEY      : Virtual Staging AI (interior renders if physical space exists)
# ADOBE_CLIENT_ID       : Firefly (interior renders for off-plan)
# ADOBE_CLIENT_SECRET   : Firefly
# WETRANSFER_API_KEY    : delivery
# CALLMEBOT_PHONE       : WhatsApp
# CALLMEBOT_API_KEY     : WhatsApp
# NOTION_TOKEN          : project tracking
```

---

## IV. FOLDER STRUCTURE

```bash
PROJECT_ID="LIOR-AV2601"   # replace per project
mkdir -p .tmp/${PROJECT_ID}-arch-viz/{00-brief,01-client-inputs/{plans,elevations,references},02-renders/{exterior,interior},03-post-processed,04-presentation,05-exports/renders}
touch .tmp/${PROJECT_ID}-arch-viz/log.txt
```

---

## V. PRODUCTION STEPS

### Step 1: Confirm Render Brief

Send render brief to human for confirmation before starting any production.

```bash
PROJECT_ID="LIOR-AV2601"
WORKDIR=".tmp/${PROJECT_ID}-arch-viz"

cat > "${WORKDIR}/00-brief/render-brief.txt" << 'EOF'
RENDER BRIEF: [PROJECT_ID]
Building type: [apartment tower / villa / commercial / other]
Context: [Dubai Marina / Palm / Business Bay / Downtown / residential community]
Target audience: [investors / planning submission / sales / marketing]
─────────────────────────────
EXTERIOR VIEWS CONFIRMED:
  [ ] Front elevation (street view)
  [ ] Rear / garden elevation
  [ ] Corner / aerial angle
  [ ] Entrance detail

INTERIOR VIEWS CONFIRMED:
  [ ] Living room: wide from corner
  [ ] Living room: toward window
  [ ] Kitchen: wide
  [ ] Master bedroom
  [ ] Bathroom / ensuite
  [ ] Staircase (if present)
  [ ] Other: ________________

TIME OF DAY / LIGHTING:
  [ ] Golden hour (warm, late afternoon)
  [ ] Midday (neutral, architectural clarity)
  [ ] Interior evening (lit interior with dark exterior)

Material palette: [text or "LIOR default for [building type]"]
Total renders confirmed: [N]
─────────────────────────────
AWAITING HUMAN CONFIRMATION BEFORE PRODUCTION STARTS
EOF

echo "Render brief draft saved. Send to human for confirmation."
```

Agent sends WhatsApp to human with brief summary, awaits sign-off. **Production does not start until brief is confirmed.**

### Step 2: Exterior Renders (Replicate SDXL + ControlNet)

Replicate is the primary tool for exterior renders: ControlNet preserves architectural geometry from the elevation drawing.

#### 2A: Upload elevation drawing to ImgBB

```bash
IMGBB_KEY=$(grep IMGBB_API_KEY /Users/cashville/.env | cut -d= -f2)
PROJECT_ID="LIOR-AV2601"
WORKDIR=".tmp/${PROJECT_ID}-arch-viz"

# Upload front elevation drawing
ELEVATION="${WORKDIR}/01-client-inputs/elevations/front-elevation.jpg"
ELEVATION_URL=$(curl -s -X POST "https://api.imgbb.com/1/upload" \
  -F "key=${IMGBB_KEY}" \
  -F "image=@${ELEVATION}" \
  | python3 -c "import sys,json; r=json.load(sys.stdin); print(r['data']['url'])")
echo "Elevation hosted: ${ELEVATION_URL}"
```

#### 2B: Run Replicate prediction

```bash
REPLICATE_KEY=$(grep REPLICATE_API_KEY /Users/cashville/.env | cut -d= -f2)
BUILDING_TYPE="apartment tower"  # adjust from brief
VIEW_NAME="front-elevation"
TIME_OF_DAY="golden hour"

# Build prompt based on building type
# Apartment tower:
PROMPT="photorealistic architectural visualization, luxury apartment tower, glass curtain wall, aluminium cladding, landscaped podium, Dubai Marina setting, ${TIME_OF_DAY}, professional architecture photography, 8k resolution, accurate proportions, no distortion"
# Villa: "photorealistic architectural visualization, contemporary villa, rendered facade, landscaped garden, palm tree entry, residential Dubai Hills setting, ${TIME_OF_DAY}, professional architecture photography, 8k"
# Commercial: "photorealistic architectural visualization, corporate facade, ground-level retail, urban street context, Business Bay Dubai, ${TIME_OF_DAY}, professional architecture photography, 8k"

PREDICTION=$(curl -s -X POST "https://api.replicate.com/v1/predictions" \
  -H "Authorization: Token ${REPLICATE_KEY}" \
  -H "Content-Type: application/json" \
  -d "{
    \"version\": \"06d6fae3b75ab68a28cd2900afa6033166910dd09fd9751047043592f8f7cc60\",
    \"input\": {
      \"image\": \"${ELEVATION_URL}\",
      \"prompt\": \"${PROMPT}\",
      \"negative_prompt\": \"cartoon, sketch, unrealistic proportions, watermark, text, blurry, distorted geometry\",
      \"controlnet_conditioning_scale\": 0.85,
      \"width\": 1920,
      \"height\": 1080,
      \"num_inference_steps\": 50,
      \"guidance_scale\": 7.5,
      \"num_outputs\": 3
    }
  }")

PREDICTION_ID=$(echo $PREDICTION | python3 -c "import sys,json; print(json.load(sys.stdin)['id'])")
echo "Prediction ID: ${PREDICTION_ID}: view: ${VIEW_NAME}"
```

#### 2C: Poll for result

```bash
REPLICATE_KEY=$(grep REPLICATE_API_KEY /Users/cashville/.env | cut -d= -f2)
# PREDICTION_ID from step 2B

while true; do
  RESULT=$(curl -s "https://api.replicate.com/v1/predictions/${PREDICTION_ID}" \
    -H "Authorization: Token ${REPLICATE_KEY}")
  STATUS=$(echo $RESULT | python3 -c "import sys,json; print(json.load(sys.stdin)['status'])")

  if [ "$STATUS" = "succeeded" ]; then
    echo "Variants:"
    echo $RESULT | python3 -c "import sys,json; [print(f'{i+1}: {u}') for i,u in enumerate(json.load(sys.stdin)['output'])]"
    break
  elif [ "$STATUS" = "failed" ]; then
    echo "FAILED: $(echo $RESULT | python3 -c \"import sys,json; print(json.load(sys.stdin).get('error',''))\")"
    break
  fi
  echo "Status: ${STATUS}: waiting 5s..."
  sleep 5
done

# Download chosen variant
CHOSEN_URL="[selected from output above]"
curl -L "${CHOSEN_URL}" -o "${WORKDIR}/02-renders/exterior/${PROJECT_ID}-ext-${VIEW_NAME}-v1.jpg"
echo "Exterior render saved: ${VIEW_NAME}"
```

**Run steps 2A–2C for each confirmed exterior view. Change VIEW_NAME and ELEVATION drawing each time.**

**Exterior render QA per view:**
- [ ] Building proportions match the elevation drawing
- [ ] No impossible geometry (floating elements, perspective collapse)
- [ ] Materials look physically accurate (glass reflects sky, concrete has texture)
- [ ] Context (street, landscape, sky) looks realistic
- [ ] No AI artifacts on facade surfaces
- [ ] Lighting direction consistent across the render
- [ ] Shadows are directionally accurate

**Replicate fallback: if model hash is deprecated:**
```bash
# Find a current model: go to https://replicate.com/explore → search "interior design SDXL controlnet"
# Select model with most recent activity → copy the version hash
# Replace the hash in the prediction call above
echo "[$(date '+%Y-%m-%d %H:%M')] FALLBACK Replicate: model hash updated to [new hash]" >> "${WORKDIR}/log.txt"
```

### Step 3: Interior Renders

Use Virtual Staging AI if a physical space exists; use Firefly for pure off-plan / new build visualization.

#### Option A: Physical space exists (use Virtual Staging AI)

```bash
IMGBB_KEY=$(grep IMGBB_API_KEY /Users/cashville/.env | cut -d= -f2)
VSTAGING_KEY=$(grep VSTAGING_API_KEY /Users/cashville/.env | cut -d= -f2)
PROJECT_ID="LIOR-AV2601"
WORKDIR=".tmp/${PROJECT_ID}-arch-viz"

ROOM="living"
PHOTO="${WORKDIR}/01-client-inputs/photos/${ROOM}-empty.jpg"

# Upload to ImgBB
IMAGE_URL=$(curl -s -X POST "https://api.imgbb.com/1/upload" \
  -F "key=${IMGBB_KEY}" -F "image=@${PHOTO}" \
  | python3 -c "import sys,json; r=json.load(sys.stdin); print(r['data']['url'])")

# Stage with Virtual Staging AI
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
    curl -L "${OUTPUT_URL}" -o "${WORKDIR}/02-renders/interior/${PROJECT_ID}-int-${ROOM}-v1.jpg"
    echo "Interior staged: ${ROOM}"
    break
  elif [ "$STATUS" = "failed" ]; then
    echo "FAILED: ${ROOM}: use Option B (Firefly)"
    echo "[$(date '+%Y-%m-%d %H:%M')] FALLBACK vstaging → Firefly for ${ROOM}" >> "${WORKDIR}/log.txt"
    break
  fi
  echo "${ROOM}: ${STATUS}: waiting 10s..."
  sleep 10
done
```

#### Option B: Off-plan / no physical space (use Firefly)

```bash
ADOBE_CLIENT_ID=$(grep ADOBE_CLIENT_ID /Users/cashville/.env | cut -d= -f2)
ADOBE_CLIENT_SECRET=$(grep ADOBE_CLIENT_SECRET /Users/cashville/.env | cut -d= -f2)

# Get token (valid 24h: run once per session)
FIREFLY_TOKEN=$(curl -s -X POST "https://ims-na1.adobelogin.com/ims/token/v3" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials&client_id=${ADOBE_CLIENT_ID}&client_secret=${ADOBE_CLIENT_SECRET}&scope=openid,AdobeID,firefly_api" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")
echo $FIREFLY_TOKEN > /tmp/firefly_token.txt

# Generate interior render
PROJECT_ID="LIOR-AV2601"
WORKDIR=".tmp/${PROJECT_ID}-arch-viz"
ROOM="living"
# Build prompt from render brief (style, materials, time of day)
PROMPT="living room, contemporary architectural interior, wide angle from corner, natural materials, warm afternoon light through floor-to-ceiling windows, Dubai luxury residential, wide angle architectural photography, shot on Phase One, 8k, photorealistic"

RESPONSE=$(curl -s -X POST "https://firefly-api.adobe.io/v3/images/generate" \
  -H "Authorization: Bearer ${FIREFLY_TOKEN}" \
  -H "X-Api-Key: ${ADOBE_CLIENT_ID}" \
  -H "Content-Type: application/json" \
  -d "{
    \"prompt\": \"${PROMPT}\",
    \"negativePrompt\": \"cheap, IKEA, cartoon, oversaturated, cold light, watermark, floating furniture, impossible geometry\",
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
curl -L "${CHOSEN_URL}" -o "${WORKDIR}/02-renders/interior/${PROJECT_ID}-int-${ROOM}-v1.jpg"
echo "Interior render saved: ${ROOM}"
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
      \"prompt\":\"${ROOM} room, contemporary architectural interior, ${MATERIALS}, wide angle architectural photography, shot on Phase One, 8k, photorealistic, Dubai residential, warm light\",
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
curl -L "${CHOSEN_URL}" -o "${WORKDIR}/02-renders/interior/${PROJECT_ID}-int-${ROOM}-v1.jpg"
```

**Interior render QA per view:**
- [ ] Room proportions match floor plan dimensions
- [ ] Ceiling height correct (standard 2.8m or as specified)
- [ ] Natural light enters from correct side (per floor plan window positions)
- [ ] Materials consistent with brief palette
- [ ] No floating objects or impossible shadows
- [ ] Furniture scale correct relative to room

### Step 4: Post-Processing (ImageMagick)

Apply color grade to hero renders only (not all variants).

```bash
PROJECT_ID="LIOR-AV2601"
WORKDIR=".tmp/${PROJECT_ID}-arch-viz"

# Golden hour exterior grade (warm boost)
for F in "${WORKDIR}/02-renders/exterior/${PROJECT_ID}-ext-"*"-v1.jpg"; do
  magick "${F}" \
    -modulate 100,105,100 \
    -color-matrix "1.05 0 0 0 1.0 0 0 0 0.88" \
    -strip \
    "${WORKDIR}/03-post-processed/$(basename ${F})"
done

# Interior evening grade (slight warm push + lifted shadows)
for F in "${WORKDIR}/02-renders/interior/${PROJECT_ID}-int-"*"-v1.jpg"; do
  magick "${F}" \
    -modulate 100,98,100 \
    -color-matrix "1.03 0 0 0 0.99 0 0 0 0.91" \
    -strip \
    "${WORKDIR}/03-post-processed/$(basename ${F})"
done

# Sharpen (structural only: not artistic)
for F in "${WORKDIR}/03-post-processed/"*".jpg"; do
  magick "${F}" \
    -unsharp 0x0.5+80+0 \
    "${F}"
done

# Upscale to 4K if source is smaller
for F in "${WORKDIR}/03-post-processed/"*".jpg"; do
  magick "${F}" -resize "3840x2160>" "${F}"
done

echo "Post-processing complete."
ls -lh "${WORKDIR}/03-post-processed/"
```

**Note: Sky replacement (exterior renders)**: if generated sky looks flat or unrealistic, replace manually in Photoshop: Edit → Sky Replacement → select from Premium Dramatic category. ImageMagick cannot do realistic sky replacement.

### Step 5: Presentation PDF Assembly

```bash
PROJECT_ID="LIOR-AV2601"
WORKDIR=".tmp/${PROJECT_ID}-arch-viz"
N_EXT="[N exterior renders]"
N_INT="[N interior renders]"

cat > "${WORKDIR}/04-presentation/presentation.md" << EOF
---
title: "Architectural Visualization: ${PROJECT_ID}"
date: "$(date -u +%B %Y)"
geometry: "margin=1.5cm"
fontsize: 11pt
---

# Architectural Visualization
**Project:** [Project name / address]
**Client:** [Name]
**Prepared by:** LIOR
**Date:** $(date -u +%B %Y)

---

## Project Overview

[Building type, context, design intent: 3–4 sentences from brief.]

---

$(for F in ${WORKDIR}/03-post-processed/${PROJECT_ID}-ext-*-v1.jpg; do
  VIEWNAME=$(basename "$F" | sed "s/${PROJECT_ID}-ext-//" | sed "s/-v1.jpg//")
  echo "## Exterior: ${VIEWNAME}"
  echo ""
  echo "![${VIEWNAME}](${F}){ width=100% }"
  echo ""
  echo "*${VIEWNAME} · golden hour · [material notes if relevant]*"
  echo ""
  echo "---"
  echo ""
done)

$(for F in ${WORKDIR}/03-post-processed/${PROJECT_ID}-int-*-v1.jpg; do
  ROOMNAME=$(basename "$F" | sed "s/${PROJECT_ID}-int-//" | sed "s/-v1.jpg//")
  echo "## Interior: ${ROOMNAME}"
  echo ""
  echo "![${ROOMNAME}](${F}){ width=100% }"
  echo ""
  echo "*${ROOMNAME} · [time of day] · [material notes if relevant]*"
  echo ""
  echo "---"
  echo ""
done)

*Questions or adjustments: contact LIOR*

EOF

pandoc "${WORKDIR}/04-presentation/presentation.md" \
  -o "${WORKDIR}/04-presentation/${PROJECT_ID}-arch-viz-presentation-v1.pdf" \
  --pdf-engine=xelatex \
  -V geometry:margin=1.5cm \
  -V fontsize=11pt

# Fallback PDF engine
# pandoc ... --pdf-engine=wkhtmltopdf

echo "Presentation PDF: ${WORKDIR}/04-presentation/${PROJECT_ID}-arch-viz-presentation-v1.pdf"
```

### Step 6: Assemble Exports

```bash
PROJECT_ID="LIOR-AV2601"
WORKDIR=".tmp/${PROJECT_ID}-arch-viz"

# Copy all post-processed renders to exports
cp "${WORKDIR}/03-post-processed/"*".jpg" "${WORKDIR}/05-exports/renders/"
cp "${WORKDIR}/04-presentation/${PROJECT_ID}-arch-viz-presentation-v1.pdf" "${WORKDIR}/05-exports/"

ls -lh "${WORKDIR}/05-exports/"
ls -lh "${WORKDIR}/05-exports/renders/"
```

---

## VI. QA CHECKLIST

Human reviews before delivery:
- [ ] All confirmed views from render brief are present
- [ ] Exterior render proportions match the elevation drawing
- [ ] No impossible geometry in any render
- [ ] Materials look physically accurate
- [ ] Interior room proportions match floor plan
- [ ] Lighting is correct for stated time of day
- [ ] No AI artifacts on any hero render
- [ ] Post-processing is consistent across all renders (not over-saturated, not cold)
- [ ] Presentation PDF is complete: one render per page, correct captions
- [ ] PDF is client-presentable (correct project name, date, no placeholder text)

---

## VII. DELIVERY

### ZIP

```bash
PROJECT_ID="LIOR-AV2601"
WORKDIR=".tmp/${PROJECT_ID}-arch-viz"

cd "${WORKDIR}"
zip -r "${PROJECT_ID}-arch-viz-v1.zip" "05-exports/"
echo "ZIP size: $(wc -c < "${PROJECT_ID}-arch-viz-v1.zip") bytes"
unzip -l "${PROJECT_ID}-arch-viz-v1.zip"
```

### WeTransfer Upload

```bash
WT_KEY=$(grep WETRANSFER_API_KEY /Users/cashville/.env | cut -d= -f2)
ZIP_FILE="${WORKDIR}/${PROJECT_ID}-arch-viz-v1.zip"
ZIP_SIZE=$(wc -c < "${ZIP_FILE}")

TRANSFER=$(curl -s -X POST "https://dev.wetransfer.com/v2/transfers" \
  -H "Content-Type: application/json" -H "x-api-key: ${WT_KEY}" \
  -d "{\"message\":\"${PROJECT_ID}: Architectural Visualization LIOR\",\"files\":[{\"name\":\"${PROJECT_ID}-arch-viz-v1.zip\",\"size\":${ZIP_SIZE}}]}")

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
N_EXT="[N]"
N_INT="[N]"
N_TOTAL="[N]"

MSG="${CLIENT_NAME}: your architectural visualizations are ready.

${N_TOTAL} renders delivered:
- ${N_EXT} exterior views
- ${N_INT} interior views
- Full presentation document

Download (valid 7 days): ${DOWNLOAD_URL}

These are ready to share with investors, planning submissions, or sales presentations. Let us know if you would like any views adjusted: time of day, material, or angle."

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
echo "[$(date '+%Y-%m-%d %H:%M')] DELIVERED: ${PROJECT_ID} | Service: arch-viz | ZIP: ${PROJECT_ID}-arch-viz-v1.zip | Renders: ${N_TOTAL} | Link: ${DOWNLOAD_URL} | WA: sent" \
  >> "${WORKDIR}/log.txt"
```

---

## VIII. WHATSAPP TEMPLATE

```
[CLIENT NAME]: your architectural visualizations are ready.

[N] renders delivered:
- [N] exterior views
- [N] interior views
- Full presentation document

Download (valid 7 days): [LINK]

These are ready to share with investors, planning submissions, or sales presentations. Let us know if you would like any views adjusted: time of day, material, or angle.
```

Rules: no "AI", no "luxury", no prices, no "drone".

---

## IX. FALLBACKS

| Tool | Failure | Fix |
|---|---|---|
| Replicate | Model hash deprecated | Go to replicate.com/explore → search "interior design SDXL controlnet" → find model with most recent activity → copy new version hash → replace in Step 2B |
| Replicate | API rate limit | Wait 60s and retry |
| Virtual Staging AI | API down | Wait 30 min. Try REimagineHome: `POST https://api.reimaginehome.ai/v1/generate`. If both down → use Firefly (Option B). |
| Adobe Firefly | Token expired | Re-run token script in Step 3. Valid 24h. |
| Firefly | API down | Replicate SDXL without ControlNet (pure text-to-image) |
| Pandoc PDF | xelatex font error | `cp [font.ttf] ~/Library/Fonts/ && fc-cache -fv`. Or use `--pdf-engine=wkhtmltopdf`. |
| WeTransfer API | Upload fails | Manual upload at wetransfer.com |
| WeTransfer | File >2GB | Split: renders ZIP + presentation PDF ZIP separately |
| WeTransfer | Down | Google Drive: share link, "anyone with link can view" |
| ImgBB | Upload fails | Retry after 30s. 32MB max per image. |

---

## X. REVISION SCOPE

**Included in base price:**
- Lighting adjustment (time of day change: re-generate with different prompt)
- Material swap (re-generate with updated palette)
- Angle tweak on existing confirmed views

**Not included: new brief line item:**
- Additional views beyond agreed render brief
- New rooms not in original brief
- Structural changes to the building design

---

## XI. LOG FORMAT

```
[YYYY-MM-DD HH:MM] STARTED: [PROJECT_ID] | views=[N ext + N int] | type=[building type]
[YYYY-MM-DD HH:MM] BRIEF CONFIRMED: by [human name]: [N] views locked
[YYYY-MM-DD HH:MM] EXTERIOR: [view name]: prediction ID [X] submitted
[YYYY-MM-DD HH:MM] EXTERIOR: [view name]: downloaded OK
[YYYY-MM-DD HH:MM] INTERIOR: [room name]: staged / rendered OK
[YYYY-MM-DD HH:MM] POST-PROCESS: applied to [N] renders
[YYYY-MM-DD HH:MM] FALLBACK: [tool] → [alternative]: [view/room]: [reason]
[YYYY-MM-DD HH:MM] QA: APPROVED by [name]
[YYYY-MM-DD HH:MM] DELIVERED: [PROJECT_ID] | ZIP: [filename] | Link: [URL] | WA: sent
```
