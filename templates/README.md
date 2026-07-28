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

## Routine

Déclencheur planifié (Routine CCR) : tous les lundis ~08:00 Paris.
Il crée un **brouillon** Slack dans `#team_sales_fr` — jamais d'envoi
automatique. Edgar relit, complète le « Focus de la semaine », ajuste et envoie.
