# CRM: Studio Partners LIOR

> Base Notion : créer une base de données "Studio Partners" avec les propriétés ci-dessous.
> Objectif : tracker les 50 partenaires de la signature au premier paiement.

---

## Propriétés de la base

| Propriété | Type | Options / Format |
|-----------|------|-----------------|
| Agency Name | Title | Nom de l'agence |
| Contact Name | Text | Prénom + Nom |
| WhatsApp | Phone | +971... |
| Email | Email | |
| City | Select | Dubai / Marbella / Singapore / Sydney / Samui |
| Tier | Select | Starter / Growth / Scale |
| Status | Select | Prospect / Contacted / Meeting Done / Agreement Sent / SIGNED / Active / Churned |
| Signed Date | Date | Date de signature MSA |
| Commitment Fee | Checkbox | Payé oui/non |
| Commitment Fee Amount | Number | AED |
| Payment Trigger Date | Date | **Toujours : 01/07/2027** |
| Projects Delivered | Number | Cumulatif depuis signature |
| Last Project Date | Date | Dernière livraison |
| Monthly Value Delivered | Formula | Projects × 8500 AED |
| Last Report Sent | Date | Dernier rapport valeur mensuel envoyé |
| Annual Prepay Offered | Checkbox | |
| Annual Prepay Signed | Checkbox | |
| Notes | Text | |

---

## Vues recommandées

### Vue 1: Pipeline (Kanban par Status)
Colonnes : Prospect → Contacted → Meeting Done → Agreement Sent → SIGNED → Active

### Vue 2: Active Partners (filtre : Status = SIGNED ou Active)
Triée par : Signed Date (plus ancien en premier)
Colonnes visibles : Agency, Tier, Projects Delivered, Last Report Sent

### Vue 3: Trigger Countdown
Filtre : Status = SIGNED
Propriété calculée : Jours avant 01/07/2027
Triée par : Jours restants (croissant)

### Vue 4: Revenue Projection
Groupée par : Tier
Somme : Monthly Value Delivered (en cours de free period)
Total projeté MRR Juillet 2027 : calculé automatiquement

---

## Routine mensuelle (1er de chaque mois)

```
Pour chaque partenaire Status = SIGNED :
1. Calculer projets livrés ce mois
2. Calculer valeur AED livrée
3. Rédiger rapport valeur (template ci-dessous)
4. Envoyer WhatsApp
5. Mettre à jour Last Report Sent
```

---

## Template rapport valeur mensuel

```
Hi [Prénom],

Your LIOR Studio Partner summary for [Mois] [Année]:

→ [N] listings delivered
→ [N] rooms virtually staged
→ [N] cinematic tours
→ Value delivered this month: AED [N × 8,500]
→ Total value since partnership: AED [cumul]

Your listings are standing out.
[Ajouter 1 stat si dispo : "average engagement on your IG posts featuring LIOR tours: X%"]

July 2027 is when your monthly partnership activates.
We'll be in touch with your invoice and annual prepay option before June 15.

Talk soon,
[Prénom]
LIOR Visual Studio
```

---

## Objectifs trimestriels

| Période | Partenaires signés cumulés | MRR projeté Juillet 2027 |
|---------|--------------------------|--------------------------|
| Fin mois 3 | 20 | $65K/mois |
| Fin mois 6 | 35 | $110K/mois |
| Fin mois 8 | 50 | $160K–$215K/mois |
| **Juillet 2027** | **50 actifs** | **$120K–$180K/mois** |

---

## Alerte automatique à configurer (n8n)

Trigger : 60 jours avant 01/07/2027 (= 01/05/2027)
Action : Envoyer message WhatsApp à chaque partenaire SIGNED

```
Hi [Prénom],

Quick heads-up: your LIOR Studio Partner free period ends June 30.
Starting July 1, your [Tier] plan activates at [AED X]/month.

We'll send your first invoice on July 1.

If you'd prefer to lock in annual pricing now (15% discount),
reply here and I'll send the details before June 15.

Looking forward to continuing the partnership.
[Prénom] · LIOR
```
