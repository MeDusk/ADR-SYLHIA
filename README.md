## <span style="color: red;">Structure du code (Prototype)</span>

**Trois dépôts, séparés par cycle de vie, et un seul module dans le hub qui porte la collecte, les écrans et l’orchestration.** Correspond à la structure déjà livrée dans le gitlab INGENIDOC.

| Dépôt | Rôle | Langage | Évolue au rythme de |
| --- | --- | --- | --- |
| `sylhia/base` | Source de vérité du schéma : lots, contrôles, tests de sécurité, ADR du projet | SQL, pgTAP | Lots revus par le responsable de la base |
| `sylhia/module-hub` | Module `modules/sylhia` : collecte, droits, validation, interface | PHP 8.4 (Laravel 13), Vue 3 + TypeScript | Versions du hub |
| `sylhia/ia` | Worker IA : extraction, synthèse, propositions validées, LLM extensions | Python 3.12 | Modèles et prompts |


```text
sylhia/                         groupe GitLab
├── base/                       source de vérité SQL
├── module-hub/                 cloné dans hub/modules/sylhia
└── ia/                         worker IA Python
```

### `sylhia/base` : la base comme contrat

```text
base/
├── README.md                   installer, reconstruire, contrôler
├── VERSION                     
├── CHANGELOG.md
├── lots/                       scripts publiés, jamais modifiés après publication
│   ├── ordre.lst               ordre d’application des lots
│   ├── 3.0.0/  3.1.0/  3.1.1-objectifs/
│   ├── rgpd-1.0.0/  acces-1.0.0/  ech-1.0.0/
│   └── mail-1.0.0/
├── controles/                  contrôle post-installation, diagnostic sécurité
├── tests/                      pgTAP : identité, RLS, API fermées, idempotence
├── jeux-essai/                 données fictives uniquement
├── outils/
│   ├── reconstruire.sh         base vide → tous les lots → contrôles
│   └── comparer-catalogue.sh   base réelle vs cartographie
├── docker/compose.yml          PostgreSQL 17 + pgTAP pour le poste
├── archives/                   SYLIA 1.1.1, 1.0.0, NC V2 : lecture seule
├── docs/
│   ├── adr/                    ADR-001 identité…
│   ├── contrat-integration.md  fonctions api.* publiques, codes d’erreur
│   └── cartographie/
└── .gitlab-ci.yml
```

### `sylhia/module-hub` : le module `modules/sylhia`

```text
module-hub/                         
├── composer.json                   dépendances propres, fusionnées par le hub
├── config/sylhia.php               connexion, descripteur chatbox, réglages
├── routes/
│   ├── web.php                     écrans
│   └── ai.php                      serveur MCP SYLHIA (outils typés)
├── src/
│   ├── SylhiaServiceProvider.php   capacités, menu, routes, MCP, tableau de bord
│   ├── Identite/
│   │   ├── ActeurResolver.php      sub du hub → hub.acteur
│   │   └── ConnexionActeur.php     connexion PostgreSQL au login de l’acteur (ADR-001)
│   ├── Base/
│   │   ├── ApiSylhia.php           seul point d’appel des fonctions api.*
│   │   ├── Idempotence.php         clé UUID par action
│   │   └── ErreursSql.php          SQLSTATE → exceptions métier
│   ├── Propositions/               lecture des propositions IA
│   ├── Validation/                 décision humaine (accepter, corriger, refuser)
│   ├── Sources/                    dépôt de fichiers adressés par SHA-256
│   ├── Gamsys/AdaptateurGamsys.php lecture → hub.identifiant_externe
│   ├── Http/Controllers/
│   └── Mcp/                        serveur et outils en lecture, sans SQL libre
├── database/migrations/            tables côté hub seulement (jamais les schémas SYLHIA)
├── resources/js/
│   ├── pages/sylhia/               Propositions.vue, Validation.vue…
│   ├── components/
│   └── widgets/                    carte de tableau de bord
├── tests/Feature  tests/Unit
└── docs/
```

### `sylhia/ia` : le worker

```text
ia/
├── pyproject.toml                  uv, ruff, mypy, pytest
├── src/sylhia_ia/
│   ├── worker.py                   prendre un travail → traiter → déposer la proposition
│   ├── base.py                     appels api.* avec le login IA
│   ├── gamsys.py                   client de l’API /gamsys du hub (jeton M2M)
│   ├── llm.py                      passerelle unique vers le modèle, journalisée
│   ├── contrats/                   schémas Pydantic des sorties
│   └── taches/                     extraction_mail.py, synthese.py…
├── prompts/<role>/<version>.md     versionnés, publiés dans hub.configuration_agent
├── evals/                          cas fictifs et résultats attendus
├── tests/
├── Dockerfile
└── .gitlab-ci.yml
```

## <span style="color: red;"> Stack technique V0 </span>

**Règle de choix : reprendre ce que INGENIDOC maintient déjà et Ajouter que Python pour l’IA.**

| Couche | Choix | Argument principal |
| --- | --- | --- |
| Base | PostgreSQL 17 | Modèle pret, avec la sécurité et la traçabilité dans la base. |
| Schéma | Lots SQL immuables, `psql`, tests pgTAP, reconstruction à vide en CI | Les lots gèrent déjà leurs refus et leurs transactions. La CI prouve à chaque fois que la base se reconstruit. |
| Application et collecte | Laravel 13, PHP 8.4, module du hub | C’est le socle du groupe. On hérite des droits, du menu, des tâches planifiées et de la file de travaux sans rien construire. |
| Interface/Web | Vue 3, Inertia 3, TypeScript, Tailwind 4, reka-ui | Imposée par le hub. |
| Identité | Passport (OIDC) et Fortify du hub, plus un login PostgreSQL par acteur | Authentification déjà durcie. La base identifie l’acteur par son login. |
| Planification et file | Planificateur Laravel et file `database` du hub, PostgreSQL pour le worker (`SKIP LOCKED`) | Aucun Redis ni courtier de messages à exploiter. |
| IA | Worker Python 3.12 : uv, Pydantic v2, psycopg 3, httpx, passerelle LLM unique, LLM bib | Validation stricte des sorties, outillage d’évaluation, et la compétence de l’équipe IA. |
| Qualité et livraison | GitLab CI : Pint, PHPUnit, ESLint, vue-tsc, ruff, mypy, pytest, Secret Detection, SAST | Les règles du hub, et une barrière automatique avant chaque fusion. |
| Poste | Windows, WSL2 (Ubuntu 24.04), VS Code, Claude Code | Même environnement que la CI, avec les extensions PHP exigées par le hub. |


## <span style="color: red;"> Architecture Cible V0 </span>


<div align="center">
  <img src="image-1.png" alt="Architecture cible" width="900">
</div>


## <span style="color: red;"> ROADMAP V0 </span>

<div align="center">
  <img src="image-2.png" alt="Architecture cible" width="900">
</div>


## <span style="color: red;"> Méthodologie : Rôles, responsabilités et règles de décision / Plane Preview </span>

> Légende RACI : **R** = Réalise · **A** = Approuve (un seul par ligne) · **C** = Consulté · **I** = Informé

## 1. Les rôles

| Rôle | Titulaire | Mission | Périmètre propre | Décide de | Ne décide pas seul de | Suppléant |
|---|---|---|---|---|---|---|
| **Directeur du site** · Responsable produit et arbitre | Christian LINOSSIER | Garantir que SYLHIA sert les priorités d’INTERFAS | Feuille de route, priorités, relation avec les services et INGENIDOC | Priorités métier, cas d’usage IA, passage des portes G0 à G4, arbitrage des désaccords | Choix techniques détaillés | — |
| **Ingénieur software** · Responsable application | Matthias DAUVEL | Livrer une application fiable et maintenable dans le hub | Dépôt `module-hub` : collecte, écrans, droits applicatifs, CI/CD | Conception et code PHP / Vue dans son périmètre | Modification de la base ou d’un ADR | Ingénieur IA |
| **Ingénieur IA** · Responsable données et IA | Mohamed NAJID | Garantir des données sûres et une IA qui propose sans décider | Dépôts `base` et `ia` : schéma SQL, sécurité des données, worker IA, évaluations | Conception et code SQL / Python dans son périmètre | Modification du module ou d’un ADR | Ingénieur software |

## 2. Matrice RACI

| Activité | Directeur du site | Ingénieur software | Ingénieur IA |
|---|:---:|:---:|:---:|
| **Pilotage** | | | |
| Priorités et feuille de route | **A** / R | C | C |
| Tri de l’Intake (demandes des services) | **A** / R | I | I |
| Contenu de chaque cycle | **A** | R | R |
| Passage des portes G0 à G4 | **A** | R | R |
| Relation avec INGENIDOC (noyau, accès, instance) | **A** | R | C |
| **Décisions d’architecture** | | | |
| ADR base, données, sécurité, IA | **A** | C | R |
| ADR application, collecte, CI/CD | **A** | R | C |
| **Base et sécurité** | | | |
| Lots SQL, contrôles, tests pgTAP | I | C | **A** / R |
| Socle de sécurité (identités, finalités, conservation, systèmes maîtres) | **A** | C | R |
| Matrice des droits par métier | **A** | C | R |
| **Collecte** | | | |
| Fiches sources | **A** | C | R |
| Connecteurs et orchestration | I | **A** / R | C |
| **IA** | | | |
| Choix des cas d’usage | **A** | C | R |
| Worker, prompts versionnés, évaluations | I | C | **A** / R |
| **Application** | | | |
| Écrans de validation, tableaux de bord, outils MCP | C | **A** / R | C |
| CI/CD, GitLab, environnements | I | **A** / R | C |
| **Mise en service** | | | |
| Recette métier et pilote | **A** | R | R |
| Exploitation (sauvegardes, supervision, procédures) | I | **A** / R | R |

> Les externes interviennent en **C**.

## 3. Règles de travail communes

- **Relecture croisée.** Toute merge request est relue par l’autre ingénieur. Personne ne fusionne seul.
- **Travail en cours limité.** Au plus 2 tâches « En cours » par developpeur.
- **Disponibilité.** Un créneau d’arbitrage fixe de 30 minutes par semaine, et une réponse aux décisions urgentes sous 48 h (A discuter au fur et é mesure).
- **Une seule source par information.** Plane dit quoi, qui et quand. GitLab contient le code et les ADR. Le rapport clair et optimisé explique pourquoi.
- **Suppléance.** Chaque developpeur sait relire et reprendre le dépôt de l’autre, ce qui évite qu’un dépôt ne dépende que d’une personne.

### 4. **Plane preview**

<div align="center">
  <img src="image-3.png" alt="Architecture cible" width="1200">
</div>


<u>___Flux d'une tache___.</u>
Chaque tâche part d'un besoin métier. Une fois sa faisabilité vérifiée, elle passe à l'état **« Prêt »** et peut démarrer. Dès qu'un développeur la prend en charge, elle passe **« En cours »**. Quand il la juge terminée, il ouvre une merge request et la tâche passe **« En revue »** : le suppléant désigné par la matrice RACI la vérifie. Deux cas se présentent alors. Soit la tâche revient au développeur pour correction, soit elle est validée et part directement en intégration continue (CI) et finalement la tache est **« Terminé »**.

## > Modelisation de Flux d'une tache

<div align="center">
  <img src="image-5.png" alt="Architecture cible" width="1200">
</div>

