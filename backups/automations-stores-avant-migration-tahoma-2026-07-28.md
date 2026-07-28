# Sauvegarde avant migration KLF200 → TaHoma Switch (Overkiz)

Date : 2026-07-28. Le KLF200 est HS (radio morte, reset usine effectué).
Migration vers TaHoma Switch + intégration Overkiz.

## Automatisations impactées (référencent les stores io ex-KLF)

Entités actuelles : `cover.store_banne_2` (« Store Banne ») et
`cover.store_veranda` (« Store Veranda »). À remplacer par les nouvelles
entités Overkiz après appairage TaHoma.

| Automatisation | id | Rôle |
|---|---|---|
| automation.volets_sud_fermeture_anti_chaleur | 1782683929809 | 11h si >26°C : ferme volets sud (RTS), déploie les stores (sauf pluie/vent) |
| automation.volets_sud_reouverture_au_coucher_du_soleil | 1782683935879 | Coucher +30min : rouvre volets sud, rentre les stores |
| automation.store_veranda_remontee_si_pluie | 1782768430336 | Pluie détectée/imminente : rentre les stores |
| automation.store_veranda_remontee_17h | 1782858153914 | 17h : rentre les stores |
| automation.stores_veranda_banne_remontee_si_vent_fort | 1784213352523 | Vent >30 / rafales >40 km/h : rentre les stores |

Les définitions complètes ont été relevées le 2026-07-28 via l'API et sont
reproduites ci-dessous (champs essentiels) pour retour arrière.

### volets_sud_fermeture_anti_chaleur (extrait actions)
- close_cover: [cover.rts_2_shutter, cover.rts_14_shutter]
- if (pas pluie, vent<30, rafales<40): open_cover: [cover.store_banne_2, cover.store_veranda]
- notify pixel_8 + nixel_8 ; input_boolean.volets_sud_fermes_auto → on

### volets_sud_reouverture_au_coucher_du_soleil (extrait actions)
- open_cover: [cover.rts_2_shutter, cover.rts_14_shutter]
- close_cover: [cover.store_banne_2, cover.store_veranda]
- input_boolean.volets_sud_fermes_auto → off

### store_veranda_remontee_si_pluie (extrait)
- conditions: store_veranda open OU store_banne_2 open
- close_cover: [cover.store_veranda, cover.store_banne_2] + notifs

### store_veranda_remontee_17h (extrait)
- conditions: un des deux open ; close_cover les deux

### stores_veranda_banne_remontee_si_vent_fort (extrait)
- triggers: wind_speed>30 ou wind_gust_speed>40 (Météo France Antony)
- close_cover les deux + notifs

## Non impactés (vérifié)
- script.volets_r1_fermer : uniquement cover.rts_8/10/12/14 (pont RTS, hors KLF)
- automation.volets_r_1_fermeture_5h : appelle ce script → OK
- Aucune automatisation ne référence les volets Velux du KLF (combles/dépendance)

## À faire après appairage TaHoma + intégration Overkiz
1. Identifier les nouvelles entités cover des deux stores (+ 4 lumières io).
2. Remplacer cover.store_banne_2 et cover.store_veranda dans les 5
   automatisations ci-dessus.
3. Proposé : condition `input_boolean.mode_vacances = off` sur la branche
   « déploiement des stores » de l'anti-chaleur (ne pas sortir les stores
   sans surveillance pendant les congés).
4. Supprimer/désactiver l'ancienne intégration Velux et ses entités orphelines.
5. Volets de toit R+1 : à ajouter aux automatisations plus tard si souhaité.
