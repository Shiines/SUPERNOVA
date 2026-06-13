# TOOLS.md — Comment utiliser chaque outil
> Référence technique partagée par tous les SOPs LIOR.
> Chaque section = un outil = comment l'utiliser de A à Z.

---

## 1. HÉBERGEMENT D'IMAGES (requis avant tout appel API)

La plupart des APIs de staging et de génération nécessitent une URL publique pour l'image source — pas un fichier local.

### ImgBB (gratuit, simple)

**Setup (une seule fois):**
1. Créer un compte sur https://imgbb.com
2. Aller dans Account → API → Generate API Key
3. Copier la clé dans `.env` : `IMGBB_API_KEY=xxx`

**Upload d'une image depuis le terminal:**
```bash
# Upload et récupération de l'URL publique
IMGBB_KEY=$(grep IMGBB_API_KEY /Users/cashville/.env | cut -d= -f2)

curl -X POST "https://api.imgbb.com/1/upload" \
  -F "key=${IMGBB_KEY}" \
  -F "image=@/path/to/empty-room.jpg" \
  | python3 -c "import sys,json; r=json.load(sys.stdin); print(r['data']['url'])"
```

**Réponse:**
```json
{
  "data": {
    "url": "https://i.ibb.co/xxxxx/empty-room.jpg",
    "display_url": "https://i.ibb.co/xxxxx/empty-room.jpg"
  },
  "success": true
}
```

Sauvegarder l'URL retournée → utiliser dans les API calls suivants.

**Limites:** 32MB max par image, images stockées 6 mois (suffisant pour un projet).

---

## 2. VIRTUAL STAGING AI

**Site:** https://virtualstaging.ai
**API doc:** https://virtualstaging.ai/api-docs

### Setup (une seule fois)
1. Créer un compte sur virtualstaging.ai
2. S'abonner à un plan Pro ou API (API requiert plan payant)
3. Aller dans Settings → API Keys → Generate
4. Copier dans `.env` : `VSTAGING_API_KEY=xxx`

### Workflow API complet

**Step 1 — Upload et stage:**
```bash
VSTAGING_KEY=$(grep VSTAGING_API_KEY /Users/cashville/.env | cut -d= -f2)
IMAGE_URL="https://i.ibb.co/xxxxx/living-room-empty.jpg"

curl -X POST "https://api.virtualstaging.ai/v1/stage" \
  -H "Authorization: Bearer ${VSTAGING_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "image_url": "'"${IMAGE_URL}"'",
    "room_type": "living_room",
    "style": "modern",
    "furnishing_level": "medium",
    "virtual_tour": false
  }'
```

**Valeurs room_type:** `living_room` · `bedroom` · `kitchen` · `dining_room` · `bathroom` · `office` · `outdoor`

**Valeurs style:** `modern` · `contemporary` · `minimalist` · `scandinavian` · `luxury` · `traditional`

**Réponse immédiate (job_id):**
```json
{
  "job_id": "job_abc123xyz",
  "status": "processing",
  "estimated_time": 45
}
```

**Step 2 — Polling pour le résultat (toutes les 10s):**
```bash
JOB_ID="job_abc123xyz"

# Poll jusqu'à ce que status == "completed"
while true; do
  RESULT=$(curl -s -X GET "https://api.virtualstaging.ai/v1/jobs/${JOB_ID}" \
    -H "Authorization: Bearer ${VSTAGING_KEY}")
  
  STATUS=$(echo $RESULT | python3 -c "import sys,json; print(json.load(sys.stdin)['status'])")
  
  if [ "$STATUS" = "completed" ]; then
    echo $RESULT | python3 -c "import sys,json; print(json.load(sys.stdin)['output_url'])"
    break
  elif [ "$STATUS" = "failed" ]; then
    echo "FAILED: $(echo $RESULT | python3 -c \"import sys,json; print(json.load(sys.stdin).get('error','unknown'))\")"
    break
  fi
  
  echo "Status: $STATUS — waiting 10s..."
  sleep 10
done
```

**Step 3 — Téléchargement du résultat:**
```bash
OUTPUT_URL="https://cdn.virtualstaging.ai/results/job_abc123xyz.jpg"

curl -L "${OUTPUT_URL}" \
  -o ".tmp/[PROJECT-ID]/02-generated/01-living-staged.jpg"
```

### Workflow Web UI (si pas d'accès API)
1. Aller sur https://virtualstaging.ai → Sign In
2. Cliquer "New Project" → "Upload Image"
3. Glisser la photo de la pièce vide
4. Sélectionner Room Type (ex: Living Room)
5. Sélectionner Style (ex: Modern)
6. Cliquer "Stage Room"
7. Attendre 60–120s
8. Télécharger le résultat (bouton Download en bas)
9. Répéter pour chaque pièce

### Limites et coûts
- Chaque staging = 1 crédit
- Prix: ~$2–4 par image selon plan
- Rate limit: 10 requêtes/minute
- Timeout API: si pas de réponse après 5 min → job failed → relancer

---

## 3. REIMAGINEHOME (fallback Virtual Staging)

**Site:** https://reimaginehome.ai
**Utiliser si:** Virtual Staging AI est down ou quota épuisé

### Workflow API
```bash
curl -X POST "https://api.reimaginehome.ai/v1/generate" \
  -H "Authorization: Bearer ${REIMAGINE_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "image_url": "'"${IMAGE_URL}"'",
    "room_type": "living_room",
    "design_style": "contemporary",
    "num_images": 3
  }'
```

Même pattern de polling que Virtual Staging AI (job_id → poll → download).

---

## 4. RUNWAY GEN-3 ALPHA TURBO (animation)

**Déjà documenté en détail dans** `sop-eggsfield.md` Phase 3.
**Référencer directement ce fichier pour l'animation.**

**Setup rapide:**
```bash
# Auth: clé API dans .env
RUNWAY_KEY=$(grep RUNWAY_API_KEY /Users/cashville/.env | cut -d= -f2)

# Appel standard
curl -X POST "https://api.dev.runwayml.com/v1/image_to_video" \
  -H "Authorization: Bearer ${RUNWAY_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gen4_turbo",
    "promptImage": "'"${STAGED_IMAGE_URL}"'",
    "promptText": "Extremely slow cinematic push forward, warm light, luxury interior, smooth steady camera, no shake, 24fps",
    "duration": 8,
    "ratio": "1280:768",
    "seed": 42
  }'
```

**Polling et download:** voir sop-eggsfield.md Phase 3 section complète.

---

## 5. MIDJOURNEY (moodboards, visualisations architecturales)

### Option A — Discord (méthode principale)

**Setup (une seule fois):**
1. Créer un compte Discord si absent
2. Rejoindre le serveur Midjourney: https://discord.gg/midjourney
3. S'abonner à Midjourney Pro: https://www.midjourney.com/account
4. Dans Discord, aller dans n'importe quel canal `#general-X`

**Générer une image:**
```
Dans Discord, taper:
/imagine prompt: [ton prompt complet] --ar 16:9 --style raw --v 6 --q 2
```

**Upscaler le meilleur résultat:**
- Midjourney génère 4 variantes (U1 U2 U3 U4 = upscale 1-4, V1 V2 V3 V4 = variante)
- Cliquer U1, U2, U3 ou U4 sous l'image pour upscaler la variante voulue
- Cliquer l'image upscalée → Open in Browser → clic droit → Save Image As

**Téléchargement automatique (si bot configuré):**
```python
# Script Python pour récupérer l'image depuis le message Discord
import discord, requests, re

# Surveiller les messages de Midjourney Bot dans le canal
# Extraire l'URL de l'image upscalée
# Télécharger en local
```

### Option B — Midjourney API (accès limité, beta)

Si accès API accordé:
```bash
curl -X POST "https://api.midjourney.com/v1/imagine" \
  -H "Authorization: Bearer ${MIDJOURNEY_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "[prompt]",
    "aspect_ratio": "16:9",
    "style_raw": true,
    "version": "6"
  }'
```

**Fallback si Midjourney inaccessible → utiliser Adobe Firefly (voir section 6).**

### Prompts types par usage

**Moodboard design intérieur:**
```
interior design moodboard, contemporary warm Dubai apartment, palette cream / warm oak / brushed brass,
linen textures, flat lay composition with material swatches and furniture references,
editorial lifestyle photography, warm afternoon light, --ar 16:9 --style raw --v 6
```

**Visualisation pièce (off-plan):**
```
[living room], contemporary warm interior design, [palette], [materials],
wide angle from corner, warm afternoon light from floor-to-ceiling windows,
luxury Dubai apartment, editorial architecture photography, photorealistic,
8k, shot on Hasselblad, --ar 16:9 --style raw --v 6 --q 2
```

**Rendu architectural extérieur:**
```
photorealistic architectural visualization, [building type], [material palette],
golden hour, professional architecture photography, Dubai skyline context,
8k resolution, --ar 16:9 --style raw --v 6
```

---

## 6. ADOBE FIREFLY API (fallback Midjourney)

**Site:** https://developer.adobe.com/firefly-api/
**Utiliser si:** Midjourney inaccessible ou pour images commerciales sans restrictions de licence.

### Setup (une seule fois)
1. Créer compte Adobe Developer: https://developer.adobe.com
2. Créer une application → activer "Firefly - Image 3"
3. Récupérer: `client_id` + `client_secret`
4. Copier dans `.env`: `ADOBE_CLIENT_ID=xxx` et `ADOBE_CLIENT_SECRET=xxx`

### Obtenir un access token (expire après 24h)
```bash
ADOBE_CLIENT_ID=$(grep ADOBE_CLIENT_ID /Users/cashville/.env | cut -d= -f2)
ADOBE_CLIENT_SECRET=$(grep ADOBE_CLIENT_SECRET /Users/cashville/.env | cut -d= -f2)

TOKEN=$(curl -X POST "https://ims-na1.adobelogin.com/ims/token/v3" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials&client_id=${ADOBE_CLIENT_ID}&client_secret=${ADOBE_CLIENT_SECRET}&scope=openid,AdobeID,firefly_enterprise,firefly_api" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

echo "Token: $TOKEN"
# Sauvegarder ce token pour les appels suivants (valide 24h)
```

### Générer une image
```bash
curl -X POST "https://firefly-api.adobe.io/v3/images/generate" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "X-Api-Key: ${ADOBE_CLIENT_ID}" \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "[ton prompt]",
    "negativePrompt": "cheap, IKEA, cartoon, oversaturated, cold light, plastic, watermark, floating furniture",
    "size": { "width": 2048, "height": 1152 },
    "numVariations": 3,
    "contentClass": "photo",
    "styles": { "presets": ["photo_realism"] }
  }'
```

**Réponse:**
```json
{
  "outputs": [
    { "image": { "url": "https://pre-signed-url.adobe.com/image1.jpg" } },
    { "image": { "url": "https://pre-signed-url.adobe.com/image2.jpg" } },
    { "image": { "url": "https://pre-signed-url.adobe.com/image3.jpg" } }
  ]
}
```

Télécharger l'URL choisie:
```bash
curl -L "[OUTPUT_URL]" -o ".tmp/[PROJECT-ID]/02-generated/moodboard-v1.jpg"
```

---

## 7. REPLICATE — STABLE DIFFUSION + CONTROLNET (visualisation architecturale avancée)

**Site:** https://replicate.com
**Utiliser pour:** rendu architectural avec contrôle précis de la géométrie (SOP-08)

### Setup (une seule fois)
1. Créer compte sur https://replicate.com
2. Account → API Tokens → Create Token
3. Copier dans `.env`: `REPLICATE_API_KEY=xxx`

### Trouver le bon modèle
Modèle recommandé pour intérieurs architecturaux:
`lucataco/sdxl-controlnet:06d6fae3b75ab68a28cd2900afa6033166910dd09fd9751047043592f8f7cc60`

### Appel API complet
```bash
REPLICATE_KEY=$(grep REPLICATE_API_KEY /Users/cashville/.env | cut -d= -f2)
CONTROL_IMAGE_URL="[URL du plan ou du croquis wireframe]"

# Lancer la prédiction
PREDICTION=$(curl -X POST "https://api.replicate.com/v1/predictions" \
  -H "Authorization: Token ${REPLICATE_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "version": "06d6fae3b75ab68a28cd2900afa6033166910dd09fd9751047043592f8f7cc60",
    "input": {
      "image": "'"${CONTROL_IMAGE_URL}"'",
      "prompt": "luxury Dubai apartment interior, contemporary warm design, natural materials, warm afternoon light, editorial architecture photography, 8k, photorealistic",
      "negative_prompt": "cheap, IKEA, cartoon, oversaturated, cold, plastic, watermark, floating furniture",
      "controlnet_conditioning_scale": 0.85,
      "width": 1920,
      "height": 1080,
      "num_inference_steps": 50,
      "guidance_scale": 7.5,
      "num_outputs": 3
    }
  }')

PREDICTION_ID=$(echo $PREDICTION | python3 -c "import sys,json; print(json.load(sys.stdin)['id'])")
echo "Prediction ID: $PREDICTION_ID"
```

### Polling
```bash
while true; do
  RESULT=$(curl -s "https://api.replicate.com/v1/predictions/${PREDICTION_ID}" \
    -H "Authorization: Token ${REPLICATE_KEY}")
  
  STATUS=$(echo $RESULT | python3 -c "import sys,json; print(json.load(sys.stdin)['status'])")
  
  if [ "$STATUS" = "succeeded" ]; then
    echo $RESULT | python3 -c "import sys,json; [print(u) for u in json.load(sys.stdin)['output']]"
    break
  elif [ "$STATUS" = "failed" ]; then
    echo "FAILED: $(echo $RESULT | python3 -c \"import sys,json; print(json.load(sys.stdin).get('error',''))\")"
    break
  fi
  
  echo "Status: $STATUS — waiting 5s..."
  sleep 5
done
```

---

## 8. ROOMSKETCHER (space planning)

**Site:** https://roomsketcher.com
**Note:** L'API est limitée aux comptes Business. Si pas d'accès API, utiliser le workflow Web UI.

### Workflow Web UI (méthode principale)
1. Aller sur https://roomsketcher.com → Sign In
2. Cliquer "New Floor Plan"
3. Entrer les dimensions de la pièce (longueur × largeur en m)
4. Dessiner les murs avec l'outil "Draw Walls"
5. Ajouter les portes et fenêtres depuis le panneau de gauche
6. Onglet "Furnish" → drag and drop les meubles
   - Sofa: chercher "sofa" → glisser dans le salon
   - Bed: chercher "bed" → glisser dans la chambre
   - Table: chercher "dining table" → etc.
7. Ajuster les dimensions de chaque meuble (double-clic → Properties)
8. Répéter pour Option B (dupliquer le projet → modifier)
9. Aller dans "3D View" pour vérifier que ça rend bien
10. File → Export → Floor Plan → JPG (2D avec meubles et dimensions)

**Export 2D avec cotations:**
- View menu → Show Dimensions: ON
- Export → Floor Plan Image → 2000px minimum

### Planner5D (alternative gratuite)
1. https://planner5d.com → New Project
2. Floor Plan → Add Room → entrer dimensions
3. Catalogue → Furniture → glisser les pièces
4. Export → Image HD

---

## 9. FFMPEG (traitement vidéo)

**Déjà documenté dans sop-eggsfield.md** (Phases 2, 4, 5, 6, 7).

**Commandes complémentaires pour les social films:**

**Extraire une image fixe d'une vidéo (pour staging depuis footage):**
```bash
ffmpeg -i walkthrough.mp4 -ss 00:00:45 -vframes 1 -q:v 2 output-frame.jpg
```

**Crop dynamique (pour trouver le meilleur cadrage vertical):**
```bash
# Prévisualiser le crop 9:16 avant de confirmer
ffmpeg -i cinematic.mp4 \
  -vf "crop=ih*9/16:ih:(iw-ih*9/16)/2:0" \
  -t 5 \
  -f mp4 preview-vertical.mp4
```

**Vérifier les specs techniques d'un fichier:**
```bash
ffprobe -v quiet -print_format json -show_streams cinematic.mp4 \
  | python3 -c "
import sys,json
d=json.load(sys.stdin)
v=[s for s in d['streams'] if s['codec_type']=='video'][0]
print(f\"Resolution: {v['width']}x{v['height']}\")
print(f\"Codec: {v['codec_name']}\")
print(f\"FPS: {v.get('r_frame_rate','unknown')}\")
print(f\"Duration: {v.get('duration','unknown')}s\")
"
```

---

## 10. PANDOC (génération PDF depuis Markdown)

**Utiliser pour:** assembler les documents de livraison (SOP-04, 07, 08, 09, 10).

### Installation
```bash
brew install pandoc
brew install --cask mactex   # Pour le moteur LaTeX (PDF haute qualité)
```

### Générer un PDF simple
```bash
pandoc input.md -o output.pdf --pdf-engine=xelatex
```

### Générer un PDF avec mise en forme (template LIOR)
Créer un fichier de variables YAML en tête du .md:
```markdown
---
title: "Design Concept — LIOR-DC2601"
date: "June 2026"
geometry: "margin=2cm"
fontsize: 11pt
mainfont: "Inter"
---

# Contenu ici...
```

```bash
pandoc brief.md -o [PROJECT-ID]-brief-v1.pdf \
  --pdf-engine=xelatex \
  -V geometry:margin=2cm \
  -V fontsize=11pt \
  --highlight-style=tango
```

### Combiner plusieurs fichiers en un seul PDF
```bash
pandoc cover.md moodboard-section.md rooms-section.md materials.md \
  -o [PROJECT-ID]-design-concept-v1.pdf \
  --pdf-engine=xelatex
```

### Intégrer des images dans le Markdown
```markdown
![Living Room Visualization](.tmp/[ID]/visualizations/living-v1.jpg){ width=100% }
```

---

## 11. PHOTOSHOP (retouche et export)

### Retouche rapide en ligne de commande (ImageMagick — alternative Photoshop)
```bash
brew install imagemagick

# Ajustement couleur (chaud +5 rouge, +3 jaune)
magick input.jpg \
  -modulate 100,95,100 \
  -color-matrix "1.03 0 0 0 0.99 0 0 0 0.91" \
  output.jpg

# Crop carré 1:1
magick input.jpg -gravity Center -crop 1:1 +repage output-sq.jpg

# Redimensionner à 2000px de large
magick input.jpg -resize 2000x output-2000px.jpg

# Suppression des métadonnées
magick input.jpg -strip output-clean.jpg
```

### Photoshop Actions (pour traitement en batch)
Si Photoshop est installé localement, une action peut être créée:
1. Window → Actions → New Action: "LIOR Color Grade"
2. Enregistrer: Image → Adjustments → Color Balance (+5R +3Y midtones)
3. Enregistrer: Curves (légère levée midtones)
4. Enregistrer: File → Export → Export As (JPG, qualité 92)
5. Stop recording
6. Batch: File → Automate → Batch → sélectionner dossier source + destination

---

## 12. LIGHTROOM (correspondance couleur entre pièces)

### Workflow de color matching
1. Ouvrir Lightroom Classic
2. File → Import → sélectionner tout le dossier `03-staging-stills/`
3. Sélectionner la meilleure pièce comme référence (en général le salon)
4. Dans Develop, ajuster:
   - Temperature: tirer vers 4800–5200K (chaud)
   - Tint: légère poussée vers le Magenta (+5)
   - Exposure: +0.1 à +0.2 (légèrement plus lumineux)
   - Highlights: -20 (récupérer les hautes lumières)
   - Shadows: +15 (lever les ombres)
   - Vibrance: -5 (légèrement désaturé — pas d'Instagram)
5. Dans Library, sélectionner toutes les autres images
6. Settings → Paste Settings → cocher White Balance + Tone
7. Vérifier chaque image — ajustements individuels si nécessaire (cuisine et salle de bain souvent plus froide — température à +100K)
8. File → Export → format JPG, qualité 92, espace colorimétrique sRGB, supprimer métadonnées

---

## 13. WETRANSFER (livraison fichiers)

### Setup (une seule fois)
1. Créer compte WeTransfer Pro: https://wetransfer.com
2. Account → API → Create App
3. Copier dans `.env`: `WETRANSFER_API_KEY=xxx`

### Upload via API
```bash
WT_KEY=$(grep WETRANSFER_API_KEY /Users/cashville/.env | cut -d= -f2)

# Step 1 — Créer le transfert
TRANSFER=$(curl -s -X POST "https://dev.wetransfer.com/v2/transfers" \
  -H "Content-Type: application/json" \
  -H "x-api-key: ${WT_KEY}" \
  -d '{
    "message": "[PROJECT-ID] — Livraison LIOR",
    "files": [
      { "name": "[PROJECT-ID]-listing-pack-v1.zip", "size": [TAILLE_EN_OCTETS] }
    ]
  }')

TRANSFER_ID=$(echo $TRANSFER | python3 -c "import sys,json; print(json.load(sys.stdin)['id'])")
UPLOAD_URL=$(echo $TRANSFER | python3 -c "import sys,json; print(json.load(sys.stdin)['files'][0]['multipart']['url'])")

echo "Transfer ID: $TRANSFER_ID"
echo "Upload URL: $UPLOAD_URL"

# Step 2 — Upload le fichier
curl -X PUT "${UPLOAD_URL}" \
  -H "Content-Type: application/octet-stream" \
  --data-binary @".tmp/[PROJECT-ID]/[PROJECT-ID]-listing-pack-v1.zip"

# Step 3 — Finaliser le transfert
RESULT=$(curl -s -X PUT "https://dev.wetransfer.com/v2/transfers/${TRANSFER_ID}/finalize" \
  -H "x-api-key: ${WT_KEY}")

DOWNLOAD_URL=$(echo $RESULT | python3 -c "import sys,json; print(json.load(sys.stdin)['url'])")
echo "Download URL: $DOWNLOAD_URL"
# → Envoyer ce DOWNLOAD_URL au client via CallMeBot
```

### Workflow manuel (si API indisponible)
1. Aller sur https://wetransfer.com
2. Cliquer "Add your files"
3. Glisser le ZIP du projet
4. Entrer le message: "[PROJECT-ID] — Livraison LIOR"
5. Cliquer "Transfer" → obtenir le lien de téléchargement
6. Copier le lien → envoyer via CallMeBot

---

## 14. CALLMEBOT — WHATSAPP NOTIFICATIONS

### Setup (une seule fois, par numéro)
1. Enregistrer le numéro CallMeBot dans les contacts WhatsApp: **+34 644 29 73 73**
2. Envoyer le message exact: `I allow callmebot to send me messages`
3. Attendre la confirmation WhatsApp (1–2 min)
4. Récupérer l'API key fournie dans la réponse
5. Copier dans `.env`: `CALLMEBOT_PHONE=+33XXXXXXXXX` et `CALLMEBOT_API_KEY=xxx`

**Note:** Ce setup est à faire une seule fois par numéro de téléphone (le tien et ceux des clients si tu veux leur notifier directement).

### Envoyer un message
```bash
PHONE=$(grep CALLMEBOT_PHONE /Users/cashville/.env | cut -d= -f2)
APIKEY=$(grep CALLMEBOT_API_KEY /Users/cashville/.env | cut -d= -f2)

MESSAGE="[CLIENT NAME] — votre commande est prête. Download : https://wetransfer.com/xxx"

# Encoder le message (espaces → +, caractères spéciaux → %XX)
ENCODED_MESSAGE=$(python3 -c "import urllib.parse; print(urllib.parse.quote('${MESSAGE}'))")

curl -s "https://api.callmebot.com/whatsapp.php?phone=${PHONE}&text=${ENCODED_MESSAGE}&apikey=${APIKEY}"
```

### Script réutilisable
```bash
#!/bin/bash
# Usage: ./notify.sh "+33XXXXXXXXX" "APIKEY" "Message texte"
# Fichier: .tmp/notify.sh

PHONE=$1
APIKEY=$2
MESSAGE=$3
ENCODED=$(python3 -c "import urllib.parse,sys; print(urllib.parse.quote(sys.argv[1]))" "${MESSAGE}")
curl -s "https://api.callmebot.com/whatsapp.php?phone=${PHONE}&text=${ENCODED}&apikey=${APIKEY}"
```

### Limites
- 1 message par 5 secondes minimum (rate limit)
- Messages longs (>300 chars) : tronquer ou diviser en 2 messages
- Format: texte uniquement (pas de fichiers, pas de liens cliquables sur tous les appareils — coller l'URL en clair)

---

## 15. ZIP ET PACKAGING

### Créer un ZIP de livraison
```bash
PROJECT_ID="LIOR-DM2601"
EXPORT_DIR=".tmp/${PROJECT_ID}/08-exports"

# ZIP avec compression normale
zip -r ".tmp/${PROJECT_ID}/${PROJECT_ID}-listing-pack-v1.zip" "${EXPORT_DIR}/"

# Vérifier le contenu
unzip -l ".tmp/${PROJECT_ID}/${PROJECT_ID}-listing-pack-v1.zip"

# Obtenir la taille en octets (nécessaire pour l'API WeTransfer)
wc -c < ".tmp/${PROJECT_ID}/${PROJECT_ID}-listing-pack-v1.zip"
```

### Vérifier que tous les fichiers attendus sont présents
```bash
# Lister les fichiers dans le ZIP avec leur taille
unzip -l ".tmp/${PROJECT_ID}/${PROJECT_ID}-listing-pack-v1.zip" | awk 'NR>3 && NF>3 {print $4, $1"B"}'
```
