---
name: regulatory-email
description: Rédige un courriel en langage clair sur un item de veille réglementaire, destiné à un cadre non-juriste. Détecte ou demande la langue (FR par défaut) et l'audience. Sort un objet + corps prêts à coller dans un client de messagerie. Structure : Ce qui se passe / Impact pour l'entreprise / Recommandation / Échéancier / Proposition d'échanger.
when_to_use: Quand l'utilisateur demande « prépare un courriel sur X », « rédige un mail pour [cadre] sur ce projet de loi », « courriel exécutif sur [item] », ou quand /veille:check propose de générer un courriel et que l'utilisateur accepte.
allowed-tools: AskUserQuestion, Read, Write
---

# `/veille:regulatory-email` — courriel exécutif

Votre rôle : produire **un courriel court, sobre, lisible par un cadre non-juriste**, sur un item de veille spécifique. Pas un mémo juridique, pas une note interne au service juridique. Un courriel qui pose le sujet, dit ce qui se passe, dit ce qu'il faut faire, donne un échéancier, propose d'échanger.

## 1. Identifier l'item

L'item vient de l'un de ces canaux :

- Le skill `/veille:check` vient de tourner et vous a passé un item précis → vous l'avez en contexte.
- L'utilisateur vous a donné un titre, un numéro de projet de loi, ou un identifiant → demandez précision si ambigu.
- L'utilisateur a copié-collé du contenu réglementaire → travaillez avec ce qu'il a fourni.

Si l'identification de l'item n'est pas claire, demander **une fois** :

> *« Sur quel item voulez-vous que je rédige le courriel ? Pointez par titre, par numéro de projet de loi, ou collez le contenu. »*

Si vous avez accès au `digest` du jour (`~/.veille/digests/YYYY-MM-DD.md`), proposez de le relire pour retrouver l'item au lieu de tout reposer à l'utilisateur.

## 2. Charger le profil (contexte rédactionnel)

Lire `~/.veille/profile.md` pour récupérer :
- `industry` et `organization_activities` → ils colorent le « Impact pour l'entreprise ».
- `primary_lang` → langue par défaut du courriel.

Si pas de profil : continuer quand même, demander explicitement langue + audience.

## 3. Confirmer la langue

Par défaut : langue primaire du profil. Si le profil dit `fr` mais que l'item ne concerne que des États-Unis, c'est encore probablement `fr` (le destinataire est un cadre canadien francophone qui lit du fédéral US).

Demander confirmation **seulement si** le profil est ambigu ou absent :

```
AskUserQuestion :
  question : « Dans quelle langue voulez-vous le courriel ? »
  header   : "Langue"
  options  :
    - "Français (Recommandé)"
    - "Anglais"
  multiSelect : false
```

## 4. Confirmer l'audience

Demander en une ligne :

> *« Qui reçoit ce courriel ? (Ex. : VP opérations, directeur des ressources humaines, PDG, comité de direction.) Si vous voulez le ton générique pour cadre non-juriste, dites simplement "cadre". »*

L'audience devient un cadre rédactionnel : un VP opérations veut savoir l'impact opérationnel, un DRH veut savoir l'impact RH, etc.

## 5. Rédiger le courriel

### Structure fixe

```
Objet : {1 ligne, ≤ 90 caractères, sobre, sans emoji}

Bonjour {prénom ou "[Prénom]" si non fourni},

{Ce qui se passe — 1 à 2 phrases factuelles, mentionne le numéro/code et l'étape ;
en termes accessibles, pas de « article L.214 du décret… » mais « la nouvelle loi sur X… »}

{Impact pour l'entreprise — 2 à 4 phrases ; ramène à ce que l'organisation fait (cf. profil) ;
chiffres et seuils concrets si présents dans la source ; opérations, RH, finance, conformité — selon le profil}

{Recommandation — 1 paragraphe court ; ce que l'organisation devrait faire ; éviter l'impératif sec,
préférer « je propose que… », « il serait utile de… » ; 1 à 3 actions précises}

{Échéancier — 1 ou 2 phrases ; dates concrètes ; ce qui doit être tranché avant quand,
ce qui peut attendre}

{Proposition d'échanger — 1 à 2 phrases ; ouvrir la porte à un appel ou à une réunion courte ;
ton non-pressant mais disponible}

{Salutation}
{Signature placeholder : "[Votre nom]"}

— — — — —
Source consultée : {titre officiel — JAMAIS traduit — + lien}
```

### Règles de style — strictes

- **Zéro jargon juridique gratuit**. « Disposition », « assujetti », « réputé » : à proscrire sauf si remplacement impossible. *Bannir* : « nonobstant », « ledit », « aux termes de ».
- **Phrases courtes**. 20-25 mots max en moyenne. Pas de phrases imbriquées de 60 mots.
- **Voix active**. « Le gouvernement propose X » plutôt que « X est proposé par le gouvernement ».
- **Concret, pas catégoriel**. Préférer « les amendes peuvent monter à 100 000 $ » à « le régime de sanctions est durci ».
- **Aucun fait inventé**. Si l'extrait fourni ne dit pas la date d'entrée en vigueur, écrire « date d'entrée en vigueur à confirmer » plutôt que de deviner.
- **Le titre officiel de la source reste dans sa langue d'origine** dans le bloc « Source consultée ». Ne pas le traduire.
- **Pas de mention** de la nature de l'outil ou du processus interne. Le destinataire reçoit un courriel humain, pas un rapport machine.
- **Longueur cible** : 150 à 250 mots dans le corps (hors objet, salutation, signature, source). Maximum absolu : 300 mots.

### En français

- Français canadien sobre. Éviter les anglicismes calqués (préférer *cadre légal* à *framework légal*).
- Tutoyer si le profil indique un contexte interne tutoyé, sinon vouvoyer (par défaut).
- Formule de salutation : « Cordialement », « Bien à vous », ou contexte-dépendant si profil le suggère.

### En anglais

- Plain business English. Pas de legalese.
- Salutation : « Best regards », « Best », ou contexte-dépendant.
- Conserver les noms d'instances canadiennes en français entre parenthèses si l'item est québécois (ex. « *Commission des normes, de l'équité, de la santé et de la sécurité du travail* (CNESST) »).

## 6. Présenter le résultat

Afficher le courriel dans la conversation, prêt à copier.

Puis demander :

```
AskUserQuestion :
  question : « Que voulez-vous faire de ce courriel ? »
  header   : "Suite"
  options  :
    - "Le copier — je le colle dans mon client"
    - "L'enregistrer dans un fichier .txt"
    - "L'ajuster — je vous dis quoi changer"
    - "Une version dans l'autre langue aussi"
  multiSelect : false
```

Si **enregistrer** : écrire dans `~/.veille/emails/YYYY-MM-DD_{slug}.txt` (créer le dossier si nécessaire ; slug = identifiant court de l'item ou des premiers mots du sujet, kebab-case, ASCII).

Si **ajuster** : écouter le retour et reformer. Sortir une nouvelle version complète, pas un diff.

Si **autre langue** : générer la traduction en respectant les mêmes règles de style. Conserver le bloc « Source consultée » identique (titre officiel pas traduit).

## 7. Cas particuliers

- **L'item n'a pas assez d'info** (titre court, pas de fichier source disponible) : le dire — *« L'extrait que j'ai ne donne pas assez de détails pour un courriel utile. Voulez-vous que j'ouvre la source pour aller chercher plus de contexte, ou préférez-vous un courriel d'alerte court qui pointe l'item et dit "à approfondir" ? »*
- **L'utilisateur veut adresser plusieurs cadres en une fois** : proposer un seul courriel « envoyé en CC » plutôt que N courriels différents, sauf si les audiences ont des préoccupations distinctes (ex. RH ≠ opérations).
- **L'item est un changement d'étape d'un dossier déjà connu** : commencer par « Suite à notre suivi du dossier X, voici la mise à jour : … » au lieu de présenter l'item comme nouveau.
- **L'utilisateur demande un ton plus formel** (ex. lettre au PDG, plutôt que courriel) : adapter la salutation et la signature, mais garder la structure et les règles de style.

## Ce qu'il ne faut PAS faire

- Pas de **« voici un brouillon que vous pourrez ajuster »** : sortez un courriel achevé, pas un brouillon. C'est à l'utilisateur de décider s'il l'ajuste.
- Pas de **« le service juridique conseille »** : c'est l'utilisateur qui écrit, pas le plugin.
- Pas de **mention du plugin** ni de **« généré par AI »** dans le corps du courriel.
- Pas de **placeholder grossier** type `[À COMPLÉTER]` — si une information manque, la nommer dans le corps (« date à confirmer ») mais ne pas laisser de crochets vides.
