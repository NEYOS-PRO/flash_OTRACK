# Nexus OS - Version 1.16.0
**Date de release** : 11/03/2026
**Type** : Mineure

## Quoi de neuf dans cette version ?

Cette version apporte deux nouvelles fonctionnalites majeures (detection de chocs
intelligente et numero de serie permanent) et resout plusieurs problemes critiques
de stabilite et de consommation batterie. Le tracker est desormais plus fiable au
redemarrage, plus econome en energie et plus facile a configurer.

## Nouvelles fonctionnalites

- **Detection de chocs par profils** : Au lieu de regler manuellement 6 parametres
  techniques, choisissez simplement un profil adapte a votre usage :
  Pieton (sensible), Voiture (filtre les vibrations), ou Course/Tout-terrain
  (ne detecte que les impacts violents). Configurable via la commande `shock`.

- **Alerte visuelle immediate sur choc** : Quand un choc est detecte, toutes
  les LEDs du tracker clignotent en rouge pendant 5 secondes pour signaler
  visuellement l'evenement, puis reviennent a leur etat normal.

- **Numero de serie permanent** : Chaque tracker peut desormais recevoir un
  numero de serie unique (Product Code + Hardware Revision + Serial Number)
  stocke dans une zone protegee de la memoire flash. Ce numero survit aux
  mises a jour firmware et aux reinitalisations. Configurable via la
  commande `pn`.

## Corrections et ameliorations

- **Stabilite au reveil** : Corrige un probleme ou le processeur pouvait
  executer des instructions corrompues au reveil depuis la veille profonde,
  causant des crashs aleatoires (errata STM32U585).

- **Precision de l'horloge** : L'horloge interne derive beaucoup moins
  (passe de ~13 minutes/jour a moins de 1 minute/jour), ce qui ameliore
  la precision des horodatages GPS et des intervalles d'envoi.

- **Gestion de l'alimentation au redemarrage** : Le tracker confirme
  maintenant son alimentation en quelques microsecondes au lieu de 13ms
  apres un crash. Elimine les cas ou le tracker mourait silencieusement
  au redemarrage sur batterie.

- **Consommation batterie** : Corrige plusieurs fuites de "verrous de veille"
  dans le module GPS qui empechaient le tracker de s'endormir correctement.
  Le systeme de veille utilise maintenant des compteurs empilables, alignes
  sur le fonctionnement natif de Zephyr RTOS.

- **Envoi de donnees et mise en veille** : La mise en veille n'est plus
  perdue si elle est demandee pendant un envoi en cours. L'ordre est mis
  en attente et execute des que l'envoi se termine.

- **Capacite de commandes FOTA** : Passe de 8 a 16 commandes par batch,
  permettant des configurations plus complexes a distance.

- **Commandes shell** : Les parametres on/off acceptent maintenant
  "true"/"false" en plus de "1"/"0".

## Changements techniques

- `b94b2b4` fix(zephyr): corrige stabilite au reveil + precision horloge interne
- `5f8a511` refactor(proc_sleep): remplace verrous on/off par des compteurs empilables
- `c6e2937` fix(gnss): elimine les fuites de verrous veille du module GPS
- `19ba9b0` fix(power): redemarrage fiable apres crash + latch alimentation immediat
- `d03bc71` feat(imu): profils de detection de choc + alerte LED visuelle
- `b077387` fix(net): mise en veille fiable pendant les envois + ameliorations reseau
- `fb525ff` feat(manufacturing): numero de serie permanent sur flash SPI externe

## Statistiques
- Fichiers modifies : 30
- Fichiers nouveaux : 5
- Commits : 7

---

## Historique des versions

| Version | Date | Type | Resume |
|---------|------|------|--------|
| **v1.16.0** | 11/03/2026 | Mineure | Profils de choc, numero de serie permanent, corrections stabilite/veille/batterie |
| v1.15.0 | - | - | Support MGA-DBD/MGA-INI pour ephemerides u-blox M8/M10 |
| v1.14.0 | - | - | Ajout du support du NBIOT |
| v1.13.0 | - | - | Default RAT CATM primary + NTN secondary |
| v1.12.0 | - | - | Fix event double-send + ajout USER_OK event type |
