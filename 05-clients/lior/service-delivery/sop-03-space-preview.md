# SOP 03: Space Preview
**Automation level: B: Auto + human gate**
**Timeline: 5 days from brief confirmation**
**Deliverable: Full space visualization · cinematic tour (renders-only) · 6–8 social assets · 2 design directions**

## I. OVERVIEW

Off-plan and pre-renovation spaces that don't exist yet: floor plan in, fully designed and visualized space out. LIOR generates two distinct creative directions, each with complete room renders, a cinematic animated tour, and social-ready cuts. Client picks the direction they prefer; that choice feeds naturally into SOP-09 or SOP-05.

---

## II. INPUTS

| # | Input | Format | Source | Blocker? |
|---|-------|--------|--------|---------|
| 1 | Floor plan | PDF or JPG, all rooms legible | Client | YES |
| 2 | Property brief | Type, size, location, target buyer | Intake form | YES |
| 3 | Number of rooms to visualize | Integer | Intake form | YES |
| 4 | Style preferences | Text, references, or Pinterest link | Intake form | No: LIOR applies default if absent |
| 5 | Existing shell photos | JPG ≥ 1920×1080 | Client | No: render-only if absent |
| 6 | Project ID | `LIOR-SP[CODE][YY][SEQ]` | Agent | Internal |

**No floor plan → no production. A phone photo of a hand sketch is acceptable if all rooms and approximate dimensions are legible.**

---

## III. SETUP (one-time)

```bash
# Install dependencies
brew install ffmpeg imagemagick pandoc

# Verify .env keys exist
grep -E "IMGBB_API_KEY|VSTAGING_API_KEY|RUNWAY_API_KEY|ADOBE_CLIENT_ID|ADOBE_CLIENT_SECRET|WETRANSFER_API_KEY|CALLMEBOT_PHONE|CALLMEBOT_API_KEY|NOTION_TOKEN" /Users/cashville/.env

# Keys needed:
# IMGBB_API_KEY       : image hosting before API calls
# VSTAGING_API_KEY    : Virtual Staging AI (if shell photos exist)
# RUNWAY_API_KEY      : Gen-3 animation
# ADOBE_CLIENT_ID     : Firefly (primary render engine for off-plan)
# ADOBE_CLIENT_SECRET : Firefly
# WETRANSFER_API_KEY  : delivery
# CALLMEBOT_PHONE     : WhatsApp notify
# CALLMEBOT_API_KEY   : WhatsApp notify
# NOTION_TOKEN        : project tracking
```

---

## IV. FOLDER STRUCTURE

```bash
PROJECT_ID="LIOR-SP2601"   # replace per project
mkdir -p .tmp/${PROJECT_ID}-space-preview/{00-brief,01-client-inputs,02-directions/{direction-A,direction-B},03-renders/{direction-A,direction-B},04-cinematic,05-social,06-exports/{Direction-A,Direction-B,Social}}
touch .tmp/${PROJECT_ID}-space-preview/log.txt
```

---

## V. PRODUCTION STEPS

### Step 1: Floor Plan Analysis

Read the floor plan image and extract the room inventory into the brief file.

```bash
PROJECT_ID="LIOR-SP2601"
WORKDIR=".tmp/${PROJECT_ID}-space-preview"

cat > "${WORKDIR}/00-brief/00-floor-plan-inventory.txt" << 'EOF'
FLOOR PLAN INVENTORY: [PROJECT_ID]
Total area: [m²]
Rooms:
  - Living / dining: [dimensions if visible]
  - Master bedroom: [yes/no, dimensions]
  - Additional bedrooms: [count]
  - Kitchen: [open or separate]
  - Bathrooms / ensuites: [count]
  - Balcony / terrace: [yes/no, size]
  - Entry / hallway: [yes/no]
Natural light direction: [based on window positions on plan]
Ceiling height: [if stated, else TBD]
Architectural features: [open plan, split levels, double height, etc.]
Rooms to visualize: [list from intake: minimum: living, master, kitchen + 1 extra]
EOF
```

Fill in the values by reading the floor plan. Note any TBD items.

### Step 2: Define Two Design Directions

Choose the direction pair appropriate to the project type:

| Project type | Direction A | Direction B |
|---|---|---|
| Off-plan investor | Modern International: neutral, broad appeal | Contemporary Warm: lifestyle, aspirational |
| Primary residence | Contemporary Warm: wood, linen, warm stone | Modern Minimal: monochrome, precise |
| Premium / developer | Curated Luxury: statement pieces, layered textures | Contemporary International: gallery-feel, curated |

For each direction, define and save:

```bash
cat > "${WORKDIR}/02-directions/direction-A/direction-A.txt" << 'EOF'
Direction A:
  Name: [1–2 words e.g. "Warm Contemporary"]
  Style: [one-line description]
  Palette: [3–5 colors: hex or material name e.g. warm white #F5F0E8, oak, warm stone]
  Key materials: floor=[material], walls=[finish], kitchen=[surface], textiles=[describe]
  Furniture character: [describe the furniture feel: low / modular / organic / etc.]
  Mood: [one sentence: what does it feel like to be in this space?]
EOF

cat > "${WORKDIR}/02-directions/direction-B/direction-B.txt" << 'EOF'
Direction B:
  Name: [1–2 words e.g. "Modern Minimal"]
  Style: [one-line description]
  Palette: [3–5 colors]
  Key materials: floor=[material], walls=[finish], kitchen=[surface], textiles=[describe]
  Furniture character: [describe]
  Mood: [one sentence]
EOF
```

### Step 3: Generate Room Visualizations (both directions)

Generate 1 hero visualization per room per direction. Minimum rooms: living, master bedroom, kitchen + 1 additional.

#### 3A: Get Adobe Firefly Token (primary engine for off-plan: no real photo needed)

```bash
ADOBE_CLIENT_ID=$(grep ADOBE_CLIENT_ID /Users/cashville/.env | cut -d= -f2)
ADOBE_CLIENT_SECRET=$(grep ADOBE_CLIENT_SECRET /Users/cashville/.env | cut -d= -f2)

FIREFLY_TOKEN=$(curl -s -X POST "https://ims-na1.adobelogin.com/ims/token/v3" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials&client_id=${ADOBE_CLIENT_ID}&client_secret=${ADOBE_CLIENT_SECRET}&scope=openid,AdobeID,firefly_api" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

echo "Firefly token obtained. Valid 24h."
# Save token for reuse in this session:
echo $FIREFLY_TOKEN > /tmp/firefly_token.txt
```

#### 3B: Generate each room × each direction

Repeat for every room × direction combination. Adjust prompt per room and direction.

```bash
FIREFLY_TOKEN=$(cat /tmp/firefly_token.txt)
ADOBE_CLIENT_ID=$(grep ADOBE_CLIENT_ID /Users/cashville/.env | cut -d= -f2)
PROJECT_ID="LIOR-SP2601"
WORKDIR=".tmp/${PROJECT_ID}-space-preview"

# Example: Living room, Direction A: Contemporary Warm
ROOM="living"
DIRECTION="A"
PROMPT="living room, contemporary warm interior design, warm white walls, oak herringbone floor, linen sofa, travertine coffee table, warm afternoon light from floor-to-ceiling windows, neutral tones with terracotta accent, wide angle from corner, Dubai apartment, editorial interior photography, photorealistic, 8k, shot on Hasselblad, natural light"

RESPONSE=$(curl -s -X POST "https://firefly-api.adobe.io/v3/images/generate" \
  -H "Authorization: Bearer ${FIREFLY_TOKEN}" \
  -H "X-Api-Key: ${ADOBE_CLIENT_ID}" \
  -H "Content-Type: application/json" \
  -d "{
    \"prompt\": \"${PROMPT}\",
    \"negativePrompt\": \"cheap, IKEA, cartoon, oversaturated, cold light, plastic, watermark, floating furniture, unrealistic proportions\",
    \"size\": { \"width\": 2048, \"height\": 1152 },
    \"numVariations\": 3,
    \"contentClass\": \"photo\",
    \"styles\": { \"presets\": [\"photo_realism\"] }
  }")

# Extract URLs
echo $RESPONSE | python3 -c "
import sys, json
data = json.load(sys.stdin)
for i, o in enumerate(data.get('outputs', [])):
    print(f'Variant {i+1}: {o[\"image\"][\"url\"]}')
"

# Download best variant (replace VAR_URL with chosen URL)
VAR_URL="[chosen URL from output above]"
curl -L "${VAR_URL}" -o "${WORKDIR}/03-renders/direction-${DIRECTION}/${PROJECT_ID}-render-${ROOM}-${DIRECTION}-v1.jpg"
```

**Prompt building blocks by room:**

| Room | Core elements to include in prompt |
|---|---|
| Living | "living room, [style], [palette], linen sofa, wide angle from corner, floor-to-ceiling windows" |
| Master bedroom | "master bedroom, [style], [palette], upholstered headboard, bedside lighting, soft morning light" |
| Kitchen | "kitchen, [style], [palette], island or peninsula, pendant lights, marble or stone countertop" |
| Bathroom | "bathroom ensuite, [style], [palette], freestanding bath or walk-in shower, natural stone" |
| Balcony | "balcony terrace, outdoor furniture, Dubai skyline or marina view, evening golden hour" |

#### 3C: Fallback: Replicate SDXL (if Firefly is unavailable)

```bash
REPLICATE_KEY=$(grep REPLICATE_API_KEY /Users/cashville/.env | cut -d= -f2)
PROMPT="luxury Dubai apartment interior, contemporary warm design, natural materials, warm afternoon light, editorial architecture photography, 8k, photorealistic"

PREDICTION=$(curl -s -X POST "https://api.replicate.com/v1/predictions" \
  -H "Authorization: Token ${REPLICATE_KEY}" \
  -H "Content-Type: application/json" \
  -d "{
    \"version\": \"06d6fae3b75ab68a28cd2900afa6033166910dd09fd9751047043592f8f7cc60\",
    \"input\": {
      \"prompt\": \"${PROMPT}\",
      \"negative_prompt\": \"cheap, IKEA, cartoon, oversaturated, cold, plastic, watermark\",
      \"width\": 1920,
      \"height\": 1080,
      \"num_inference_steps\": 50,
      \"guidance_scale\": 7.5,
      \"num_outputs\": 3
    }
  }")

PREDICTION_ID=$(echo $PREDICTION | python3 -c "import sys,json; print(json.load(sys.stdin)['id'])")

# Poll for result
while true; do
  RESULT=$(curl -s "https://api.replicate.com/v1/predictions/${PREDICTION_ID}" \
    -H "Authorization: Token ${REPLICATE_KEY}")
  STATUS=$(echo $RESULT | python3 -c "import sys,json; print(json.load(sys.stdin)['status'])")
  if [ "$STATUS" = "succeeded" ]; then
    echo $RESULT | python3 -c "import sys,json; [print(u) for u in json.load(sys.stdin)['output']]"
    break
  elif [ "$STATUS" = "failed" ]; then
    echo "FAILED"
    break
  fi
  echo "Status: $STATUS: waiting 5s..."
  sleep 5
done
```

### Step 4: Animate Renders via Runway Gen-3 (cinematic tour)

No real footage exists for off-plan. The cinematic tour is built entirely from animated renders. Produce two tours: one per direction.

#### 4A: Upload render to ImgBB (Runway needs a public URL)

```bash
IMGBB_KEY=$(grep IMGBB_API_KEY /Users/cashville/.env | cut -d= -f2)
PROJECT_ID="LIOR-SP2601"
WORKDIR=".tmp/${PROJECT_ID}-space-preview"

# Upload one render (repeat for each render to animate)
PHOTO="${WORKDIR}/03-renders/direction-A/${PROJECT_ID}-render-living-A-v1.jpg"
IMAGE_URL=$(curl -s -X POST "https://api.imgbb.com/1/upload" \
  -F "key=${IMGBB_KEY}" \
  -F "image=@${PHOTO}" \
  | python3 -c "import sys,json; r=json.load(sys.stdin); print(r['data']['url'])")
echo "Hosted URL: ${IMAGE_URL}"
```

#### 4B: Animate with Runway Gen-3

```bash
RUNWAY_KEY=$(grep RUNWAY_API_KEY /Users/cashville/.env | cut -d= -f2)
STAGED_IMAGE_URL="${IMAGE_URL}"  # from step 4A
ROOM="living"
DIRECTION="A"

JOB=$(curl -s -X POST "https://api.runwayml.com/v1/image_to_video" \
  -H "Authorization: Bearer ${RUNWAY_KEY}" \
  -H "Content-Type: application/json" \
  -d "{
    \"model\": \"gen4_turbo\",
    \"promptImage\": \"${STAGED_IMAGE_URL}\",
    \"promptText\": \"Extremely slow cinematic push forward, warm interior light, smooth steady camera, no shake, 24fps, photorealistic\",
    \"duration\": 8,
    \"ratio\": \"1280:768\",
    \"seed\": 42
  }")

JOB_ID=$(echo $JOB | python3 -c "import sys,json; print(json.load(sys.stdin)['id'])")
echo "Runway job ID: ${JOB_ID}"

# Poll until complete
while true; do
  RESULT=$(curl -s "https://api.runwayml.com/v1/tasks/${JOB_ID}" \
    -H "Authorization: Bearer ${RUNWAY_KEY}")
  STATUS=$(echo $RESULT | python3 -c "import sys,json; print(json.load(sys.stdin).get('status','unknown'))")
  if [ "$STATUS" = "SUCCEEDED" ]; then
    VIDEO_URL=$(echo $RESULT | python3 -c "import sys,json; print(json.load(sys.stdin)['output'][0])")
    curl -L "${VIDEO_URL}" -o "${WORKDIR}/04-cinematic/${PROJECT_ID}-clip-${ROOM}-${DIRECTION}-v1.mp4"
    echo "Downloaded: ${ROOM} Direction ${DIRECTION}"
    break
  elif [ "$STATUS" = "FAILED" ]; then
    echo "FAILED: ${ROOM} Direction ${DIRECTION}"
    break
  fi
  echo "Status: $STATUS: waiting 15s..."
  sleep 15
done
```

**Runway fallback: Ken Burns (if Runway fails after 3 attempts):**

```bash
STILL="${WORKDIR}/03-renders/direction-A/${PROJECT_ID}-render-living-A-v1.jpg"
ffmpeg -loop 1 -i "${STILL}" \
  -vf "scale=3840:2160,zoompan=z='min(zoom+0.0006,1.12)':x='iw/2-(iw/zoom/2)':y='ih/2-(ih/zoom/2)':d=192:s=1920x1080,fps=24" \
  -t 8 -pix_fmt yuv420p \
  "${WORKDIR}/04-cinematic/${PROJECT_ID}-clip-living-A-fallback.mp4"
echo "Ken Burns fallback applied"
echo "[$(date '+%Y-%m-%d %H:%M')] FALLBACK Runway → Ken Burns for living Direction A [${PROJECT_ID}]" >> "${WORKDIR}/log.txt"
```

#### 4C: Assemble cinematic tour per direction

```bash
PROJECT_ID="LIOR-SP2601"
WORKDIR=".tmp/${PROJECT_ID}-space-preview"
DIRECTION="A"  # run again for B

# Create concat list
cat > /tmp/concat-${DIRECTION}.txt << EOF
file '${WORKDIR}/04-cinematic/${PROJECT_ID}-clip-living-${DIRECTION}-v1.mp4'
file '${WORKDIR}/04-cinematic/${PROJECT_ID}-clip-master-${DIRECTION}-v1.mp4'
file '${WORKDIR}/04-cinematic/${PROJECT_ID}-clip-kitchen-${DIRECTION}-v1.mp4'
file '${WORKDIR}/04-cinematic/${PROJECT_ID}-clip-room4-${DIRECTION}-v1.mp4'
EOF

# Add dissolve transitions and assemble
ffmpeg -f concat -safe 0 -i /tmp/concat-${DIRECTION}.txt \
  -vf "xfade=transition=dissolve:duration=0.6:offset=7.4,xfade=transition=dissolve:duration=0.6:offset=15.2" \
  -c:v libx264 -preset slow -crf 20 -pix_fmt yuv420p \
  "${WORKDIR}/06-exports/Direction-${DIRECTION}/${PROJECT_ID}-cinematic-direction-${DIRECTION}-v1.mp4"
echo "Cinematic tour Direction ${DIRECTION} assembled"
```

**Assembly structure (renders-only):**
- Fade in → Room 1 animated → Room 2 animated → Room 3 animated → Room 4 animated → Fade out
- Transitions: dissolve 0.6s between rooms
- Total target: 40–55s (shorter than hybrid tour: no connective real footage)

### Step 5: Social Assets

#### Still crops

```bash
PROJECT_ID="LIOR-SP2601"
WORKDIR=".tmp/${PROJECT_ID}-space-preview"

for DIRECTION in A B; do
  HERO="${WORKDIR}/03-renders/direction-${DIRECTION}/${PROJECT_ID}-render-living-${DIRECTION}-v1.jpg"

  # 16:9 hero still (already at correct ratio from render)
  cp "${HERO}" "${WORKDIR}/06-exports/Social/${PROJECT_ID}-hero-${DIRECTION}-v1.jpg"

  # 1:1 square crop for Instagram
  magick "${HERO}" -gravity Center -crop 1:1 +repage \
    "${WORKDIR}/06-exports/Social/${PROJECT_ID}-sq-${DIRECTION}-v1.jpg"
done
```

#### 30s vertical reel per direction

```bash
for DIRECTION in A B; do
  ffmpeg -i "${WORKDIR}/06-exports/Direction-${DIRECTION}/${PROJECT_ID}-cinematic-direction-${DIRECTION}-v1.mp4" \
    -vf "crop=ih*9/16:ih:(iw-ih*9/16)/2:0, scale=1080:1920" \
    -t 30 -c:v libx264 -preset slow -crf 24 \
    -c:a aac -b:a 128k -movflags +faststart \
    "${WORKDIR}/06-exports/Social/${PROJECT_ID}-reel-direction-${DIRECTION}-v1.mp4"
done
```

### Step 6: Color Grade (ImageMagick)

```bash
for DIRECTION in A B; do
  for ROOM in living master kitchen room4; do
    INPUT="${WORKDIR}/03-renders/direction-${DIRECTION}/${PROJECT_ID}-render-${ROOM}-${DIRECTION}-v1.jpg"
    if [ -f "${INPUT}" ]; then
      magick "${INPUT}" \
        -modulate 100,95,100 \
        -color-matrix "1.03 0 0 0 0.99 0 0 0 0.91" \
        -strip \
        "${WORKDIR}/06-exports/Direction-${DIRECTION}/${PROJECT_ID}-render-${ROOM}-${DIRECTION}-v1.jpg"
    fi
  done
done
echo "Color grade applied to all hero renders"
```

---

## VI. QA CHECKLIST

Before delivery, human verifies:

**Renders:**
- [ ] Both directions feel clearly distinct from each other
- [ ] Renders are photorealistic: not visibly AI-generated
- [ ] No artifacts or floating objects in hero images
- [ ] Color palette from each direction brief is visible in the renders
- [ ] Client would recognize the space from their floor plan (rooms match layout)
- [ ] Natural light enters from correct side (matches floor plan window positions)

**Tours:**
- [ ] Both cinematic tours play correctly end-to-end
- [ ] Transitions are smooth
- [ ] No clip has camera shake or visual glitches
- [ ] Direction A and B tours feel tonally different

**Social assets:**
- [ ] Reel crops correctly (subject not cut off)
- [ ] Square crops centered on hero element
- [ ] No watermarks or artifacts

---

## VII. DELIVERY

### ZIP

```bash
PROJECT_ID="LIOR-SP2601"
WORKDIR=".tmp/${PROJECT_ID}-space-preview"

cd "${WORKDIR}"
zip -r "${PROJECT_ID}-space-preview-v1.zip" "06-exports/"
echo "ZIP size: $(wc -c < "${PROJECT_ID}-space-preview-v1.zip") bytes"
unzip -l "${PROJECT_ID}-space-preview-v1.zip"
```

### WeTransfer Upload

```bash
WT_KEY=$(grep WETRANSFER_API_KEY /Users/cashville/.env | cut -d= -f2)
ZIP_FILE="${WORKDIR}/${PROJECT_ID}-space-preview-v1.zip"
ZIP_SIZE=$(wc -c < "${ZIP_FILE}")

TRANSFER=$(curl -s -X POST "https://dev.wetransfer.com/v2/transfers" \
  -H "Content-Type: application/json" -H "x-api-key: ${WT_KEY}" \
  -d "{\"message\":\"${PROJECT_ID}: Space Preview LIOR\",\"files\":[{\"name\":\"${PROJECT_ID}-space-preview-v1.zip\",\"size\":${ZIP_SIZE}}]}")

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
DIR_A_NAME="[Direction A name]"
DIR_A_DESC="[Direction A one-line description]"
DIR_B_NAME="[Direction B name]"
DIR_B_DESC="[Direction B one-line description]"

MSG="${CLIENT_NAME}: your Space Preview is ready.

Two complete design directions for your space:

Direction A: ${DIR_A_NAME}: ${DIR_A_DESC}
Direction B: ${DIR_B_NAME}: ${DIR_B_DESC}

Each includes room visualizations, a cinematic tour, and social-ready content.

Download (valid 7 days): ${DOWNLOAD_URL}

Let us know which direction resonates: or if you would like to mix elements from both."

ENCODED=$(python3 -c "import urllib.parse,sys; print(urllib.parse.quote(sys.argv[1]))" "${MSG}")
curl -s "https://api.callmebot.com/whatsapp.php?phone=${PHONE}&text=${ENCODED}&apikey=${APIKEY}"
echo "WhatsApp sent."
```

### Notion Update

```bash
NOTION_TOKEN=$(grep NOTION_TOKEN /Users/cashville/.env | cut -d= -f2)
PAGE_ID="[ID_NOTION: from project Notion page]"

curl -s -X PATCH "https://api.notion.com/v1/pages/${PAGE_ID}" \
  -H "Authorization: Bearer ${NOTION_TOKEN}" \
  -H "Notion-Version: 2022-06-28" \
  -H "Content-Type: application/json" \
  -d "{\"properties\":{\"Status\":{\"select\":{\"name\":\"Delivered\"}},\"Delivery Link\":{\"url\":\"${DOWNLOAD_URL}\"},\"Delivered At\":{\"date\":{\"start\":\"$(date -u +%Y-%m-%d)\"}}}}"

echo "Notion updated."
```

### Delivery Log

```bash
echo "[$(date '+%Y-%m-%d %H:%M')] DELIVERED: ${PROJECT_ID} | Service: space-preview | ZIP: ${PROJECT_ID}-space-preview-v1.zip | Link: ${DOWNLOAD_URL} | WA: sent" \
  >> "${WORKDIR}/log.txt"
```

---

## VIII. WHATSAPP TEMPLATE

```
[CLIENT NAME]: your Space Preview is ready.

Two complete design directions for your space:

Direction A: [name]: [one-line description]
Direction B: [name]: [one-line description]

Each includes room visualizations, a cinematic tour, and social-ready content.

Download (valid 7 days): [LINK]

Let us know which direction resonates: or if you would like to mix elements from both.
```

Rules: no "AI", no "luxury", no prices, no "drone".

---

## IX. FALLBACKS

| Tool | Failure | Fix |
|---|---|---|
| Adobe Firefly | Token expired | Re-run token script (Step 3A). Token valid 24h. |
| Adobe Firefly | API down | Use Replicate SDXL (Step 3C) |
| Runway Gen-3 | Job failed after 3 attempts | Use Ken Burns FFmpeg fallback (Step 4B). Log it. |
| WeTransfer API | Upload fails | Go to wetransfer.com manually, upload ZIP, copy link |
| WeTransfer API | File >2GB | Split into two ZIPs: renders in one, videos in another |
| WeTransfer down | Completely unavailable | Upload to Google Drive, enable "anyone with link can view", share that link |
| CallMeBot | Phone not registered | Send "I allow callmebot to send me messages" to +34 644 29 73 73 on WhatsApp |
| CallMeBot | Rate limit | Wait 5 seconds between messages |
| CallMeBot | Message too long | Truncate to 300 characters max |
| ImgBB | Upload fails | Try again after 30s. 32MB max per image. |

---

## X. LOG FORMAT

```
[YYYY-MM-DD HH:MM] STARTED: [PROJECT_ID] | rooms=[N] | directions=A+B
[YYYY-MM-DD HH:MM] RENDERS: Direction A complete: living, master, kitchen, [room4]
[YYYY-MM-DD HH:MM] RENDERS: Direction B complete: living, master, kitchen, [room4]
[YYYY-MM-DD HH:MM] ANIMATION: Direction A tour assembled: [duration]s
[YYYY-MM-DD HH:MM] ANIMATION: Direction B tour assembled: [duration]s
[YYYY-MM-DD HH:MM] FALLBACK: [tool] → [alternative] for [room] [PROJECT_ID]
[YYYY-MM-DD HH:MM] QA: APPROVED by [name]
[YYYY-MM-DD HH:MM] DELIVERED: [PROJECT_ID] | ZIP: [filename] | Link: [URL] | WA: sent
```

---

## UPSELL TRIGGER

If client selects a preferred direction, agent sends 24h later:

```
Glad Direction [A/B] resonates. We can take this into a full design concept:
every material specified, every room finalized, contractor-ready documents.
Would that be useful for your project?
```

→ Leads to SOP-09 (Design Concept) or SOP-05 (Full Interior Design).
