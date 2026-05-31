# Veille — plugin Claude Code

> Veille réglementaire personnelle pour juristes. Profil utilisateur, sweep de sources publiques, digest classé par pertinence et urgence, courriels prêts à envoyer en langage clair. Bilingue FR/EN, autonome, 100 % local.

## Concept

Un plugin Claude Code qu'un juriste installe pour **sa propre veille**. Il :

1. **Apprend votre profil** une fois (rôle, secteur, juridictions, critères d'urgence) — via `/veille:profile`.
2. **À la demande** (`/veille:check`), parcourt les sources gouvernementales publiques pertinentes et vous présente uniquement **ce qui est nouveau** depuis le dernier passage, trié par priorité.
3. **Quand un item mérite d'être communiqué** à un cadre non-juriste : dites « prépare un courriel sur X » et le plugin rédige en langage clair, dans la langue voulue.

**Pas de backend. Pas de DB. Pas de cron.** Tout passe par `WebFetch` sur des sites gouvernementaux publics, et l'état reste sur votre machine sous `~/.veille/`.

## Installation

Dans Claude Code :

```
/plugin marketplace add paulmurat31-sketch/veille-plugin
/plugin install veille@veille-plugin
```

Remplacez `paulmurat31-sketch` par le namespace GitHub du dépôt (ou le chemin local pour un test : `/plugin marketplace add ./veille-plugin`).

Puis :

```
/veille:profile
```

Comptez 2–3 minutes pour répondre. À la fin, vous avez un `~/.veille/profile.md` modifiable à la main.

Ensuite, à la fréquence qui vous convient :

```
/veille:check
```

## Commandes

| Commande | Quand l'utiliser |
|---|---|
| `/veille:profile` | Première fois, ou pour mettre à jour votre profil |
| `/veille:check` | Refresh manuel ; voir les nouveautés depuis le dernier passage |
| (auto) `/veille:regulatory-email` | Déclenchée par `/veille:check` ou en disant « prépare un courriel sur X » |

Le skill `regulatory-watch` est interne — il sert de moteur partagé aux commandes ci-dessus, vous n'avez pas à l'invoquer directement.

### Flags pour `/veille:check`

| Flag | Effet |
|---|---|
| (aucun) | Sources **core** actives ∩ juridictions du profil — recommandé |
| `--include-best-effort` | Ajoute les sources scraping HTML (plus fragiles) |
| `--all` | Inclut toutes les sources, même celles désactivées par défaut |
| `--source <id>` | Sweep d'une seule source (`legisinfo`, `gazette_ca_part1`, …) |

## Sources couvertes

32 sources gouvernementales canadiennes et US sont préchargées dans le registre `data/sources.default.json` :

- **Canada fédéral** : LEGISinfo (projets de loi), Gazette du Canada Parties I & II, Consultations gouvernementales, Open Government datasets.
- **Provinces et territoires** : projets de loi et gazettes officielles pour QC, ON, BC, AB, MB, SK, NS, NB, NL, PE, YT, NT, NU.
- **US fédéral** : Federal Register (rules, proposed rules, notices).

**8 sources sont activées par défaut** au lancement du plugin — celles qui exposent une API ou un flux RSS structuré et qui répondaient au sweep d'installation. Les 24 autres (dont 20 scrapers HTML) sont présentes dans le registre mais désactivées. Voir [`NOTES.md`](NOTES.md) pour le détail par source.

### Activer ou personnaliser une source

Créer un fichier `~/.veille/sources.json` (qui surcharge `data/sources.default.json` du plugin). Format identique. Quelques scénarios :

```jsonc
// Activer le scraper « ola_bills » (Ontario)
{
  "sources": [
    { "id": "ola_bills", "enabled_by_default": true /* + autres champs identiques */ }
  ]
}
```

```jsonc
// Ajouter une source perso
{
  "sources": [
    { "id": "ma-source", "name": "...", "jurisdiction": "CA_QC",
      "url": "https://...", "adapter_type": "rss",
      "reliability": "core", "enabled_by_default": true,
      "fetch_notes": "..." }
  ]
}
```

La structure complète est documentée dans [`skills/regulatory-watch/SKILL.md`](skills/regulatory-watch/SKILL.md) §3.

## Confidentialité

- **Aucune donnée n'est envoyée nulle part** sauf à Anthropic pour le raisonnement de Claude — comportement standard de Claude Code.
- Les sources interrogées sont **toutes des sites gouvernementaux publics** (parlements, gazettes officielles, registres de consultation). Aucun accès authentifié.
- Le profil utilisateur et l'historique « déjà vu » restent sous `~/.veille/` sur votre machine. Le plugin ne contient jamais d'état utilisateur.
- Partager le plugin (fork, marketplace public) **ne partage jamais** votre profil ni votre état.
- Pas de cron, pas de daemon, pas de processus en tâche de fond : le plugin agit uniquement quand vous invoquez une commande.

## Extension future — connecteurs API privés

Le plugin est conçu **standalone**. Une extension future pourrait y greffer des connecteurs à des bases internes (DMS, registre interne, etc.) en ajoutant des `adapter_type` supplémentaires dans le moteur. Hors scope v0.1.

## État du build

Voir [`NOTES.md`](NOTES.md) pour les décisions, le statut des sources au sweep d'installation, et la dette assumée.

## Licence

MIT — voir [`LICENSE`](LICENSE).
