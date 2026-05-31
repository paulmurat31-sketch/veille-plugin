---
name: profile
description: Entretien structuré pour capturer le profil de l'utilisateur (rôle, secteur, juridictions, critères d'urgence, langues, format de sortie). Écrit le résultat dans ~/.veille/profile.md. À lancer en premier après l'installation du plugin, ou pour mettre à jour le profil.
when_to_use: Quand l'utilisateur tape /veille:profile, ou demande à configurer/réviser sa veille, ou quand /veille:check détecte l'absence d'un profil utilisable.
allowed-tools: AskUserQuestion, Read, Write
---

# `/veille:profile` — entretien d'amorçage

Votre rôle : conduire un entretien guidé, sobre, qui produit un **profil utilisateur** complet et exploitable par `/veille:check`. Pas de notes inutiles, pas de questions ouvertes vagues. À la fin : un fichier `~/.veille/profile.md` propre, et une explication claire de la suite.

## Avant de commencer

1. Vérifier si `~/.veille/profile.md` existe déjà.
   - Si oui : afficher un résumé en deux lignes et demander : *« Profil existant détecté. Le **mettre à jour** (on reprend là où on est) ou le **remplacer** (entretien complet depuis zéro) ? »*
   - Si non : annoncer *« Aucun profil. On en construit un — comptez 2 à 3 minutes. »*
2. Préciser que les réponses restent **100 % locales** (rien envoyé ailleurs qu'à Anthropic pour la conversation elle-même, comme tout usage Claude Code).
3. Ne pas demander tout d'un coup. Structurer en **trois segments** courts (1, 2, 3 ci-dessous), avec une transition entre chaque.

## Segment 1 — Identité et rôle (conversationnel + 1 AskUserQuestion)

Posez les questions ouvertes une par une (ou groupées par deux si elles sont brèves) :

- *« Quel est votre rôle ? Un titre suffit, par exemple : avocat senior en droit du travail, conseiller juridique à l'interne, parajuriste en propriété intellectuelle. »*
- *« Quels sont vos domaines de pratique principaux ? (deux à quatre suffisent) »*
- *« Quel secteur d'activité ou industrie souhaitez-vous surveiller ? Si vous êtes en cabinet, mettez le secteur dominant de votre portefeuille. Si vous êtes à l'interne, le secteur de votre organisation. »*
- *« En une phrase ou deux, qu'est-ce que votre organisation fait concrètement ? (produits, services, marchés). C'est ce qui me permet de juger la pertinence d'un règlement pour vous. »*

Puis une question discrète avec `AskUserQuestion` :

```
Question : « Quel est votre niveau de séniorité ? »
Header  : "Séniorité"
Options :
  - "Junior (0–4 ans)"
  - "Intermédiaire (5–9 ans)"
  - "Senior (10+ ans) (Recommandé si vous portez la décision)"
  - "Partner / GC / chef de pratique"
multiSelect: false
```

Si la séniorité « ne s'applique pas » au sens classique, l'utilisateur cliquera *Other*.

## Segment 2 — Couverture (conversationnel + 1 AskUserQuestion)

Présentez les codes de juridictions disponibles, puis demandez conversationnel :

> Les juridictions que je peux surveiller :
>
> - **Canada fédéral** — `CA_FED`
> - **Provinces** : `CA_QC` `CA_ON` `CA_BC` `CA_AB` `CA_MB` `CA_SK` `CA_NS` `CA_NB` `CA_NL` `CA_PE`
> - **Territoires** : `CA_YT` `CA_NT` `CA_NU`
> - **US fédéral** — `US_FED`
>
> Lesquelles voulez-vous suivre ? Listez les codes séparés par des espaces ou des virgules. (Si c'est pour le Canada en entier : tapez « tout-CA ». Pour CA + US fédéral : « tout-CA + US_FED ».)

Validez ce que l'utilisateur écrit. Refusez les codes inconnus avec une suggestion. Au moins une juridiction est obligatoire.

Puis une question discrète :

```
Question : « Dans quelle langue voulez-vous votre digest ? »
Header  : "Langue digest"
Options :
  - "Français — défaut canadien (Recommandé)"
  - "Anglais"
  - "FR avec sources EN intactes"
  - "EN avec sources FR intactes"
multiSelect: false
```

Note : peu importe la langue du digest, les **titres officiels** dans la rubrique « Source » ne sont jamais traduits — on garde la langue d'origine.

## Segment 3 — Critères d'urgence et format (2 AskUserQuestion + validation conversationnelle des mots-clés)

```
Question : « À quel horizon une entrée en vigueur (ou fin de consultation) devient-elle urgente pour vous ? »
Header  : "Horizon urgent"
Options :
  - "30 jours — réactif"
  - "60 jours"
  - "90 jours (Recommandé — plus serein)"
  - "180 jours — défensif"
multiSelect: false
```

```
Question : « Quels autres événements doivent automatiquement déclencher une alerte urgente ? »
Header  : "Déclencheurs"
multiSelect: true
Options :
  - "Nouveau projet de loi dans votre périmètre"
  - "Échéance de consultation publique imminente"
  - "Passage à la sanction royale ou à l'entrée en vigueur"
  - "Aucun — j'utilise seulement le critère d'horizon"
```

Puis, conversationnel :

> *« Pour conclure : à partir de votre rôle, secteur et activités, je vous propose une liste de mots-clés FR et EN qui serviront à filtrer le bruit avant la classification. Validez ou ajustez. »*

Générer **5 à 10 mots-clés FR** et **5 à 10 mots-clés EN** en s'appuyant sur les `practice_areas`, `industry` et `organization_activities` recueillis. Les présenter en deux listes claires, puis demander :

> *« Quelque chose à ajouter, retirer, ou reformuler ? »*

Itérer jusqu'à validation.

Finir avec :

```
Question : « Quel format de digest préférez-vous ? »
Header  : "Format digest"
Options :
  - "Court — 1 à 2 phrases par item (Recommandé)"
  - "Moyen — un paragraphe par item"
  - "Complet — avec citations de la source"
multiSelect: false
```

## Écriture du profil

Construire `~/.veille/profile.md` selon le squelette ci-dessous. La section humaine doit être agréable à relire ; le bloc YAML en fin de fichier est ce que les autres skills parsent.

```markdown
# Profil de veille

> Profil construit par /veille:profile le {ISO_DATE}. Vous pouvez l'éditer à la main — gardez le bloc YAML en fin de fichier intact.

## En bref

- **Rôle** : {role}
- **Séniorité** : {seniority}
- **Domaines de pratique** : {practice_areas, joined}
- **Secteur / industrie** : {industry}
- **Juridictions surveillées** : {jurisdictions, joined}
- **Langue du digest** : {primary_lang}

## Activités de l'organisation

{organization_activities}

## Critères d'urgence

- Horizon : un item devient urgent si sa date d'entrée en vigueur ou de fin de consultation tombe dans **{urgent_horizon_days} jours** ou moins.
- Déclencheurs additionnels : {also_urgent_on, joined ou « aucun »}.

## Mots-clés utilisés au pré-filtre

### Français
{liste à puces}

### Anglais
{liste à puces}

## Format de sortie préféré

{output_preferences description en clair}

---

```yaml
{bloc YAML structuré selon le schéma de regulatory-watch §2}
```
```

**Important** : la dernière section du fichier doit être le bloc ` ```yaml ` (le moteur parse le **dernier** bloc YAML rencontré).

## Écriture du fichier

1. Créer le dossier `~/.veille/` s'il n'existe pas.
2. Écrire le fichier `~/.veille/profile.md` (créer ou remplacer).
3. Annoncer le chemin exact (« Profil enregistré dans `~/.veille/profile.md` »).

## Confirmation et explication de la suite

Conclure par un récap en 4 lignes :

```
✓ Profil enregistré dans ~/.veille/profile.md
  • Juridictions : {liste}
  • Langue digest : {primary}
  • Sources actives pour vous : {N sources core ∩ juridictions}

Prochaine étape : /veille:check
  • Lance un sweep manuel des sources publiques pertinentes
  • Vous montre uniquement les nouveautés depuis le dernier passage
  • Trié par priorité (P0 cette semaine / P1 ce mois-ci / P2 surveillance)
```

Si l'utilisateur a coché des juridictions sans source core active (ex. `CA_NS` qui est désactivé), le signaler :

> *« Note : pour {juridiction}, aucune source n'est active par défaut au moment de l'écriture du plugin (URL en panne au sweep d'installation). Voir `NOTES.md` du plugin. Vous pouvez activer manuellement les sources `best_effort` correspondantes via `~/.veille/sources.json`. »*

## Cas limites

- **L'utilisateur veut sauter une question** : autoriser, mettre une valeur par défaut raisonnable (séniorité=`mid`, horizon=`90`), et le mentionner dans le récap final.
- **L'utilisateur n'a aucune juridiction connue** : proposer `CA_FED` par défaut et expliquer qu'il pourra élargir.
- **L'utilisateur dit « plus tard »** : ne PAS écrire de profil vide. Dire que sans profil, `/veille:check` ne peut pas fonctionner, et inviter à revenir.
