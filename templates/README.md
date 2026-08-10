# Récap Sales hebdomadaire — automatisation

Génère chaque **lundi matin** un brouillon Slack pré-rempli du récap Sales
(chiffres de la semaine passée + AG de la semaine à venir), déposé dans les
**Brouillons** de `#team_sales_fr` pour qu'Edgar le complète et l'envoie.

- Template : [`recap-sales-hebdo.md`](./recap-sales-hebdo.md)
- Périmètre : pipeline **Property Management (copro)**, **France uniquement**
- Canal cible : `#team_sales_fr` (`CCGP59ZJA`), en **brouillon** (jamais d'envoi auto)

## Contenu du récap

**Semaine passée** — on élit d'abord l'🏆 *AE de la semaine passée* (plus gros
ARR signé), puis :
- nombre de deals signés + ARR total
- podium ARR, plus gros volume de deals

**Semaine à venir** — on élit d'abord l'🌟 *AE de la semaine à venir* (le plus
d'AG en volume ET en valeur), puis :
- nombre d'AG à jouer + ARR que ça représente
- qui a le plus d'AG en **volume** et en **valeur (ARR)**
- la plus grosse AG en **ARR** (montant, pas nb de lots)
- qui a le plus de **démos planifiées**
- 👀 l'AE qui démarre avec le plus de **tâches en retard**

## Fenêtres de dates (au déclenchement, un lundi J)

- **Semaine passée** = lundi J-7 → dimanche J-1
- **Semaine à venir** = lundi J → dimanche J+6

Les filtres de date HubSpot (`closedate`, `date_d_ag`, `demo_date`) attendent des
**millisecondes epoch UTC** — convertir les bornes de dates avant la requête.

## Source de données — HubSpot (API Search)

⚠️ **Omni Analytics n'est plus utilisable** : les crédits IA du modèle sont épuisés
(`AI credit limit reached`). Tout le récap passe désormais **100 % par HubSpot**.

⚠️ **Utiliser `search_crm_objects` (API Search), PAS `query_crm_data`.** L'endpoint
SQL renvoie `403 (after trying upscoping)` — scope manquant sur le connecteur. L'API
Search fonctionne avec le scope accordé ; on agrège côté client (pas de `GROUP BY`,
on lit le champ `total` ou on somme les `results`).

### Filtres communs à toutes les requêtes DEAL

| Filtre | Valeur |
|---|---|
| Pipeline copro | `pipeline = 'default'` |
| **France uniquement** | `market = 'fr'` (⚠️ le pipeline mélange FR et DE — EXCLURE `market='de'`) |
| Deal signé | `dealstage = 'closedwon'` |
| Stage AG imminente | `dealstage = 'contractsent'` (Waiting for vote) |
| Montant / ARR | `amount_in_home_currency` (+ `deal_currency_code`) |
| Date d'AG | `date_d_ag` |
| Date de démo | `demo_date` |
| Commercial | `hubspot_owner_id` (→ noms + `isActive` via `search_owners`) |

### 1. Semaine passée — deals signés + ARR par AE
`DEAL` où `pipeline='default'` ET `market='fr'` ET `dealstage='closedwon'` ET
`closedate` ∈ [lundi J-7, lundi J[. Agréger par `hubspot_owner_id` : nb deals + somme
`amount_in_home_currency`.

→ total deals, total ARR, 🏆 AE de la semaine (top ARR), podium ARR, plus gros volume.

⚠️ **Deals sans `hubspot_owner_id`** : on en voit apparaître par lots (timestamps de
création/signature identiques → import en masse). Les **exclure** du total et les
**mentionner à part** — ils ne sont attribuables à aucun AE.

### 2. AG à venir — volume + valeur par AE
`DEAL` où `pipeline='default'` ET `market='fr'` ET `dealstage='contractsent'` ET
`date_d_ag` ∈ [lundi J, lundi J+7[. Agréger par owner : nb AG + somme ARR.

→ total AG, total ARR en jeu, 🌟 AE de la semaine à venir (le plus d'AG en volume ET
valeur), classements volume et valeur.

### 3. La plus grosse AG en ARR
Même filtre que (2), mais lister les deals individuels (deal name, owner, ARR), trier
par ARR décroissant.

→ 🏔️ la plus grosse AG (AE + copropriété + **ARR en €**, PAS le nb de lots).

### 4. Démos planifiées par AE — via `MEETING_EVENT`
⚠️ **Ne PAS utiliser `demo_date` sur DEAL** : le champ est quasi jamais alimenté pour
les démos à venir (0 deal daté après le 01/08 dans tout le CRM lors des tests). Les
démos vivent dans l'objet **`MEETING_EVENT`**.

`MEETING_EVENT` où `hs_meeting_start_time` ∈ [lundi J, lundi J+7[ ET
`hs_meeting_title` CONTAINS_TOKEN `Découvrez` (le meeting démo FR s'intitule
**« Découvrez Matera avec [AE] »** ; l'Allemagne = « Matera entdecken … », exclue de
fait). Exclure `hs_meeting_outcome` = `CANCELED` et `NO_SHOW`.

⚠️ **Attribution : par le prénom dans le TITRE, PAS par `hubspot_owner_id`.** Le
propriétaire du meeting est souvent le SDR qui book, pas l'AE qui présente. L'AE réel
est nommé dans le titre après « avec » (ex. « Découvrez Matera avec Jérôme »). Extraire
ce prénom (ignorer « notre expert », « ! ») et le rapprocher du roster AE, puis compter
par AE. → 💻 le plus de démos planifiées (top 3).

> Réf. 10→16/08 : Joseph Nys ≈ 16, Jérôme Hantzberg ≈ 8, Erwan Montfort ≈ 5.

### 5. Tâches en retard par AE — 👀
`TASK` où `hs_task_is_overdue = true`.

⚠️ **Le total brut est énorme (~97 000) et pollué par des tâches système** (« 🚨 Ticket
Call Center… », alertes data-quality, etc.). Ne PAS parcourir la masse. **Compter par
AE** : pour chaque AE, requête `hs_task_is_overdue=true` ET `hubspot_owner_id=<id>`
(`limit 1`) et lire le champ `total`.

**Scope obligatoire → AE closers EN ACTIVITÉ.** Boucler sur les owners apparus comme
`deal owner` en (1)/(2), + Nicolas Mysliwiak (`645804627`), en gardant `isActive=true`
(via `search_owners`). Prendre le plus haut total.

> Réf. 10/08 : Erwan Montfort 62, Nicolas Mysliwiak 59, Jérôme Hantzberg 35,
> Benjamin Dechelette 25, Mattéo Cornée 16. (Les chiffres bougent chaque semaine.)

## Routine — état

✅ **La routine est active** : `Récap Sales hebdo — lundi matin`, cron `5 6 * * 1`
(lundi 06:05 UTC = **08:05 Paris l'été / 07:05 l'hiver**). Elle est **rattachée à la
session** qui porte les connecteurs Slack + HubSpot (self-bind) : à chaque
déclenchement, la session se réveille, les connecteurs se reconnectent, et le récap se
génère puis se dépose en brouillon.

**Fragilité connue** : si l'autorisation **HubSpot** expire entre deux lundis, le run
ne peut pas sortir les chiffres. Dans ce cas la routine **prévient Edgar de reconnecter
HubSpot** (claude.ai → Connecteurs) et **n'invente aucun chiffre**.

> ⚠️ Une routine créée par un agent ne peut pas embarquer de nouveaux connecteurs pour
> cette org (limitation côté plateforme). C'est pour ça qu'elle est rattachée à une
> session existante déjà connectée, plutôt que de lancer une session neuve. Si un jour
> la session n'est plus résumable, recréer la routine depuis une session connectée (ou
> depuis l'UI Routines de claude.ai avec Slack + HubSpot activés) avec le prompt
> ci-dessous.

### Prompt de la routine (référence / à recoller si besoin)

```
⏰ RÉCAP SALES HEBDO — déclenchement automatique du lundi matin.

Génère le Récap Sales de la semaine et dépose-le en BROUILLON Slack dans
#team_sales_fr (channel_id CCGP59ZJA) via slack_send_message_draft. NE JAMAIS
ENVOYER. Edgar (U02QU1SPRHR) relit, complète le "Focus" et envoie.

Périmètre : pipeline Property Management copro (pipeline='default'), FRANCE uniquement
(market='fr' ; EXCLURE 'de'). Source : HubSpot via search_crm_objects (PAS
query_crm_data → 403). Omni indisponible (crédits épuisés). Ne jamais inventer.

Dates (lundi J) : passée = J-7→J-1 ; à venir = J→J+6. Bornes en millisecondes epoch
UTC pour les filtres de date.

1) Semaine passée : DEAL pipeline='default', market='fr', dealstage='closedwon',
   closedate ∈ [J-7, J[. Par owner : nb deals + ARR (amount_in_home_currency). Total
   deals + ARR. 🏆 AE = top ARR. Podium ARR. Plus gros volume. Deals sans
   hubspot_owner_id : exclure + mentionner à part.
2) AG à venir : DEAL pipeline='default', market='fr', dealstage='contractsent',
   date_d_ag ∈ [J, J+7[. Par owner : nb AG + ARR. Total AG + ARR. 🌟 AE = le plus d'AG
   en volume ET valeur. Classement volume + valeur. 🏔️ plus grosse AG en ARR (montant,
   pas lots).
3) 💻 Démos : MEETING_EVENT, hs_meeting_start_time ∈ [J, J+7[, hs_meeting_title
   CONTAINS_TOKEN 'Découvrez' (démo FR "Découvrez Matera avec [AE]" ; DE "Matera
   entdecken" exclu). Exclure hs_meeting_outcome CANCELED/NO_SHOW. Compter par le
   PRÉNOM dans le titre (après "avec"), PAS par hubspot_owner_id (souvent le SDR). Top 3.
4) 👀 Tâches en retard : TASK hs_task_is_overdue=true. Le total brut (~97k) inclut des
   tâches système → compter PAR AE (filtre hubspot_owner_id=<id>, lire 'total').
   Boucler sur les AE closers actifs (owners de (1)/(2) + Nicolas Mysliwiak 645804627,
   isActive=true). Prendre le max.

Noms via search_owners. Rédiger en français, format Slack, selon
templates/recap-sales-hebdo.md, AE élu en tête de chaque section.
slack_send_message_draft(channel_id='CCGP59ZJA', message=<récap>). Si
draft_already_exists : signaler, ne pas forcer. Si HubSpot 403/token expiré : prévenir
Edgar de ré-autoriser HubSpot, ne pas inventer de chiffres.
```

## Référence HubSpot

| Concept | Propriété / valeur |
|---|---|
| Pipeline copro | `pipeline = 'default'` (Property Management) |
| France | `market = 'fr'` (DE = `'de'`, à exclure) |
| Deal signé | `dealstage = 'closedwon'` |
| Stage AG imminente | `dealstage = 'contractsent'` (Waiting for vote) |
| Date d'AG | `date_d_ag` |
| Date de démo | `demo_date` |
| Montant / ARR | `amount_in_home_currency` (+ `deal_currency_code`) |
| Commercial | `hubspot_owner_id` (→ noms via `search_owners`) |
| Tâche en retard | objet `TASK`, `hs_task_is_overdue = true` |
