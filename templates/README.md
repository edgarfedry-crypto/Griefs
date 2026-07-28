# Récap Sales hebdomadaire — automatisation

Génère chaque **lundi matin** un brouillon Slack pré-rempli du récap Sales
(chiffres de la semaine passée + AG de la semaine à venir), déposé dans les
**Brouillons** de `#team_sales_fr` pour qu'Edgar le complète et l'envoie.

- Template : [`recap-sales-hebdo.md`](./recap-sales-hebdo.md)
- Périmètre : pipeline **Property Management (copro)**, **France uniquement**
- Canal cible : `#team_sales_fr` (`C046Y4TCV5M`… voir ci-dessous), en **brouillon**

## Définition des fenêtres de dates (au moment où la routine se déclenche)

Le lundi J :
- **Semaine passée** = lundi J-7 → dimanche J-1
- **Semaine à venir** = lundi J → dimanche J+6

## Sources de données

Tout passe par **Omni Analytics** (connecteur `Omni Analytics`, modèle « Matera »,
topic `sales_deals_property_management`). Omni est privilégié à HubSpot car
il expose le modèle sémantique métier (ARR closé, AG, lots) et reste
disponible même quand le token HubSpot expire.

### 1. Chiffres de la semaine passée — deals signés + ARR par commercial
> Prompt Omni :
> « For the Property Management (copro) pipeline, buildings in France ONLY,
> list per sales rep (deal owner / AE) the number of deals signed (won) and the
> total ARR closed, for deals signed (won) between {LUN_PASSE} and {DIM_PASSE}.
> Sort by total ARR closed descending. »

Alimente : total deals, total ARR, podium ARR, plus gros volume, table par commercial.

### 2. AG de la semaine à venir — nombre + lots par commercial
> « For the Property Management (copro) pipeline, buildings in France, list per
> sales rep the number of deals whose general assembly date (date_d_ag) falls
> between {LUN_VENIR} and {DIM_VENIR}, and the total number of main units
> (nombre_de_lots). Sort by number of assemblies descending. »

Alimente : total AG, total lots, « qui en a le plus ».

### 3. La plus grosse AG à venir (détail)
> « … list individual deals whose general assembly date is between {LUN_VENIR}
> and {DIM_VENIR}. Show deal name, owner name, number of main units, assembly
> type (ag_type), assembly date. Sort by number of main units descending. Limit 10. »

Alimente : « la plus grosse AG » + table « Top des AG à surveiller ».

## Points de vigilance (à fiabiliser)

- **Rdv physiques** (`ag_type = Physique`) : le filtre `ag_type` n'est **pas**
  pris en compte de façon fiable par Omni en langage naturel (il renvoie le
  total, pas le sous-ensemble physique). Deux options :
  - laisser ce champ en `[à compléter]` dans le brouillon, **ou**
  - le tirer via HubSpot en direct :
    `SELECT hubspot_owner_id, COUNT(*) FROM DEAL WHERE pipeline='default'
    AND ag_type='Physique' AND date_d_ag BETWEEN '{LUN_VENIR}' AND '{DIM_VENIR}'
    GROUP BY hubspot_owner_id ORDER BY COUNT(*) DESC`
    (nécessite le connecteur HubSpot ré-authentifié).
- **Qualité des dates d'AG** : `date_d_ag` contient des dates par défaut (ex.
  beaucoup de deals calés au 1er du mois) et couvre tous les stages. Pour un
  chiffre « AG réellement à venir » plus fin, restreindre au stage
  *Waiting for vote* (`contractsent`) et/ou aux dates confirmées
  (`date_d_ag_confirmee = true`). À ajuster selon le besoin d'Edgar.

## Référence HubSpot (si besoin de requêter en direct)

| Concept | Propriété / valeur |
|---|---|
| Pipeline copro | `pipeline = 'default'` (Property Management) |
| Deal signé | `dealstage = 'closedwon'` |
| Date d'AG | `date_d_ag` |
| Type d'AG | `ag_type` (`Physique`, `Visio`, `Vote par correspondance`, `PV direct`, `Perdu avant AG`) |
| Taille d'AG | `nombre_de_lots` |
| Montant | `amount_in_home_currency` / `amount_tax_excluded` |
| Commercial | `hubspot_owner_id` |

## Routine — comment l'installer

⚠️ **Important** : une Routine créée par un agent n'embarque pas les connecteurs
(Slack / Omni). Les sessions déclenchées tourneraient donc sans accès aux données
ni à Slack. Il faut créer la Routine **depuis l'UI Routines de claude.ai**, où
tes connecteurs sont automatiquement rattachés aux sessions déclenchées.

**Étapes :**
1. Sur claude.ai → **Routines** (ou Paramètres → Routines / Tâches planifiées).
2. Nouvelle routine, planification **tous les lundis vers 08:00 (Europe/Paris)**.
   Note : si le champ est en UTC, mettre **06:05 UTC** (= 08:05 été / 07:05 hiver).
3. Vérifier que les connecteurs **Slack** et **Omni Analytics** sont activés pour
   la routine.
4. Coller le prompt ci-dessous.

### Prompt à coller

```
Session fraîche déclenchée un LUNDI matin. Objectif : produire le "Récap Sales
hebdomadaire" et le déposer en BROUILLON Slack dans #team_sales_fr
(channel_id CCGP59ZJA). NE JAMAIS ENVOYER — uniquement un brouillon via
slack_send_message_draft. Edgar (U02QU1SPRHR) relit, complète le "Focus" et envoie.

Périmètre : pipeline Property Management (copro), FRANCE uniquement.

1) Dates (aujourd'hui = lundi J) :
   Semaine passée : P1 = date -d '-7 days', P2 = date -d '-1 day'
   Semaine à venir : V1 = aujourd'hui, V2 = date -d '+6 days'  (format YYYY-MM-DD, + affichage JJ/MM)

2) Omni Analytics (getData ; modelId 225379a7-7597-48e1-a675-2777f3d42275 ;
   topic sales_deals_property_management) :
   a. "For the Property Management pipeline, buildings in France ONLY, list per
      sales rep (deal owner) the number of deals signed (won) and the total ARR
      closed, for deals signed (won) between {P1} and {P2}. Sort by total ARR
      closed descending."
   b. "For the Property Management pipeline, buildings in France, considering ONLY
      deals currently in the Waiting for vote deal stage, list per sales rep the
      number of deals whose general assembly date (date_d_ag) falls between {V1}
      and {V2}, and the total number of main units (nombre_de_lots). Sort by
      number of deals descending."
   Rdv physiques : filtre ag_type='Physique' PAS fiable via Omni → laisser
   [à compléter], ou via HubSpot si dispo :
   SELECT hubspot_owner_id, COUNT(*) FROM DEAL WHERE pipeline='default'
   AND ag_type='Physique' AND date_d_ag BETWEEN '{V1}' AND '{V2}'
   GROUP BY hubspot_owner_id ORDER BY COUNT(*) DESC   (puis noms via search_owners)

3) Calculs : totaux deals/ARR (a) ; totaux AG/lots (b) ; top ARR, top volume (a) ;
   "en a le plus" = top nb AG ; top volume de lots (b).

4) Rédige en français, format Slack (emojis :shortcode:, gras *…*, sans tableaux
   markdown), selon templates/recap-sales-hebdo.md.

5) slack_send_message_draft(channel_id="CCGP59ZJA", message=<récap>). Si
   draft_already_exists : le signaler, ne pas forcer.

Règle : si Omni ou Slack indisponible, le signaler — ne JAMAIS inventer de chiffres.
```

La routine crée un **brouillon** Slack dans `#team_sales_fr` — jamais d'envoi
automatique. Edgar relit, complète le « Focus de la semaine », ajuste et envoie.
