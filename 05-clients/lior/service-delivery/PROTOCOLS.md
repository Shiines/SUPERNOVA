# PROTOCOLS.md: Livraison · Communication · Fallbacks
> Référence partagée par tous les SOPs LIOR.
> Trois sections : livraison complète de A à Z · communication client exacte · que faire quand ça échoue.

---

## SECTION 1: LIVRAISON COMPLÈTE (de A à Z)

Protocole identique pour tous les services. S'exécute après que tous les fichiers sont produits et que la QA est passée.

### Étape 1: Vérifier que tous les fichiers sont là

```bash
PROJECT_ID="LIOR-DM2601"
SERVICE="listing-pack"  # ou: staging / space-planning / design-concept / etc.
EXPORT_DIR=".tmp/${PROJECT_ID}/08-exports"

# Lister les fichiers
ls -lh "${EXPORT_DIR}/"
echo "---"
echo "Nombre de fichiers: $(ls "${EXPORT_DIR}/" | wc -l)"
echo "Taille totale: $(du -sh "${EXPORT_DIR}/")"
```

Vérifier que chaque fichier attendu est présent (voir liste dans le SOP du service concerné).
Si un fichier manque → relancer la production de ce composant avant de continuer.

### Étape 2: Nommer les fichiers correctement

Convention stricte : `[PROJECT-ID]-[TYPE]-[ROOM/VARIANT]-v[N].[ext]`

```bash
# Renommage en masse si les fichiers sortent des outils avec des noms automatiques
# Exemple: renommer 3 fichiers JPG dans le dossier exports

i=1
for f in "${EXPORT_DIR}"/*.jpg; do
  ROOM=$(basename "$f" | sed 's/.*-//' | sed 's/\.jpg//')
  mv "$f" "${EXPORT_DIR}/${PROJECT_ID}-staged-${ROOM}-v1.jpg"
  ((i++))
done
```

### Étape 3: Créer le ZIP

```bash
cd .tmp/${PROJECT_ID}/
zip -r "${PROJECT_ID}-${SERVICE}-v1.zip" "08-exports/"

# Vérifier le ZIP
echo "ZIP size: $(wc -c < "${PROJECT_ID}-${SERVICE}-v1.zip") bytes"
echo "Contents:"
unzip -l "${PROJECT_ID}-${SERVICE}-v1.zip"
```

### Étape 4: Upload WeTransfer et récupérer le lien

```bash
WT_KEY=$(grep WETRANSFER_API_KEY /Users/cashville/.env | cut -d= -f2)
ZIP_FILE=".tmp/${PROJECT_ID}/${PROJECT_ID}-${SERVICE}-v1.zip"
ZIP_SIZE=$(wc -c < "${ZIP_FILE}")
ZIP_NAME="${PROJECT_ID}-${SERVICE}-v1.zip"

# Créer le transfert
TRANSFER=$(curl -s -X POST "https://dev.wetransfer.com/v2/transfers" \
  -H "Content-Type: application/json" \
  -H "x-api-key: ${WT_KEY}" \
  -d "{
    \"message\": \"${PROJECT_ID}: Livraison LIOR\",
    \"files\": [{\"name\": \"${ZIP_NAME}\", \"size\": ${ZIP_SIZE}}]
  }")

TRANSFER_ID=$(echo $TRANSFER | python3 -c "import sys,json; print(json.load(sys.stdin)['id'])")
UPLOAD_URL=$(echo $TRANSFER | python3 -c "import sys,json; print(json.load(sys.stdin)['files'][0]['multipart']['url'])")

# Upload
curl -s -X PUT "${UPLOAD_URL}" \
  -H "Content-Type: application/octet-stream" \
  --data-binary @"${ZIP_FILE}"

# Finaliser
RESULT=$(curl -s -X PUT "https://dev.wetransfer.com/v2/transfers/${TRANSFER_ID}/finalize" \
  -H "x-api-key: ${WT_KEY}")

DOWNLOAD_URL=$(echo $RESULT | python3 -c "import sys,json; print(json.load(sys.stdin)['url'])")
echo "DOWNLOAD LINK: ${DOWNLOAD_URL}"
# → Sauvegarder ce lien: il sera envoyé au client
```

**Si WeTransfer API échoue → upload manuel sur https://wetransfer.com, copier le lien.**

### Étape 5: Notifier le client par WhatsApp

```bash
PHONE=$(grep CALLMEBOT_PHONE /Users/cashville/.env | cut -d= -f2)
APIKEY=$(grep CALLMEBOT_API_KEY /Users/cashville/.env | cut -d= -f2)
CLIENT_NAME="Omar"  # depuis le brief
DOWNLOAD_URL="https://wetransfer.com/xxx"

# Composer le message (adapter selon le service: voir templates section 2)
MESSAGE="${CLIENT_NAME}: votre commande LIOR est prête. Téléchargement (7 jours) : ${DOWNLOAD_URL}"

ENCODED=$(python3 -c "import urllib.parse,sys; print(urllib.parse.quote(sys.argv[1]))" "${MESSAGE}")
curl -s "https://api.callmebot.com/whatsapp.php?phone=${PHONE}&text=${ENCODED}&apikey=${APIKEY}"
echo "WhatsApp sent."
```

### Étape 6: Mettre à jour Notion

```bash
NOTION_TOKEN=$(grep NOTION_TOKEN /Users/cashville/.env | cut -d= -f2)
LEADS_DB=$(grep NOTION_LEADS_DB_ID /Users/cashville/.env | cut -d= -f2)
PAGE_ID="[ID de la page Notion du projet: récupérer à la création du projet]"

# Mettre à jour le statut et ajouter le lien de livraison
curl -s -X PATCH "https://api.notion.com/v1/pages/${PAGE_ID}" \
  -H "Authorization: Bearer ${NOTION_TOKEN}" \
  -H "Notion-Version: 2022-06-28" \
  -H "Content-Type: application/json" \
  -d "{
    \"properties\": {
      \"Status\": { \"select\": { \"name\": \"Delivered\" } },
      \"Delivery Link\": { \"url\": \"${DOWNLOAD_URL}\" },
      \"Delivered At\": { \"date\": { \"start\": \"$(date -u +%Y-%m-%d)\" } }
    }
  }"
echo "Notion updated."
```

### Étape 7: Logger la livraison

```bash
echo "[$(date '+%Y-%m-%d %H:%M')] DELIVERED: ${PROJECT_ID} | Service: ${SERVICE} | ZIP: ${ZIP_NAME} | Link: ${DOWNLOAD_URL} | WA: sent" \
  >> ".tmp/${PROJECT_ID}/log.txt"
```

---

## SECTION 2: TEMPLATES WHATSAPP PAR SERVICE

Copier-coller + remplacer les variables entre crochets.

**Note:** Ne jamais mentionner "AI", "drone", "FPV", "luxury". Ne jamais inclure de prix.

---

### SOP-01: Virtual Staging
```
[NOM]: vos pièces aménagées sont prêtes.

[N] pièces livrées avec les formats sociaux inclus.

Téléchargement (7 jours) : [LIEN]

Dites-nous si vous souhaitez ajuster quelque chose.
```

---

### SOP-02: Full Listing Pack
```
[NOM]: votre Full Listing Pack est prêt.

• [N] pièces aménagées (JPG + format carré)
• 1 tour cinématique (16:9 · vertical · carré)
• 3 films sociaux (30s reel · 15s teaser · 30s listing)

Téléchargement (7 jours) : [LIEN]

N'hésitez pas si vous voulez ajuster quoi que ce soit.
```

---

### SOP-03: Space Preview
```
[NOM]: votre Space Preview est prêt.

Deux directions créatives pour votre espace :

Direction A: [nom] : [description une ligne]
Direction B: [nom] : [description une ligne]

Chacune inclut les visuels des pièces, un tour cinématique et les formats sociaux.

Téléchargement (7 jours) : [LIEN]

Dites-nous quelle direction vous parle le plus.
```

---

### SOP-04: Interior Design Brief
```
[NOM]: votre brief de design intérieur est prêt.

Le document couvre :
• Direction de style et moodboard
• 2 options de layout
• Visuels de [N] pièces clés
• Sélection matériaux et mobilier complète

Téléchargement (7 jours) : [LIEN]

Ce document est prêt à partager directement avec un entrepreneur ou un architecte.
```

---

### SOP-05: Full Interior Design
```
[NOM]: votre dossier de design complet est prêt.

Il contient tout ce dont votre équipe a besoin :
• Visuels de toutes les pièces
• Plans d'espace avec mobilier
• Cahier des matériaux et finitions
• Descriptif des travaux par pièce

Téléchargement (7 jours) : [LIEN]

Nous sommes disponibles pendant le chantier pour toute clarification.
```

---

### SOP-06: Property Management (rapport mensuel)
```
[NOM]: rapport [MOIS] prêt.

• Occupation : [X]% ([N] nuits)
• Revenu : AED [X]
• Note clients : [X]/5

[Une ligne highlight ou alerte]

Rapport complet : [LIEN]
```

---

### SOP-07: Space Planning
```
[NOM]: votre étude d'espace est prête.

Deux options de layout :
• Option A: [nom] : [description courte]
• Option B: [nom] : [description courte]

Chaque option inclut le plan coté avec mobilier et le raisonnement détaillé.

Téléchargement (7 jours) : [LIEN]

Dites-nous quelle direction vous convient ou si vous voulez explorer autrement.
```

---

### SOP-08: Architectural Visualization
```
[NOM]: vos rendus architecturaux sont prêts.

[N] rendus livrés :
• [N] vues extérieures
• [N] vues intérieures
• Présentation complète en PDF

Téléchargement (7 jours) : [LIEN]

Prêt à partager avec vos investisseurs, votre équipe commerciale ou pour dépôt de permis.
```

---

### SOP-09: Design Concept
```
[NOM]: votre concept design est prêt.

Le document couvre tout ce dont un entrepreneur a besoin :
• Direction créative et moodboard
• Visuels de [N] pièces clés
• Cahier des matériaux et finitions complet

Téléchargement (7 jours) : [LIEN]

À transmettre directement à votre prestataire ou architecte.
```

---

### SOP-10: Pre-Renovation Brief
```
[NOM]: votre dossier de rénovation est prêt.

Il contient :
• Visuels avant/après de toutes les pièces concernées
• Descriptif des travaux pièce par pièce
• Cahier des matériaux et finitions
• Budget indicatif par poste

Téléchargement (7 jours) : [LIEN]

À transmettre directement à vos entrepreneurs pour demande de devis.
```

---

### Demande de photos manquantes (toutes pièces)
```
[NOM]: pour [PIÈCE], la photo que nous avons reçue est [raison : trop sombre / floue / angle insuffisant].

Pourriez-vous la reprendre depuis l'angle opposé de la pièce, en journée, téléphone horizontal à hauteur de poitrine ?

Nous relançons dès réception.
```

---

### Demande de vidéo manquante
```
[NOM]: pour démarrer votre commande, nous avons besoin d'une vidéo de visite de votre bien.

5 minutes, n'importe quel smartphone. Instructions : [LIEN VERS ANNEX A ou questionnaire.html]

Nous relançons dès réception.
```

---

## SECTION 3: FALLBACKS (que faire quand ça échoue)

### Virtual Staging AI: API down ou quota épuisé
```
1. Attendre 30 min → réessayer (souvent une interruption temporaire)
2. Si toujours down après 30 min → passer à REimagineHome (voir TOOLS.md section 3)
3. Si REimagineHome aussi indisponible → utiliser le workflow Web UI de l'un ou l'autre
4. Logger: [DATE] FALLBACK virtualstaging.ai → reimaginehome.ai pour [PROJECT-ID]
```

### Runway Gen-3: timeout ou artefacts sur 3 tentatives
```
1. Changer le seed (seed: 42 → seed: 123)
2. Raccourcir le prompt de motion (versions trop longues peuvent dégrader le résultat)
3. Si toujours échoué après 3 tentatives → Ken Burns FFmpeg (voir sop-higgfield Phase 3)
   ffmpeg -loop 1 -i [still.jpg] \
     -vf "scale=3840:2160,zoompan=z='min(zoom+0.0006,1.12)':x='iw/2-(iw/zoom/2)':y='ih/2-(ih/zoom/2)':d=192:s=1920x1080,fps=24" \
     -t 8 -pix_fmt yuv420p \
     04-animated-clips/[room]-staged-fallback.mp4
4. Logger: [DATE] FALLBACK Runway → Ken Burns pour [room] [PROJECT-ID]
```

### Midjourney: Discord inaccessible ou API non disponible
```
1. Passer directement à Adobe Firefly API (voir TOOLS.md section 6)
2. Si Firefly aussi down → Replicate avec SDXL (voir TOOLS.md section 7)
3. Logger le fallback utilisé
```

### Replicate: modèle déprécié ou erreur de version
```
1. Aller sur https://replicate.com/explore → rechercher "interior design SDXL"
2. Trouver le modèle avec le plus d'utilisations récentes
3. Copier le nouveau version hash → remplacer dans le script
4. Relancer
```

### WeTransfer: upload échoué
```
1. Vérifier la taille du ZIP (WeTransfer free: max 2GB, Pro: max 200GB)
2. Si trop lourd → séparer en deux ZIPs (staged rooms / videos séparément)
3. Si API échoue → upload manuel sur https://wetransfer.com
4. Si WeTransfer totalement indisponible → upload sur Google Drive:
   - Créer un dossier partagé [PROJECT-ID]
   - Activer le partage "anyone with the link can view"
   - Copier le lien → envoyer au client
```

### CallMeBot: message non envoyé
```
Erreurs communes:
- "Phone number not registered": refaire le setup WhatsApp (envoyer "I allow callmebot to send me messages" au +34 644 29 73 73)
- "Rate limit": attendre 5 secondes entre chaque message
- "Message too long": tronquer à 300 caractères maximum

Test de diagnostic:
curl -s "https://api.callmebot.com/whatsapp.php?phone=${PHONE}&text=Test+LIOR&apikey=${APIKEY}"

Réponse attendue: "Message sent" ou code d'erreur explicite.
```

### Photo client rejetée (qualité insuffisante)
```
1. Identifier le problème précis (trop sombre / flou / angle / occupée)
2. Envoyer la demande de reshoot (template section 2)
3. Logger: [DATE] BLOCKED [PROJECT-ID]: [pièce] en attente reshoot
4. Ne pas bloquer les autres pièces: continuer la production sur les photos valides
5. Quand la photo reshoot arrive → reprendre seulement cette pièce
```

### Pandoc: erreur de génération PDF
```
Erreur fréquente: police manquante (xelatex ne trouve pas Inter ou Cormorant Garamond)

Fix:
1. Installer les polices: cp [font.ttf] ~/Library/Fonts/
2. Mettre à jour la cache: fc-cache -fv
3. Réessayer: pandoc ... --pdf-engine=xelatex

Alternative (sans dépendance LaTeX):
pandoc input.md -o output.pdf --pdf-engine=wkhtmltopdf
brew install wkhtmltopdf
```

### API key expirée ou invalide (n'importe quel service)
```
1. Aller dans le dashboard du service concerné → régénérer la clé
2. Mettre à jour /Users/cashville/.env
3. Relancer le script
4. Ne jamais hardcoder les clés dans les scripts: toujours lire depuis .env
```

---

## SECTION 4: CHECKLIST DE LIVRAISON (universelle)

À cocher avant chaque livraison, quel que soit le service.

**Fichiers:**
- [ ] Tous les fichiers attendus présents dans `exports/`
- [ ] Nommage correct: `[PROJECT-ID]-[type]-[variant]-v[N].[ext]`
- [ ] Résolutions correctes (voir spec du service)
- [ ] Pas de filigrane, pas d'artefact visible sur les fichiers livrés
- [ ] Métadonnées supprimées (GPS, device info)

**ZIP:**
- [ ] ZIP créé et nommé correctement
- [ ] Contenu du ZIP vérifié (`unzip -l`)
- [ ] Taille du ZIP < 2GB (sinon diviser)

**Communication:**
- [ ] Message WhatsApp compose avec le bon template (section 2)
- [ ] Aucun prix dans le message
- [ ] Aucun "AI", "drone", "luxury" dans le message
- [ ] Lien WeTransfer inclus dans le message

**Logistique:**
- [ ] Notion mis à jour: statut → `Delivered`
- [ ] Lien de livraison sauvegardé dans Notion
- [ ] Log.txt mis à jour avec l'entrée de livraison
