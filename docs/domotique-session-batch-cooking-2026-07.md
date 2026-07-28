# État domotique — session « batch-cooking » (juillet 2026)

Synthèse des travaux Home Assistant menés dans la session Claude locale (répertoire `batch-cooking`), via le connecteur MCP Home Assistant. Complète `etat-domotique-2026-07.md` (saga KLF200 → TaHoma). HA : Beelink N150 / Proxmox, Alarmo, Zigbee2MQTT, accès distant Nabu Casa.

## Accès distant
Le connecteur ha-mcp pointait sur l'IP locale `192.168.0.64:8123` → injoignable hors du réseau. Basculé sur l'**URL Nabu Casa** (`https://<id>.ui.nabu.casa`) dans `claude_desktop_config.json` (clé `HOMEASSISTANT_URL`), token longue durée inchangé (valide ~10 ans). Permet le pilotage à distance. Rebasculer sur l'IP locale à la maison si besoin de latence minimale.

## Clavier RFID Frient KEYZB-110 (+ Alarmo)
**Décision d'architecture (à ne pas casser)** : Alarmo reste **SANS code requis** pour ne pas casser les désarmements automatiques (5h30, nounou, ménage, bouton porte) qui appellent `alarm_control_panel.alarm_disarm` sans code. La **validation du code se fait dans l'automation** `automation.clavier_frient_alarmo`, pas dans Alarmo.

- Topic MQTT : `zigbee2mqtt/Clavier Alarme`. PIN saisi → `action_code = "3577"`. Badge RFID → `action_code = "+<UID>"` (UID = partie **après** le `+`).
- Codes : PIN 3577 (Nathan), 2402 (Pauline). Badges UID : `B8DDE1F8`, `B8EB98F8`, `E80003F8`, `C804D5F8` (4 enrôlés, 2 restants).
- **Ajouter un badge** : le tapoter **en armant en même temps** (sinon l'UID ne remonte pas), lire l'UID après le `+`, l'ajouter à la condition de l'automation.
- Mapping : arm_all_zones→away, arm_night_zones→night, arm_day_zones→away (armed_home non activé), disarm→disarm. 2ᵉ automation : retour d'état Alarmo → LED clavier.

## Alarme (Alarmo)
- **Armement auto au départ** (`zone.home`=0 pendant 5 min, 7h-22h) → mode Absence.
- Capteurs ajoutés : ouvertures Fenêtre Cuisine + Véranda (Absent+Nuit), mouvements Couloir + Salon (Absent, individuels).
- **Groupe « Mouvements Intérieurs »** dans Alarmo : capteurs mouvement exposés aux fenêtres (Terrasse/Entrée/Véranda) déclenchent seulement si **2 se confirment en 120 s** → élimine les fausses alarmes solaires. Couloir/Salon (sans fenêtre) restent individuels/instantanés.
- **Sirène Heiman HS2WD-E** : `duration: 300` dans l'action Alarmo + max_duration device 300 s.
- Snapshots caméras (Reolink Terrasse + Foscam Entrée/Véranda + Portillon) envoyés à Nathan+Pauline au déclenchement.

## Caméra Reolink
Intégration Reolink (device « Camera-Terrasse ») → `camera.exterieur_camera_terrasse_fluide` + détection IA (personne/véhicule/animal). Sur le dashboard (section Sécurité).

## Énergie
`ha-linky` (add-on) branché sur **Conso API `conso.boris.sh`** (PAS MyElectricalData — c'est LA cause des échecs « token invalide » : ha-linky utilise la passerelle de bokub, pas MED). Import ~1 an d'historique horaire. Dashboard Énergie + prix 0,1955 €/kWh TTC (Base). Coûts définis dans le bloc `costs` de ha-linky. Electricity Maps (empreinte carbone) ajouté.

## Arrosage GIEX 2 voies (device « Arrosage Automatique »)
- Voie 1 = **potager** (`switch.arrosage_automatique_valve_1`), voie 2 = **massifs fleurs** (`valve_2`, tuyau préperforé).
- Mécanisme : régler `number.arrosage_automatique_timer_X` (minutes) puis activer la vanne → arrose puis **coupure auto** (timer intégré).
- Automations météo-adaptatives (via `weather.get_forecasts` Météo-France) : **Potager** 1j/2 à 6h30 (tous les jours si prévision >30°C), 25 min, skip si pluie ≥4mm. **Massifs** 1j/2 à 6h00, 20 min, skip si pluie. Parité `now().toordinal() % 2`.
- **Piles NiMH 1,2V** (au lieu de 1,5V alcaline attendu) → la jauge **% affiche ~10% en permanence** (artefact de la courbe alcaline), ce n'est PAS une vraie décharge. Fonctionnel validé (vanne s'ouvre/ferme). NiMH lâchent brutalement → surveiller. Ce capteur batterie est hors de l'alerte batterie faible.
- Dashboard : section Arrosage (vannes, curseurs durée, décomptes, batterie).

## Dashboard (dashboard-maison)
Sections : Ouvrants, Sécurité (alarme, mode vacances, caméras), Lumières, Volets (12 volets nommés), Température (thermostat clim salon + T° int/ext), Arrosage. Clim pilotable (`climate.salon_clim_salon_climate`).

## Ménage post-migration KLF200 → TaHoma (fait dans cette session)
Le remplacement KLF→TaHoma (autre session) a rendu orphelins des objets créés ici :
- **Supprimés** : covers template inversés `cover.store_banne_2` + `cover.store_veranda`, groupe lumière `light.store_banne_led` (pointaient sur les entités KLF disparues).
- **Dashboard repointé** vers `cover.jardin_store_banne`, `cover.salle_a_manger_store_veranda`, `light.jardin_lumiere_banne_1` (Overkiz). Lumières store banne 2/3/4 jugées inutiles → seule `banne_1` conservée.
- Les 5 automations stores avaient déjà été migrées vers les covers Overkiz par l'autre session.

## Divers
- **Archivage notifications** : `automation.archive_notifications` logue chaque notif mobile dans le logbook.
- **Bouton garage nocturne** : déclenchements MQTT parasites (rapports batterie) → condition ajoutée pour ne réagir qu'aux vraies actions.
- **Alerte batterie faible** Zigbee (26 capteurs, seuil 20%, Nathan uniquement).

## Reste à faire
- Onduleur Eaton : à déplacer dans la baie de brassage.
- KLF/Velux (septembre) : reset + appairage volets Velux (combles, toit, dépendance — prévenir Somfy/VELUX), réveil des 3 récepteurs lumière, ménage entités Velux orphelines dans HA.
- Badges RFID 5 et 6 à enrôler.
- Durée sirène (test sonore), scénarios CozyTouch (radiateurs Atlantic, Overkiz intégré), 2ᵉ bloc clim Mitsubishi, migration Roborock Fantasio (S5), flash clé Zigbee ezsp→ember (exclusion Defender).
