# Home Assistant — pilotage depuis Claude Code

Ce dépôt sert à piloter Home Assistant depuis des sessions Claude Code
(notamment lancées depuis le smartphone). Deux canaux sont disponibles :

## 1. Serveur MCP `ha-mcp` (contrôle des appareils)

Configuré dans `.mcp.json` : la session lance `uvx ha-mcp@latest` (le même
serveur que la configuration Claude Desktop de l'utilisateur), qui se
connecte à Home Assistant via les variables `HA_URL` et `HA_TOKEN`.
Utiliser ses outils en priorité pour lire les états et contrôler les
appareils. Si ha-mcp expose aussi des outils de gestion des automatisations,
ils peuvent être utilisés, mais l'API REST ci-dessous reste la référence.

## 2. API REST (configuration et automatisations)

Pour créer, modifier ou supprimer des automatisations, utiliser l'API REST
de Home Assistant avec `curl`. Les variables d'environnement `HA_URL` et
`HA_TOKEN` sont définies dans les réglages de l'environnement Claude Code
(voir `docs/home-assistant-setup.md`).

En-têtes à utiliser pour tous les appels :

```bash
curl -sS -H "Authorization: Bearer $HA_TOKEN" -H "Content-Type: application/json" ...
```

Endpoints utiles :

| Action | Méthode et endpoint |
|---|---|
| Vérifier la connexion | `GET $HA_URL/api/` |
| Lister tous les états (dont `automation.*`) | `GET $HA_URL/api/states` |
| Lire une automatisation | `GET $HA_URL/api/config/automation/config/{id}` |
| Créer/modifier une automatisation | `POST $HA_URL/api/config/automation/config/{id}` (corps JSON : `alias`, `trigger`, `condition`, `action`, `mode`) |
| Supprimer une automatisation | `DELETE $HA_URL/api/config/automation/config/{id}` |
| Recharger les automatisations | `POST $HA_URL/api/services/automation/reload` |
| Vérifier la configuration | `POST $HA_URL/api/config/core/check_config` |
| Appeler un service | `POST $HA_URL/api/services/{domaine}/{service}` |

Notes :
- L'`{id}` d'une automatisation existante se trouve dans `attributes.id`
  de l'entité `automation.*` (via `/api/states`). Pour une nouvelle
  automatisation, générer un id unique (par ex. un timestamp).
- L'API de configuration nécessite un token d'un compte administrateur.
- Après un `POST` sur `/api/config/automation/config/...`, l'automatisation
  est rechargée automatiquement ; sinon appeler `automation/reload`.
- Toujours relire l'automatisation existante avant de la modifier : le
  `POST` remplace la définition complète, il ne fusionne pas.

## Règles de prudence

- Avant de supprimer ou remplacer une automatisation, afficher sa
  définition actuelle et la sauvegarder dans `backups/` (créer le dossier
  si besoin) en la committant, pour pouvoir revenir en arrière.
- Ne jamais écrire le token dans un fichier du dépôt ni dans un message
  de commit.
