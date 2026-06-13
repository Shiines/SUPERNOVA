# SOP 01 — Virtual Staging
**Automation: Full auto (A)**  ·  **Timeline: 48h**  ·  **Livrable: jusqu'à 5 pièces meublées + crops sociaux**

---

## I. OVERVIEW

Photos vides du client → pièces meublées photorealistic → formats sociaux inclus.
Pipeline : hébergement image → API staging → polling → download → color grade → crop → ZIP → WeTransfer → WhatsApp.

---

## II. INPUTS

| # | Input | Format | Bloquant |
|---|-------|--------|---------|
| 1 | Photos vides (toutes pièces) | JPG/PNG ≥ 1920×1080 | OUI |
| 2 | Nombre de pièces (1–5) | Entier | OUI |
| 3 | Style | Texte ou keyword | Non — défaut auto |
| 4 | Type de bien + acheteur cible | Texte | Non — défaut auto |
| 5 | Project ID | `LIOR-[CODE][AA][SEQ]` | Interne |

**Photo manquante ou rejetée → envoyer ce message WhatsApp :**
```
[NOM] — pour [PIÈCE], la photo reçue est [raison : trop sombre / floue / angle insuffisant].
Pourriez-vous la reprendre depuis l'angle opposé, en journée, téléphone horizontal à hauteur de poitrine ?
On relance dès réception.
```

**Critères de validation photo :**
| Critère | Passe | Échec |
|---------|-------|-------|
| Résolution | ≥ 1920×1080 | < 1280×720 |
| Sol visible | Visible, dégagé | Coupé ou caché |
| Angle | Coin ou diagonal | Face à un mur |
| Exposition | Équilibrée | Brûlée ou noire |
| Netteté | Nette | Floutée |
| Contenu | Vide ou quasi-vide | Meublée / encombrée |

---

## III. SETUP (une seule fois par machine)

### 3.1 ImgBB (hébergement image)
```bash
# 1. Créer un compte sur https://imgbb.com
# 2. Account → API → Generate API Key
# 3. Ajouter dans /Users/cashville/.env :
echo "IMGBB_API_KEY=votre_cle" >> /Users/cashville/.env
```

### 3.2 Virtual Staging AI
```bash
# 1. Créer un compte sur https://virtualstaging.ai → plan Pro/API
# 2. Settings → API Keys → Generate
# 3. Ajouter dans /Users/cashville/.env :
echo "VSTAGING_API_KEY=votre_cle" >> /Users/cashville/.env
```

### 3.3 WeTransfer
```bash
# 1. Créer un compte WeTransfer Pro sur https://wetransfer.com
# 2. Account → API → Create App → copier la clé
echo "WETRANSFER_API_KEY=votre_cle" >> /Users/cashville/.env
```

### 3.4 CallMeBot (WhatsApp)
```bash
# 1. Enregistrer ce numéro dans vos contacts WhatsApp : +34 644 29 73 73
# 2. Envoyer exactement ce message WhatsApp : I allow callmebot to send me messages
# 3. Attendre la réponse (1–2 min) — noter l'API key fournie
echo "CALLMEBOT_PHONE=+33XXXXXXXXX" >> /Users/cashville/.env
echo "CALLMEBOT_API_KEY=votre_cle" >> /Users/cashville/.env
```

### 3.5 Vérifier l'installation des outils
```bash
which ffmpeg || brew install ffmpeg
which magick || brew install imagemagick
python3 -c "import json" && echo "python3 OK"
```

---

## IV. PRODUCTION

### Étape 0 — Créer la structure de dossiers

```bash
PROJECT_ID="LIOR-DM2601"  # ← adapter

mkdir -p .tmp/${PROJECT_ID}-staging/{01-inputs,02-generated,03-retouched,04-exports}
touch .tmp/${PROJECT_ID}-staging/log.txt
touch .tmp/${PROJECT_ID}-staging/url-map.txt
touch .tmp/${PROJECT_ID}-staging/job-map.txt
touch .tmp/${PROJECT_ID}-staging/output-map.txt

echo "[$(date '+%Y-%m-%d %H:%M')] START ${PROJECT_ID} — Virtual Staging" >> .tmp/${PROJECT_ID}-staging/log.txt
```

Copier les photos client dans `01-inputs/` avec nommage normalisé :
```bash
# Exemple de renommage si les fichiers arrivent avec des noms aléatoires
mv .tmp/${PROJECT_ID}-staging/01-inputs/IMG_1234.jpg .tmp/${PROJECT_ID}-staging/01-inputs/living.jpg
mv .tmp/${PROJECT_ID}-staging/01-inputs/IMG_1235.jpg .tmp/${PROJECT_ID}-staging/01-inputs/master.jpg
# etc. — noms valides: living / master / kitchen / dining / ensuite / balcony
```

---

### Étape 1 — Héberger les photos (obtenir des URLs publiques)

Les APIs de staging n'acceptent pas les fichiers locaux — il faut des URLs.

```bash
IMGBB_KEY=$(grep IMGBB_API_KEY /Users/cashville/.env | cut -d= -f2)

for PHOTO in .tmp/${PROJECT_ID}-staging/01-inputs/*.jpg; do
  ROOM=$(basename "$PHOTO" .jpg)
  
  URL=$(curl -s -X POST "https://api.imgbb.com/1/upload" \
    -F "key=${IMGBB_KEY}" \
    -F "image=@${PHOTO}" \
    | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['data']['url'])")
  
  echo "${ROOM}=${URL}" >> .tmp/${PROJECT_ID}-staging/url-map.txt
  echo "  ✓ Hébergé: ${ROOM} → ${URL}"
done

echo "--- URL MAP ---"
cat .tmp/${PROJECT_ID}-staging/url-map.txt
```

---

### Étape 2 — Sélectionner le style

Si le client n'a pas précisé de style, appliquer le défaut selon le type de bien :

| Type de bien | Style API | Densité |
|-------------|-----------|---------|
| Studio / 1BR | `minimalist` | `low` |
| 2BR / 3BR mid-range | `contemporary` | `medium` |
| 3BR+ premium | `modern` | `medium` |
| Penthouse / Villa | `luxury` | `medium` |
| Off-plan / neuf | `modern` | `low` |

**Palette imposée Dubai (dans tous les cas) :** base blanc chaud / crème · bois chaud · laiton brossé · pierre naturelle · lin / bouclé.

---

### Étape 3 — Lancer le staging (une pièce à la fois)

```bash
VSTAGING_KEY=$(grep VSTAGING_API_KEY /Users/cashville/.env | cut -d= -f2)
STYLE="contemporary"    # ← adapter selon étape 2
DENSITY="medium"        # ← adapter selon étape 2

while IFS='=' read -r ROOM IMAGE_URL; do
  [ -z "$ROOM" ] && continue
  echo "=== Staging: ${ROOM} ==="

  # Détecter le type de pièce automatiquement
  case "${ROOM}" in
    living*|salon*) ROOM_TYPE="living_room" ;;
    master*|chambre*) ROOM_TYPE="bedroom" ;;
    kitchen*|cuisine*) ROOM_TYPE="kitchen" ;;
    dining*|salle*) ROOM_TYPE="dining_room" ;;
    bath*|ensuite*|salle_de_bain*) ROOM_TYPE="bathroom" ;;
    balcony*|balcon*|terrasse*) ROOM_TYPE="outdoor" ;;
    *) ROOM_TYPE="living_room" ;;
  esac

  # Lancer 3 variantes (seeds différents)
  for SEED in 42 123 789; do
    RESPONSE=$(curl -s -X POST "https://api.virtualstaging.ai/v1/stage" \
      -H "Authorization: Bearer ${VSTAGING_KEY}" \
      -H "Content-Type: application/json" \
      -d "{
        \"image_url\": \"${IMAGE_URL}\",
        \"room_type\": \"${ROOM_TYPE}\",
        \"style\": \"${STYLE}\",
        \"furnishing_level\": \"${DENSITY}\",
        \"seed\": ${SEED}
      }")

    JOB_ID=$(echo $RESPONSE | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('job_id','ERROR'))")
    
    if [ "$JOB_ID" = "ERROR" ]; then
      echo "  ✗ Erreur API: $(echo $RESPONSE)"
      echo "[$(date '+%Y-%m-%d %H:%M')] ERROR ${ROOM} seed=${SEED}: ${RESPONSE}" >> .tmp/${PROJECT_ID}-staging/log.txt
    else
      echo "${ROOM}_seed${SEED}=${JOB_ID}" >> .tmp/${PROJECT_ID}-staging/job-map.txt
      echo "  → Job lancé: ${JOB_ID} (seed ${SEED})"
    fi
    
    sleep 2  # rate limit
  done

done < .tmp/${PROJECT_ID}-staging/url-map.txt
```

---

### Étape 4 — Polling et téléchargement des résultats

```bash
VSTAGING_KEY=$(grep VSTAGING_API_KEY /Users/cashville/.env | cut -d= -f2)
mkdir -p .tmp/${PROJECT_ID}-staging/02-generated

while IFS='=' read -r KEY JOB_ID; do
  [ -z "$KEY" ] && continue
  echo "=== Polling: ${KEY} (${JOB_ID}) ==="
  ATTEMPTS=0

  while true; do
    RESULT=$(curl -s "https://api.virtualstaging.ai/v1/jobs/${JOB_ID}" \
      -H "Authorization: Bearer ${VSTAGING_KEY}")
    STATUS=$(echo $RESULT | python3 -c "import sys,json; print(json.load(sys.stdin).get('status','unknown'))")

    if [ "$STATUS" = "completed" ]; then
      OUTPUT_URL=$(echo $RESULT | python3 -c "import sys,json; print(json.load(sys.stdin)['output_url'])")
      curl -s -L "${OUTPUT_URL}" -o ".tmp/${PROJECT_ID}-staging/02-generated/${KEY}.jpg"
      echo "  ✓ Téléchargé: ${KEY}"
      echo "${KEY}=${OUTPUT_URL}" >> .tmp/${PROJECT_ID}-staging/output-map.txt
      break
    elif [ "$STATUS" = "failed" ]; then
      echo "  ✗ FAILED: ${KEY}"
      echo "[$(date '+%Y-%m-%d %H:%M')] FAILED ${KEY}" >> .tmp/${PROJECT_ID}-staging/log.txt
      break
    fi

    ((ATTEMPTS++))
    if [ $ATTEMPTS -ge 36 ]; then  # 36 × 10s = 6 min max
      echo "  ✗ TIMEOUT: ${KEY}"
      break
    fi

    echo "  Status: ${STATUS} (${ATTEMPTS}/36) — attente 10s..."
    sleep 10
  done

done < .tmp/${PROJECT_ID}-staging/job-map.txt
```

---

### Étape 5 — QA : choisir la meilleure variante par pièce

Pour chaque pièce, comparer les 3 fichiers `[room]_seed42.jpg`, `[room]_seed123.jpg`, `[room]_seed789.jpg`.

**Checklist (une seule variante par pièce doit passer TOUS les critères) :**

Structurel — échec = régénérer :
- [ ] Meubles posés sur le sol (pas flottants)
- [ ] Aucun meuble qui traverse un mur
- [ ] Proportions correctes (canapé = taille canapé)
- [ ] Pas d'objets dupliqués identiques
- [ ] Proportions de la pièce préservées

Visuel :
- [ ] Direction lumière cohérente avec les fenêtres
- [ ] Sol original préservé (non retexturé)
- [ ] Pas d'artefacts AI sur les murs
- [ ] Vue des fenêtres préservée

Style :
- [ ] Palette chaude (pas de dominante froide/grise)
- [ ] Mobilier premium, pas entrée de gamme
- [ ] Art mural neutre, pas de logo visible

**Copier la meilleure variante dans le dossier retouche :**
```bash
# Exemple : la variante seed42 du salon est la meilleure
cp .tmp/${PROJECT_ID}-staging/02-generated/living_seed42.jpg \
   .tmp/${PROJECT_ID}-staging/02-generated/living-best.jpg
# Répéter pour chaque pièce
```

**Si aucune variante ne passe :** vérifier la photo source → si la photo est bonne, relancer avec style différent.
```bash
# Relancer avec style luxury au lieu de contemporary
STYLE="luxury"
# Puis reprendre depuis étape 3 pour cette seule pièce
```

---

### Étape 6 — Color grade + color matching

```bash
mkdir -p .tmp/${PROJECT_ID}-staging/03-retouched

# Appliquer le grade LIOR (chaud, légèrement désaturé) à toutes les meilleures variantes
for FILE in .tmp/${PROJECT_ID}-staging/02-generated/*-best.jpg; do
  ROOM=$(basename "$FILE" -best.jpg)

  magick "${FILE}" \
    -color-matrix "1.03 0 0  0 0.99 0  0 0 0.91" \
    -modulate 102,93,100 \
    -unsharp 0x0.8+60+0.03 \
    -strip \
    ".tmp/${PROJECT_ID}-staging/03-retouched/${ROOM}-graded.jpg"

  echo "  ✓ Color grade: ${ROOM}"
done

echo "--- Color grade done ---"
ls -lh .tmp/${PROJECT_ID}-staging/03-retouched/
```

---

### Étape 7 — Export final (16:9 + crop 1:1 social)

```bash
mkdir -p .tmp/${PROJECT_ID}-staging/04-exports

for FILE in .tmp/${PROJECT_ID}-staging/03-retouched/*-graded.jpg; do
  ROOM=$(basename "$FILE" -graded.jpg)

  # Image principale — min 2000px de large
  magick "${FILE}" \
    -resize "2000x>" \
    ".tmp/${PROJECT_ID}-staging/04-exports/${PROJECT_ID}-staged-${ROOM}-v1.jpg"

  # Crop carré 1:1 pour Instagram / TikTok
  magick "${FILE}" \
    -gravity Center \
    -crop 1:1 +repage \
    -resize "1080x1080" \
    ".tmp/${PROJECT_ID}-staging/04-exports/${PROJECT_ID}-staged-${ROOM}-sq-v1.jpg"

  echo "  ✓ Exporté: ${ROOM} (16:9 + 1:1)"
done

echo "--- EXPORTS ---"
ls -lh .tmp/${PROJECT_ID}-staging/04-exports/
echo "Nombre de fichiers: $(ls .tmp/${PROJECT_ID}-staging/04-exports/ | wc -l)"
```

---

## V. QA FINALE

Avant de livrer, ouvrir chaque fichier dans `04-exports/` et vérifier :
- [ ] Tous les fichiers attendus présents (N pièces × 2 formats = 2N fichiers)
- [ ] Chaque image ≥ 2000px de large (vérifier: `magick identify -format "%wx%h\n" 04-exports/*.jpg`)
- [ ] Pas d'artefact visible (objet flottant, mur déformé, meuble coupé)
- [ ] Palette cohérente entre toutes les pièces (même tonalité chaude)
- [ ] Métadonnées supprimées (vérifier: `exiftool 04-exports/[fichier].jpg` → GPS = absent)

```bash
# Vérification automatique des résolutions
magick identify -format "%f: %wx%h\n" .tmp/${PROJECT_ID}-staging/04-exports/*.jpg
```

---

## VI. DELIVERY

### 6.1 Créer le ZIP
```bash
cd .tmp/${PROJECT_ID}-staging/
zip -r "${PROJECT_ID}-staging-v1.zip" "04-exports/"
echo "ZIP size: $(du -sh ${PROJECT_ID}-staging-v1.zip)"
unzip -l "${PROJECT_ID}-staging-v1.zip" | tail -5
```

### 6.2 Upload WeTransfer
```bash
WT_KEY=$(grep WETRANSFER_API_KEY /Users/cashville/.env | cut -d= -f2)
ZIP_FILE=".tmp/${PROJECT_ID}-staging/${PROJECT_ID}-staging-v1.zip"
ZIP_SIZE=$(wc -c < "${ZIP_FILE}")
ZIP_NAME="${PROJECT_ID}-staging-v1.zip"

# Créer le transfert
TRANSFER=$(curl -s -X POST "https://dev.wetransfer.com/v2/transfers" \
  -H "Content-Type: application/json" \
  -H "x-api-key: ${WT_KEY}" \
  -d "{\"message\":\"${PROJECT_ID} — Virtual Staging LIOR\",\"files\":[{\"name\":\"${ZIP_NAME}\",\"size\":${ZIP_SIZE}}]}")

TRANSFER_ID=$(echo $TRANSFER | python3 -c "import sys,json; print(json.load(sys.stdin)['id'])")
UPLOAD_URL=$(echo $TRANSFER | python3 -c "import sys,json; print(json.load(sys.stdin)['files'][0]['multipart']['url'])")

# Upload
curl -s -X PUT "${UPLOAD_URL}" \
  -H "Content-Type: application/octet-stream" \
  --data-binary @"${ZIP_FILE}"

# Finaliser et récupérer le lien
DOWNLOAD_URL=$(curl -s -X PUT "https://dev.wetransfer.com/v2/transfers/${TRANSFER_ID}/finalize" \
  -H "x-api-key: ${WT_KEY}" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['url'])")

echo "DOWNLOAD LINK: ${DOWNLOAD_URL}"
```

### 6.3 Notifier le client via WhatsApp
```bash
PHONE=$(grep CALLMEBOT_PHONE /Users/cashville/.env | cut -d= -f2)
APIKEY=$(grep CALLMEBOT_API_KEY /Users/cashville/.env | cut -d= -f2)
CLIENT_NAME="[NOM DU CLIENT]"
N_ROOMS=5  # ← adapter
MSG="${CLIENT_NAME} — vos pieces amenagees sont pretes. ${N_ROOMS} pieces livrees avec formats sociaux inclus. Telechargement (7 jours) : ${DOWNLOAD_URL}"
ENCODED=$(python3 -c "import urllib.parse,sys; print(urllib.parse.quote(sys.argv[1]))" "${MSG}")
curl -s "https://api.callmebot.com/whatsapp.php?phone=${PHONE}&text=${ENCODED}&apikey=${APIKEY}"
echo "WhatsApp envoyé."
```

### 6.4 Mettre à jour Notion
```bash
NOTION_TOKEN=$(grep NOTION_TOKEN /Users/cashville/.env | cut -d= -f2)
PAGE_ID="[ID_PAGE_NOTION_DU_PROJET]"  # ← récupérer à la création du projet

curl -s -X PATCH "https://api.notion.com/v1/pages/${PAGE_ID}" \
  -H "Authorization: Bearer ${NOTION_TOKEN}" \
  -H "Notion-Version: 2022-06-28" \
  -H "Content-Type: application/json" \
  -d "{
    \"properties\": {
      \"Status\": {\"select\": {\"name\": \"Delivered\"}},
      \"Delivery Link\": {\"url\": \"${DOWNLOAD_URL}\"},
      \"Delivered At\": {\"date\": {\"start\": \"$(date -u +%Y-%m-%d)\"}}
    }
  }"
echo "Notion mis à jour."
```

### 6.5 Logger
```bash
echo "[$(date '+%Y-%m-%d %H:%M')] DELIVERED ${PROJECT_ID} | ZIP: ${ZIP_NAME} | Link: ${DOWNLOAD_URL} | WA: sent | Notion: Delivered" \
  >> .tmp/${PROJECT_ID}-staging/log.txt
```

---

## VII. RÉVISIONS

- Fenêtre : 48h après livraison
- Scope inclus : changement de style, swap de pièce, correction couleur
- Process : reprendre depuis étape 3 pour les pièces concernées uniquement
- Livraison révision : ZIP nommé `v2`, `v3`

---

## VIII. FALLBACKS

| Problème | Action |
|---------|--------|
| Virtual Staging AI down | Attendre 30 min → réessayer. Si toujours down : utiliser REimagineHome (`POST https://api.reimaginehome.ai/v1/generate` — même structure d'appel) |
| Job failed après 3 tentatives | Vérifier la photo source (angle, exposition). Changer le style (`luxury`). Si toujours échoué → flag humain avec le meilleur résultat disponible |
| ImgBB upload échoue | Vérifier la taille (max 32MB). Réduire si nécessaire : `magick input.jpg -resize 3000x output.jpg` |
| WeTransfer API échoue | Upload manuel sur https://wetransfer.com → copier le lien → envoyer via CallMeBot |
| CallMeBot ne répond pas | Vérifier setup (re-envoyer "I allow callmebot to send me messages"). Rate limit : attendre 5s entre messages |
| ImageMagick non installé | `brew install imagemagick` |

---

## IX. LOG FORMAT

```
[2026-06-13 10:00] START LIOR-DM2601 — Virtual Staging
[2026-06-13 10:05] UPLOADED living → https://i.ibb.co/xxx
[2026-06-13 10:06] JOB living_seed42=job_abc123
[2026-06-13 10:08] COMPLETED living_seed42 → https://cdn.virtualstaging.ai/xxx
[2026-06-13 10:35] QA living: selected=seed42 | retouch=none | approved=YES
[2026-06-13 11:20] EXPORTED: 5 rooms × 2 formats = 10 files
[2026-06-13 11:25] DELIVERED LIOR-DM2601 | ZIP: LIOR-DM2601-staging-v1.zip | Link: https://wetransfer.com/xxx | WA: sent | Notion: Delivered
```
