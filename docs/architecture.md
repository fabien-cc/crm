# Architecture

Document de décision. Les choix déjà tranchés sont en haut. Pour chaque choix restant, coche une option (`[x]`) et remplis la ligne **Décision**. L'option que je recommande est signalée, sans engagement de ta part.

Ce que je dis des outils tiers reflète l'état connu au 2026-10-04 : à revérifier au moment de trancher, l'écosystème local-first bouge vite.

## Décisions prises

| Domaine | Décision | Date |
|---|---|---|
| Langage | TypeScript strict, front et back | 2026-10-04 |
| Runtime serveur | **Bun, dernière version stable**, figée à une version exacte (1.4.2 est la dernière que j'ai trouvée, à vérifier) | 2026-10-04 |
| Dépendance à Bun | Assumée : on peut utiliser les API spécifiques à Bun, et le retour à Node n'est plus un objectif | 2026-10-04 |
| Modèle d'application | **Local-first** : l'UI lit et écrit dans une copie locale des données, puis la synchronisation se fait en arrière-plan | 2026-10-04 |
| Notifications de sync | **SSE** du serveur vers le client. Le push et le pull des données restent en HTTP | 2026-10-04 |

### Garde-fous liés à Bun

Bun 1.4 (août 2026) est une réécriture complète de Zig vers Rust. Les premiers retours sont bons, mais il y a peu de recul en production (voir les sources plus bas).

- La version de Bun est figée dans `package.json` (`"packageManager": "bun@x.y.z"`), dans l'image Docker et en CI. Les montées de version sont volontaires, après passage de toute la suite de tests.
- L'image Docker utilise une base **Debian (glibc), pas Alpine (musl)** : un crash du ramasse-miettes sur musl n'a été corrigé qu'en 1.4.2.
- Pas de `bun test --parallel` tant que l'issue [#41357](https://github.com/oven-sh/bun/issues/41357) (crash des workers parallèles) n'est pas corrigée.
- Avant la mise en production, un **test d'endurance de 24 h** sous charge réaliste vérifie que la mémoire (RSS) redescend : API, quelques centaines de flux SSE ouverts, sync email.

## Socle local-first

Ce qui suit vaut quel que soit le moteur de sync choisi plus bas.

- L'interface ne lit et n'écrit que dans la copie locale. Elle ne bloque jamais sur le réseau.
- **Le serveur reste la source de vérité.** Les écritures du client sont des *mutations* nommées : elles décrivent une intention, comme `deal.moveStage {dealId, stageId, position}`, et non un état de ligne. Le serveur les réapplique, vérifie les droits et les règles métier, puis les valide ou les rejette.
- **Un seul chemin d'écriture côté serveur.** L'UI, la sync email, les webhooks, l'import CSV et les automatisations passent tous par les mêmes mutations. Tout est donc versionné, synchronisé et soumis aux mêmes règles.
- Quand des données changent, le serveur envoie un **SSE** au client (un « poke », sans données) avec la nouvelle version. Le client récupère alors les changements par HTTP (pull). Il envoie aussi ses mutations en attente par HTTP (push), avec des envois idempotents qu'on peut rejouer.
- Les IDs sont des **UUIDv7 générés par le client** : on peut créer hors ligne sans conflit d'identifiant.
- L'ordre des cartes du kanban utilise des **clés d'indexation fractionnaire** : une insertion ne modifie qu'une ligne.
- La **source** des contacts et les **transitions d'étape** des deals sont enregistrées dès le départ, car le reporting en dépend.

## Ordre conseillé pour trancher

Les choix 1 à 4 dépendent les uns des autres et conditionnent tout le reste. Les autres peuvent attendre et se changent plus facilement.

---

## 1. Fondations local-first

### 1.1 Niveau de fonctionnement hors ligne

- [ ] **A. Hors ligne complet (lecture et écriture)** (recommandé, c'est ta demande initiale)
  - Pour : on peut tout faire dans le train ou en rendez-vous sans réseau, puis la sync se fait au retour.
  - Contre : il faut une file de mutations persistante, une résolution des conflits et la gestion des vieilles versions de l'app.
- [ ] **B. Lecture hors ligne, écriture en ligne seulement**
  - Pour : pas de conflits d'écriture hors ligne, beaucoup plus simple.
  - Contre : impossible de noter un compte rendu de RDV sans réseau, ce qui frustre au pire moment.
- [ ] **C. Instantané en ligne, sans hors ligne garanti**
  - Pour : on garde la vitesse du local-first sans les cas limites du hors ligne long.
  - Contre : si le réseau coupe, l'app dépend du cache ; le comportement doit être explicite.

**Décision :**

### 1.2 Moteur de sync

*Dépend de 1.1. Conditionne 1.3 et 1.4.*

- [ ] **A. Protocole maison (modèle Replicache)** (recommandé)
  - Pour : maîtrise totale, règles de conflit métier, aucune dépendance au cœur du produit. La décision SSE s'y intègre directement. Le volume par start-up est petit, donc on réplique tout le workspace et on évite le problème le plus dur (la réplication partielle). Chaque mutation est écrite une fois et tourne côté client (optimiste) et côté serveur (fait foi).
  - Contre : c'est le composant le plus risqué à écrire. Il faut une simulation de convergence avec plusieurs clients, des coupures réseau et des réordonnancements, **avant** toute fonctionnalité.
- [ ] **B. PowerSync** (Postgres répliqué vers un SQLite local)
  - Pour : mûr, écritures hors ligne prêtes à l'emploi, édition auto-hébergeable.
  - Contre : un service de plus à opérer, une licence source-available (pas open source), son propre transport (pas notre SSE), des règles de sync (« sync rules ») à apprendre.
- [ ] **C. ElectricSQL + TanStack DB**
  - Pour : très bon pour la lecture (des « shapes » Postgres en HTTP, avec un mode live en SSE), facile à mettre en cache, TanStack DB gère l'optimiste.
  - Contre : les écritures hors ligne et la file de mutations restent à notre charge. Un service Electric de plus à opérer.
- [ ] **D. Zero (Rocicorp)**
  - Pour : excellente expérience de développement, requêtes réactives.
  - Contre : pensé pour « paraître instantané », pas pour écrire hors ligne. Transport WebSocket. Incompatible avec 1.1 A.
- [ ] **E. CRDT (Automerge, Yjs, Jazz)**
  - Pour : convergence garantie mathématiquement, sans serveur arbitre.
  - Contre : les règles métier (unicité, droits, maximum 7 étapes) s'expriment mal sans autorité centrale. Les données grossissent et le reporting SQL devient difficile.

**Décision :**

### 1.3 Stratégie de conflit

*Dépend de 1.2.*

- [ ] **A. Serveur arbitre + dernière écriture gagnante par champ** (recommandé)
  - Pour : simple à comprendre, les règles métier sont appliquées au même endroit, les rejets sont visibles dans une boîte « Conflits ».
  - Contre : quand deux personnes modifient le même champ, une modification est perdue. C'est rare dans un CRM, et c'est visible.
- [ ] **B. Dernière écriture gagnante par ligne**
  - Pour : encore plus simple.
  - Contre : deux personnes qui modifient des champs différents du même contact s'écrasent l'une l'autre.
- [ ] **C. CRDT partout**
  - Pour : aucune modification perdue.
  - Contre : coût disproportionné. À réserver aux notes longues, le jour où l'édition à plusieurs devient un vrai besoin.

Règles proposées avec l'option A :

| Cas | Règle |
|---|---|
| Même champ modifié sur 2 appareils | Le dernier écrit gagne, dans l'ordre où le serveur applique les mutations ; les patchs sont partiels |
| Deal déplacé sur 2 appareils | Le dernier déplacement gagne ; les deux transitions sont journalisées |
| Doublon créé hors ligne (même email) | Le serveur fusionne dans le contact existant et enregistre une redirection d'ID |
| Modification d'une entité supprimée | La suppression gagne ; la mutation est rejetée et affichée dans « Conflits » |
| Mutation invalide (droits, règle métier) | Rejet définitif, jamais retenté en boucle ; retour arrière dans l'UI |

**Décision :**

### 1.4 Stockage local

*Dépend de 1.2. Avec PowerSync, c'est SQLite d'office.*

- [ ] **A. Store en mémoire + IndexedDB** (recommandé)
  - Pour : lectures synchrones, donc rendu immédiat. Simple. C'est le modèle de Linear.
  - Contre : tout le jeu de données répliqué est en RAM, ce qui peut poser problème sur iPhone en gros volume (d'où 1.5). Pas de SQL local.
- [ ] **B. SQLite-WASM sur OPFS (wa-sqlite)**
  - Pour : gros volumes sans pression mémoire, recherche plein texte FTS5, reporting en SQL local.
  - Contre : requêtes asynchrones depuis un worker, réactivité à construire soi-même, comportement d'OPFS à valider sur Safari.
- [ ] **C. PGlite (Postgres compilé en WASM)**
  - Pour : le même dialecte SQL que le serveur.
  - Contre : plusieurs Mo de WASM, démarrage plus lent.

**Décision :**

### 1.5 Périmètre de réplication

- [ ] **A. Tout le workspace, sauf les corps d'emails et les activités de plus de 12 mois** (recommandé)
  - Pour : tout ce qui sert au quotidien est disponible hors ligne. Les éléments lourds sont chargés à la demande, puis mis en cache.
  - Contre : l'historique ancien n'est consultable qu'en ligne.
- [ ] **B. Tout le workspace**
  - Pour : le plus simple, tout est hors ligne.
  - Contre : la mémoire et le stockage grossissent sans limite avec la sync email.
- [ ] **C. À la demande (ce qui a été consulté)**
  - Pour : très léger.
  - Contre : le hors ligne devient imprévisible ; il faut gérer la réplication partielle.

**Décision :**

### 1.6 Séparation entre start-up

- [ ] **A. Base partagée, `workspace_id` partout + RLS Postgres** (recommandé)
  - Pour : un seul déploiement, bascule entre start-up depuis la palette, un utilisateur membre de plusieurs. Une requête qui oublie le filtre ne fait pas fuiter de données. Une base locale par workspace côté client.
  - Contre : chaque requête passe dans une transaction avec `SET LOCAL`, et il faut mettre `workspace_id` en tête de chaque index.
- [ ] **B. Un schéma ou une base par start-up**
  - Pour : isolation forte, export et suppression simples.
  - Contre : migrations à jouer N fois, pool de connexions par start-up.
- [ ] **C. Une instance par start-up**
  - Pour : isolation totale, aucun code multi-tenant.
  - Contre : N déploiements, N connexions, aucune vue transverse.

**Décision :**

### 1.7 Permissions

- [ ] **A. Tous les membres d'un workspace voient tout** (recommandé en v1)
  - Pour : la sync réplique le workspace entier, sans filtrage par utilisateur.
  - Contre : impossible de cacher un deal sensible à un membre.
- [ ] **B. Permissions par enregistrement dès le départ**
  - Pour : contrôle fin.
  - Contre : transforme la sync en problème de réplication partielle, nettement plus complexe.

**Décision :**

---

## 2. Serveur

### 2.1 Framework HTTP

- [ ] **A. Elysia** (pressenti)
  - Pour : conçu pour Bun et le plus rapide dessus. Client typé de bout en bout avec Eden Treaty. Validation intégrée (TypeBox), OpenAPI généré, SSE par générateurs.
  - Contre : écosystème plus petit que Hono. Évolue vite (version 1.4.x en 2026). La porte vers Node n'est pas totalement fermée : un adaptateur `@elysiajs/node` existe, encore en bêta.
- [ ] **B. Hono**
  - Pour : multi-runtime, très répandu, client RPC typé, helper SSE.
  - Contre : un peu moins rapide sur Bun ; ne tire pas parti des spécificités de Bun.

**Décision :**

### 2.2 Validation et schémas partagés

*Les arguments des mutations sont validés côté client et côté serveur.*

- [ ] **A. TypeBox** (recommandé si Elysia)
  - Pour : natif dans Elysia, très rapide, produit directement du JSON Schema, donc OpenAPI.
  - Contre : API moins agréable que Zod, écosystème plus petit.
- [ ] **B. Zod**
  - Pour : le standard de fait, très lisible, énormément d'intégrations. Elysia l'accepte via Standard Schema.
  - Contre : plus lourd dans le bundle client, un peu plus lent.
- [ ] **C. Valibot**
  - Pour : modulaire, le plus léger côté client.
  - Contre : moins répandu.

**Décision :**

### 2.3 Base de données serveur

- [ ] **A. Postgres** (recommandé)
  - Pour : contraintes, RLS, LISTEN/NOTIFY (qui déclenche les SSE), file de jobs dans la base, SQL analytique pour le reporting.
  - Contre : un service à opérer et à sauvegarder.
- [ ] **B. SQLite (WAL + Litestream)**
  - Pour : un seul fichier, Bun a SQLite intégré (`bun:sqlite`), sauvegarde continue triviale.
  - Contre : un seul process écrivain, pas de RLS ni de LISTEN/NOTIFY, jobs à construire soi-même.

**Décision :**

### 2.4 Accès aux données

- [ ] **A. Drizzle (driver `drizzle-orm/bun-sql`)** (recommandé)
  - Pour : schéma en TS, migrations générées, proche du SQL, gère les politiques RLS. Utilise le client Postgres natif de Bun.
  - Contre : l'outil de migration a quelques aspérités.
- [ ] **B. `Bun.sql` brut + migrations SQL écrites à la main**
  - Pour : zéro dépendance, SQL explicite, le plus rapide.
  - Contre : pas de types générés depuis le schéma ; il faut les maintenir à la main.
- [ ] **C. Kysely**
  - Pour : le meilleur query builder typé.
  - Contre : pas de schéma déclaratif ni de migrations générées.

**Décision :**

### 2.5 Jobs et planification

- [ ] **A. pg-boss** (recommandé)
  - Pour : file dans Postgres, cron, retries, throttling, jobs uniques. Pas de Redis.
  - Contre : la compatibilité avec Bun 1.4 reste à vérifier.
- [ ] **B. graphile-worker**
  - Pour : très rapide, réveil par LISTEN/NOTIFY.
  - Contre : moins d'outils de contrôle de flux.
- [ ] **C. `Bun.cron` + une table de jobs maison**
  - Pour : natif depuis Bun 1.4, zéro dépendance.
  - Contre : retries, verrouillage et reprise après crash à écrire soi-même.

**Décision :**

### 2.6 Auth

- [ ] **A. Better Auth : passkeys + magic link** (recommandé)
  - Pour : bibliothèque TS auto-hébergée, organisations (multi-workspace) incluses, intégration Elysia.
  - Contre : projet jeune, API encore en mouvement.
- [ ] **B. Fournisseur d'identité auto-hébergé (Zitadel, Authentik, Keycloak)**
  - Pour : SSO, gestion complète des utilisateurs.
  - Contre : un service lourd, excessif pour quelques start-up.
- [ ] **C. Auth maison**
  - Pour : zéro dépendance.
  - Contre : passkeys et sécurité à porter seul.

Contrainte du local-first : la session doit rester valable hors ligne. Si elle a expiré au retour du réseau, les mutations en attente sont conservées jusqu'à la reconnexion.

**Décision :**

---

## 3. Client

### 3.1 Framework UI

- [ ] **A. React + React Compiler** (recommandé)
  - Pour : le plus grand écosystème (cmdk, Radix, TanStack). C'est ce que les LLM écrivent le mieux.
  - Contre : la performance demande de la discipline (sélecteurs fins, virtualisation).
- [ ] **B. SolidJS**
  - Pour : réactivité fine par défaut, bundle plus petit.
  - Contre : écosystème plus maigre.
- [ ] **C. Svelte 5**
  - Pour : lisible, rapide.
  - Contre : moins de briques orientées clavier.

**Décision :**

### 3.2 Routing

- [ ] **A. TanStack Router** (recommandé)
  - Pour : entièrement typé, paramètres de recherche typés, préchargement.
  - Contre : API plus dense.
- [ ] **B. React Router (mode SPA)**
  - Pour : le plus connu.
  - Contre : typage moins poussé.

**Décision :**

### 3.3 Composants

- [ ] **A. shadcn/ui (Radix) + Tailwind** (recommandé)
  - Pour : le code des composants vit dans le repo et se modifie librement. Accessibilité de Radix, cmdk intégré.
  - Contre : rendu générique tant qu'on ne le retravaille pas.
- [ ] **B. React Aria Components**
  - Pour : la meilleure gestion du clavier, du tactile et de l'accessibilité.
  - Contre : API verbeuse, tout le style est à faire.

**Décision :**

### 3.4 Recherche locale

*Si 1.4 = SQLite, FTS5 devient l'option naturelle.*

- [ ] **A. MiniSearch** (recommandé avec 1.4 A)
  - Pour : petit, mises à jour incrémentales, recherche par préfixe et approximative.
  - Contre : moins rapide que FlexSearch sur de très gros index.
- [ ] **B. FlexSearch**
  - Pour : le plus rapide.
  - Contre : API plus rugueuse.
- [ ] **C. FTS5 (SQLite)**
  - Pour : intégré à la base locale.
  - Contre : seulement avec 1.4 B.

**Décision :**

### 3.5 Forme de l'app

- [ ] **A. PWA** (recommandé)
  - Pour : une seule base de code, pas de store, déploiement instantané. Le service worker met le shell en cache pour démarrer hors ligne.
  - Contre : sur iOS, le Web Push et la durabilité du stockage exigent l'installation sur l'écran d'accueil.
- [ ] **B. Capacitor (PWA emballée en app native)**
  - Pour : le même code, plus un stockage durable et des push natifs. C'est la porte de sortie si les tests iOS déçoivent.
  - Contre : publication en store et cycle de review.
- [ ] **C. Expo / React Native**
  - Pour : sensation native.
  - Contre : une seconde interface à maintenir, desktop clavier mal servi.

**Décision :**

---

## 4. Intégrations

### 4.1 Accès aux emails et à l'agenda

- [ ] **A. APIs natives Gmail et Microsoft Graph** (recommandé)
  - Pour : gratuit, sync incrémentale fiable (`historyId`, requêtes delta), aucun tiers ne voit les données.
  - Contre : deux intégrations à maintenir. Les scopes Gmail en lecture sont « restreints » : il faut déclarer l'app OAuth comme « Internal » dans le Google Workspace de chaque start-up pour éviter l'audit de sécurité.
- [ ] **B. Agrégateur (Nylas, Unipile)**
  - Pour : une seule API. Unipile couvre aussi la messagerie LinkedIn.
  - Contre : SaaS payant à l'utilisateur, données chez un tiers.
- [ ] **C. IMAP / JMAP**
  - Pour : universel.
  - Contre : pas de sync incrémentale propre chez Gmail, pas d'agenda.

**Décision :**

---

## 5. Outillage et opérations

### 5.1 Monorepo

- [ ] **A. Bun workspaces seuls** (recommandé)
  - Pour : rien à ajouter.
  - Contre : pas de cache de tâches.
- [ ] **B. Bun workspaces + Turborepo**
  - Pour : cache local et distant des builds et tests.
  - Contre : une configuration de plus, inutile au début.

**Décision :**

### 5.2 Lint et formatage

- [ ] **A. Biome** (recommandé)
  - Pour : un seul outil, très rapide.
  - Contre : moins de règles que l'écosystème ESLint.
- [ ] **B. ESLint + Prettier**
  - Pour : le plus complet.
  - Contre : lent, deux outils à configurer.
- [ ] **C. Biome + oxlint**
  - Pour : rapidité et règles supplémentaires.
  - Contre : deux outils.

**Décision :**

### 5.3 Tests

- [ ] **A. `bun test` + Playwright** (recommandé)
  - Pour : intégré à Bun, rapide. Playwright pour les parcours, les raccourcis et le hors ligne (`context.setOffline(true)`). fast-check pour simuler la convergence de la sync.
  - Contre : régressions récentes du test runner en 1.4 (voir les garde-fous).
- [ ] **B. Vitest + Playwright**
  - Pour : très mûr, excellent mode watch.
  - Contre : un outil de plus alors que Bun en fournit un.

**Décision :**

### 5.4 Déploiement

- [ ] **A. Docker Compose + Caddy sur un VPS** (recommandé)
  - Pour : lisible, HTTPS automatique (nécessaire pour le service worker et les webhooks). Caddy transmet les SSE sans les retenir en tampon.
  - Contre : déploiements et rollbacks à scripter soi-même.
- [ ] **B. Kamal**
  - Pour : déploiement sans coupure et rollback en une commande.
  - Contre : un outil Ruby de plus.
- [ ] **C. Coolify ou Dokploy**
  - Pour : PaaS auto-hébergé avec interface, déploiement sur `git push`.
  - Contre : une couche de plus à maintenir et sécuriser.

**Décision :**

### 5.5 Sauvegardes Postgres

- [ ] **A. WAL-G vers un stockage compatible S3** (recommandé)
  - Pour : simple, sauvegarde continue, restauration à un instant précis.
  - Contre : moins d'outils de vérification que pgBackRest.
- [ ] **B. pgBackRest**
  - Pour : rétention, vérification, sauvegardes parallèles.
  - Contre : plus de configuration.

**Décision :**

---

## À valider d'abord

Des vérifications courtes, à faire avant le développement des fonctionnalités. Chacune peut remettre en cause un choix ci-dessus.

1. **Prototype du moteur de sync** sur deux tables (contact, deal), avec une simulation de convergence et le canal SSE.
2. **Mémoire et stockage sur iPhone** avec un jeu de données réaliste : 10 000 contacts, 2 000 deals, 100 000 activités.
3. **Durabilité du stockage sur Safari** : la PWA installée conserve-t-elle bien ses données dans la durée ?
4. **Compatibilité Bun 1.4** avec la pile choisie (pg-boss, Better Auth, Drizzle) et test d'endurance de 24 h avec SSE.
5. **Web Push sur iOS** pour les relances.

## Sources sur Bun en production (relevées le 2026-10-03)

- [Bun 1.4 (blog officiel)](https://bun.com/blog/bun-v1.4)
- [Bun v1.4.2](https://bun.com/blog/bun-v1.4.2)
- [Bun 1.4 at Scrydon](https://scrydon.com/insights/2026/08/21/bun-1-4-at-scrydon/) : staging, 9 services
- [Prisma Compute sur le portage Rust](https://www.prisma.io/blog/bun-rust-rewrite-prisma-compute)
- [Bun 1.4 : changements cassants (ecorpit)](https://ecorpit.com/bun-1-4-rust-rewrite-breaking-changes-production-2026/)
- [Elysia : adaptateur Node](https://npmjs.com/package/%40elysiajs%2Fnode)
