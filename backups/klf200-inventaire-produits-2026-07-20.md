# Inventaire des produits KLF200 — sauvegarde avant reset usine

Date : 2026-07-20
Source : interface web du KLF200 (`klf200.velux/#/products`) + entités Home Assistant.

Objectif : en cas de reset usine du KLF200, ré-enregistrer les produits et les
renommer **exactement à l'identique** pour que les entités Home Assistant
(`cover.*`, `light.*`) soient recréées avec les mêmes identifiants et que les
automatisations continuent de fonctionner.

## Produits enregistrés (15)

| Nom exact dans le KLF200 | Type (Product group) | Zone |
|---|---|---|
| Dependance_Fenetre_Salon | Window operator | Dépendance |
| Dependance_Fenetre_SDB | Window operator | Dépendance |
| Combles_Volet_Escalier | Roller shutter | Combles |
| Combles_Volet_Lit | Roller shutter | Combles |
| Combles_Volet_SDB | Roller shutter | Combles |
| Dependance_Volet_Salon | Roller shutter | Dépendance |
| Dependance_Volet_SDB | Roller shutter | Dépendance |
| Store_Banne_Lumière_1 | Light | Terrasse |
| Store_Banne_Lumière_2 | Light | Terrasse |
| Store_Banne_Lumière_3 | Light | Terrasse |
| Store_Banne_Lumière_4 | Light | Terrasse |
| RDC_Store_Veranda | Horizontal awning | RDC |
| Store_Banne | Horizontal awning | Terrasse |

Noms confirmés à l'identique par l'utilisateur le 2026-07-25, juste avant le
reset usine. Firmware déjà en 0.2.0.0.71.0 (dernière version) : pas de mise à
jour à refaire après le reset.

## Produits à ajouter (objectif de l'opération)

| Nom prévu | Type attendu | Zone |
|---|---|---|
| R+1_Volet_Toit_June | Roller shutter (SSL solaire) | R+1 / toit |
| R+1_Volet_Toit_(2e, nom à définir) | Roller shutter (SSL solaire) | R+1 / toit |

Une entrée fantôme `R+1_Volet_Toit_June` de type `ptype52` (enregistrement
corrompu, Identify sans réponse) était présente : à supprimer, ne pas la
reproduire. Les deux volets ont été validés sains (appairage réussi du premier
coup sur une Somfy Situo 5 io Pure II, canaux 3 et 4).

## Entités Home Assistant correspondantes (constatées le 2026-07-20)

- `cover.dependance_volet_salon` (Dependance_Volet_Salon)
- `cover.combles_volet_escalier` (Combles_Volet_Escalier)
- `cover.store_banne` (Store_Banne)
- `cover.rdc_store_veranda` (RDC_Store_Veranda)
- (les autres produits KLF suivent le même schéma de nommage ;
  les `cover.rts_*` sont des volets Somfy RTS hors KLF200, non concernés)

## Procédure de ré-enregistrement après reset usine

1. Mettre à jour le firmware du KLF200 (0.2.0.0.71.0) AVANT tout
   ré-enregistrement (téléchargement sur velux.com, installation via
   l'interface web).
2. Les commandes de la maison sont toutes monodirectionnelles (KLI 313,
   Situo 5 io Pure II) : la « copie de télécommande » ne fonctionne pas,
   utiliser exclusivement la découverte (« Discover products »).
3. Par zone : ouvrir l'enregistrement de chaque produit (touche engrenage/PROG
   de sa commande jusqu'au va-et-vient — fenêtre de 10 minutes), possible pour
   plusieurs produits à la suite, puis lancer UNE découverte sur le KLF200 :
   elle trouve tous les produits ouverts d'un coup.
4. Renommer immédiatement chaque produit avec le nom exact du tableau
   ci-dessus (attention aux accents : `Lumière`).
5. Volets solaires SSL : faire la découverte en journée, batterie chargée,
   réveiller le volet par une commande juste avant d'ouvrir l'enregistrement.
6. Vérifier dans Home Assistant que les entités réapparaissent avec les mêmes
   identifiants, puis tester les automatisations qui les utilisent.
