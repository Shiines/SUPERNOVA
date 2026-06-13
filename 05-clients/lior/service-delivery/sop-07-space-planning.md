# SOP 07 — Space Planning
**Automation: Full auto (A)**  ·  **Timeline: 48h**  ·  **Livrable: 2 plans d'espace cotés + rationale annoté**

---

## I. OVERVIEW

Plan existant + brief d'usage → 2 options de layout avec mobilier placé à l'échelle + rationale PDF.
Pipeline : parsing brief → analyse plan → générer 2 layouts (RoomSketcher web UI) → annoter → compiler PDF (Pandoc) → livrer.

---

## II. INPUTS

| # | Input | Format | Bloquant |
|---|-------|--------|---------|
| 1 | Plan existant | PDF ou JPG, lisible | OUI |
| 2 | Dimensions des pièces | Sur le plan ou liste en m | OUI |
| 3 | Brief d'usage | Comment ils veulent utiliser chaque pièce | OUI |
| 4 | Profil occupants | Couple / famille / pro seul / priorité réception | OUI |
| 5 | Mobilier à conserver | Photos + dimensions si existant | Non |
| 6 | Project ID | `LIOR-[CODE][AA][SEQ]` | Interne |

**Si dimensions manquantes sur le plan → envoyer ce message :**
```
[NOM] — votre plan est bien reçu mais il manque les dimensions des pièces.
Des mesures approximatives suffisent (ex: "salon environ 6m × 4m").
Pourriez-vous nous les confirmer rapidement ?
```

---

## III. SETUP (une seule fois)

```bash
# Pandoc pour la génération PDF
which pandoc || brew install pandoc
which wkhtmltopdf || brew install --cask wkhtmltopdf
# Note: wkhtmltopdf est le moteur PDF le plus simple, pas besoin de LaTeX

# WeTransfer et CallMeBot
grep -E "WETRANSFER_API_KEY|CALLMEBOT_PHONE|CALLMEBOT_API_KEY|NOTION_TOKEN" \
  /Users/cashville/.env
```

---

## IV. STRUCTURE DE DOSSIERS

```bash
PROJECT_ID="LIOR-SP2601"  # ← adapter

mkdir -p .tmp/${PROJECT_ID}-spaceplan/{01-inputs,02-layouts,03-rationale,04-exports}
touch .tmp/${PROJECT_ID}-spaceplan/log.txt

echo "[$(date '+%Y-%m-%d %H:%M')] START ${PROJECT_ID} — Space Planning" \
  >> .tmp/${PROJECT_ID}-spaceplan/log.txt
```

---

## V. ÉTAPE 1 — PARSING DU BRIEF

Extraire du formulaire d'intake et construire le brief structuré :

```bash
cat > .tmp/${PROJECT_ID}-spaceplan/01-inputs/brief.txt << 'EOF'
PROJECT BRIEF — [PROJECT_ID]
─────────────────────────────────────
Occupants: [ex: couple, pas d'enfants]
Usage salon: [ex: cocooning TV + réception occasionnelle]
Usage chambre principale: [ex: espace dressing souhaité]
Usage cuisine: [ex: cuisinier quotidien, cuisine sociale]
Usage autre: [ex: coin bureau dans chambre 2]
Mobilier à conserver: [ex: canapé d'angle gris 280cm]
Contraintes fixes: [ex: mur porteur côté est, plomberie non déplaçable]
Style mobilier: [ex: contemporain chaud]
─────────────────────────────────────
DIMENSIONS (à compléter depuis le plan):
Salon: [L × l] m
Chambre principale: [L × l] m
Cuisine: [L × l] m
Salle de bain: [L × l] m
[autres pièces]
─────────────────────────────────────
EOF
```

---

## VI. ÉTAPE 2 — ANALYSE DU PLAN

Lire le plan et noter dans le log :

```bash
cat >> .tmp/${PROJECT_ID}-spaceplan/log.txt << 'EOF'
ANALYSE PLAN
─────────────────────
Surface totale: [m²]
Orientation lumière: [fenêtres N/S/E/O]
Flux principal: [entrée → salon → ...]
Contraintes identifiées:
  - Murs porteurs suspectés: [position]
  - Plomberie fixe: [WC, cuisine]
  - Tableau électrique: [position]
Goulots d'étranglement: [couloir étroit, portes gênantes, etc.]
─────────────────────
EOF
```

**Principes appliqués aux deux options :**
- Circulations principales : min 90cm dégagés
- Tour de lit : min 60cm chaque côté
- Autour table à manger : min 75cm tous côtés
- Canapé vers source lumière (pas dos à la fenêtre)
- Lit : tête contre mur plein (pas sous fenêtre)

---

## VII. ÉTAPE 3 — GÉNÉRER LES 2 LAYOUTS (RoomSketcher)

**Workflow web — RoomSketcher (https://roomsketcher.com)**

Ouvrir le navigateur, aller sur https://roomsketcher.com → Sign In.

**Créer Option A :**
1. Cliquer "New Floor Plan"
2. Choisir "Draw your own" → entrer les dimensions de chaque pièce
3. Dessiner les murs avec l'outil "Draw Walls" (cliquer-glisser, entrer les mesures)
4. Ajouter les portes : panneau gauche → "Doors" → glisser en position
5. Ajouter les fenêtres : panneau gauche → "Windows" → glisser en position
6. Onglet "Furnish" en haut
7. Placer le mobilier :
   - Barre de recherche : taper "sofa" → glisser dans le salon
   - Double-clic sur le meuble → Properties → ajuster les dimensions exactes
   - Taper "bed" → glisser dans la chambre principale → orienter (rotation avec la flèche)
   - Taper "dining table" → glisser → dimensions selon brief
   - Répéter pour chaque pièce

**Principes Option A (Lumière & Circulation ouverte) :**
- Mobilier orienté vers la lumière naturelle
- Sightline depuis l'entrée vers le point de vue principal
- Espace de vie maximal autour des pièces principales

**Principes Option B (Fonctionnel & Zones définies) :**
- Zones distinctes par activité (TV / lecture / travail clairement séparés)
- Rangements intégrés à chaque zone
- Meilleure séparation acoustique si enfants ou télétravail

8. Vérifier la vue 3D : bouton "3D View" en haut → confirmer que rien n'est flottant ou incohérent
9. Retour en vue 2D → File → Export → "Floor Plan Image" → JPG, résolution 2000px minimum
10. Cocher "Show dimensions" avant l'export
11. Sauvegarder : `.tmp/${PROJECT_ID}-spaceplan/02-layouts/option-A.jpg`

**Créer Option B :**
- Dupliquer le projet : File → "Duplicate Project" → renommer "Option B"
- Réorganiser le mobilier selon les principes Option B
- Même export → sauvegarder : `.tmp/${PROJECT_ID}-spaceplan/02-layouts/option-B.jpg`

---

## VIII. ÉTAPE 4 — RATIONALE ANNOTÉ

Rédiger 5–7 points d'explication par option dans un fichier Markdown :

```bash
cat > .tmp/${PROJECT_ID}-spaceplan/03-rationale/rationale.md << 'EOF'
---
title: "Space Planning — [PROJECT_ID]"
date: "[DATE]"
geometry: "margin=2.5cm"
fontsize: 11pt
---

# Space Planning
## [Adresse propriété]

---

## Option A — [Nom, ex: "Lumière & Ouverture"]

![Option A](.tmp/[PROJECT_ID]-spaceplan/02-layouts/option-A.jpg){ width=100% }

- **Canapé orienté vers la baie vitrée** — maximise la lumière naturelle dans la zone principale de séjour
- **Table à manger côté cuisine** — réduit les déplacements pour servir et débarrasser
- **Mur TV sur la façade est** — pas de contre-jour en soirée (heures de visionnage principales)
- **Lit aligné avec la fenêtre** — lumière naturelle du matin côté réveil
- **Circulation entrée maintenue à 95cm** — passage aisé avec bagages ou poussette
- **Vue de l'entrée vers le balcon** — sensation de profondeur et d'espace dès l'entrée

---

## Option B — [Nom, ex: "Fonctionnel & Zones"]

![Option B](.tmp/[PROJECT_ID]-spaceplan/02-layouts/option-B.jpg){ width=100% }

- **Salon et salle à manger clairement séparés** — chaque zone a sa propre identité visuelle
- **Coin bureau intégré dans l'alcôve de la chambre** — porte fermable pour focus
- **Réorganisation îlot cuisine** — espace de préparation agrandi, comptoir social
- **Chambre 2 : bureau + rangement prioritaires** — optimisé pour télétravail ou invités
- **TV et canapé groupés plus serrés** — expérience cinématique plus intimiste
- **Trade-off noté** : couloir légèrement réduit (85cm) pour permettre ouverture porte chambre plus large

---

## Récapitulatif

| Pièce | Option A | Option B | Notre recommandation |
|-------|----------|----------|---------------------|
| Salon | Lumineux, ouvert | Intime, fonctionnel | A si réception · B si vie quotidienne |
| Chambre principale | Maximise espace | Intègre dressing | B si stockage prioritaire |
| Cuisine | Standard | Social avec îlot | B si cuisinier actif |

---

*Document préparé par LIOR · [DATE]*
EOF
```

---

## IX. ÉTAPE 5 — GÉNÉRER LE PDF

```bash
pandoc .tmp/${PROJECT_ID}-spaceplan/03-rationale/rationale.md \
  -o .tmp/${PROJECT_ID}-spaceplan/04-exports/${PROJECT_ID}-space-planning-v1.pdf \
  --pdf-engine=wkhtmltopdf \
  -V margin-top=20mm \
  -V margin-bottom=20mm \
  -V margin-left=20mm \
  -V margin-right=20mm

echo "PDF généré : $(ls -lh .tmp/${PROJECT_ID}-spaceplan/04-exports/)"
```

**Si erreur pandoc/wkhtmltopdf :**
```bash
# Vérifier l'installation
which wkhtmltopdf
# Si absent : brew install --cask wkhtmltopdf

# Alternative : utiliser pdfkit via Python
python3 -c "
import pdfkit
pdfkit.from_file('.tmp/rationale.html', '.tmp/output.pdf')
"
# Nécessite: pip install pdfkit
```

---

## X. EXPORT FINAL

```bash
EXPORT_DIR=".tmp/${PROJECT_ID}-spaceplan/04-exports"
mkdir -p "${EXPORT_DIR}"

# Copier les layouts avec le bon nommage
cp .tmp/${PROJECT_ID}-spaceplan/02-layouts/option-A.jpg \
   "${EXPORT_DIR}/${PROJECT_ID}-layout-option-A-v1.jpg"

cp .tmp/${PROJECT_ID}-spaceplan/02-layouts/option-B.jpg \
   "${EXPORT_DIR}/${PROJECT_ID}-layout-option-B-v1.jpg"

# Le PDF est déjà dans exports
ls -lh "${EXPORT_DIR}/"
# Doit afficher 3 fichiers: layout-A.jpg · layout-B.jpg · space-planning.pdf
```

---

## XI. DELIVERY

### ZIP
```bash
cd .tmp/${PROJECT_ID}-spaceplan/
zip -r "${PROJECT_ID}-space-planning-v1.zip" "04-exports/"
echo "ZIP: $(du -sh ${PROJECT_ID}-space-planning-v1.zip)"
```

### WeTransfer
```bash
WT_KEY=$(grep WETRANSFER_API_KEY /Users/cashville/.env | cut -d= -f2)
ZIP_FILE=".tmp/${PROJECT_ID}-spaceplan/${PROJECT_ID}-space-planning-v1.zip"
ZIP_SIZE=$(wc -c < "${ZIP_FILE}")
ZIP_NAME="${PROJECT_ID}-space-planning-v1.zip"

TRANSFER=$(curl -s -X POST "https://dev.wetransfer.com/v2/transfers" \
  -H "Content-Type: application/json" \
  -H "x-api-key: ${WT_KEY}" \
  -d "{\"message\":\"${PROJECT_ID} — Space Planning LIOR\",\"files\":[{\"name\":\"${ZIP_NAME}\",\"size\":${ZIP_SIZE}}]}")

TRANSFER_ID=$(echo $TRANSFER | python3 -c "import sys,json; print(json.load(sys.stdin)['id'])")
UPLOAD_URL=$(echo $TRANSFER | python3 -c "import sys,json; print(json.load(sys.stdin)['files'][0]['multipart']['url'])")
curl -s -X PUT "${UPLOAD_URL}" -H "Content-Type: application/octet-stream" --data-binary @"${ZIP_FILE}"

DOWNLOAD_URL=$(curl -s -X PUT "https://dev.wetransfer.com/v2/transfers/${TRANSFER_ID}/finalize" \
  -H "x-api-key: ${WT_KEY}" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['url'])")

echo "DOWNLOAD LINK: ${DOWNLOAD_URL}"
```

### WhatsApp
```bash
PHONE=$(grep CALLMEBOT_PHONE /Users/cashville/.env | cut -d= -f2)
APIKEY=$(grep CALLMEBOT_API_KEY /Users/cashville/.env | cut -d= -f2)
CLIENT_NAME="[NOM]"

MSG="${CLIENT_NAME} — votre etude d'espace est prete. Option A : [nom court]. Option B : [nom court]. Chaque option inclut le plan cote et le raisonnement detaille. Telechargement (7 jours) : ${DOWNLOAD_URL}"
ENCODED=$(python3 -c "import urllib.parse,sys; print(urllib.parse.quote(sys.argv[1]))" "${MSG}")
curl -s "https://api.callmebot.com/whatsapp.php?phone=${PHONE}&text=${ENCODED}&apikey=${APIKEY}"
```

### Notion
```bash
NOTION_TOKEN=$(grep NOTION_TOKEN /Users/cashville/.env | cut -d= -f2)
PAGE_ID="[ID_NOTION]"
curl -s -X PATCH "https://api.notion.com/v1/pages/${PAGE_ID}" \
  -H "Authorization: Bearer ${NOTION_TOKEN}" \
  -H "Notion-Version: 2022-06-28" \
  -H "Content-Type: application/json" \
  -d "{\"properties\":{\"Status\":{\"select\":{\"name\":\"Delivered\"}},\"Delivery Link\":{\"url\":\"${DOWNLOAD_URL}\"},\"Delivered At\":{\"date\":{\"start\":\"$(date -u +%Y-%m-%d)\"}}}}"

echo "[$(date '+%Y-%m-%d %H:%M')] DELIVERED ${PROJECT_ID} | Link: ${DOWNLOAD_URL}" \
  >> .tmp/${PROJECT_ID}-spaceplan/log.txt
```

---

## XII. FALLBACKS

| Problème | Action |
|---------|--------|
| RoomSketcher inaccessible | Utiliser Planner5D (https://planner5d.com) — même workflow web UI |
| PDF wkhtmltopdf échoue | `pip install pdfkit && python3 -c "import pdfkit; pdfkit.from_file('input.html','output.pdf')"` |
| Dimensions manquantes | Envoyer demande (template section II) — attendre — ne pas estimer les dimensions |
| Plan illisible | Demander une photo de chaque pièce avec un mètre visible au sol |

---

## XIII. UPSELL (48h après livraison)

```bash
MSG="[NOM] — une fois le layout choisi, l'etape naturelle est un concept design complet — materiaux, finitions, mobilier source. Ca evite generalement du temps et du cout pendant les travaux. On vous explique ce que ca couvre ?"
ENCODED=$(python3 -c "import urllib.parse,sys; print(urllib.parse.quote(sys.argv[1]))" "${MSG}")
curl -s "https://api.callmebot.com/whatsapp.php?phone=${PHONE}&text=${ENCODED}&apikey=${APIKEY}"
```
