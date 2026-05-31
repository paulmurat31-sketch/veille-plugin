---
name: check
description: Refresh manuel de la veille réglementaire. Charge le profil utilisateur, sweepe les sources publiques pertinentes, ne montre que ce qui est nouveau (ou a changé d'étape) depuis le dernier passage, trié par priorité (P0 / P1 / P2). Bilingue selon le profil. À la fin, propose de générer un courriel sur un item.
when_to_use: Quand l'utilisateur tape /veille:check, ou demande à « regarder les nouveautés », « faire une veille », « refresh », « digest réglementaire ».
argument-hint: "[--all | --include-best-effort | --source <id>]"
allowed-tools: WebFetch, Read, Write, AskUserQuestion
---

# `/veille:check` — refresh manuel

Vous orchestrez un cycle de veille complet de bout en bout. Tous les détails techniques (schémas d'état, stratégies de fetch par type d'adapter, format de présentation, règles de dédup, matrice pertinence × urgence) sont définis dans le skill `regulatory-watch` — **consultez-le et appliquez-le** : c'est le moteur partagé.

Référence interne : `${CLAUDE_PLUGIN_ROOT}/skills/regulatory-watch/SKILL.md`. Si vous n'avez pas déjà ce contenu chargé en contexte, lisez-le maintenant.

## Arguments acceptés

L'utilisateur peut passer ces flags via `$ARGUMENTS` :

| Flag | Effet |
|---|---|
| (aucun) | Sweep par défaut : sources `core` actives ∩ juridictions du profil |
| `--all` | Inclut **toutes** les sources, y compris `best_effort` et désactivées |
| `--include-best-effort` | Inclut les sources `best_effort` (sans toucher aux désactivées) |
| `--source <id>` | Sweep d'une seule source (override les filtres) |

Si plusieurs flags se contredisent, expliquer le conflit et demander confirmation.

## Procédure pas à pas

### 1. Précharger le moteur

Si `regulatory-watch/SKILL.md` n'est pas en contexte, le lire. Vous en aurez besoin pour :
- les schémas `profile.md`, `seen.json`, `sources.default.json`
- les stratégies de fetch par `adapter_type`
- la matrice pertinence × urgence → priorité
- le format de présentation des items
- l'archivage du digest

### 2. Charger le profil utilisateur

Lire `~/.veille/profile.md` (ou `$VEILLE_STATE_DIR/profile.md` si défini).

- **Profil absent** : dire *« Aucun profil détecté. Lancez d'abord `/veille:profile` pour configurer votre contexte. »* et arrêter.
- **Profil présent, YAML invalide** : dire *« Profil détecté mais malformé (bloc YAML manquant ou invalide). Relancez `/veille:profile` pour le reconstruire. »* et arrêter.
- **Profil OK** : récapituler en deux lignes (juridictions, langue) pour confirmer à l'utilisateur ce qui va être appliqué.

### 3. Charger seen.json

- Lire `~/.veille/seen.json` s'il existe.
- S'il n'existe pas, initialiser en mémoire avec `{ "schema_version": 1, "last_check_iso": null, "items": {} }` et noter que c'est un **premier check** (tous les items seront « nouveaux »).

### 4. Charger le registre de sources

Cascade :
1. Si `~/.veille/sources.json` existe → c'est le registre actif.
2. Sinon → lire `${CLAUDE_PLUGIN_ROOT}/data/sources.default.json`.

### 5. Filtrer les sources

Appliquer les règles de `regulatory-watch` §4 (juridiction ∩ activation) puis les flags utilisateur de la section *Arguments* ci-dessus.

**Annoncer explicitement** avant le fetch :

```
Sources sélectionnées : N
  - actives par défaut : N
  - ajoutées par flag : N
Écartées : M
  - hors juridiction du profil : K
  - désactivées (URL en panne ou anti-bot) : K — détaillées dans NOTES.md du plugin
```

Si zéro source → arrêter avec un message clair (voir §4 du moteur).

### 6. Fetcher les sources

Pour chaque source sélectionnée, utiliser `WebFetch` avec le prompt approprié à son `adapter_type` (cf. `regulatory-watch` §5). Vous pouvez paralléliser les WebFetch dans une seule réponse pour aller plus vite.

Pour chaque source qui échoue, consigner l'incident (raison, ne fait pas crasher le run).

### 7. Pré-filtrer, dédupliquer, classer

Pour chaque item retourné par les adapters :

1. **Pré-filtre keyword** (cf. moteur §6) : la liste de mots-clés à utiliser est construite à partir du profil.
2. **Dédup** contre `seen.json` :
   - item inconnu → marquer `NEW`
   - item connu, même `stage` → ignorer (mettre à jour `last_seen_iso` dans seen.json)
   - item connu, `stage` différent → marquer `STAGE_CHANGED` (à présenter avec préfixe « ⏩ Étape changée »)
3. **Classer** les `NEW` et `STAGE_CHANGED` selon la matrice du moteur §7 — la classification est **pilotée par le profil**, pas par une taxonomie figée. Pour chaque item, produire pertinence + urgence + priorité (P0 / P1 / P2) + `pourquoi_concerne` + `actions`.

Token-conscience : si on a >50 items à classer, prévenir l'utilisateur et lui demander s'il veut continuer ou limiter (top-50 par exemple).

### 8. Présenter le digest

Suivre le format de `regulatory-watch` §9.

Ordre :
1. **P0 (🔴)** — à traiter cette semaine.
2. **P1 (🟡)** — à traiter ce mois-ci.
3. **P2 (🟢)** — surveillance.

Pour chaque item :

```
### {🔴|🟡|🟢} {titre — préfixé "⏩ Étape changée : " si STAGE_CHANGED}

- **Ce qui se passe** : 1-2 phrases factuelles, code/numéro + étape
- **Pourquoi ça vous concerne** : lien explicite au profil (1-2 phrases)
- **Actions recommandées** :
  - …
  - …
- Source : [{source_name}]({url})
```

Inclure en fin de digest une section **« Sources balayées »** :

```
## Sources balayées

| Source | Items | Statut |
|---|---|---|
| LEGISinfo | 7 | ok |
| Gazette CA I | 5 | ok |
| US Federal Register | 100 | ok (tronqué à 100) |
| Gazette CA II | — | ⚠ HTTP 503 |
| ... | ... | ... |
```

### 9. Cas « rien de neuf »

Si aucun item ne passe à présenter (zéro nouveau, zéro changement d'étape), le **dire proprement** — pas de faux contenu, pas de remplissage :

```
✓ Rien de neuf depuis le dernier passage.
  • {N} items vus au total ({M} nouveaux, 0 à signaler — tous écartés par votre pré-filtre)
  • Dernier check : {date}
  • Sources balayées : {N} (voir tableau ci-dessous)
```

Inclure quand même le tableau « Sources balayées » pour confirmer ce qui a été regardé.

### 10. Persister `seen.json`

Mettre à jour le fichier avec :
- `last_check_iso` = maintenant
- pour chaque item rencontré : ajout ou mise à jour selon les règles de dédup du moteur §8
- inclure les items pré-filtrés (avec `"prefiltered": true`) pour éviter de les reclasser au prochain run

Écrire `~/.veille/seen.json` en JSON pretty-printé (indent 2).

### 11. Archiver le digest

Écrire le digest markdown complet dans `~/.veille/digests/YYYY-MM-DD.md` (créer le dossier si nécessaire). Si un digest existe déjà pour aujourd'hui, l'écraser.

Préfixer avec le bloc de métadonnées du run (cf. moteur §10).

### 12. Proposer un courriel

À la fin, **toujours** demander :

```
AskUserQuestion :
  question : « Voulez-vous que je vous rédige un courriel sur un de ces points pour un cadre non-juriste ? »
  header   : "Courriel"
  options  :
    - "Oui — un courriel sur un item P0"
    - "Oui — choisir un autre item"
    - "Non, c'est tout pour aujourd'hui"
    - "Plus tard, je reviendrai"
  multiSelect : false
```

Si l'utilisateur dit oui :
- Lui demander de pointer l'item (par titre ou numéro) si plusieurs candidats.
- Basculer sur le skill `regulatory-email` avec l'item choisi en contexte.

Si l'utilisateur dit non : conclure en deux lignes (date du prochain check suggéré : selon le rythme habituel, ou « à votre rythme »).

## Cas particuliers

- **Premier check (seen.json absent)** : annoncer *« Premier check — tous les items rencontrés seront marqués comme nouveaux. Au prochain passage, vous ne verrez que les changements. »*
- **Profil très large** (toutes juridictions, horizon 180 jours, beaucoup de keywords) : prévenir que le digest risque d'être long et proposer un filtre additionnel.
- **Une seule source ciblée (`--source`)** : ne pas chercher de delta sur les autres ; le tableau « Sources balayées » n'aura qu'une ligne.
- **Erreurs réseau massives** (>50 % des sources en échec) : signaler que la connectivité semble dégradée et que le digest est incomplet.

## Ce qu'il ne faut PAS faire

- Ne **pas** inventer un item qui n'existe pas dans le flux fetché.
- Ne **pas** classer un item avec une priorité plus alarmante que ce que le profil supporte (la classification est pilotée par le profil — si rien dans le profil ne touche l'item, c'est `P2` ou pré-filtré).
- Ne **pas** afficher d'items déjà vus (seul le delta intéresse).
- Ne **pas** écrire le digest s'il est vide — l'archive du jour ne doit exister que s'il y a du contenu signifiant (ou si c'est explicitement le premier check).
