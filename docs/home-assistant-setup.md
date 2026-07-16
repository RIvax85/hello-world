# Mise en place : piloter Home Assistant depuis Claude Code sur mobile

Ce guide décrit la configuration à faire **une seule fois** pour que les
sessions Claude Code lancées depuis le smartphone (app Claude → Code)
puissent contrôler vos appareils et gérer vos automatisations.

Les sessions mobiles s'exécutent dans le cloud, pas sur le téléphone :
votre instance Home Assistant doit donc être joignable depuis Internet.

## Étape 1 — Rendre Home Assistant accessible publiquement (HTTPS)

Au choix :

- **Nabu Casa Cloud** (le plus simple, solution officielle payante) :
  votre URL est du type `https://xxxxxxxx.ui.nabu.casa`.
- **Cloudflare Tunnel** (gratuit) : add-on "Cloudflared" dans Home
  Assistant + un domaine chez Cloudflare.
- **Reverse proxy** (nginx, Caddy…) avec un certificat HTTPS valide.

Une adresse locale (`http://192.168.x.x:8123`) ne fonctionnera **pas**.

## Étape 2 — Rien à installer dans Home Assistant

Le serveur MCP utilisé est `ha-mcp` (le même que dans la configuration
Claude Desktop) : il est lancé **dans la session cloud** via
`uvx ha-mcp@latest` (voir `.mcp.json`) et dialogue avec Home Assistant par
son API, avec le token. Aucune intégration ni add-on n'est donc requis
côté Home Assistant.

Alternative : l'intégration officielle "Model Context Protocol Server"
de Home Assistant (endpoint `https://votre-url/mcp_server/sse`) reste
utilisable si on préfère un serveur hébergé par Home Assistant, mais elle
ne couvre que le contrôle des appareils exposés à Assist.

## Étape 3 — Créer un token d'accès longue durée

1. Dans Home Assistant : cliquer sur votre profil (en bas à gauche)
   → onglet **Sécurité** → **Jetons d'accès longue durée** → Créer.
2. Utiliser un compte **administrateur** (nécessaire pour l'API de
   configuration des automatisations).
3. Copier le token : il ne sera plus affiché ensuite.

## Étape 4 — Configurer l'environnement Claude Code

Sur [claude.ai/code](https://claude.ai/code) (ou dans l'app) :

1. Ouvrir les réglages de l'**environnement** utilisé pour ce dépôt.
2. Ajouter deux **variables d'environnement** :
   - `HA_URL` = `https://votre-url-home-assistant` (sans `/` final)
   - `HA_TOKEN` = le token créé à l'étape 3
3. Régler la **politique réseau** (Network access) sur **Custom** :
   - ajouter votre domaine Home Assistant dans « Allowed domains »
     (par ex. `xxxxxxxx.ui.nabu.casa`) ;
   - **cocher « Also include default list of common package managers »** :
     indispensable pour que `uvx` puisse télécharger `ha-mcp` depuis PyPI
     au démarrage de la session.

Le fichier `.mcp.json` du dépôt référence ces variables : aucun secret
n'est stocké dans le dépôt.

## Étape 5 — Tester depuis le smartphone

Lancer une nouvelle session Claude Code sur ce dépôt et demander par
exemple :

- « Liste mes automatisations Home Assistant » (API REST)
- « Éteins les lumières du salon » (serveur MCP)
- « Crée une automatisation qui allume la terrasse au coucher du soleil »

## Dépannage

| Symptôme | Cause probable |
|---|---|
| Le serveur MCP n'apparaît pas | `HA_URL`/`HA_TOKEN` absents des réglages de l'environnement, ou session lancée avant leur ajout |
| `uvx` ne télécharge pas `ha-mcp` | Case « Also include default list of common package managers » non cochée dans la politique réseau Custom |
| Erreur 401 | Token invalide ou expiré → en générer un nouveau |
| Timeout / connexion refusée | Instance non accessible publiquement, ou domaine Nabu Casa absent des « Allowed domains » |
| 401/403 sur `/api/config/automation/...` | Le token n'appartient pas à un compte administrateur |
