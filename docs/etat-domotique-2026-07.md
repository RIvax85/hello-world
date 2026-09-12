# État de la domotique — juillet 2026 (après migration KLF200 → TaHoma)

Synthèse de référence pour toute session future. Détails opérationnels dans
`backups/`.

## Architecture actuelle

| Couche | Matériel | Intégration HA |
|---|---|---|
| io-homecontrol (stores, volets VELUX à terme) | Somfy TaHoma Switch | Overkiz (cloud, compte Somfy Europe) |
| RTS (volets roulants maison) | Pont RTS existant (`cover.rts_*`) | inchangé |
| Zigbee (capteurs, relais) | Zigbee2MQTT | inchangé |
| Alarme | Alarmo | inchangé |
| VELUX KLF200 | **retiré du service** (voir historique) | intégration Velux à supprimer |

## Ce qui marche (2026-07-28 au soir)

- TaHoma : Store Banne (« Jardin »), Store Veranda (« Salle à manger »),
  Lumière Banne 1. Entités HA : `cover.jardin_store_banne`,
  `cover.salle_a_manger_store_veranda`, `light.jardin_lumiere_banne_1`.
- Les 5 automatisations stores migrées et testées (pluie, vent fort, 17h,
  coucher du soleil, anti-chaleur 11h). L'anti-chaleur ne déploie plus les
  stores si `input_boolean.mode_vacances` est actif.
- Sémantique vérifiée : `close_cover` = rentrer le store.
- Commandes locales : KLI 313 (volets VELUX), Situo 5 io Pure II
  (canal 1 = store banne, canal 2 = lumières), commande store véranda.

## Historique et leçon clé (saga KLF200, juillet 2026)

1. Le KLF200 n'arrivait pas à enregistrer 2 volets de toit SSL neufs
   (produit fantôme « ptype52 »).
2. Après un reset usine du KLF200, plus AUCUN produit n'était découvrable —
   diagnostic final : **clé de sécurité io**. Chaque box 2W inscrit sa clé
   dans les moteurs ; le reset a détruit la clé sans la retirer des moteurs,
   qui refusaient toute nouvelle box. Le KLF n'était probablement pas HS.
3. Le KLF200 étant un produit arrêté (directive RED 2025, successeur KLF150
   sans API locale), remplacement par TaHoma Switch.
4. Récupération des moteurs Somfy io des stores : **double coupure 2-8-2
   (2 s OFF / 8 s ON / 2 s OFF / ON) puis PROG 7 s** = efface mémoire radio
   ET clé io. Puis : prise en main (Montée+Descente bref simultané),
   fins de course (my+Descente / my / my+Montée / my long), enregistrement
   (PROG), appairage TaHoma (parcours « io 1-way »).
5. Pièges rencontrés : la fenêtre usine s'applique à TOUS les appareils du
   même disjoncteur (un PROG bref pendant la fenêtre s'inscrit partout) ;
   dissociation par la bascule PROG (ouvrir via un canal exclusif, appui
   bref sur le canal à retirer) ; un récepteur lumière effacé s'allume en
   permanence.

## Ajouts du 2026-07-29 (jour du départ)

- `automation.eclairage_exterieur_commutateur_sous_escalier` : alimente les
  lampes extérieures (détecteurs intégrés) du coucher du soleil à 1h.
- `automation.alarme_echec_d_armement_annonce_des_ouvrants` : sur événement
  `alarmo_failed_to_arm`, notifie les téléphones + annonce vocale (enceinte
  séjour) du ou des ouvrants qui bloquent. Testé OK (fenêtre bureau).
- `automation.alarme_sirene_continue_pendant_le_declenchement` : la sirène
  Zigbee s'arrêtait après 10 s (durée par défaut des commandes warning
  Zigbee2MQTT, `siren.sirene` est en assumed_state). Relance toutes les 8 s
  tant qu'Alarmo est « triggered », plafond ~3 min. Testé OK (>30 s,
  coupure immédiate au désarmement).
- Test négatif : les détecteurs de fumée ne réagissent pas à une commande
  MQTT `warning` → pas de sonnerie multi-étages via les détecteurs.
- ⚠️ « Détecteur de fumée Palier » : tous états `unknown` dans HA —
  probablement hors réseau Zigbee ou pile morte. Bouton test à vérifier.

## Ajustements vacances du 2026-07-30

Incident : l'automatisation « Nounou — Désarmement alarme » a désarmé
l'alarme le jeudi 30/07 à 17h45, maison vide et famille en vacances.
L'armement automatique au départ n'a pas repris la main (son déclencheur
est le *front* de départ, déjà passé). Alarme restée désarmée ~1 h.

Corrections appliquées :

- `automation.nounou_desarmement_alarme` : condition permanente
  `input_boolean.mode_vacances = off`. Le désarmement nounou est
  désormais automatiquement neutralisé à chaque période de vacances.
- `automation.femme_de_menage_desarmement_alarme` et
  `..._rearmement_alarme` : condition
  `mode_vacances = off OU date = 2026-08-11` — le passage du 11 août
  (un mardi) est maintenu, tous les autres mardis de l'été sont neutralisés.
- `automation.alarme_desarmement_automatique_5h30` : garde
  `mode_vacances = off` ajoutée (ne pouvait pas se déclencher en mode
  Absence, mais cohérence de principe).
- Nouveau `automation.vacances_filet_de_securite_rearmement_alarme` :
  garde-fou générique contre tout désarmement imprévu pendant une absence.
  Version finale (tenant compte des visiteurs attendus — belle-mère,
  femme de ménage — qui désarment manuellement au bouton porte ou au
  clavier RFID et ne sont PAS suivis par `zone.home`) :
  déclencheur `time_pattern` toutes les 15 min ; réarme seulement si
  mode vacances actif, alarme désarmée depuis > 20 min, `zone.home < 1`,
  ET aucune activité intérieure depuis 30 min (mouvement salon / entrée /
  couloir / véranda / WC, ou ouverture porte principale / salon / véranda /
  garage). Un visiteur présent bloque donc le réarmement ; il repart, la
  maison redevient calme, l'alarme se réarme seule ~30 min plus tard.
  Exception sur le créneau ménage du 11/08 (9h-14h).
  ⚠️ Le capteur mouvement Terrasse est volontairement exclu (extérieur,
  risque de blocage permanent).
- Nouveau `automation.vacances_alerte_desarmement_manuel` : notifie les
  deux téléphones (avec photo caméra Entrée) dès qu'un désarmement a lieu
  pendant le mode vacances.
- `automation.portillon_sonnette` : la notification embarque désormais une
  photo de la caméra portillon (`/api/camera_proxy/camera.portillon_main`),
  priorité haute.

## Constat du 2026-08-12 — simulation de présence incomplète

Vérification en cours de vacances : l'intégration Presence Simulation
(`switch.simulation_vacances`, active depuis le 29/07) ne pilote en réalité
que **Volet Salon** (`cover.rts_4_shutter`) et **Volet Entrée**
(`cover.rts_6_shutter`). Les cinq autres volets étaient figés fermés :

| Volet | Fermé depuis (au 12/08 12h15) |
|---|---|
| SdB (`rts_8`) | 397 h (~16 j) |
| June (`rts_14`) | 175 h (~7 j) |
| Terrasse (`rts_2`) | 109 h (~4,5 j) |
| Malo (`rts_10`) | 79 h (~3,3 j) |
| Robin (`rts_12`) | 79 h (~3,3 j) |

Cause : `automation.volets_r_1_fermeture_5h` ferme SdB/Malo/Robin/June
chaque matin à 5h (elle tourne toujours, dernier passage le 12/08 à 5h) et,
en mode vacances, plus rien ne les rouvre — la réouverture au coucher du
soleil ne concerne que Terrasse + June et dépend du flag
`volets_sud_fermes_auto`, resté `off` (l'anti-chaleur n'a plus tourné
depuis le 04/08). Résultat : maison visiblement fermée côté rue, soit
l'inverse de l'effet recherché.

Correctif : `automation.vacances_volets_simulation_de_presence` — pendant
le mode vacances, ouvre Terrasse/SdB/Malo/Robin/June le matin (08h15),
les referme à la mi-journée si T° > 28 °C, les rouvre en soirée (19h) et
les ferme pour la nuit (22h), avec un décalage aléatoire de 0 à 45 min à
chaque étape. Salon et Entrée restent gérés par l'intégration.

Autres observations du 12/08 :
- L'alarme est en **mode Nuit** depuis 19 h (probablement un appui simple
  sur le bouton porte par un visiteur, qui arme en Nuit et non en Absence).
  En mode Nuit les capteurs de mouvement intérieurs ne sont pas armés.
- `last_triggered` de l'alarme = 11/08 08h31 : déclenchement le jour du
  ménage, la femme de ménage étant arrivée avant le désarmement de 9h.
  → à la rentrée, avancer le désarmement du mardi à 8h15.
- Mouvement isolé « Véranda » le 12/08 à 12h09, sans aucune ouverture de
  porte associée (dernière ouverture il y a 19 h) : probable faux positif
  (soleil/chaleur dans la véranda).

Actions prises le 12/08 à 12h51 :
- Alarme repassée en **mode Absence** (`armed_away`).
- Le service `alarmo.disable_sensor` n'existe pas dans cette version
  d'Alarmo : impossible d'exclure un capteur par API. Exclusion à faire
  manuellement le cas échéant via le panneau Alarmo → Capteurs →
  « Capteur Mouvement Veranda Occupation » → décocher le mode Absence.
- En attendant, `automation.alarme_filtre_faux_positif_veranda` : si un
  déclenchement survient alors que seul le capteur véranda a bougé et
  qu'aucun ouvrant n'a changé d'état depuis 10 min, l'alarme est désarmée
  puis immédiatement réarmée en Absence, avec notification photo. Une
  intrusion réelle passant nécessairement par un ouvrant, la sirène reste
  normale dans ce cas.

## Notifications — remplacement du Pixel 8 (2026-08-13)

Le Pixel 8 de Nathan est hors service. Le téléphone pro **Samsung-A34** a
été ajouté à l'app compagnon (`notify.mobile_app_samsung_a34`,
`device_tracker.samsung_a34`, testé OK).

Solution retenue : `automation.notifications_relais_pixel_8_vers_samsung_a34`
écoute les événements `call_service` du domaine `notify`, filtre ceux
destinés à `mobile_app_pixel_8` et les retransmet à l'identique (titre,
message, et bloc `data` avec image/priorité) au Samsung-A34.

Pourquoi un relais plutôt qu'un remplacement du service dans chaque
automatisation : 21 automatisations appellent `notify.mobile_app_pixel_8`,
il faudrait toutes les réécrire puis les réécrire à nouveau au retour du
Pixel. Le relais se désactive d'un clic. Effet de bord bienvenu :
`automation.archive_notifications` (qui journalise les notifications
Pixel dans le logbook) continue de fonctionner sans modification.

⚠️ À la réparation du Pixel : désactiver ou supprimer ce relais, sinon
Nathan recevra chaque notification en double.

⚠️ Observation : `sensor.nixel_8_battery_level` (téléphone de Pauline) ne
remonte plus rien depuis 13 jours (~30/07). Les notifications push
peuvent malgré tout arriver, mais à vérifier — sinon Pauline ne reçoit
plus rien non plus.

## Corrections du 2026-08-13 (retours de Pauline)

### Stores déployés pendant les vacances — cause racine

⚠️ **Diagnostic initial ERRONÉ, corrigé le 13/08.** J'avais attribué les
appels `set_cover_position` sur les deux stores (12/08 à 12h09, 13/08 à
11h26) à l'intégration Presence Simulation. Vérification faite,
`aa46a3c352f3473f8b9c10bf56e54ec0` est le **compte de Pauline**
(`person.pauline`) : c'est elle qui déployait les stores depuis l'app,
et qui refermait dans la foulée les volets Salon et Malo — une routine
anti-canicule manuelle. Les stores n'ont jamais été dans la simulation
(confirmé par Nathan).

Correspondance des `context_user_id` observés :
- `23bae9a71ce047ad9966e861cbdc20d2` → Nathan (compte utilisé par ha-mcp)
- `aa46a3c352f3473f8b9c10bf56e54ec0` → Pauline
- `52d64c7c190944bf8f2cbdf4d3a1ba5f` → intégration Presence Simulation
  (pilote les volets RTS : rts_4, rts_6…)

Le 12/08 à 16h36 l'automatisation pluie a bien rentré les stores (état
`closed` à 16h37, notification reçue) ; Pauline les a redéployés le
lendemain matin.

Facteur aggravant : les stores passent régulièrement en `unavailable`
(coupures cloud TaHoma — 12/08 14h19→14h34, 13/08 08h51→08h57). Une
commande envoyée pendant ces fenêtres peut ne pas aboutir.

Correctifs :
- Stores rentrés manuellement le 13/08 à 11h39 (banne était à 43 %,
  véranda à 100 %) — vérifiés `closed` / position 0.
- Nouveau `automation.vacances_stores_toujours_rentres` : en mode
  vacances, tout store restant déployé plus de 90 s est rentré, avec
  jusqu'à 3 tentatives et attente de confirmation d'état (absorbe les
  passages en `unavailable`). Notification si les 3 tentatives échouent.
- Retrait des stores de la Presence Simulation : **sans objet**, Nathan a
  vérifié le 13/08 qu'ils n'y figuraient pas. (Le retrait via l'API aurait
  de toute façon été impossible : l'intégration se configure par config
  entry, UI uniquement.)

### Température extérieure du dashboard

La tuile affichait `sensor.salon_melcloudhome_527f_47e6_outdoor_temperature`,
c'est-à-dire la sonde de l'unité extérieure de la clim (23,5 °C relevés
contre 30,4 °C chez Météo France). Remplacée par l'attribut `temperature`
de l'entité `weather.meteo_france_...antony`. La sonde clim est conservée
en dessous, renommée « Sonde ext. clim (indicative) ».

### Stores : règles définitives (13/08, demande de Nathan)

Comportement voulu : pas de déploiement par la simulation de présence,
mais déploiement conservé en cas de forte chaleur — **véranda 100 %,
banne 40 % maximum** — et rentrée maintenue en cas de vent ou de pluie.

Mise en œuvre :
- Nouveau `input_boolean.stores_deployes_pour_chaleur` : distingue un
  déploiement légitime (chaleur) d'un déploiement parasite (simulation).
- `automation.volets_sud_fermeture_anti_chaleur` : la condition
  `mode_vacances = off` a été **retirée** (le déploiement chaleur est
  désormais autorisé aussi en vacances) ; `cover.open_cover` remplacé par
  `cover.set_cover_position` — véranda 100, banne 40 ; le drapeau est levé
  avant les commandes. Les garde-fous pluie/vent restent en condition.
- `automation.vacances_stores_toujours_rentres` : ne rentre les stores que
  si le drapeau chaleur est éteint **et** si le déploiement ne vient pas
  d'un humain (`trigger.to_state.context.user_id is none`) — ajouté le
  13/08 après avoir découvert que Pauline pilotait les stores à la main :
  le filet ne doit jamais contrarier une décision humaine. Il ne subsiste
  que pour un déploiement d'origine automatique inattendue.
- Nouveau `automation.stores_fin_du_deploiement_chaleur` : éteint le
  drapeau dès que les deux stores sont rentrés (pluie, vent, 17h ou
  coucher du soleil), ce qui réarme le garde-fou.
- Inchangé : `store_veranda_remontee_si_pluie`,
  `stores_veranda_banne_remontee_si_vent_fort`, `store_veranda_remontee_17h`
  et la rentrée au coucher du soleil.

Vérifié en réel le 13/08 à 11h47 (30,4 °C, vent 3,6 km/h) : déclenchement
manuel de l'anti-chaleur → drapeau levé, les deux stores se déploient.

### Dashboard — section Température (13/08)

Nathan a retiré la tuile « Sonde ext. clim ». Harmonisation du rendu :
la température extérieure était une carte `entity` (rendu différent des
tuiles voisines) car Météo-France n'expose **pas** de capteur de
température — seulement UV, précipitations, humidité, couverture
nuageuse, risques. La température n'existe qu'en attribut de l'entité
`weather`. Passée en carte `tile` avec `state_content: temperature`
(possible en HA 2026.7.4), donc rendu identique aux autres.

Section Température finale : thermostat Clim Salon (carte de contrôle,
volontairement différente), puis quatre tuiles au même format —
Température Salon, Température Extérieure, Vigilance Météo-France,
Température Combles.

Vigilance Météo-France au 13/08 : **Orange = Canicule** (vent violent et
orages au vert), d'où le maintien des stores déployés.

## Reste à faire (rentrée septembre 2026)

1. **Volets VELUX** (3 combles, 2 toit R+1, dépendance : 2 volets +
   2 fenêtres) : encore porteurs de l'ancienne clé KLF → reset moteur
   physique (bouton P ~10 s) puis appairage TaHoma. Appeler avant :
   support Somfy 0 820 055 055 (manipulations « clé perdue ») et VELUX.
   Prévenir le locataire pour la dépendance.
2. **3 récepteurs lumière muets** : PROG du récepteur ~2 s puis PROG bref
   Situo canal 2, puis découverte TaHoma.
3. **Ménage HA** : supprimer l'intégration Velux et les entités orphelines
   (`cover.combles_*`, `cover.dependance_*`, `cover.store_banne`,
   `cover.rdc_store_veranda`, `light.store_banne_*`, et statuer sur
   `cover.store_banne_2` / `cover.store_veranda`).
4. Optionnel : API locale Overkiz (mode développeur Somfy + jeton) ;
   vérifier l'appairage du capteur vent Eolis du banne ; ajouter les volets
   de toit aux automatisations une fois appairés.
5. Ressusciter « Détecteur de fumée Palier » (pile / ré-appairage Zigbee) —
   point de sécurité incendie.
6. Mise à jour firmware du Shelly portail (reportée volontairement avant
   le départ).
7. Sirène : régler proprement la durée par défaut du warning côté
   Zigbee2MQTT (l'automatisation-relance devient alors une redondance).
8. Script central `notifier` : fan-out vers les 2 téléphones + trace
   `logbook.log` pour un historique des notifications consultable dans HA,
   puis migration progressive des automatisations.

## 2026-08-23 — Alarme qui s'arme alors que la maison est occupée

Symptôme : `automation.alarme_armement_automatique_au_depart` s'est
déclenchée à 12h50 alors que Nathan était à la maison.

Cause : `person.nathan_dondey` suit **deux** trackers —
`device_tracker.nixel_8` (son Pixel, revenu de réparation, état `home`,
batterie 94 %, remontée il y a 2 min) et `device_tracker.samsung_a34`
(téléphone pro, état `not_home`). Home Assistant a retenu le Samsung
comme `source` du person entity, donc Nathan est apparu `not_home`,
`zone.home` est tombé à 0, et l'armement automatique s'est déclenché
5 min plus tard.

⚠️ Correspondance téléphones ↔ personnes (rétablie, mon hypothèse
précédente était inversée) :
- `person.nathan_dondey` → `device_tracker.nixel_8` (« Nixel 8 » = le
  Pixel 8 de Nathan) + `device_tracker.samsung_a34` (pro)
- `person.pauline` → `device_tracker.pixel_8`
Donc `notify.mobile_app_nixel_8` = Nathan, `notify.mobile_app_pixel_8`
= Pauline. La note du 13/08 disant l'inverse était fausse.

Actions :
- `automation.alarme_armement_automatique_au_depart` : condition ajoutée
  — n'arme pas si un téléphone du foyer (nixel_8, pixel_8, samsung_a34)
  est à l'état `home`, même si l'entité `person` dit `not_home`.
- `automation.notifications_relais_pixel_8_vers_samsung_a34` : **désactivée**
  (le Pixel de Nathan est réparé). Conservée, réactivable si besoin.
- **À faire par Nathan (bloqué côté outillage)** : retirer
  `device_tracker.samsung_a34` de `person.nathan_dondey` —
  Paramètres → Personnes → Nathan → retirer l'appareil. Sans ça, le
  téléphone pro continuera de fausser `zone.home` (et donc la clim, le
  robot, l'alarme, tout ce qui dépend de la présence).

### Correction des fiches Personne (23/08, après précision de Nathan)

Pauline a cassé son Pixel 8 et utilise désormais le Samsung-A34 ; le Pixel
de Nathan (« Nixel 8 ») est réparé. Fiches corrigées via l'API websocket
(`person/update` — non exposé en REST) :

| Personne | Avant | Après |
|---|---|---|
| Nathan | nixel_8 + samsung_a34 | **nixel_8** |
| Pauline | pixel_8 (cassé) | **samsung_a34** |

Vérifié juste après : Nathan `home` (source nixel_8), Pauline `home`
(source samsung_a34), `zone.home` = 2. La présence est de nouveau fiable,
donc aussi tout ce qui en dépend (armement/désarmement alarme, clim,
robot aspirateur).

Téléphones actifs et services de notification :
- Nathan → `device_tracker.nixel_8` / `notify.mobile_app_nixel_8`
- Pauline → `device_tracker.samsung_a34` / `notify.mobile_app_samsung_a34`
- `pixel_8` : hors service, ne plus s'y fier.

`automation.notifications_relais_pixel_8_vers_samsung_a34` : **réactivée**,
description mise à jour — elle sert désormais son vrai objectif, faire
suivre à Pauline les notifications que les ~21 automatisations envoient
encore à `notify.mobile_app_pixel_8`.

`automation.alarme_armement_automatique_au_depart` : liste des téléphones
du garde-fou réduite à nixel_8 + samsung_a34 (pixel_8 exclu, pour qu'un
état périmé `home` ne puisse pas bloquer l'armement indéfiniment).

Chantier propre à la rentrée : remplacer `notify.mobile_app_pixel_8` par
`notify.mobile_app_samsung_a34` dans les automatisations et supprimer le
relais — idéalement en passant par le script central `notifier` déjà
prévu dans la liste ci-dessous.

## 2026-09-12 — Vérification arrosage + inventaire des mises à jour

### Arrosage : la routine fonctionne, mais Nathan n'en voyait rien

Les deux automatisations sont actives et se sont déclenchées ce matin
(06h00 et 06h30) ; elles ont sauté leur exécution car le 12/09 est un
jour « impair » — comportement normal, l'arrosage est programmé un jour
sur deux.

Dernier arrosage réel confirmé par l'état des vannes :
- `switch.arrosage_automatique_valve_1` (potager) : `last_changed`
  11/09 06h55 locale → arrosé le 11/09 de 06h30 à 06h55 (25 min).
- `switch.arrosage_automatique_valve_2` (massifs) : `last_changed`
  11/09 06h20 locale → arrosé le 11/09 de 06h00 à 06h20 (20 min).
- Arrosage supplémentaire du potager le 06/09 (jour impair mais
  T°max > 30 °C, la clause canicule a joué).

**Cause de l'impression « ça n'a pas marché » : les deux automatisations
ne notifiaient que `notify.mobile_app_pixel_8`**, c'est-à-dire le Pixel
cassé de Pauline (relayé vers son Samsung). Nathan n'a donc jamais reçu
les confirmations d'arrosage. Corrigé : ajout de
`notify.mobile_app_nixel_8` dans les deux automatisations (sauvegardes
dans `backups/automation-1784990097733-avant-2026-09-12.json` et
`backups/automation-1784990109169-avant-2026-09-12.json`).

⚠️ Rappel : ~21 automatisations notifient encore `mobile_app_pixel_8`.
Le chantier `notifier` centralisé reste la vraie solution.

### Anomalie d'historique à surveiller

Le recorder fonctionne (158 changements du capteur mouvement salon
enregistrés sur 24 h), mais le logbook présente des trous : la requête
sur `switch.arrosage_automatique_valve_1` sur 168 h ne renvoie que les
événements du 06/09, alors que l'état de l'entité prouve un changement
le 11/09 à 04h55 UTC. Même symptôme sur les entités `automation.*`
(`volets_r_1_fermeture_5h`, qui tourne tous les jours, n'a qu'une seule
entrée en 7 jours). À investiguer à tête reposée : purge du recorder,
exclusions dans `configuration.yaml`, ou base corrompue.

### Autres points relevés

- `sensor.arrosage_automatique_battery` = 79 % mais **plus aucune
  remontée depuis le 06/08** (37 jours). À surveiller : pile du boîtier
  GIEX ou remontée Zigbee défaillante.
- **123 entités indisponibles**, dont 51 scènes, 25 boutons et 9 `cover`
  (probablement les reliquats VELUX/KLF200 jamais nettoyés). Ménage
  toujours en attente.

### Mises à jour en attente (13)

| Composant | Actuel | Disponible |
|---|---|---|
| Home Assistant Core | 2026.7.4 | 2026.9.2 |
| Home Assistant OS | 18.1 | 18.2 |
| Zigbee2MQTT | 2.12.1-1 | 2.14.1-1 |
| Mosquitto broker | 7.1.0 | 7.1.1 |
| File editor | 6.0.0 | 6.1.0 |
| Home Assistant MCP Server | 7.14.2 | 8.4.3 |
| Linky | 1.7.0 | 1.8.0 |
| Dahua | 0.9.83 | 0.9.93 |
| MELCloud Home | v2.3.5 | v2.5.0 |
| Alarmo | v1.10.18 | v1.10.19 |
| Rika Firenet | v2.29.38 | v2.29.40 |
| Thermostat 0xc09b9efffeaae32d | 4864 | 5124 |
| Bouton Porte | 33566472 | 33576193 |

Ordre conseillé : sauvegarde complète → add-ons (Mosquitto, Z2M, File
editor) → intégrations HACS → HA Core → HA OS → firmwares Zigbee en
dernier (risque de brique, à faire quand on est sur place).

## 2026-09-12 (soir) — Script de notification centralisé

Fin du bricolage : toutes les notifications passent désormais par un point
d'entrée unique, `script.notifier`.

### Le script

Champs : `titre` (requis), `message` (requis), `extra` (bloc `data` de la
notification mobile : image, priority, ttl…), `cible` (`tous` par défaut,
ou `nathan` / `pauline`). Il envoie aux téléphones actifs du foyer puis
journalise le message via `logbook.log`.

Séquence : `notify.mobile_app_nixel_8` (Nathan) et/ou
`notify.mobile_app_samsung_a34` (Pauline) selon `cible`. **Le jour où un
téléphone change, c'est la seule chose à modifier dans toute
l'installation.**

### Migration

23 automatisations migrées, 24 appels `script.notifier` créés (le Garage
en a deux, une par branche). Vérifié après coup : plus aucun appel
`notify.mobile_app_*` ailleurs que dans le relais désactivé, aucune
configuration illisible, aucune automatisation en état anormal.

Les paires `notify` n'étaient pas toujours adjacentes (anti-chaleur) et
parfois imbriquées dans une boucle (snapshots caméras) ou dans les
branches d'un `choose` (garage) : la migration a été faite par parcours
récursif, en regroupant les appels de même charge utile.

Trois automatisations ne notifiaient **que** le Pixel mort de Pauline —
`Capteurs — Alerte batterie faible`, `Clim Salon — Arrêt si ouverture RDC`
et `Clim Salon — Démarrage automatique si chaud` : Nathan n'en recevait
rien. Elles notifient maintenant les deux téléphones.

### Rustines retirées

- `automation.notifications_relais_pixel_8_vers_samsung_a34` : **désactivée**
  (plus personne n'écrit vers `mobile_app_pixel_8`). Conservée au cas où.
- `automation.archive_notifications` : **désactivée** — elle journalisait
  les appels vers `mobile_app_pixel_8` ; le script fait désormais le
  `logbook.log` lui-même, sans risque de doublon.

### Sauvegardes

Les 43 configurations d'automatisation d'avant migration sont dans
`backups/automations-2026-09-12/` (une par identifiant). Pour revenir en
arrière sur une automatisation :
`curl -X POST -H "Authorization: Bearer $HA_TOKEN" -H "Content-Type: application/json" \
  -d @backups/automations-2026-09-12/<id>.json "$HA_URL/api/config/automation/config/<id>"`

### Vérifié en réel

- Appel direct du script avec image → exécution propre.
- Déclenchement de `automation.portillon_sonnette` → `script.notifier`
  appelé dans la même milliseconde.

## 2026-09-12 (soir) — Ménage des entités fantômes

### Correction de mon propre chiffre

J'avais annoncé « 125 entités indisponibles ». C'était faux : ce compte
mélangeait `unavailable` et `unknown`, or `unknown` est l'état **normal
au repos** des domaines `scene`, `button`, `number`, `select`, `stt` et
`tts` — ils n'ont pas d'état tant qu'on ne s'en sert pas. Les 43 « scènes
Hue cassées » et les 20 « entités MQTT cassées » n'avaient rien de cassé.

Compte réel avant ménage : **33 entités `unavailable`**, dont 26 issues
du KLF200.

### Supprimé

- **Intégration `velux` (VELUX_KLF_C770)**, entry_id
  `01KW4P71CGPSZQGEAC129BRVY7` → 26 entités disparues (13 `button`,
  8 `cover`, 4 `light`, dont les fantômes `cover.combles_volet_*`,
  `cover.dependance_*`, `cover.rdc_store_veranda`, `cover.store_banne`
  et `light.store_banne_lumiere_1..4`). Vérifié avant suppression :
  aucune automatisation ne les référençait.
- **Intégration `zha` (SONOFF Zigbee 3.0 USB Dongle Plus V2)**, entry_id
  `01KW0CQR735YNQHKV4SP63EZEX`, en `not_loaded` — elle ne pouvait pas
  démarrer, le dongle étant utilisé par Zigbee2MQTT. Reliquat d'une
  ancienne configuration.
- 3 tuiles du dashboard Maison (Combles Escalier / Lit / SdB) qui
  pointaient vers les entités supprimées. Sauvegarde de la configuration
  précédente : `backups/dashboard-maison-2026-09-12.json`.

Résultat : **743 → 716 entités**, **33 → 7 `unavailable`**.

### 🔴 Bug majeur trouvé au passage : « Bonne Nuit » n'armait pas l'alarme

`script.bonne_nuit` appelait `cover.open_cover` sur
`cover.rdc_store_veranda` — l'entité VELUX morte depuis le retrait du
KLF200 en juillet. L'appel sur une entité `unavailable` lève une erreur
et **interrompt le script**, si bien que les deux dernières étapes,
`climate.turn_off` et surtout `alarm_control_panel.alarm_arm_night`,
n'étaient jamais exécutées.

C'est l'explication du problème signalé par Pauline fin juillet
(« l'automatisme Google Bonne nuit ne met pas l'alarme »), resté non
diagnostiqué pendant sept semaines.

Corrections :
- entité remplacée par `cover.salle_a_manger_store_veranda` (Overkiz),
  même action `open_cover` — à confirmer par Nathan : l'intention est
  bien de **déployer** le store véranda pour la nuit ?
- `continue_on_error: true` ajouté sur toutes les étapes de pilotage
  (lumières, volets, store, clim) : un appareil injoignable ne peut plus
  empêcher l'armement de l'alarme.
- Sauvegarde de l'ancien script : `backups/scripts-2026-09-12/bonne_nuit.json`
  (les 15 scripts ont été archivés).

### Les 7 entités encore `unavailable` — à regarder, pas à supprimer

| Entité | Lecture |
|---|---|
| `light.jardin_lumiere_banne_1` + ses 3 boutons | La **seule** lumière du store banne récupérée en juillet est de nouveau hors ligne. À vérifier côté TaHoma. |
| `light.veranda_lampadaire_veranda` | Lampe Hue de la véranda injoignable (débranchée ?). |
| `binary_sensor.portillon_alarm_local` | Capteur du portier Dahua. |
| `media_player.freebox_player_pop` | Freebox Player en veille — normal. |

### À investiguer plus tard

**Trois entrées Overkiz** coexistent : deux `ndondey@gmail.com` (chargées)
et une `Passerelle : 2047-0740-8349` en `not_loaded`
(entry_id `01KW0CQHGPV7C7BRYEDW1MHR35`). Probable reliquat d'une tentative
d'API locale. Non touché ce soir, faute de pouvoir vérifier quelles
entités dépendent de quelle entrée sans risque.
