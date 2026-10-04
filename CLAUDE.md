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
