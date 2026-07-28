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

## Sources de données — Omni Analytics

Modèle « Matera » (`modelId 225379a7-7597-48e1-a675-2777f3d42275`).
Omni est privilégié à HubSpot : modèle sémantique métier + reste dispo quand le
token HubSpot expire.

### 1. Semaine passée — deals signés + ARR par AE (topic `sales_deals_property_management`)
> « For the Property Management pipeline, buildings in France ONLY, list per
> sales rep (deal owner) the number of deals signed (won) and the total ARR
> closed, for deals signed (won) between {P1} and {P2}. Sort by total ARR
> closed descending. »

→ total deals, total ARR, 🏆 AE de la semaine (top ARR), podium, plus gros volume.

### 2. AG à venir — volume + valeur par AE (topic `sales_deals_property_management`)
> « For the Property Management pipeline, buildings in France, considering ONLY
> deals that entered the Waiting for vote stage and have not yet exited it, list
> per sales rep the number of deals AND the total ARR (sum of deal amount) whose
> general assembly date (date_d_ag) falls between {V1} and {V2}. Sort by total
> ARR descending. »

→ total AG, total ARR en jeu, 🌟 AE de la semaine à venir, "le plus d'AG"
volume + valeur. (Restreindre au stage *Waiting for vote* est essentiel : sinon
`date_d_ag` remonte des centaines de deals avec des dates par défaut.)

### 3. La plus grosse AG en ARR (topic `sales_deals_property_management`)
> « … considering ONLY deals that entered the Waiting for vote stage and have not
> yet exited it, list individual deals whose general assembly date is between
> {V1} and {V2}. Show deal name, owner name, and ARR (deal amount). Sort by ARR
> descending. Limit 5. »

→ 🏔️ la plus grosse AG (AE + copropriété + ARR).

### 4. Démos planifiées par AE (topic `sales_deals_property_management`)
> « For the Property Management pipeline, buildings in France, count per sales rep
> the number of demos scheduled with a demo date between {V1} and {V2}. Sort
> descending. » _(Omni mappe sur `next_scheduled_demo_at`.)_

→ 💻 le plus de démos planifiées.

### 5. Tâches en retard par AE — 👀 (HubSpot préféré, sinon Omni)

**Source préférée = HubSpot** (objet `TASK`, propriété booléenne `hs_task_is_overdue`) :
> `SELECT hubspot_owner_id, COUNT(*) FROM TASK WHERE hs_task_is_overdue = true`
> `GROUP BY hubspot_owner_id ORDER BY COUNT(*) DESC LIMIT 60`
> puis résoudre les `hubspot_owner_id` en noms via `search_owners` (qui renvoie
> aussi `isActive`).

**Fallback Omni** (topic `sales_crm__engagements`) si HubSpot indisponible :
> « Count per task owner the number of open, not-completed tasks overdue as of {V1}
> (due before {V1}), restricted to the Sales team on the Property Management
> pipeline in France. Sort descending. Limit 25. »

**Scoping obligatoire → AE closers EN ACTIVITÉ.** Le classement brut contient des
comptes non-closers avec des volumes anormaux (team leads, sales ops, comptes de
rotation — p. ex. Jean-Baptiste Pluchet ~2 180, Matthieu Troesch ~93, un owner
vide ~409). Ne garder QUE les AE closers actifs = ceux qui apparaissent comme
`deal owner` dans les requêtes 1–4 (deals signés / AG / démos). Prendre le 1er de
cette liste filtrée. _(Réf. semaine du 27/07 : Joseph Miraux = 43, 1er closer actif.)_

## Routine — comment l'installer

⚠️ **Important** : une Routine créée par un agent n'embarque pas les connecteurs
(Slack / Omni). Les sessions déclenchées tourneraient sans accès aux données ni à
Slack. Il faut créer la Routine **depuis l'UI Routines de claude.ai**, où les
connecteurs sont rattachés aux sessions déclenchées.

**Étapes :**
1. claude.ai → **Routines** (Paramètres → Routines / Tâches planifiées).
2. Planification **tous les lundis vers 08:00 (Europe/Paris)**. Si le champ est en
   UTC : **06:05 UTC** (= 08:05 été / 07:05 hiver).
3. Activer les connecteurs **Slack** et **Omni Analytics**.
4. Coller le prompt ci-dessous.

### Prompt à coller

```
Session fraîche déclenchée un LUNDI matin. Objectif : produire le "Récap Sales
hebdomadaire" et le déposer en BROUILLON Slack dans #team_sales_fr
(channel_id CCGP59ZJA). NE JAMAIS ENVOYER — uniquement un brouillon via
slack_send_message_draft. Edgar (U02QU1SPRHR) relit, complète le "Focus" et envoie.

Périmètre : pipeline Property Management (copro), FRANCE uniquement.
Modèle Omni : modelId 225379a7-7597-48e1-a675-2777f3d42275.

1) Dates (aujourd'hui = lundi J) : P1 = date -d '-7 days', P2 = date -d '-1 day',
   V1 = aujourd'hui, V2 = date -d '+6 days' (YYYY-MM-DD + affichage JJ/MM).

2) Omni getData (topic sales_deals_property_management) :
   a. "For the Property Management pipeline, buildings in France ONLY, list per
      sales rep the number of deals signed (won) and total ARR closed, for deals
      signed between {P1} and {P2}. Sort by total ARR descending."
   b. "For the Property Management pipeline, buildings in France, considering ONLY
      deals that entered the Waiting for vote stage and have not yet exited it,
      list per sales rep the number of deals AND the total ARR (sum of deal amount)
      whose general assembly date (date_d_ag) is between {V1} and {V2}. Sort by
      total ARR descending."
   c. Même filtre que (b) mais deals individuels : "…list individual deals with
      deal name, owner name, and ARR (deal amount). Sort by ARR descending. Limit 5."
   d. "For the Property Management pipeline, buildings in France, count per sales
      rep the number of demos scheduled with a demo date between {V1} and {V2}.
      Sort descending."
   e. Tâches en retard (👀). PRÉFÉRER HubSpot si le connecteur est autorisé :
      SELECT hubspot_owner_id, COUNT(*) FROM TASK WHERE hs_task_is_overdue = true
      GROUP BY hubspot_owner_id ORDER BY COUNT(*) DESC LIMIT 60 ; puis noms via
      search_owners. Sinon fallback Omni topic sales_crm__engagements : "Count per
      task owner the number of open not-completed tasks overdue as of {V1} (due
      before {V1}), restricted to the Sales team on the Property Management pipeline
      in France. Sort descending. Limit 25." Dans TOUS les cas : ne garder que les
      AE closers EN ACTIVITÉ = ceux présents comme deal owner dans (a)-(d) ; écarter
      les non-closers à volumes anormaux (team leads, ops, comptes de rotation).

3) Calculs : totaux deals/ARR (a) ; total AG (b) + total ARR (somme b) ;
   AE semaine passée = top ARR (a) ; AE semaine à venir = top ARR d'AG (b) ;
   plus d'AG volume (b, tri count) et valeur (b, tri ARR) ; plus grosse AG (c) ;
   plus de démos (d) ; plus de tâches en retard (e, nettoyé).

4) Rédige en français, format Slack (emojis :shortcode:, gras *…*, sans tableaux
   markdown), selon templates/recap-sales-hebdo.md — élire l'AE en tête de chaque
   section, avant les détails.

5) slack_send_message_draft(channel_id="CCGP59ZJA", message=<récap>). Si
   draft_already_exists : le signaler, ne pas forcer.

Règle : si Omni/Slack indisponible, le signaler — ne JAMAIS inventer de chiffres.
```

## Référence HubSpot (si requête directe nécessaire)

| Concept | Propriété / valeur |
|---|---|
| Pipeline copro | `pipeline = 'default'` (Property Management) |
| Deal signé | `dealstage = 'closedwon'` |
| Stage AG imminente | `dealstage = 'contractsent'` (Waiting for vote) |
| Date d'AG | `date_d_ag` |
| Taille d'AG | `nombre_de_lots` |
| Montant / ARR | `amount_in_home_currency` / `amount_tax_excluded` |
| Commercial | `hubspot_owner_id` (→ noms via `search_owners`) |
| Tâche en retard | objet `TASK`, `hs_task_is_overdue = true` |
