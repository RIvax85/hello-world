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
