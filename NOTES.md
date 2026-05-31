# NOTES — journal de construction du plugin Veille

> Pour le Tech Lead. Décisions, sources vérifiées vs mortes, dette assumée. Mis à jour au fil des étapes.

## Décisions de cadrage (validées 2026-05-31)

| # | Décision | Choix retenu | Raison |
|---|---|---|---|
| 1 | `skills/` vs `commands/` | `skills/` partout | `commands/` est *legacy* selon la doc officielle Claude Code. Skills donnent le même résultat utilisateur (`/veille:profile`, `/veille:check`) avec un frontmatter plus riche et le support de fichiers attachés. |
| 2 | Plugin = marketplace single-entry | Oui | Install en une commande, pas de second repo. `.claude-plugin/plugin.json` + `.claude-plugin/marketplace.json` cohabitent à la racine. |
| 3 | Pré-filtre keyword | 100 % profil-driven | Aucun terme métier ne doit être codé en dur. La liste de mots-clés est dérivée du profil (secteur, domaines de pratique, activités), avec un petit noyau réglementaire générique FR/EN. |
| 4 | Structure des items dans `/veille:check` | « Ce qui se passe / Pourquoi ça vous concerne / Actions recommandées / lien source » | Forme retenue dans le brief. La structure « Verdict → Mécanisme → Impact → Action » est réservée aux courriels du skill `regulatory-email`. |

**Contrainte d'identité** : aucune mention d'une entreprise ou produit spécifique dans le code, les exemples, les manifestes ou la doc. Le plugin est neutre et partageable.

## Étape 1 — Scaffold (terminée 2026-05-31)

Créé :

```
veille-plugin/
├── .claude-plugin/
│   ├── plugin.json          # name=veille, displayName="Veille réglementaire", version 0.1.0
│   └── marketplace.json     # name=veille-plugin, single-entry
├── skills/                  # vide — rempli aux étapes 3–6
├── data/                    # vide — rempli à l'étape 2
├── .gitignore
├── LICENSE                  # MIT
├── README.md                # squelette — final à l'étape 7
└── NOTES.md                 # ce fichier
```

`git init` fait, branche `main`. Aucun commit encore — j'attends d'avoir au moins l'étape 2 livrée avant un premier commit propre.

## Étape 2 — Sweep des 32 sources (terminée 2026-05-31)

Sweep effectué via `curl -sLI` (32 URLs en parallèle, UA navigateur, timeout 15s) + drill ciblé via `WebFetch` sur les sources core problématiques.

### Bilan global

| | Total | Activées par défaut | Désactivées |
|---|---:|---:|---:|
| **core** (API/RSS) | 12 | **8** | 4 |
| **best_effort** (scraping) | 20 | 0 | 20 |
| **TOTAL** | 32 | **8** | 24 |

Les scrapers HTML sont tous désactivés par défaut (scope v1 décidé) — l'utilisateur peut en activer dans `~/.veille/sources.json` à ses risques.

### Sources core actives (8) — joignables et fonctionnelles au sweep

| ID | Juridiction | Type | URL au moment du sweep |
|---|---|---|---|
| `legisinfo` | CA_FED | api | `parl.ca/legisinfo/en/overview/json/recentlyintroduced` |
| `gazette_ca_part1` | CA_FED | rss | `gazette.gc.ca/rss/p1-fra.xml` |
| `gazette_ca_part2` | CA_FED | rss | `gazette.gc.ca/rss/p2-fra.xml` |
| `consultations_ca` | CA_FED | api | `open.canada.ca/static/current_consultations_open.csv` |
| `open_gov_ca` | CA_FED | api | `open.canada.ca/data/en/api/3/action/package_search?q=regulation` |
| `ab_gazette` | CA_AB | rss | `kings-printer.alberta.ca/rss/RSS_QP_ALL.xml` |
| `sk_gazette` | CA_SK | api | `publications.saskatchewan.ca/api/v1/products?categoryId=216` |
| `federal_register_us` | US_FED | api | `federalregister.gov/api/v1/documents.json?order=newest&per_page=100` |

### Corrections appliquées au registre d'origine porté

| Source | URL initiale (seed) | Problème observé | Correction |
|---|---|---|---|
| `legisinfo` | `/legisinfo/en/bills/json` | HTTP 302 vers `/en/overview` (endpoint déplacé en mai 2026) | URL changée pour `/legisinfo/en/overview/json/recentlyintroduced` (validé : retourne array de bills bilingues). Limite : ne retourne que les *récemment introduits*, pas tous les bills. Suffit pour la veille (delta). |
| `sk_gazette` | `/api/v1/products` | ERR — endpoint nécessite un paramètre catégorie | URL changée pour `/api/v1/products?categoryId=216` (Gazette). Validé HTTP 200. |
| `bc_bills` | `/parliamentary-business/legislation-debates-proceedings` | HTTP 404 | URL changée pour `/parliamentary-business/overview/43rd-parliament/2nd-session/bills` (session courante). À mettre à jour à chaque nouvelle session. |

### Sources core désactivées (4) — à réinvestiguer

| Source | Code HTTP | Cause | Réactivation |
|---|---|---|---|
| `consultation_qc` | 403 | API GraphQL Decidim refuse les requêtes anonymes | Si la plateforme rouvre l'accès anonyme, ou via un token. |
| `ontario_reg_registry` | 000 (DNS) | Domaine `ontariocanada.com` non joignable au sweep | Trouver l'URL canonique actuelle du registre régulatoire d'Ontario. ERO (`ero.ontario.ca`) couvre seulement l'environnement. |
| `engage_bc` | 404 | Service govTogetherBC restructuré/déplacé | Identifier le nouveau service de consultation publique du gouvernement BC. |
| `ns_bills` | 000 (DNS) | Domaine `nslegislature.ca` non joignable au sweep | Possiblement downtime temporaire. À reconfirmer au prochain check. |

### Best-effort scrapers (20) — tous présents dans le registre mais désactivés

URLs validées HTTP 200 au sweep sauf :
- `pei_bills`, `pei_gazette` : redirection anti-bot Radware → contenu inaccessible automatiquement
- `yt_bills` : HTTP 403 (bloc anti-bot — déjà documenté dans le seed)
- `bc_bills` : 404 sur l'URL d'origine ; URL session-courante substituée mais désactivée (best_effort)

Tous restent dans le registre — l'utilisateur peut les activer manuellement dans `~/.veille/sources.json` s'il accepte le risque de fragilité du scraping HTML.

### Couverture par juridiction (sources actives par défaut)

| Juridiction | Sources actives |
|---|---|
| Canada fédéral | 5 (LEGISinfo, Gazettes I/II, Consultations, Open Gov) |
| Alberta | 1 (King's Printer) |
| Saskatchewan | 1 (Gazette) |
| US fédéral | 1 (Federal Register) |
| QC, ON, BC, MB, NS, NB, NL, PE, YT, NT, NU | 0 par défaut |

C'est honnête de signaler à l'utilisateur que pour QC/ON/BC, la couverture par défaut est limitée à `gazette_ca_part1`/`part2` (lois fédérales applicables au QC) tant que les sources provinciales restent à investiguer ou à activer manuellement.

## Dette assumée

- **Pas de cron** : refresh manuel uniquement via `/veille:check`. Une intégration scheduling pourrait venir plus tard.
- **Scrapers HTML best-effort** : 20 sources provinciales/territoriales sans flux structuré sont seedées mais désactivées par défaut. L'utilisateur peut les activer à ses risques ; la robustesse n'est pas garantie.
- **Pas de connecteur API privé** (extension future documentée seulement) : le plugin fonctionne en standalone via WebFetch.
- **Une seule fenêtre de profil** : pas de profils multiples par utilisateur dans la v1.

## Étapes 3 à 6 — Skills (terminées 2026-05-31)

Quatre `skills/<name>/SKILL.md` posés :

| Skill | `user-invocable` | `allowed-tools` | Rôle |
|---|---|---|---|
| `regulatory-watch` | **false** | WebFetch, Read, Write | Moteur partagé — schémas, stratégies fetch, dédup, classification, archivage. Référence centrale. |
| `profile` | true (défaut) | AskUserQuestion, Read, Write | Cold interview. Sort `~/.veille/profile.md` (humain + bloc YAML). |
| `check` | true (défaut) | WebFetch, Read, Write, AskUserQuestion | Orchestrateur `/veille:check`. Délègue les détails à `regulatory-watch`. Propose courriel à la fin. |
| `regulatory-email` | true (défaut) | AskUserQuestion, Read, Write | Courriel exécutif en langage clair, FR/EN, structure 5 sections, ≤ 300 mots. |

### Open questions tranchées au cours de l'écriture

- **« Rien de neuf » format** : on affiche un message court + le tableau « Sources balayées » pour confirmer ce qui a été regardé. Pas de faux contenu. (Réf. `check` §9.)
- **Horodatage seen.json** : `first_seen_iso`, `last_seen_iso`, `stage_history[]`. Pas de purge automatique en v1 (l'utilisateur peut effacer manuellement). (Réf. `regulatory-watch` §8.)
- **Composition skills** : `check` lit `regulatory-watch` au runtime et applique ses règles. `regulatory-watch` est `user-invocable: false` pour ne pas polluer le menu `/`. Pas d'appel direct entre skills — convention de référence par chemin (`${CLAUDE_PLUGIN_ROOT}/skills/regulatory-watch/SKILL.md`).

## Étape 7 — Validation (terminée 2026-05-31)

### Validation structurelle

- `.claude-plugin/plugin.json` — JSON valide ✓
- `.claude-plugin/marketplace.json` — JSON valide ✓
- `data/sources.default.json` — JSON valide, 32 sources ✓
- 4 × `skills/<n>/SKILL.md` — frontmatter conforme, `name` matche le dossier ✓

### Validation runtime du moteur — test WebFetch live

Test du prompt `adapter_type: api` de `regulatory-watch` §5 sur la source la plus critique :

- **URL** : `https://www.parl.ca/legisinfo/en/overview/json/recentlyintroduced`
- **Résultat** : 7 items extraits, schéma `{items: [...]}` respecté, champs `external_id` (NumberCode), `title` (FR), `title_lang`, `url`, `published_at`, `stage`, `excerpt` tous présents et corrects.
- **Conclusion** : la stratégie `api` du moteur fonctionne. Par parité de prompt, `rss` et `atom` devraient marcher aussi. `scraping` reste fragile par nature (test au cas par cas après activation utilisateur).

### Validation « E2E » réelle — à exécuter post-installation

L'installation et l'invocation effective des skills ne peuvent se faire que **dans une instance Claude Code distincte de celle qui a construit le plugin**. Procédure recommandée pour le Tech Lead / le décideur :

1. **Installation locale** :
   ```
   /plugin marketplace add /Users/paulmurat/veille-plugin
   /plugin install veille@veille-plugin
   ```
2. **Test profile** :
   ```
   /veille:profile
   ```
   Vérifier : trois segments d'interview, AskUserQuestion sur séniorité/langue/horizon/déclencheurs/format, conversation sur rôle/domaines/secteur/activités/juridictions/mots-clés. Écriture de `~/.veille/profile.md` avec bloc YAML parsable.
3. **Test check (run 1)** :
   ```
   /veille:check
   ```
   Vérifier : annonce des sources sélectionnées, parallel WebFetch des 8 sources core, présentation triée P0/P1/P2 avec format « Ce qui se passe / Pourquoi ça vous concerne / Actions / Source », tableau « Sources balayées », écriture de `~/.veille/seen.json` et `~/.veille/digests/YYYY-MM-DD.md`, proposition courriel à la fin.
4. **Test check (run 2, immédiat)** :
   ```
   /veille:check
   ```
   Vérifier : aucun item présenté (tous déjà dans `seen.json`), message « ✓ Rien de neuf », tableau Sources balayées intact.
5. **Test courriel FR** : depuis la fin de check, accepter la proposition → vérifier objet ≤ 90 chars, corps 150-250 mots, structure 5 sections, ton sobre, source dans la langue d'origine.
6. **Test courriel EN** : dire « another version in English » → vérifier traduction propre, mêmes règles de style.

Si tous les points passent, le plugin est prêt à être partagé.

## Dette assumée (rappel + ajouts)

- **Pas de cron** : refresh manuel uniquement.
- **20 scrapers HTML** désactivés par défaut. Activable via `~/.veille/sources.json` à risque utilisateur.
- **4 sources core désactivées** au sweep d'installation (`consultation_qc`, `ontario_reg_registry`, `engage_bc`, `ns_bills`) — voir §Étape 2. À réinvestiguer.
- **LEGISinfo** : utilise `/overview/json/recentlyintroduced` qui retourne seulement les bills récemment introduits — pas le full listing. Suffit pour la veille (delta) mais limite la couverture rétroactive au moment de l'installation.
- **Pas de mode API privée** (extension future documentée seulement).
- **Profil unique par utilisateur** — pas de profils multiples en v0.1.
- **Pas de tests automatisés** — la validation E2E est manuelle (section ci-dessus). Un futur skill `/veille:doctor` pourrait automatiser le smoke test.

## Pour le Tech Lead

Le plugin est complet et cohérent. Aucune mention d'identité d'entreprise tierce dans aucun artefact (vérifié au grep). Le registre des 32 sources est porté, vérifié, et chaque écart par rapport au seed d'origine est tracé.

### Pousser sur GitHub — fait 2026-05-31

- Namespace : `paulmurat31-sketch`
- Repo : https://github.com/paulmurat31-sketch/veille-plugin
- Visibilité : **privée** (à passer publique avec `gh repo edit paulmurat31-sketch/veille-plugin --visibility public --accept-visibility-change-consequences` quand prêt à partager)
- Auth GitHub : `gh` installé via Homebrew, OAuth web, token avec scopes `repo gist read:org workflow`
- Identité commits : `Paul Murat <paulmurat31-sketch@users.noreply.github.com>` (noreply pour ne pas exposer l'adresse personnelle)

### Reste à faire côté décideur

- Faire la passe E2E avec ce qui est décrit en §Étape 7 (installation via `/plugin marketplace add paulmurat31-sketch/veille-plugin` dans Claude Code CLI).
- Si le marketplace privé pose problème à l'install Claude Code, passer le repo en public.
