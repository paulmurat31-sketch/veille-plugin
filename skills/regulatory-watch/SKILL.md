---
name: regulatory-watch
description: Moteur partagé du plugin Veille. Définit les schémas d'état (profil, seen.json, registre de sources), les stratégies de fetch par type d'adapter (api / atom / rss / scraping), le pré-filtre keyword profil-driven, la rubrique de classification pertinence × urgence, les clés de dédup et la dégradation gracieuse. Invoqué par /veille:check ; pas destiné à un appel utilisateur direct.
when_to_use: À charger automatiquement quand un autre skill du plugin Veille (notamment /veille:check) doit fetcher des sources, dédupliquer contre l'historique vu, ou classer un item selon le profil de l'utilisateur. Ne pas invoquer pour autre chose.
user-invocable: false
allowed-tools: WebFetch, Read, Write
---

# Moteur Veille — `regulatory-watch`

Ce document est la **référence opérationnelle partagée** du plugin Veille. Les autres skills (`/veille:check` en particulier) suivent les conventions, schémas et procédures décrits ici.

Il n'y a pas de code exécutable côté plugin : tout passe par le main agent Claude qui orchestre `WebFetch`, `Read` et `Write`. La présence d'un `SKILL.md` dédié garantit que les choix techniques sont centralisés (un seul endroit à modifier pour faire évoluer le moteur).

## 1. État local sur le poste de l'utilisateur

Tout vit sous `~/.veille/` (créer le dossier si absent au premier passage). Aucun fichier n'est partagé entre utilisateurs ; le plugin lui-même ne contient jamais d'état utilisateur.

```
~/.veille/
├── profile.md            ← Profil de l'utilisateur. Créé par /veille:profile. Lisible humain + bloc YAML parsable.
├── sources.json          ← (optionnel) Surcharge du registre de sources. Si absent, on lit data/sources.default.json du plugin.
├── seen.json             ← Historique des items déjà vus. Maintenu par /veille:check.
└── digests/
    └── YYYY-MM-DD.md     ← Archive du digest du jour (un par check si le jour change).
```

Si l'utilisateur veut un emplacement différent, il peut exporter `VEILLE_STATE_DIR=/chemin/autre` ; le skill lit cette variable d'environnement si elle est définie, sinon `~/.veille/`.

## 2. Schéma du profil (`~/.veille/profile.md`)

Le fichier est en deux parties : une **section humaine** en haut (que l'utilisateur peut éditer à la main), un **bloc YAML structuré** en bas (que les skills parsent).

Exemple de bloc YAML (cible) :

```yaml
profile_version: 1
created_at: "2026-05-31"
updated_at: "2026-05-31"

# Identité et rôle
role: "Avocat senior en droit du travail"
seniority: "senior"          # junior | mid | senior | partner | in-house | other
practice_areas:              # libellés libres ; servent à dériver des mots-clés
  - "droit du travail"
  - "santé et sécurité au travail"
industry: "Manufacture aérospatiale"
organization_activities: |
  Conception, fabrication et vente de composants aérospatiaux ;
  export vers US, EU et Asie-Pacifique ; ~800 employés au Québec.

# Couverture
jurisdictions:               # codes du registre (CA_FED, CA_QC, ..., US_FED)
  - CA_FED
  - CA_QC
  - CA_ON
  - US_FED
languages:
  primary: fr                # fr | en
  secondary_accepted:        # langues acceptables si la source n'a pas la primary
    - en

# Ce qui rend un item « urgent » pour cet utilisateur
urgency_criteria:
  urgent_horizon_days: 90    # entrée en vigueur dans < N jours → urgent
  also_urgent_on:            # déclencheurs additionnels
    - new_bill_in_scope
    - consultation_deadline_imminent
    - stage_change_to_royal_assent_or_in_force

# Mots-clés dérivés du profil (servent au pré-filtre du moteur)
keywords_seed:
  fr:
    - "travail"
    - "normes du travail"
    - "santé et sécurité"
    - "aérospatiale"
    - "manufacture"
    - "export contrôlé"
  en:
    - "labour"
    - "employment standards"
    - "occupational health"
    - "aerospace"
    - "manufacturing"
    - "export controls"

# Préférences de présentation
output_preferences:
  digest_length: short       # short | medium | full
  structure:                 # ordre des sections par item
    - whats_happening
    - why_relevant
    - actions
    - source_link
```

Au moment du parsing : extraire le **dernier** bloc ` ```yaml ` du fichier (l'utilisateur peut avoir mis d'autres blocs `yaml` en illustration). Si le fichier n'a pas de bloc YAML parsable, lever l'erreur « Profil malformé — relancer `/veille:profile` ».

## 3. Schéma du registre de sources

Lecture en cascade :

1. Si `~/.veille/sources.json` existe → c'est le registre actif (surcharge totale).
2. Sinon → lire `${CLAUDE_PLUGIN_ROOT}/data/sources.default.json` (livré avec le plugin).

Schéma de chaque source (cf. `data/sources.default.json` pour les exemples) :

```json
{
  "id": "string-stable-kebab-case",
  "name": "Nom lisible (FR)",
  "jurisdiction": "CA_FED | CA_QC | CA_ON | CA_BC | CA_AB | CA_MB | CA_SK | CA_NS | CA_NB | CA_NL | CA_PE | CA_YT | CA_NT | CA_NU | US_FED | EU | UK",
  "url": "https://...",
  "adapter_type": "api | atom | rss | scraping",
  "reliability": "core | best_effort",
  "enabled_by_default": true | false,
  "fetch_notes": "Comment parser, particularités, clé de dédup. Lu et affiché à l'utilisateur si demande de debug."
}
```

## 4. Sélection des sources à interroger

Étant donnés un profil et un registre :

1. **Filtre juridictions** : ne garder que les sources dont `jurisdiction` est dans `profile.jurisdictions`.
2. **Filtre activation** : par défaut, ne garder que `enabled_by_default: true`. Flags du skill `check` peuvent surcharger :
   - `--all` : inclure aussi les sources `enabled_by_default: false`.
   - `--source <id>` : sweeper uniquement cette source (override tous les filtres).
   - `--include-best-effort` : inclure les `reliability: best_effort` mais pas les `disabled`.
3. **Annonce explicite** : avant de fetcher, dire à l'utilisateur combien de sources sont sélectionnées, et combien ont été écartées par les filtres (avec la raison).

Si le résultat est zéro source : arrêter et expliquer pourquoi (pas de juridiction commune ? toutes les sources de la juridiction sont disabled ?).

## 5. Stratégies de fetch par `adapter_type`

Toutes les requêtes passent par l'outil `WebFetch`. Pour chaque source :

### `adapter_type: api`

Endpoint JSON ou CSV. Prompt WebFetch :

> « Extrais la liste des items du flux à cette URL. Pour chaque item retourne un objet : `external_id` (identifiant stable selon `fetch_notes`), `title` (préfère la langue primary du profil), `title_lang` (`fr` ou `en`), `url` (lien vers la fiche/document), `published_at` (ISO 8601 si disponible), `stage` (étape législative ou statut si disponible), `excerpt` (résumé ou abstract si présent, max 2000 caractères). Si le payload est un CSV, lis-le comme une table. Retourne au maximum 100 items, triés du plus récent au plus ancien. Réponds en JSON strict, format `{"items": [...]}` sans commentaire. »

### `adapter_type: rss` / `adapter_type: atom`

Flux XML. Prompt WebFetch :

> « Cette URL est un flux RSS ou Atom. Pour chaque `<item>` (RSS) ou `<entry>` (Atom) retourne : `external_id` (préfère `<guid>`, sinon `<id>`, sinon `<link>`), `title`, `title_lang`, `url`, `published_at` (depuis `<pubDate>` ou `<published>`), `excerpt` (depuis `<description>` ou `<summary>`, max 2000 chars). Pas de `stage` sauf si explicitement mentionné dans le titre. Retourne tout (jusqu'à 100), du plus récent au plus ancien. JSON strict `{"items": [...]}`. »

### `adapter_type: scraping`

HTML non structuré. Plus fragile. Prompt WebFetch :

> « Cette URL est une page HTML listant des éléments réglementaires (projets de loi, règlements, ou avis de gazette). Identifie la liste principale et pour chaque entrée extraire : `external_id` (le numéro, code, ou identifiant unique visible sur la ligne — si rien d'évident, utilise un slug stable de l'URL), `title`, `title_lang`, `url` (lien absolu vers la fiche), `published_at` (si une date est visible), `stage` (si un statut est visible). Si la page ne ressemble pas à une liste, ou ne semble pas être la bonne URL, dis-le explicitement plutôt que d'inventer. JSON strict `{"items": [...]}`. »

### Dégradation gracieuse — règles dures

- Une source qui échoue (timeout, erreur HTTP, parsing impossible) **ne fait pas échouer le digest**. On consigne l'incident et on continue avec les autres.
- En fin de check, lister explicitement les sources qui ont échoué avec un motif d'une ligne (ex. « `engage_bc` — HTTP 404, source possiblement déplacée »).
- Si **toutes** les sources échouent : dire « aucune source joignable — vérifier la connexion réseau ou les URLs du registre ».
- Limite de tokens : si une source retourne >100 items, n'en garder que les 100 plus récents et noter le tronquage.

## 6. Pré-filtre keyword profil-driven

Avant de classer un item (étape coûteuse), faire un filtre simple sur les mots-clés du profil pour écarter le bruit massif (santé publique, agriculture, pêcheries… selon le profil).

**Construction de la liste de mots-clés** :

1. Noyau réglementaire générique FR/EN (toujours inclus) :
   - FR : `loi`, `règlement`, `projet de loi`, `consultation`, `entrée en vigueur`, `arrêté`, `décret`
   - EN : `act`, `bill`, `regulation`, `consultation`, `notice`, `proposed rule`, `final rule`
2. **Mots-clés du profil** (`profile.keywords_seed.fr` + `profile.keywords_seed.en`).
3. **Mots-clés dérivés** :
   - Chaque `practice_area` → casse + minuscule (ex. « droit du travail » → terme tel quel)
   - `industry` → minuscule, mots de >3 caractères extraits
   - Tokens significatifs de `organization_activities`

**Application** :

- Construire un texte de recherche : `f"{title.lower()} {excerpt.lower()}"`.
- Si **aucun** mot-clé du profil ne match → marquer l'item `prefiltered: true` et ne PAS le présenter (mais le mémoriser dans `seen.json` pour ne pas le re-considérer).
- Si **au moins un** mot-clé match → passer à la classification.

Cette logique épargne ~70 % des items typiquement non-pertinents tout en restant volontairement large.

## 7. Classification — pertinence × urgence → priorité

Pour chaque item qui a passé le pré-filtre, demander au main agent Claude de **classer en s'appuyant sur le profil** (pas sur une taxonomie figée) :

Pour chaque item, produire :

- `pertinence` : `🔴 forte` | `🟡 modérée` | `🟢 marginale`
- `urgence` : `🔴 immédiate` | `🟡 court terme` | `🟢 surveillance`
- `priorite` (dérivée du croisement, voir matrice ci-dessous)
- `pourquoi_concerne` : une à deux phrases expliquant le lien avec le profil
- `actions` : 1 à 3 actions concrètes recommandées (ex. « ajouter à la veille des prochaines semaines », « consulter le service paie », « préparer une réponse à la consultation avant le YYYY-MM-DD »)

### Matrice de croisement pertinence × urgence → `priorite`

|                  | 🔴 immédiate | 🟡 court terme | 🟢 surveillance |
|------------------|-------------|----------------|-----------------|
| 🔴 **forte**     | 🔴 P0       | 🔴 P0          | 🟡 P1           |
| 🟡 **modérée**   | 🔴 P0       | 🟡 P1          | 🟡 P1           |
| 🟢 **marginale** | 🟡 P1       | 🟢 P2          | 🟢 P2           |

- `P0` (🔴) — à traiter cette semaine.
- `P1` (🟡) — à traiter ce mois-ci.
- `P2` (🟢) — surveillance passive.

### Critères d'urgence — pilotés par le profil

Lire `profile.urgency_criteria` :
- `urgent_horizon_days` : si la date d'entrée en vigueur (ou de fin de consultation) tombe dans < N jours → `urgence: immédiate`.
- `also_urgent_on` : déclencheurs additionnels (ex. nouveau projet de loi dans le périmètre, changement d'étape vers `Royal Assent` ou `In Force`).

S'il n'y a aucune date claire, retomber sur `surveillance` par défaut (et le signaler).

### Critères de pertinence — pilotés par le profil

Le main agent doit raisonner en s'appuyant sur :
- `practice_areas` (un item directement dans un domaine listé → `forte`)
- `industry` et `organization_activities` (un item touche le secteur listé ou une activité explicitement nommée → `forte`)
- `jurisdictions` (déjà filtré, mais une mesure transversale fédérale qui touche la juridiction de l'utilisateur → `forte`)
- Mention indirecte / pourrait s'appliquer → `modérée`
- Mention très tangentielle → `marginale`

Ne jamais inventer de fait. Si une donnée importante manque dans l'extrait, l'indiquer ouvertement (« contenu pas assez détaillé pour conclure — à lire en source »).

## 8. Schéma `~/.veille/seen.json`

Source de vérité pour le delta entre deux runs.

```json
{
  "schema_version": 1,
  "last_check_iso": "2026-05-31T18:00:00Z",
  "items": {
    "legisinfo:S-248": {
      "source_id": "legisinfo",
      "external_id": "S-248",
      "first_seen_iso": "2026-05-31T18:00:00Z",
      "last_seen_iso": "2026-05-31T18:00:00Z",
      "last_stage": "Senate second reading",
      "stage_history": [
        { "iso": "2026-05-31", "stage": "Senate second reading" }
      ],
      "last_priority": "P1",
      "last_title": "An Act respecting Caribbean Heritage Month"
    }
  }
}
```

**Clé de dédup** : `{source_id}:{external_id}`. L'`external_id` doit être stable selon les conventions par source (cf. `fetch_notes` du registre).

**Règles de mise à jour** :

- Item **jamais vu** → l'ajouter à `items`, marquer `first_seen_iso = last_seen_iso = now`, à présenter à l'utilisateur dans le digest.
- Item **déjà vu, même `stage`** → mettre à jour `last_seen_iso` uniquement, ne PAS présenter (rien de nouveau).
- Item **déjà vu, `stage` différent** → mettre à jour `last_stage`, pousser sur `stage_history`, **présenter** comme « changement d'étape ».
- Item retourné par le pré-filtre (donc rejeté par mots-clés) → mémoriser avec une marque `"prefiltered": true` pour ne pas le reclasser au prochain run.

**Hygiène** : pas de purge automatique en v1. L'utilisateur peut effacer `seen.json` manuellement pour repartir à zéro.

## 9. Format de présentation des items dans le digest

Pour chaque item à présenter :

```
### {priorite} {titre}

- **Ce qui se passe** : {1-2 phrases factuelles, mentionne le code/numéro et l'étape}
- **Pourquoi ça vous concerne** : {1-2 phrases liant explicitement au profil de l'utilisateur}
- **Actions recommandées** :
  - {action 1}
  - {action 2 (max 3)}
- Source : [{source_name}]({url})
```

Si un item est un **changement d'étape** d'un item déjà connu, préfixer le titre par « ⏩ Étape changée : ». Inclure une mention de l'ancienne → nouvelle étape.

L'ordre dans le digest :

1. D'abord les `P0` triés par juridiction du profil (primaire d'abord), puis par titre.
2. Puis les `P1` mêmes critères.
3. Puis les `P2` mêmes critères.
4. En fin de digest, une section « Sources balayées » avec le compte par source + signalement des sources en erreur.

## 10. Archivage du digest

À la fin d'un run réussi, écrire le digest complet (markdown) dans `~/.veille/digests/YYYY-MM-DD.md` (créer le dossier si nécessaire). Si un digest existe déjà pour la date du jour, l'écraser par le nouveau (le dernier run du jour fait foi).

Ajouter en tête du fichier les métadonnées du run :

```yaml
---
run_iso: "2026-05-31T18:00:00Z"
profile_jurisdictions: [CA_FED, CA_QC, CA_ON, US_FED]
sources_swept: 8
sources_failed: 0
items_seen_total: 47
items_prefiltered: 33
items_new_or_changed: 5
items_p0: 0
items_p1: 2
items_p2: 3
---
```

## 11. Token-conscience

- Par défaut, n'interroger que les sources `enabled_by_default: true` (8 actuellement) — c'est ce qui tient à l'échelle.
- Signaler en début de digest combien de sources ont été **sautées** par filtre (juridiction, désactivation), pour que l'utilisateur sache ce qu'il ne voit pas.
- Si l'utilisateur a inclus `--all` ou `--include-best-effort` et qu'on dépasse 20 sources, prévenir : « ça va prendre N appels WebFetch ».
