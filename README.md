# demo-itsm-servicenow

[![CI](https://github.com/xavier-manclieres/demo-itsm-servicenow-Jira/actions/workflows/ci.yml/badge.svg)](https://github.com/xavier-manclieres/demo-itsm-servicenow-Jira/actions/workflows/ci.yml)

Détection d'incidents récurrents et rédaction de REX assistée, en **infrastructure et workflows as code**.

Chaque nuit, n8n repère dans ServiceNow (ou Jira Service Management) les incidents qui se répètent et propose des problèmes à valider. Quand un incident critique (P1) est résolu, il rédige un brouillon de REX à relire. Airtable sert d'espace de validation, Claude de rédacteur, Power BI de restitution.

> Règle de conception : l'outil ITSM reste la source de vérité, et rien n'y est créé sans validation humaine.

## Architecture

```
ServiceNow  <-- API REST / webhook -->  n8n  <-->  Claude (tri, rédaction)
                                         |
                                         +--> Airtable (validation humaine)
                                         +--> GET /kpi --> Power BI
```

| Workflow | Rôle | Déclencheur | État |
|---|---|---|---|
| WF0 | Jeu de données de démo | manuel, une fois | construit, testé |
| WF0b | Incident P1 terminé, pour tester WF2 (Jira) | manuel | construit, testé |
| WF1 | Détection des incidents récurrents | chaque nuit à 2 h | construit, testé |
| WF1b | Création du problème dans ServiceNow | candidat passé à « Validé » (vérifié toutes les 5 min) | construit, testé |
| WF2 | Brouillon de REX | incident P1 résolu | construit, testé |
| WF2b | Publication du REX | brouillon passé à « Validé » (vérifié toutes les 5 min) | construit, testé |
| WF3 | Indicateurs pour Power BI | actualisation du rapport (GET /webhook/kpi) | construit, testé |

« Testé » signifie exécuté sur une instance ServiceNow de développement et sur un site Jira Service Management Cloud, avec des données fictives.

## Ce que le DevOps apporte

- **Infra as code** : n8n et Postgres démarrent avec `docker compose up`.
- **Deux outils ITSM, une seule base** : Jira et ServiceNow partagent les tables Airtable ; le champ « Outil » sépare leurs candidats et leurs brouillons, chaque workflow ne voit que les siens.
- **Workflows as code** : exports JSON sans identifiants, relus par diff. Un dossier par outil ITSM (`servicenow/workflows/`, `jira/workflows/`) et `workflows/` pour ce qui ne dépend d'aucun outil (WF3).
- **Prompts versionnés** : `prompts/`, chaque changement est un commit.
- **Tests** : `code-nodes/` exécute le code des nœuds Code **directement depuis les exports** de workflows. Le code testé est exactement celui qui tourne dans n8n, sans copie à maintenir.
- **CI** : validation du JSON, tests et recherche de secrets à chaque push.

## Démarrage

Prérequis : Docker, Node.js 22 ou plus, Git.

```bash
cp .env.example .env        # puis renseigner les valeurs
docker compose -f infra/docker-compose.yml --env-file .env up -d
```

n8n est ensuite disponible sur http://localhost:5678.

```bash
npm test                    # tests des nœuds Code
```

### Importer les workflows

Dans n8n : *Workflows › Import from File*, puis choisir un fichier de `servicenow/workflows/` (et `workflows/` pour WF3). Ensuite :

1. Créer les identifiants **ServiceNow** (OAuth2 : application « n8n » dans System OAuth → Application Registry, redirection vers `…/rest/oauth2-credential/callback` ; la Basic Auth est bloquée par défaut sur les instances récentes — « Basic Auth Restriction »), **Airtable** (jeton d'accès), **Anthropic** et, pour WF3, un identifiant **Basic Auth** dédié à Power BI, puis les associer aux nœuds.
2. Remplacer l'identifiant de base Airtable (`app…`) et de table (`tbl…`) par ceux de votre base, construite selon `airtable/schema.md`.
3. WF0 crée lui-même les trois CI de la démo s'ils manquent, puis 22 incidents marqués `demo-itsm` (ID de corrélation). WF0b (Jira) crée un incident P1 terminé pour tester WF2.

### Brancher Power BI

WF3 renvoie un JSON en trois parties : `synthese` (une ligne d'indicateurs), `problemes` et `rex` (une ligne par enregistrement). Le guide pas à pas (requêtes Power Query, mesures DAX, mise en page) est dans [`powerbi/README.md`](powerbi/README.md).

![Rapport Power BI sur données fictives](powerbi/rapport%20Power%20BI%202026-10-10%20104811.png)

Le rapport se construit d'abord sur `powerbi/exemple-kpi.json`, une réponse de WF3 sur données fictives, produite par le vrai code du nœud (un test vérifie qu'elle reste à jour). On bascule ensuite sur l'adresse de WF3 en changeant une seule ligne.

| Indicateur | Calcul |
|---|---|
| Taux de validation | validés (y compris créés) / (validés + rejetés) |
| Incidents regroupés | somme des incidents de tous les candidats |
| Délai de relecture | de la création du brouillon à sa publication, en heures |
| MTTR P1 | moyenne des durées ouverture → résolution des P1, en minutes |

## Arborescence

```
workflows/     exports JSON communs aux outils ITSM (WF3, indicateurs)
code-nodes/    JavaScript des nœuds Code + tests
prompts/       prompts du tri et du REX
servicenow/    workflows WF0 à WF2b, Business Rule, jeu de données
jira/          workflows WF0 à WF2b pour Jira Service Management (+ WF0b, incident P1 de test)
airtable/      schéma des tables
powerbi/       données d'exemple et guide du rapport
infra/         docker-compose, déploiement
.github/       CI
```

## Sécurité

- Aucun secret dans le dépôt : `.env` est ignoré, seul `.env.example` est versionné.
- Les exports de workflows ne doivent contenir aucun identifiant (vérifier avant chaque commit).
- Avant tout envoi à Claude, WF2 masque les adresses IP et les secrets passés en clair, et remplace les noms des intervenants par « Intervenant 1, 2… ». Un test vérifie qu'aucune de ces données ne fuit.
- WF3 n'expose que des comptes, des durées et des statuts : ni texte de REX, ni hypothèse, ni sys_id. Son adresse est protégée par un identifiant Basic Auth que Power BI garde dans son coffre : il n'apparaît jamais dans le code du rapport versionné. Un test vérifie l'absence de fuite.
- Toutes les données de démo sont fictives.

## Feuille de route

- [x] Semaines 1-2 : dépôt, CI, base Airtable, WF0
- [x] Semaines 3-4 : WF1 et WF1b, tests sur le code réel des nœuds
- [x] Semaines 5-6 : WF2, WF2b et WF3
- [x] Rapport Power BI (projet `.pbip` versionné, construit sur données fictives)
- [x] Instance ServiceNow de développement, connexion OAuth2
- [x] Second outil ITSM : Jira Service Management, sur la même base Airtable
- [ ] Webhook ServiceNow (Business Rule) à la place du sondage de WF2
- [ ] Déploiement Ansible, post LinkedIn

## Licence

MIT
