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

## Étape 2 — Activer l'intégration MCP Server dans Home Assistant

1. Paramètres → Appareils et services → Ajouter une intégration.
2. Chercher **"Model Context Protocol Server"** et l'installer.
3. Choisir ce que l'assistant peut contrôler via les paramètres
   d'exposition d'Assist (Paramètres → Assistants vocaux → Exposer).

L'endpoint MCP est alors disponible sur `https://votre-url/mcp_server/sse`.

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
3. Vérifier que la **politique réseau** de l'environnement autorise le
   trafic sortant vers votre domaine Home Assistant (ajouter le domaine
   à la liste blanche si la politique est restrictive).

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
| Erreur 401 | Token invalide ou expiré → en générer un nouveau |
| Timeout / connexion refusée | Instance non accessible publiquement, ou domaine bloqué par la politique réseau de l'environnement |
| 404 sur `/mcp_server/sse` | Intégration MCP Server non installée dans Home Assistant |
| 401/403 sur `/api/config/automation/...` | Le token n'appartient pas à un compte administrateur |
