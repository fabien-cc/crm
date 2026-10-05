# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## État du repo

Projet au stade zéro : aucun code, aucune commande de build/test pour l'instant. Mettre à jour ce fichier (commandes, architecture) dès que le premier code existe.

**`docs/architecture.md` est la référence des choix d'architecture.** Décidé : TypeScript, Bun (dernière version stable, figée, dépendance à Bun assumée), local-first, SSE pour notifier la sync. Les autres choix sont en cours : l'utilisateur coche ses décisions directement dans ce fichier. Ne pas initialiser de projet ni adopter une bibliothèque dont le choix n'y est pas tranché. Le `.gitignore` actuel est le template Cargo et devra être remplacé.

## Objectif

CRM auto-hébergé, très opérationnel, pour les start-up de l'utilisateur (plusieurs entreprises). Il évolue au fil des besoins réels : privilégier des incréments petits et utilisables plutôt qu'un design exhaustif en amont.

## Exigences UX (non négociables)

Cible : des utilisateurs très techniques, des gros nerds. L'UX est un différenciateur, pas une finition à faire en fin de projet.

- **Mobile friendly** : chaque écran est conçu pour le téléphone et le dektop (usage à une main, entre deux rendez-vous), et enrichi pour le desktop.
- **Extrêmement rapide** : les interactions doivent paraître instantanées. Pas de spinner bloquant pour une action courante ; mises à jour optimistes ; navigation sans rechargement de page.
- **Local-first** (décision du 2026-10-04, qui annule celle du 2026-10-02) : l'UI lit et écrit dans une copie locale, le serveur reste la source de vérité. Toute fonctionnalité doit définir son comportement hors ligne et sa règle de conflit (niveau de hors ligne : voir `docs/architecture.md`).
- **Keyboard-first sur desktop** : toute action fréquente a un raccourci clavier, avec une palette de commandes (Cmd/Ctrl+K) qui donne accès à tout. Les raccourcis sont faciles à découvrir (affichés dans l'UI, aide accessible au clavier). Toute nouvelle fonctionnalité arrive avec ses raccourcis, pas après.

## Priorités produit (dans l'ordre)

L'ordre guide les arbitrages : ne pas construire une priorité basse au détriment d'une plus haute.

1. **Contacts et comptes** — base unique et propre : coordonnées, historique des échanges, source du contact. C'est le socle ; la qualité des données (dédoublonnage, source toujours renseignée) prime.
2. **Pipeline de vente visuel** — kanban par étapes (nouveau lead, qualifié, démo, proposition, signé/perdu). Contrainte forte : **5 à 7 étapes maximum**.
3. **Tâches et relances** — rappels automatiques pour qu'aucun prospect ne refroidisse ; la plupart des deals perdus le sont par oubli de relance.
4. **Intégration email et agenda** — sync Gmail/Outlook + calendrier pour que les échanges se loguent automatiquement dans l'historique du contact. Facteur clé d'adoption.
5. **Reporting simple** — leads par source, taux de conversion entre étapes, durée du cycle de vente, CA prévisionnel pondéré.
6. **Automatisations de base** — séquences d'emails, attribution automatique des leads, notifications. Pas indispensable au départ.
7. **Capture de leads** — formulaires web, import CSV, connecteurs d'acquisition (LinkedIn, publicité, chat).

## Implications de conception à garder en tête

- La **source** d'un contact/lead et l'**historique des étapes** d'un deal (dates d'entrée/sortie) doivent être stockés dès le départ : le reporting (priorité 5) en dépend et ne peut pas être reconstitué a posteriori.
- Les échanges (emails, RDV, notes, appels) convergent vers un même historique rattaché au contact/compte, qu'ils soient saisis à la main ou synchronisés (priorité 4).
- Usage pour plusieurs start-up : la séparation des données entre entreprises reste à décider (`docs/architecture.md`, §1.6).

## Harnais anti-dérive (exigence, pas encore en place)

Ce code est écrit en grande partie par des LLM. Le risque principal est la dérive : patterns incohérents, code mort, erreurs avalées, tests affaiblis pour passer. La qualité ne doit donc pas reposer sur les consignes de ce fichier, qui restent indicatives, mais sur des **vérifications déterministes impossibles à contourner**. Le harnais sera mis en place lors de l'initialisation du projet, avant le premier code applicatif. Les outils précis sont à choisir dans `docs/architecture.md` §6.

- **Une seule commande `verify`** (nom à fixer) qui enchaîne tout. Elle est utilisée à l'identique par Claude, les hooks git et la CI.
- **TypeScript au plus strict** : `strict`, plus `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, `noPropertyAccessFromIndexSignature`, `noImplicitOverride`, `noImplicitReturns`, `noFallthroughCasesInSwitch`, `noUnusedLocals`, `noUnusedParameters`, `allowUnreachableCode: false`, `allowUnusedLabels: false`, `verbatimModuleSyntax`.
- **Lint avec information de types** : interdiction de `any`, des promesses non attendues, des accès `unsafe`, de l'assertion non nulle `!`. Exhaustivité obligatoire des `switch` sur les unions.
- **Règles d'architecture vérifiées par outil** : couches et sens des imports (le client n'importe jamais le serveur, les mutations restent pures), pas de cycles. Détection du code mort, des exports et dépendances inutilisés, et de la duplication.
- **Tests** : la CI échoue si une suite ne contient aucun test. Seuils de couverture. Tests de mutation sur le cœur métier et le moteur de sync. Tests de convergence de la sync.
- **Compteur de dérogations à cliquet** : le nombre de `@ts-expect-error`, de désactivations de lint et de casts `as` est mesuré. La vérification échoue s'il augmente.
- **Hooks git** : un pre-commit rapide (format et lint des fichiers modifiés) et un pre-push qui lance `verify` en entier. Rien n'atteint le remote sans que `verify` passe. La CI rejoue `verify` côté serveur, donc un contournement local ne suffit pas.
- **Hooks Claude Code** :
  - après chaque édition, typecheck et lint du fichier modifié ;
  - un hook Stop qui bloque la fin du tour tant que `verify` échoue ;
  - un PreToolUse qui interdit de modifier les configs du harnais et de lancer des commandes avec `--no-verify`.

## Règles de code

Chaque règle est destinée à être appliquée par le harnais ci-dessus. En attendant, elle s'applique telle quelle.

- **IMPORTANT : ne jamais affaiblir le harnais pour faire passer une vérification.** Cela inclut :
  - modifier `tsconfig`, les configs de lint, d'architecture, de tests ou les hooks ;
  - ajouter `any`, `@ts-ignore`, `@ts-expect-error`, un `as` de contournement, une assertion `!` ou une désactivation de lint ;
  - utiliser `--no-verify`.
  
  Si une dérogation semble nécessaire : s'arrêter, expliquer pourquoi et demander. Une dérogation acceptée porte un commentaire qui en donne la raison.
- **Ne jamais modifier, affaiblir ou supprimer un test existant pour qu'il passe.** Si un test paraît faux, le signaler et demander. Un test vérifie un comportement, pas une valeur codée en dur pour lui.
- **Aucune nouvelle dépendance sans accord explicite.** Vérifier d'abord si le besoin est couvert par une dépendance existante, par Bun ou par la plateforme.
- **Réutiliser avant de créer.** Chercher la fonction, le type ou le pattern existant. Si deux patterns existants se contredisent, le signaler plutôt que les mélanger.
- **Faire le changement le plus petit qui résout la tâche.** Pas d'abstraction spéculative, pas de paramètre « pour plus tard », pas de couche de compatibilité : rien n'est encore en production.
- **Les erreurs restent visibles.** Pas de `catch` vide, pas de `catch` qui journalise puis continue comme si de rien n'était, pas de valeur par défaut silencieuse qui masque une donnée manquante.
- **Valider aux frontières, faire confiance aux types à l'intérieur.** Les entrées réseau, stockage et webhooks sont validées par schéma. Pas de vérifications défensives sur des valeurs déjà typées.
- **Modéliser les états par des unions discriminées.** Pas de booléens combinés. Chaque `switch` sur une union se termine par un cas `never` exhaustif.
- **Commentaires** : expliquer pourquoi, jamais ce que fait le code. Pas de code commenté, pas de commentaire qui raconte la modification.
- **Une tâche n'est terminée que quand `verify` passe.** Montrer la commande et sa sortie, ne jamais affirmer un succès sans preuve. Après deux tentatives infructueuses sur la même erreur, s'arrêter et exposer le problème plutôt que de contourner.
- **Les choix d'architecture de `docs/architecture.md` font foi.** Pour s'en écarter, proposer d'abord une modification du document, jamais directement dans le code.
