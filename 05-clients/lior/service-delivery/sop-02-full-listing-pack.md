# SOP 02: Full Listing Pack
**Automation: Auto + gate humain (B)**  ·  **Timeline: 72h**  ·  **Livrable: 8 pièces meublées · 1 tour cinématique · 3 films sociaux**

---

## I. OVERVIEW

Pipeline : photos vides + vidéo walkthrough → 3 tracks parallèles → gate humain → livraison.

```
Track A: 8 pièces staged (SOP-01 × 8)
Track B: Tour cinématique (sop-higgfield.md complet)  ← dépend Track A
Track C: Extraction raw footage (dès réception vidéo)
                    ↓
              Gate humain
                    ↓
         3 films sociaux (coupes du tour)
                    ↓
                Livraison
```

---

## II. INPUTS

| # | Input | Format | Bloquant |
|---|-------|--------|---------|
| 1 | Photos vides toutes pièces | JPG/PNG ≥ 1920×1080 | OUI |
| 2 | Vidéo walkthrough | MP4/MOV ≥ 720p, 3–7 min brute | OUI |
| 3 | Brief propriété | Adresse, pièces, acheteur cible | OUI |
| 4 | Style | Keyword | Non |

**Si photos manquantes → envoyer le brief photo (Annex B de sop-higgfield.md).**
**Si vidéo manquante → envoyer le brief filmage (Annex A de sop-higgfield.md).**
Les deux doivent être reçus avant de démarrer quelque production que ce soit.

---

## III. SETUP (une seule fois par machine)

```bash
# Vérifier que tous les outils sont installés
which ffmpeg    || brew install ffmpeg
which magick    || brew install imagemagick
which ffprobe   || brew install ffmpeg  # inclus avec ffmpeg
python3 -c "import json" && echo "python3 OK"

# Vérifier les clés dans .env
grep -E "IMGBB_API_KEY|VSTAGING_API_KEY|RUNWAY_API_KEY|WETRANSFER_API_KEY|CALLMEBOT_PHONE|CALLMEBOT_API_KEY|NOTION_TOKEN" \
  /Users/cashville/.env

# Si une clé manque :
# IMGBB: https://imgbb.com → Account → API
# VSTAGING: https://virtualstaging.ai → Settings → API Keys
# RUNWAY: https://app.runwayml.com → Account → API Keys
# WETRANSFER: https://wetransfer.com → Account → API
# CALLMEBOT: envoyer "I allow callmebot to send me messages" au +34 644 29 73 73
```

---

## IV. STRUCTURE DE DOSSIERS

```bash
PROJECT_ID="LIOR-DM2601"  # ← adapter

mkdir -p .tmp/${PROJECT_ID}-pack/{00-brief,01-client-inputs/{photos,video},02-staging,03-raw-cuts,04-animated-clips,05-assembly,06-audio,07-social,08-exports}
touch .tmp/${PROJECT_ID}-pack/log.txt

echo "[$(date '+%Y-%m-%d %H:%M')] START ${PROJECT_ID}: Full Listing Pack" >> .tmp/${PROJECT_ID}-pack/log.txt
```

Placer les fichiers client :
```bash
# Copier les photos dans 01-client-inputs/photos/ avec nommage normalisé
# living.jpg · master.jpg · kitchen.jpg · dining.jpg · ensuite.jpg · study.jpg · bedroom2.jpg · balcony.jpg

# Copier la vidéo dans 01-client-inputs/video/
# walkthrough.mp4
```

---

## V. TRACK A: STAGING (8 pièces)

Exécuter exactement **SOP-01** pour les 8 pièces.
Dossier de sortie : `.tmp/${PROJECT_ID}-pack/02-staging/`

Ajustements vs SOP-01 :
- 8 pièces (pas 5)
- Le style doit être **identique pour les 8 pièces**: le verrouiller après validation de la première
- Le color matching Lightroom/ImageMagick est **obligatoire** (pas optionnel) sur 8 pièces

**Ordre de priorité si moins de 8 photos utilisables :**
1. Salon · 2. Chambre principale · 3. Cuisine · 4. Salle de bain principale · 5. Salle à manger · 6. Chambre 2 · 7. Bureau · 8. Balcon

**Track A complète quand :** 8 JPGs approuvés dans `02-staging/`, color-matchés.

```bash
# Vérifier que les 8 pièces sont bien là
ls -1 .tmp/${PROJECT_ID}-pack/02-staging/*.jpg | wc -l
# Doit afficher 8
```

---

## VI. TRACK C: EXTRACTION FOOTAGE (démarre en parallèle de Track A)

Pendant que le staging génère, extraire et ralentir les clips raw.

### 6.1 Inventaire footage
```bash
# Vérifier les specs de la vidéo client
ffprobe -v quiet -print_format json -show_streams \
  .tmp/${PROJECT_ID}-pack/01-client-inputs/video/walkthrough.mp4 \
  | python3 -c "
import sys,json
d=json.load(sys.stdin)
v=[s for s in d['streams'] if s['codec_type']=='video'][0]
print('Resolution:', v['width'], 'x', v['height'])
print('Duration:', float(v.get('duration',0)), 's')
print('FPS:', v.get('r_frame_rate','?'))
"

# Regarder la vidéo, noter les timecodes dans log.txt :
# [ROOM] [START] [END] [QUALITÉ: KEEP/B-ROLL/SKIP]
```

### 6.2 Extraire les clips
```bash
# Extraction avec timestamps: adapter selon inventaire
ffmpeg -i .tmp/${PROJECT_ID}-pack/01-client-inputs/video/walkthrough.mp4 \
  -ss 00:03:28 -t 00:00:34 -c:v copy -c:a copy \
  .tmp/${PROJECT_ID}-pack/03-raw-cuts/00-opening.mp4

ffmpeg -i .tmp/${PROJECT_ID}-pack/01-client-inputs/video/walkthrough.mp4 \
  -ss 00:00:28 -t 00:00:37 -c:v copy -c:a copy \
  .tmp/${PROJECT_ID}-pack/03-raw-cuts/01-living-empty.mp4

# → Répéter pour chaque pièce identifiée dans l'inventaire
```

### 6.3 Ralentir les clips (80% de la vitesse originale)
```bash
for CLIP in .tmp/${PROJECT_ID}-pack/03-raw-cuts/*.mp4; do
  NAME=$(basename "$CLIP" .mp4)
  ffmpeg -i "${CLIP}" \
    -vf "setpts=1.25*PTS" \
    -af "atempo=0.8" \
    ".tmp/${PROJECT_ID}-pack/03-raw-cuts/${NAME}-slow.mp4"
  echo "  ✓ Ralenti: ${NAME}"
done
```

**Track C complète quand :** tous les clips `-slow.mp4` sont dans `03-raw-cuts/`.

---

## VII. TRACK B: TOUR CINÉMATIQUE

**Démarre uniquement après Track A et Track C complètes.**

Exécuter **`sop-higgfield.md` Phases 0 → 7** en intégralité.

Paramètres spécifiques Full Listing Pack :
- Staging stills input : `02-staging/*.jpg` (8 pièces)
- Raw cuts input : `03-raw-cuts/*-slow.mp4`
- Durée cible : 60–70s (plus long que le minimum: 8 pièces)
- Output : `.tmp/${PROJECT_ID}-pack/08-exports/${PROJECT_ID}-cinematic-v1.mp4`

**Animer chaque staging still via Runway Gen-3 :**

```bash
RUNWAY_KEY=$(grep RUNWAY_API_KEY /Users/cashville/.env | cut -d= -f2)
IMGBB_KEY=$(grep IMGBB_API_KEY /Users/cashville/.env | cut -d= -f2)

# Motion prompts par type de pièce
declare -A MOTION_PROMPTS
MOTION_PROMPTS[living]="Extremely slow cinematic push forward into the living room, natural warm light, shallow depth of field, luxury interior photography, smooth steady camera, no shake, 24fps"
MOTION_PROMPTS[master]="Very slow push into bedroom through doorway, soft diffused light on bedding, luxury hotel quality, steady smooth camera, warm tones, cinematic"
MOTION_PROMPTS[kitchen]="Slow pan right across kitchen, warm light on surfaces, island in foreground, smooth tracking motion, architectural photography style"
MOTION_PROMPTS[dining]="Very slow orbit around dining table, warm pendant light above, elegant, smooth cinematic tracking"
MOTION_PROMPTS[ensuite]="Slow reveal push into bathroom, warm soft light on tiles, spa quality, mirror reflection, smooth cinematic movement"
MOTION_PROMPTS[balcony]="Slow push forward toward the view, glass railing catching light, city visible beyond, smooth glide, golden hour quality"

for STILL in .tmp/${PROJECT_ID}-pack/02-staging/*.jpg; do
  ROOM=$(basename "$STILL" .jpg)
  PROMPT="${MOTION_PROMPTS[$ROOM]:-Extremely slow cinematic push forward, natural warm light, luxury interior, smooth steady camera, no shake, 24fps}"

  # Héberger l'image
  IMG_URL=$(curl -s -X POST "https://api.imgbb.com/1/upload" \
    -F "key=${IMGBB_KEY}" \
    -F "image=@${STILL}" \
    | python3 -c "import sys,json; print(json.load(sys.stdin)['data']['url'])")

  # Lancer l'animation (X-Runway-Version requis, api.runwayml.com sans "dev")
  RESPONSE=$(curl -s -X POST "https://api.runwayml.com/v1/image_to_video" \
    -H "Authorization: Bearer ${RUNWAY_KEY}" \
    -H "X-Runway-Version: 2024-11-06" \
    -H "Content-Type: application/json" \
    -d "{
      \"model\": \"gen4_turbo\",
      \"promptImage\": \"${IMG_URL}\",
      \"promptText\": \"${PROMPT}\",
      \"duration\": 8,
      \"ratio\": \"1280:768\",
      \"seed\": 42
    }")

  TASK_ID=$(echo $RESPONSE | python3 -c "import sys,json; print(json.load(sys.stdin).get('id','ERROR'))")
  echo "${ROOM}=${TASK_ID}" >> .tmp/${PROJECT_ID}-pack/runway-jobs.txt
  echo "  → Runway lancé: ${ROOM} (${TASK_ID})"
  sleep 3
done
```

**Polling Runway :**
```bash
RUNWAY_KEY=$(grep RUNWAY_API_KEY /Users/cashville/.env | cut -d= -f2)
mkdir -p .tmp/${PROJECT_ID}-pack/04-animated-clips

while IFS='=' read -r ROOM TASK_ID; do
  [ -z "$ROOM" ] && continue
  echo "=== Polling Runway: ${ROOM} (${TASK_ID}) ==="
  ATTEMPTS=0

  while true; do
    RESULT=$(curl -s "https://api.runwayml.com/v1/tasks/${TASK_ID}" \
      -H "Authorization: Bearer ${RUNWAY_KEY}" \
      -H "X-Runway-Version: 2024-11-06")
    STATUS=$(echo $RESULT | python3 -c "import sys,json; print(json.load(sys.stdin).get('status','unknown'))")

    if [ "$STATUS" = "SUCCEEDED" ]; then
      VIDEO_URL=$(echo $RESULT | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['output'][0])")
      curl -s -L "${VIDEO_URL}" -o ".tmp/${PROJECT_ID}-pack/04-animated-clips/${ROOM}-staged.mp4"
      echo "  ✓ Animé: ${ROOM}"
      break
    elif [ "$STATUS" = "FAILED" ]; then
      echo "  ✗ Runway FAILED: ${ROOM} → Ken Burns fallback"
      # Fallback Ken Burns
      ffmpeg -loop 1 -i ".tmp/${PROJECT_ID}-pack/02-staging/${ROOM}.jpg" \
        -vf "scale=3840:2160,zoompan=z='min(zoom+0.0006,1.12)':x='iw/2-(iw/zoom/2)':y='ih/2-(ih/zoom/2)':d=192:s=1920x1080,fps=24" \
        -t 8 -pix_fmt yuv420p \
        ".tmp/${PROJECT_ID}-pack/04-animated-clips/${ROOM}-staged.mp4"
      echo "  ✓ Ken Burns fallback: ${ROOM}"
      break
    fi

    ((ATTEMPTS++))
    [ $ATTEMPTS -ge 60 ] && echo "TIMEOUT ${ROOM}" && break
    echo "  ${STATUS} (${ATTEMPTS}/60): 10s..."
    sleep 10
  done

done < .tmp/${PROJECT_ID}-pack/runway-jobs.txt
```

**Assemblage, color grade, audio, export : suivre sop-higgfield.md Phases 4→7 exactement.**
Output final dans `08-exports/` :
- `${PROJECT_ID}-cinematic-v1.mp4` (16:9, 60–70s)
- `${PROJECT_ID}-cinematic-reels-v1.mp4` (9:16, 30–45s)
- `${PROJECT_ID}-cinematic-sq-v1.mp4` (1:1, 30–45s)

---

## VIII. GATE HUMAIN

Avant de produire les films sociaux, révision humaine :
- [ ] 8 pièces staged : style cohérent, pas d'artefact visible
- [ ] Tour cinématique joue correctement du début au loop
- [ ] Durée : 60–70s
- [ ] Colour grade unifie raw footage et stills AI

**Si approbation → continuer. Si non → instructions précises à l'agent → re-run du composant concerné uniquement.**

---

## IX. FILMS SOCIAUX (3 films: après gate)

**Film 1: Reel Instagram/TikTok (30s, 9:16)**
```bash
ffmpeg -i .tmp/${PROJECT_ID}-pack/08-exports/${PROJECT_ID}-cinematic-v1.mp4 \
  -vf "crop=ih*9/16:ih:(iw-ih*9/16)/2:0, scale=1080:1920" \
  -t 30 \
  -c:v libx264 -preset slow -crf 24 \
  -c:a aac -b:a 128k \
  -movflags +faststart \
  .tmp/${PROJECT_ID}-pack/07-social/${PROJECT_ID}-reel-30s-v1.mp4
```

**Film 2: Teaser 15s (9:16)**
```bash
ffmpeg -i .tmp/${PROJECT_ID}-pack/08-exports/${PROJECT_ID}-cinematic-v1.mp4 \
  -ss 00:00:00 -t 00:00:15 \
  -vf "crop=ih*9/16:ih:(iw-ih*9/16)/2:0, scale=1080:1920" \
  -c:v libx264 -preset slow -crf 24 \
  -c:a aac -b:a 128k \
  -movflags +faststart \
  .tmp/${PROJECT_ID}-pack/07-social/${PROJECT_ID}-teaser-15s-v1.mp4
```

**Film 3: Listing portals (30s, 16:9)**
```bash
ffmpeg -i .tmp/${PROJECT_ID}-pack/08-exports/${PROJECT_ID}-cinematic-v1.mp4 \
  -ss 00:00:00 -t 00:00:30 \
  -c:v libx264 -preset slow -crf 23 \
  -c:a aac -b:a 128k \
  -movflags +faststart \
  .tmp/${PROJECT_ID}-pack/07-social/${PROJECT_ID}-listing-30s-v1.mp4
```

**Copier les films sociaux dans exports :**
```bash
cp .tmp/${PROJECT_ID}-pack/07-social/*.mp4 .tmp/${PROJECT_ID}-pack/08-exports/
```

**Exporter aussi les crops carrés des staged stills (depuis Track A) :**
```bash
for FILE in .tmp/${PROJECT_ID}-pack/02-staging/*.jpg; do
  ROOM=$(basename "$FILE" .jpg)
  
  magick "${FILE}" -resize "2000x>" \
    ".tmp/${PROJECT_ID}-pack/08-exports/${PROJECT_ID}-staged-${ROOM}-v1.jpg"

  magick "${FILE}" -gravity Center -crop 1:1 +repage -resize "1080x1080" \
    ".tmp/${PROJECT_ID}-pack/08-exports/${PROJECT_ID}-staged-${ROOM}-sq-v1.jpg"
done
```

---

## X. CHECKLIST FICHIERS FINAUX

```
08-exports/ doit contenir 22 fichiers :
├── 8 × [ID]-staged-[room]-v1.jpg          (16:9)
├── 8 × [ID]-staged-[room]-sq-v1.jpg       (1:1)
├── [ID]-cinematic-v1.mp4                  (16:9, 60–70s)
├── [ID]-cinematic-reels-v1.mp4            (9:16, 30–45s)
├── [ID]-cinematic-sq-v1.mp4               (1:1, 30–45s)
├── [ID]-reel-30s-v1.mp4                   (9:16, 30s)
├── [ID]-teaser-15s-v1.mp4                 (9:16, 15s)
└── [ID]-listing-30s-v1.mp4                (16:9, 30s)
```

```bash
echo "Fichiers dans exports:"
ls -1 .tmp/${PROJECT_ID}-pack/08-exports/ | wc -l
ls -lh .tmp/${PROJECT_ID}-pack/08-exports/
```

---

## XI. DELIVERY

### 11.1 ZIP
```bash
cd .tmp/${PROJECT_ID}-pack/
zip -r "${PROJECT_ID}-listing-pack-v1.zip" "08-exports/"
echo "ZIP: $(du -sh ${PROJECT_ID}-listing-pack-v1.zip)"
```

### 11.2 WeTransfer

```bash
WT_KEY=$(grep WETRANSFER_API_KEY /Users/cashville/.env | cut -d= -f2)
ZIP_FILE=".tmp/${PROJECT_ID}-pack/${PROJECT_ID}-listing-pack-v1.zip"
ZIP_SIZE=$(wc -c < "${ZIP_FILE}")
ZIP_NAME="${PROJECT_ID}-listing-pack-v1.zip"

# 1: JWT Bearer token (obligatoire)
WT_TOKEN=$(curl -s -X POST "https://dev.wetransfer.com/v2/authorize" \
  -H "Content-Type: application/json" \
  -H "x-api-key: ${WT_KEY}" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['token'])")

# 2: Créer le transfert
TRANSFER=$(curl -s -X POST "https://dev.wetransfer.com/v2/transfers" \
  -H "Content-Type: application/json" \
  -H "x-api-key: ${WT_KEY}" \
  -H "Authorization: Bearer ${WT_TOKEN}" \
  -d "{\"message\":\"${PROJECT_ID}: Full Listing Pack LIOR\",\"files\":[{\"name\":\"${ZIP_NAME}\",\"size\":${ZIP_SIZE}}]}")

TRANSFER_ID=$(echo $TRANSFER | python3 -c "import sys,json; print(json.load(sys.stdin)['id'])")
FILE_ID=$(echo $TRANSFER | python3 -c "import sys,json; print(json.load(sys.stdin)['files'][0]['id'])")
UPLOAD_URL=$(echo $TRANSFER | python3 -c "import sys,json; print(json.load(sys.stdin)['files'][0]['multipart']['url'])")

# 3: Upload
curl -s -X PUT "${UPLOAD_URL}" -H "Content-Type: application/octet-stream" --data-binary @"${ZIP_FILE}"

# 4: Marquer le fichier comme uploadé (OBLIGATOIRE avant finalize)
curl -s -X PUT "https://dev.wetransfer.com/v2/transfers/${TRANSFER_ID}/files/${FILE_ID}/upload-complete" \
  -H "x-api-key: ${WT_KEY}" \
  -H "Authorization: Bearer ${WT_TOKEN}"

# 5: Finaliser
DOWNLOAD_URL=$(curl -s -X PUT "https://dev.wetransfer.com/v2/transfers/${TRANSFER_ID}/finalize" \
  -H "x-api-key: ${WT_KEY}" \
  -H "Authorization: Bearer ${WT_TOKEN}" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['url'])")

echo "DOWNLOAD LINK: ${DOWNLOAD_URL}"
```

### 11.3 WhatsApp
```bash
PHONE=$(grep CALLMEBOT_PHONE /Users/cashville/.env | cut -d= -f2)
APIKEY=$(grep CALLMEBOT_API_KEY /Users/cashville/.env | cut -d= -f2)
CLIENT_NAME="[NOM]"

MSG="${CLIENT_NAME}: votre Full Listing Pack est pret. 8 pieces meublees + tour cinematique + 3 films sociaux. Telechargement (7 jours) : ${DOWNLOAD_URL}"
ENCODED=$(python3 -c "import urllib.parse,sys; print(urllib.parse.quote(sys.argv[1]))" "${MSG}")
curl -s "https://api.callmebot.com/whatsapp.php?phone=${PHONE}&text=${ENCODED}&apikey=${APIKEY}"
echo "WhatsApp envoyé."
```

### 11.4 Notion
```bash
NOTION_TOKEN=$(grep NOTION_TOKEN /Users/cashville/.env | cut -d= -f2)
PAGE_ID="[ID_NOTION]"

curl -s -X PATCH "https://api.notion.com/v1/pages/${PAGE_ID}" \
  -H "Authorization: Bearer ${NOTION_TOKEN}" \
  -H "Notion-Version: 2022-06-28" \
  -H "Content-Type: application/json" \
  -d "{\"properties\":{\"Status\":{\"select\":{\"name\":\"Delivered\"}},\"Delivery Link\":{\"url\":\"${DOWNLOAD_URL}\"},\"Delivered At\":{\"date\":{\"start\":\"$(date -u +%Y-%m-%d)\"}}}}"
```

---

## XII. TIMELINE (72h)

| Heure | Action |
|-------|--------|
| H+0 | Inputs validés: Tracks A, B, C lancés |
| H+0→H+4 | Track A : staging 8 pièces |
| H+0→H+2 | Track C : extraction raw footage |
| H+4 | Track A OK → Track B Phase 3 (animation Runway) |
| H+4→H+9 | Track B : animation 8 stills |
| H+9→H+14 | Track B : assemblage + grade + audio + export |
| H+14 | Gate humain |
| H+15→H+17 | Films sociaux |
| H+17 | Livraison |

---

## XIII. FALLBACKS

| Problème | Action |
|---------|--------|
| Virtual Staging AI down | REimagineHome API: même structure d'appel |
| Runway FAILED sur une pièce | Ken Burns FFmpeg (voir commande dans section VII) |
| Runway down > 2h | Toutes les animations en Ken Burns |
| WeTransfer API échoue | Upload manuel https://wetransfer.com |
| Révision sur une pièce staged impacte le tour | Re-animer uniquement ce clip Runway → re-assembler le tour → re-exporter les 3 formats |
