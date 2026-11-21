<p align="center"><img src="docs/img/logo.png" width="240" alt="Hallego"></p>

# Hallego, vitrine d'architecture

Hallego est une marketplace de seconde main hyperlocale pour les familles avec enfants, disponible sur Google Play en Turquie. J'ai conçu et réalisé seul tout le produit : API, application mobile, site web, panneau d'administration, infrastructure et modération par IA.

Le code source est privé car c'est un produit en production. Ce dépôt montre comment il est construit, avec les chiffres et une sélection de décisions d'architecture (ADR).

English version: [README.md](./README.md)

## Liens

- Application : [Hallego sur Google Play](https://play.google.com/store/apps/details?id=com.hallego.app)
- Site web : [hallego.com](https://hallego.com)
- Package open source extrait du projet : [@mdsiha369/adonis-mercure](https://www.npmjs.com/package/@mdsiha369/adonis-mercure) ([code source](https://github.com/mdsiha/adonis-mercure))
- Démo en ligne de mon projet précédent Kaybaz (sans compte) : [kaybaz.md-x.dev](https://kaybaz.md-x.dev)

## Vue d'ensemble

Voir le schéma dans [README.md](./README.md#system-overview) (en anglais, les noms sont les mêmes).

## En chiffres

Mesurés sur le code le 2 novembre 2026.

| Élément                  | Chiffre                                                      |
| ------------------------ | ------------------------------------------------------------ |
| API                      | 30 modules, 199 endpoints, 125 migrations de base de données |
| Application mobile       | 60 écrans, 45 BLoC (Flutter)                                 |
| Tests automatisés        | 1 428 API, 1 186 Flutter, 34 site web, tous réussis          |
| Décisions d'architecture | 27 ADR, 5 publiés ici                                        |

## Technologies

- **API** : Node.js 22, AdonisJS 6, TypeScript, PostgreSQL 17 avec PostGIS, Redis, BullMQ
- **Mobile** : Flutter, BLoC
- **Web** : Next.js 16, React 19, Tailwind CSS, dans un monorepo pnpm (site public et panneau d'administration)
- **Temps réel** : Mercure en Server-Sent Events
- **IA** : Anthropic Claude derrière une couche d'abstraction (OpenAI et Gemini supportés)
- **Infrastructure** : Docker, Coolify sur Hetzner, GitHub Actions, Cloudflare

## L'application

<p>
<img src="docs/img/01_home.jpg" width="30%" alt="Accueil"> <img src="docs/img/03_filters.jpg" width="30%" alt="Filtres"> <img src="docs/img/04_results.jpg" width="30%" alt="Résultats de recherche">
</p>

L'interface est en turc.

## Points à regarder

**Modération assistée par IA.** Chaque nouvelle annonce passe par un pipeline asynchrone. Un modèle de vision analyse les photos et produit un score de confiance, qui oriente l'annonce vers la publication ou vers un modérateur humain. Le pipeline a un arrêt automatique en cas de panne du modèle, des prompts versionnés, un corpus de référence pour comparer les versions et une console d'administration qui suit les coûts, la latence et la qualité.

**Messagerie temps réel.** Le chat utilise Mercure (SSE) plutôt que des WebSockets, avec des sujets privés protégés par jeton. Voir l'[ADR-0002](./docs/adr/0002-mercure-sse-over-websockets.md).

**Structure backend propre.** Chaque module suit une organisation Action, Controller, Repository pour garder des contrôleurs minces. Voir l'[ADR-0005](./docs/adr/0005-action-controller-repository.md).

**Choix d'infrastructure pragmatiques.** Recherche plein texte PostgreSQL plutôt qu'Elasticsearch ([ADR-0003](./docs/adr/0003-postgresql-fts-over-elasticsearch.md)), Redis et BullMQ plutôt qu'une file de messages cloud ([ADR-0004](./docs/adr/0004-bullmq-redis-over-cloud-queues.md)).

**Protection des données dès la conception.** Le produit respecte la loi turque sur les données personnelles (KVKK) : chiffrement au repos, journaux d'audit, suppression logique et brouillage de la position exacte côté serveur.

<p><img src="docs/img/06_location.jpg" width="40%" alt="Position approximative d'une annonce"><br><sub>L'adresse exacte n'est jamais affichée : l'application montre un cercle de 750 m calculé côté serveur.</sub></p>

## Kaybaz, une démo en ligne à essayer

Kaybaz est la marketplace turque que j'ai construite seul à partir de 2018. Elle a tourné en production jusqu'en 2022, et je l'ai remise en ligne en 2026 comme démo publique. Hallego est mon produit actuel, Kaybaz montre un projet plus ancien et la façon dont je sécurise un vieux système avant de l'exposer. Le code source est privé. L'interface est en turc, le bandeau de la démo est en anglais.

**À essayer en deux minutes, sans compte.** Ouvrez [kaybaz.md-x.dev](https://kaybaz.md-x.dev), allez sur la page de connexion, cliquez sur « Try as buyer », ouvrez une annonce et achetez-la. Le paiement est simulé, aucun argent ne bouge : les fonds restent sous séquestre jusqu'à la confirmation de réception. Déconnectez-vous ensuite et cliquez sur « Try as seller » pour voir la commande côté vendeur. « Try as admin » ouvre l'espace d'administration en lecture seule.

**Ce que la démo montre.** Un paiement avec séquestre et remboursements automatiques planifiés, la livraison avec retours et litiges, une messagerie temps réel avec Socket.IO, une recherche par localisation sur quatre niveaux de données administratives turques avec une carte Leaflet, et une PWA installable. Stack de Kaybaz : AdonisJS 5, Vue 3 avec Quasar, PostgreSQL 17, Docker, Coolify sur Hetzner, Cloudflare. Environ 55 000 lignes de code.

**Rendre une démo publique sûre.** Un seul interrupteur `DEMO_MODE` transforme chaque intégration sortante en simulation locale : passerelle de paiement, facturation, e-mail. L'inscription est fermée et les comptes de démo sont verrouillés : pas de changement de mot de passe, pas de modification d'annonce, pas d'envoi de fichier. Les données sont anonymisées et les photos ne portent aucune métadonnée EXIF. Un script restaure chaque nuit une copie propre de la base en construisant une nouvelle base à côté de la base active, en la vérifiant, puis en échangeant les noms : un reset qui échoue laisse la démo intacte.

## Contact

Michael DAS, développeur full-stack, en cours d'installation à Ottawa (Canada).

&copy; 2026 Michael DAS. Tous droits réservés. Les textes et images de ce dépôt sont partagés à titre de présentation seulement.
