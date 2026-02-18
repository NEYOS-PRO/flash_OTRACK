# NSE - Commandes Gerees

> Nexus Shell Engine - Reference des commandes `--key=value`
> Ce fichier est mis a jour au fur et a mesure de l'ajout de modules.

---

## Format General

```
<commande> [subcommand] <action> [--param1=valeur1] [--param2=valeur2] ...
```

| Action | Description |
|--------|-------------|
| `set`  | Appliquer parametres (validate-all-then-apply) |
| `get`  | Lire config (format copier-collable, reutilisable en set) |
| `get -s` | Lire config en noms courts (ex: `--R=5000` au lieu de `--rate=5000`) |
| `view` | Tableau humanise : Parametre, Valeur, Autorise, Commande, Description |
| `help` | Aide auto-generee avec valeurs et exemples |

### Noms courts (short names)

Chaque parametre possede un **nom long** (ex: `rate`) et un **alias court** (ex: `R`).

| Contexte | Comportement |
|----------|-------------|
| `set` | Accepte les deux : `--rate=5000` ou `--R=5000` |
| `get` | Noms longs par defaut. `get -s` pour noms courts |
| `view` | Affiche `rate(R)` dans la colonne Parametre |
| `help` | Affiche `--rate(R)=<50..86400000>` |
| `config get` | Noms longs par defaut. `config get -s` pour noms courts |

## Codes de Retour

| Code | Nom | Signification |
|------|-----|---------------|
| 0 | FAIL | Erreur hardware/driver (timeout, etc.) |
| 1 | OK | Parametre applique avec succes |
| 2 | UNKNOWN_PARAM | Parametre inconnu (`--rat` au lieu de `--rate`) |
| 3 | INVALID_VALUE | Valeur non numerique (`--rate=abc`) |
| 4 | OUT_OF_RANGE | Valeur hors limites (`--rate=10`, min=50) |
| 5 | BAD_FORMAT | Syntaxe invalide (manque `--`, manque `=`) |

## Atomicite

Toutes les commandes `set` utilisent le pattern **validate-all-then-apply** :
si un seul parametre est invalide, **rien** n'est applique.

## Colonne "Autorise" dans view

La commande `view` affiche une colonne **Autorise** intelligente :
- **Enums** (plage < 16 valeurs) : enumere `0=Label, 1=Label, ...`
- **Numeriques** (grande plage) : affiche `min - max` humanise

La colonne **Commande** montre un exemple copier-collable avec la valeur courante.

## Commande `config`

La commande `config get` affiche **toute la config** de tous les modules NSE enregistres.
Chaque module NSE est automatiquement inclus (via `STRUCT_SECTION_ITERABLE`).

```bash
config get       # Toutes les configs en noms longs
config get -s    # Toutes les configs en noms courts
```

## Persistence

Les configurations sont modifiees **en memoire uniquement** par `set`.
La sauvegarde NVS se fait **au shutdown** (pattern projet : `nvs_delete` + `nvs_common_save_data`).

---

# ========== MODULE : gnss (GNSS Configuration) ==========

**Commande** : `gnss`
**Fichier** : `nexus_shell_engine_gnss.c`
**Status** : Operationnel

### Parametres

| Param | Court | Type | Range | Defaut | Description |
|-------|-------|------|-------|--------|-------------|
| `rate` | `R` | uint32 | 50 - 86400000 ms | 30000 (30s) | Intervalle acquisition GPS |
| `lastpos` | `L` | uint32 | 0 - 1 | 1 | Politique si pas de fix GPS |
| `mode` | `M` | uint32 | 0, 2-10 | 0 | Modele dynamique UBX |

### Detail : --rate (--R)

Intervalle entre deux acquisitions GNSS en millisecondes.

| Exemple | Affichage humanise |
|---------|--------------------|
| 1000 | 1.000s |
| 5000 | 5.000s |
| 60000 | 1m00s |
| 3600000 | 1h00m00s |

### Detail : --lastpos (--L)

Que fait le GNSS quand il n'obtient **pas de fix valide** a un cycle d'acquisition.

| Valeur | Label | Interne | Comportement |
|--------|-------|---------|--------------|
| 0 | Ne pas enregistrer | GNSS_NO_FIX_POLICY_SKIP | Rien n'est ecrit dans le buffer |
| 1 | Repeter derniere pos | GNSS_NO_FIX_POLICY_REPEAT_LAST | Derniere position valide recopiee (tag NO_FIX) |

### Detail : --mode (--M)

Modele dynamique UBX - optimise les filtres Kalman du GPS selon le type de mouvement.
La valeur est la valeur UBX directe (pas de remapping).

| Valeur | Label | Cas d'usage |
|--------|-------|-------------|
| 0 | Portable (general) | Usage polyvalent |
| 2 | Stationnaire | Balise fixe, point de reference |
| 3 | Pieton | Marche, faible vitesse |
| 4 | Vehicule | Route, voiture |
| 5 | Maritime | Bateau |
| 6 | Aerien <1g | Drone, ballon |
| 7 | Aerien <2g | Avion leger |
| 8 | Aerien <4g | Avion rapide |
| 9 | Montre | Poignet |
| 10 | Velo | Cyclisme |

**Note** : Valeur 1 reservee UBX, non utilisee.

### Exemples

```bash
# Configurer tout d'un coup (noms longs)
gnss set --rate=5000 --lastpos=1 --mode=4

# Configurer en noms courts
gnss set --R=5000 --L=1 --M=4

# Changer seulement le rate
gnss set --rate=10000

# Lire la config (noms longs)
gnss get
# -> gnss --rate=5000 --lastpos=1 --mode=4

# Lire la config (noms courts)
gnss get -s
# -> gnss --R=5000 --L=1 --M=4

# Voir le tableau humanise (affiche les deux noms)
gnss view
# +-----------+----------------------+----------------------------------------------+-------------------------------+-------------------------------+
# | Parametre | Valeur               | Autorise                                     | Commande                      | Description                   |
# +-----------+----------------------+----------------------------------------------+-------------------------------+-------------------------------+
# | rate(R)   | 30.000s              | 50ms - 24h00m00s                             | gnss set --rate=30000         | Intervalle acquisition        |
# | lastpos(L)| Repeter derniere pos | 0=Ne pas enregistrer, 1=Repeter derniere pos | gnss set --lastpos=1          | Politique si pas de fix       |
# | mode(M)   | Vehicule             | 0=Portable, 2=Stationnaire, ..., 10=Velo     | gnss set --mode=4             | Modele dynamique UBX (0,2-10) |
# +-----------+----------------------+----------------------------------------------+-------------------------------+-------------------------------+

# Aide complete (affiche les deux noms)
gnss help
```

### OTA (via FOTA)

```
Serveur envoie: "gnss set --rate=1000 --mode=3"
   ou (short): "gnss set --R=1000 --M=3"
Device execute: shell_execute_cmd()
ACK retour: code NSE (0-5)
```

---

# ========== MODULE : modem (Modem Configuration) ==========

**Commande** : `modem`
**Fichier** : `nexus_shell_engine_modem.c`
**Status** : Operationnel
**API interne** : `API_tracker_config_*()`, `API_nexus_net_config_*()`

### Parametres

| Param | Court | Type | Range | Defaut | Description |
|-------|-------|------|-------|--------|-------------|
| `network_1` | `N1` | uint32 | 0 - 2 | 0 (CAT-M) | Reseau primaire |
| `network_2` | `N2` | uint32 | 0 - 2 | 1 (NTN) | Reseau secondaire (fallback) |
| `rate_catm` | `RC` | uint32 | 5000 - 86400000 ms | 60000 (1min) | Intervalle envoi CAT-M |
| `rate_nbiot` | `RB` | uint32 | 5000 - 86400000 ms | 15000 (15s) | Intervalle envoi NB-IoT |
| `rate_ntn` | `RN` | uint32 | 30000 - 86400000 ms | 60000 (1min) | Intervalle envoi NTN satellite |
| `rate_lte` | `RL` | uint32 | 5000 - 86400000 ms | - | Raccourci CATM + NBIOT |
| `first_connexion_catm` | `FC` | uint32 | 60000 - 600000 ms | 240000 (4min) | Timeout 1ere connexion CAT-M |
| `first_connexion_nbiot` | `FB` | uint32 | 60000 - 600000 ms | 60000 (1min) | Timeout 1ere connexion NB-IoT |
| `first_connexion_ntn` | `FN` | uint32 | 180000 - 900000 ms | 300000 (5min) | Timeout 1ere connexion NTN |
| `first_connexion_lte` | `FL` | uint32 | 60000 - 600000 ms | - | Raccourci CATM + NBIOT |
| `max_time_no_connection_catm` | `XC` | uint32 | 60000 - 600000 ms | 70000 (1m10s) | Timeout runtime CAT-M |
| `max_time_no_connection_nbiot` | `XB` | uint32 | 60000 - 600000 ms | 120000 (2min) | Timeout runtime NB-IoT |
| `max_time_no_connection_ntn` | `XN` | uint32 | 180000 - 900000 ms | 180000 (3min) | Timeout runtime NTN |
| `max_time_no_connection_lte` | `XL` | uint32 | 60000 - 600000 ms | - | Raccourci CATM + NBIOT |
| `live_catm` | `LC` | uint32 | 0 - 2 | 0 (ALL) | Mode envoi CAT-M |
| `live_nbiot` | `LB` | uint32 | 0 - 2 | 0 (ALL) | Mode envoi NB-IoT |
| `live_ntn` | `LN` | uint32 | 0 - 2 | 1 (LIVE) | Mode envoi NTN |
| `live_lte` | `LL` | uint32 | 0 - 2 | - | Raccourci CATM + NBIOT |

### Convention noms courts modem

Schema `[Action][RAT]` :

| Prefixe | Action | Suffixe | RAT |
|---------|--------|---------|-----|
| N | Network | 1/2 | Primaire/Secondaire |
| R | Rate | C/B/N/L | CATM/NBIOT/NTN/LTEM |
| F | First connexion | C/B/N/L | CATM/NBIOT/NTN/LTEM |
| X | maX time no connection | C/B/N/L | CATM/NBIOT/NTN/LTEM |
| L | Live/send mode | C/B/N/L | CATM/NBIOT/NTN/LTEM |

### Detail : --network_1 / --network_2 (--N1 / --N2)

Selectionne le type de reseau (RAT = Radio Access Technology).
Les valeurs sont les `rat_type_t` directes du code (pas de remapping).

| Valeur | Enum code | Label | Description |
|--------|-----------|-------|-------------|
| 0 | RAT_CATM | CAT-M | LTE-M terrestre, rapide, faible latence |
| 1 | RAT_NTN | NTN | 5G NTN satellite (non-terrestre) |
| 2 | RAT_NBIOT | NB-IoT | NB-IoT terrestre, latence moyenne |

**ATTENTION - Difference spec vs code** :
La spec serveur avait NB-IoT=1 et NTN=2. Le code fait l'inverse.
On suit le code : **1=NTN, 2=NBIOT**. Mettre a jour le fichier serveur.

### Detail : --rate_catm / --rate_nbiot / --rate_ntn (--RC / --RB / --RN)

Intervalle d'envoi des donnees en millisecondes, par RAT, en mode NORMAL.

### Detail : --rate_lte / --first_connexion_lte / --max_time_no_connection_lte / --live_lte (--RL / --FL / --XL / --LL)

Raccourcis qui appliquent la **meme valeur** a CATM ET NBIOT en une seule commande.

| Action | Comportement |
|--------|-------------|
| `SET` | Applique la valeur aux deux RATs terrestres (CATM + NBIOT) |
| `GET` | Retourne la valeur CATM (source de verite terrestres) |

### Detail : --first_connexion_* (--FC / --FB / --FN / --FL)

Timeout de recherche initiale reseau au boot ou lors d'un changement de RAT.
Si la connexion echoue une fois ce delai depasse, un autre reseau est tente.

### Detail : --max_time_no_connection_* (--XC / --XB / --XN / --XL)

Timeout de recherche reseau en fonctionnement normal (runtime), apres perte de connexion.
Si la connexion n'est pas retablie dans ce delai, le modem bascule sur un autre reseau.

### Detail : --live_* (--LC / --LB / --LN / --LL)

Mode d'envoi des donnees par RAT. Valeurs = `nexus_net_send_mode_t` directes du code.

| Valeur | Enum code | Label | Description |
|--------|-----------|-------|-------------|
| 0 | SEND_MODE_ALL | Toutes les donnees | Envoie tous les elements accumules |
| 1 | SEND_MODE_LIVE | En direct | Envoie seulement le dernier element |
| 2 | SEND_MODE_NONE | Aucune | N'envoie rien (skip) |

**ATTENTION - Difference spec vs code** :
La spec serveur avait 0=En direct(LIVE), 1=Toutes(ALL). Le code fait l'inverse.
On suit le code : **0=ALL, 1=LIVE, 2=NONE**. Mettre a jour le fichier serveur.

### Exemples

```bash
# Config complete (noms longs)
modem set --network_1=0 --network_2=1 --rate_catm=60000 --rate_ntn=600000

# Config complete (noms courts)
modem set --N1=0 --N2=1 --RC=60000 --RN=600000

# Raccourci CATM+NBIOT
modem set --rate_lte=120000
modem set --RL=120000              # equivalent court

# Timeout 1ere connexion
modem set --first_connexion_lte=240000 --first_connexion_ntn=300000
modem set --FL=240000 --FN=300000  # equivalent court

# Mode envoi
modem set --live_lte=0 --live_ntn=1
modem set --LL=0 --LN=1           # equivalent court

# Lire la config (noms longs)
modem get

# Lire la config (noms courts)
modem get -s

# Voir le tableau humanise
modem view
```

### OTA (via FOTA)

```
Serveur envoie: "modem set --N1=0 --N2=1 --RL=120000 --RN=600000 --LL=0 --LN=1"
Device execute: shell_execute_cmd()
ACK retour: code NSE (0-5)
```

---

# ========== MODULE : alert (Button Configuration - 3 boutons) ==========

**Commande** : `alert`
**Sous-commandes** : `B1`, `B2`, `B3`
**Fichier** : `nexus_shell_engine_alert.c`
**Status** : Operationnel
**API interne** : `DO_button_handler_set_*()` / `CK_button_handler_get_*()`
**Persistence** : NVS ID 901, save au shutdown via `API_button_handler_save()`

### Architecture

3 boutons physiques avec detection long-press et publication event configurable.

```
GPIO PE10/PD7/PG4 (bouton vers GND)
    | Zephyr gpio-keys driver (debounce 50ms)
    | Input subsystem (CONFIG_INPUT + CONFIG_GPIO_KEYS)
    v
button_handler.c
    | INPUT_CALLBACK_DEFINE -> DO_button_input_callback
    | press -> k_work_schedule(timing_pressed ms)
    | release -> k_work_cancel
    | work -> check pressed + cooldown -> API_event_publish_simple(event_type)
    v
Event System -> BUTTON_1/BUTTON_2/BUTTON_3 events
```

### Hardware

| Bouton | GPIO | Pin STM32 | Nom physique |
|--------|------|-----------|--------------|
| button_1 | PE10 | GPIOE pin 10 | Bouton carre |
| button_2 | PD7 | GPIOD pin 7 | Bouton croix |
| button_3 | PG4 | GPIOG pin 4 | Bouton triangle |

Configuration : Pull-up interne + Active-low (bouton connecte vers GND).
Note : EXTI4 partage entre PG4 (button_3) et PC4 (stm6601 INT). PC4 INT desactive pour liberer EXTI4.

### Commandes

```bash
# Global (les 3 boutons)
alert view                      # Tableau comparatif des 3 boutons
alert get                       # Get des 3 boutons (3 lignes copier-collables)
alert get -s                    # Get des 3 boutons en noms courts
alert status                    # GPIO pin status + runtime state (debug)

# Par bouton (B1, B2, B3)
alert B1 set --param=val        # Configurer bouton 1
alert B1 get                    # Lire config bouton 1 (copier-collable)
alert B1 get -s                 # Lire config bouton 1 en noms courts
alert B1 view                   # Tableau humanise bouton 1
alert B1 help                   # Aide complete bouton 1
```

### Parametres (par bouton)

| Param | Court | Type | Range | Defaut B1 | Defaut B2 | Defaut B3 | Description |
|-------|-------|------|-------|-----------|-----------|-----------|-------------|
| `event_send` | `ES` | uint32 | 0 - 1 | 0 (Actif) | 0 (Actif) | 0 (Actif) | Activation envoi event |
| `event_type` | `ET` | uint32 | 1 - 31 | 29 (BUTTON_1) | 30 (BUTTON_2) | 31 (BUTTON_3) | Type event a publier |
| `timing_pressed` | `TP` | uint32 | 1000 - 10000 ms | 3000 (3s) | 3000 (3s) | 3000 (3s) | Duree d'appui long-press |
| `timing_between_click` | `TB` | uint32 | 30000 - 7200000 ms | 60000 (1min) | 60000 (1min) | 60000 (1min) | Cooldown entre clics |

### Detail : --event_send (--ES)

Active ou desactive l'envoi d'event quand le bouton est appuye.

| Valeur | Label | Comportement |
|--------|-------|--------------|
| 0 | Actif (envoie event) | Appui long -> publie l'event configure |
| 1 | Desactive | Appui long -> ignore, rien ne se passe |

### Detail : --event_type (--ET)

Type d'event publie lors d'un appui long valide. Chaque bouton peut publier n'importe quel event du systeme.

| Valeur | Event | Cas d'usage |
|--------|-------|-------------|
| 1 | POWER_ON | - |
| 2 | POWER_OFF | - |
| 3 | SHOCK | - |
| 4 | SHOCK_PRIO | - |
| 5 | RESERVED | (ancien BUTTON_1, reserve) |
| 6 | SOS | Alerte urgence |
| 7-28 | (autres events) | Voir event_types.h |
| 29 | **BUTTON_1** | Defaut bouton 1 (carre) |
| 30 | **BUTTON_2** | Defaut bouton 2 (croix) |
| 31 | **BUTTON_3** | Defaut bouton 3 (triangle) |

Exemple : `alert B3 set --event_type=6` ou `alert B3 set --ET=6` -> bouton triangle envoie SOS.

### Detail : --timing_pressed (--TP)

Duree d'appui en millisecondes pour declencher le long-press. Si le bouton est relache avant ce delai, rien ne se passe.

| Exemple | Affichage | Comportement |
|---------|-----------|--------------|
| 1000 | 1.000s | Appui rapide (1 seconde) |
| 3000 | 3.000s | Appui standard (3 secondes, defaut) |
| 5000 | 5.000s | Appui moyen |
| 10000 | 10.000s | Appui tres long (10 secondes) |

### Detail : --timing_between_click (--TB)

Cooldown minimum entre deux envois d'event pour le meme bouton. Empeche le spam.

| Exemple | Affichage | Description |
|---------|-----------|-------------|
| 30000 | 30.000s | Minimum (30 secondes) |
| 60000 | 1m00s | 1 minute (defaut) |
| 300000 | 5m00s | 5 minutes |
| 3600000 | 1h00m00s | 1 heure |
| 7200000 | 2h00m00s | Maximum (2 heures) |

### Exemples

```bash
# Vue globale des 3 boutons (tableau comparatif)
alert view

# Get global (3 lignes copier-collables, noms longs)
alert get
# alert B1 --event_send=0 --event_type=29 --timing_pressed=3000 --timing_between_click=60000
# alert B2 --event_send=0 --event_type=30 --timing_pressed=3000 --timing_between_click=60000
# alert B3 --event_send=0 --event_type=31 --timing_pressed=3000 --timing_between_click=60000

# Get global (3 lignes, noms courts)
alert get -s
# alert B1 --ES=0 --ET=29 --TP=3000 --TB=60000
# alert B2 --ES=0 --ET=30 --TP=3000 --TB=60000
# alert B3 --ES=0 --ET=31 --TP=3000 --TB=60000

# Config bouton 1 : envoyer SOS (noms longs ou courts)
alert B1 set --event_type=6
alert B1 set --ET=6

# Config bouton 3 : desactiver + timing long
alert B3 set --event_send=1 --timing_pressed=5000
alert B3 set --ES=1 --TP=5000

# Config bouton 2 : cooldown 5 minutes
alert B2 set --timing_between_click=300000
alert B2 set --TB=300000

# Debug GPIO
alert status
```

### Fonctionnement bouton

1. **Appui** : `k_work_schedule()` programme un timer de `timing_pressed` ms
2. **Relache avant timing_pressed** : `k_work_cancel()`, rien ne se passe
3. **Timer expire (bouton encore appuye)** :
   - Verifie `event_send == 0` (actif)
   - Verifie cooldown `timing_between_click` respecte
   - Publie `API_event_publish_simple(event_type)`
   - Event visible dans les logs (boite cyan EVENT PUBLISHED)
4. **Cooldown** : Si on reappuie trop vite, log "cooldown Xs remaining"

### NVS

| ID | Contenu | Taille |
|----|---------|--------|
| 901 | `button_config_t` (magic + version + 3x button_single_config_t) | ~36 bytes |

Save au shutdown uniquement (via `PROC_stm6601_shutdown_sequence` -> `API_button_handler_save()`).

### OTA (via FOTA)

```
Serveur envoie: "alert B1 set --ES=0 --ET=6 --TP=3000 --TB=60000"
Device execute: shell_execute_cmd()
ACK retour: code NSE (0-5)
```

---

# ========== MODULE : led (LED Button Status - 3 boutons x 3 etats) ==========

**Commande** : `led`
**Sous-commandes** : `b1_pres`, `b1_send`, `b1_sent`, `b2_pres`, `b2_send`, `b2_sent`, `b3_pres`, `b3_send`, `b3_sent`
**Fichier** : `nexus_shell_engine_led_status.c`
**Status** : Operationnel
**API interne** : `DO_led_button_set_*()` / `CK_led_button_get_*()`

### Architecture

Feedback visuel LED pour chaque bouton, avec 3 etats dans le cycle de vie d'un appui :

```
Bouton appuye                    Event envoye           ACK recu du serveur
    |                                |                       |
    v                                v                       v
[pressed]  ----timeout----->  [sending]  ----ACK----->  [sent]
 LED feedback                  LED feedback              LED feedback
 pendant appui                 en attente                confirmation
```

### Commandes

```bash
# Global (9 modules d'un coup)
led view                                    # Tableau comparatif 3x3
led get                                     # Get 9 lignes (noms longs)
led get -s                                  # Get 9 lignes (noms courts)

# Par bouton+etat (set/get/view/help)
led b1_pres set --led=0 --mode=1 --speed=1 --color=3
led b1_pres get
led b1_pres get -s
led b1_pres view
led b1_pres help

led b2_send set --L=0 --M=2 --S=1 --C=5   # noms courts
led b3_sent set --led=0 --mode=0 --speed=0 --color=1 --timeout=3000
```

### Parametres (pressed/sending : 4 params)

| Param | Court | Type | Range | Description |
|-------|-------|------|-------|-------------|
| `led` | `L` | uint32 | 0 - 6 | Choix LED physique |
| `mode` | `M` | uint32 | 0 - 3 | Mode animation |
| `speed` | `S` | uint32 | 0 - 2 | Vitesse animation |
| `color` | `C` | uint32 | 0 - 9 | Couleur LED |

### Parametres (sent : 5 params, +timeout)

| Param | Court | Type | Range | Description |
|-------|-------|------|-------|-------------|
| `led` | `L` | uint32 | 0 - 6 | Choix LED physique |
| `mode` | `M` | uint32 | 0 - 3 | Mode animation |
| `speed` | `S` | uint32 | 0 - 2 | Vitesse animation |
| `color` | `C` | uint32 | 0 - 9 | Couleur LED |
| `timeout` | `T` | uint32 | 500 - 30000 ms | Duree affichage sent |

### Detail : --led (--L)

| Valeur | Label | Description |
|--------|-------|-------------|
| 0 | LED 1 (Rouge) | LED individuelle rouge |
| 1 | LED 2 (Verte) | LED individuelle verte |
| 2 | LED 3 (Bleue) | LED individuelle bleue |
| 3 | LED RGB | LED RGB (couleur via --color) |
| 4 | LED All | Toutes les LEDs |
| 5 | LED None | Aucune LED (desactive) |
| 6 | LED Status | LED de statut systeme |

### Detail : --mode (--M)

| Valeur | Label | Description |
|--------|-------|-------------|
| 0 | Off | LED eteinte |
| 1 | Solid | LED allumee en continu |
| 2 | Blink | Clignotement |
| 3 | Pulse | Pulsation (fade in/out) |

### Detail : --speed (--S)

| Valeur | Label | Description |
|--------|-------|-------------|
| 0 | Slow | Lent |
| 1 | Medium | Moyen |
| 2 | Fast | Rapide |

### Detail : --color (--C)

| Valeur | Label |
|--------|-------|
| 0 | Off |
| 1 | Red |
| 2 | Green |
| 3 | Blue |
| 4 | Yellow |
| 5 | Cyan |
| 6 | Magenta |
| 7 | White |
| 8 | Orange |
| 9 | Purple |

### Detail : --timeout (--T)

Duree d'affichage de la LED en etat "sent" (confirmation ACK), en millisecondes.
Apres ce delai, la LED revient a son etat normal.

### Exemples

```bash
# Vue globale (tableau 3x3)
led view
# +------------+-------------------+-------------------+-----------+-------------------+-----------+
# | Etat       | led               | mode              | speed     | color             | timeout   |
# +------------+-------------------+-------------------+-----------+-------------------+-----------+
# | B1 pressed | 3 (LED RGB)       | 1 (Solid)         | 0 (Slow)  | 3 (Blue)          | -         |
# | B1 sending | 3 (LED RGB)       | 2 (Blink)         | 1 (Medium)| 3 (Blue)          | -         |
# | B1 sent    | 3 (LED RGB)       | 1 (Solid)         | 0 (Slow)  | 2 (Green)         | 3000      |
# +------------+-------------------+-------------------+-----------+-------------------+-----------+
# | B2 pressed | ...               | ...               | ...       | ...               | ...       |
# ...

# Get global (9 lignes)
led get
# led b1_pres --led=3 --mode=1 --speed=0 --color=3
# led b1_send --led=3 --mode=2 --speed=1 --color=3
# led b1_sent --led=3 --mode=1 --speed=0 --color=2 --timeout=3000
# ...

# Get global (noms courts)
led get -s
# led b1_pres --L=3 --M=1 --S=0 --C=3
# led b1_send --L=3 --M=2 --S=1 --C=3
# led b1_sent --L=3 --M=1 --S=0 --C=2 --T=3000
# ...

# Configurer B1 pressed : LED RGB, solid, bleu
led b1_pres set --led=3 --mode=1 --color=3
led b1_pres set --L=3 --M=1 --C=3  # equivalent court

# Configurer B2 sent : LED RGB, solid, vert, timeout 5s
led b2_sent set --led=3 --mode=1 --color=2 --timeout=5000
```

### OTA (via FOTA)

```
Serveur envoie: "led b1_pres set --L=3 --M=1 --S=0 --C=3"
Device execute: shell_execute_cmd()
ACK retour: code NSE (0-5)
```

---

## Modules Existants - Resume

| Module | Commande | Short Names | Params | Description |
|--------|----------|-------------|--------|-------------|
| GNSS | `gnss` | R, L, M | 3 | rate, lastpos, mode |
| Modem | `modem` | N1/N2, RC/RB/RN/RL, FC/FB/FN/FL, XC/XB/XN/XL, LC/LB/LN/LL | 18 | network, rate, first_connexion, max_time_no_connection, live (par RAT) |
| Alert | `alert B1/B2/B3` | ES, ET, TP, TB | 4 x 3 | event_send, event_type, timing_pressed, timing_between_click |
| LED Status | `led bN_pres/send/sent` | L, M, S, C, T | 4-5 x 9 | led, mode, speed, color, timeout (sent only) |
| Config | `config` | - | - | Aggregateur : affiche tous les modules |

## Reference Short Names Complete

| Module | Long | Court | Description |
|--------|------|-------|-------------|
| gnss | rate | R | Intervalle acquisition |
| gnss | lastpos | L | Politique no-fix |
| gnss | mode | M | Modele dynamique UBX |
| modem | network_1 | N1 | Reseau primaire |
| modem | network_2 | N2 | Reseau secondaire |
| modem | rate_catm | RC | Intervalle CAT-M |
| modem | rate_nbiot | RB | Intervalle NB-IoT |
| modem | rate_ntn | RN | Intervalle NTN |
| modem | rate_lte | RL | Intervalle CATM+NBIOT |
| modem | first_connexion_catm | FC | Timeout 1ere co CAT-M |
| modem | first_connexion_nbiot | FB | Timeout 1ere co NB-IoT |
| modem | first_connexion_ntn | FN | Timeout 1ere co NTN |
| modem | first_connexion_lte | FL | Timeout 1ere co CATM+NBIOT |
| modem | max_time_no_connection_catm | XC | Timeout runtime CAT-M |
| modem | max_time_no_connection_nbiot | XB | Timeout runtime NB-IoT |
| modem | max_time_no_connection_ntn | XN | Timeout runtime NTN |
| modem | max_time_no_connection_lte | XL | Timeout runtime CATM+NBIOT |
| modem | live_catm | LC | Mode envoi CAT-M |
| modem | live_nbiot | LB | Mode envoi NB-IoT |
| modem | live_ntn | LN | Mode envoi NTN |
| modem | live_lte | LL | Mode envoi CATM+NBIOT |
| alert | event_send | ES | Activation envoi |
| alert | event_type | ET | Type event |
| alert | timing_pressed | TP | Duree d'appui |
| alert | timing_between_click | TB | Cooldown entre clics |
| led | led | L | Choix LED |
| led | mode | M | Mode animation |
| led | speed | S | Vitesse animation |
| led | color | C | Couleur LED |
| led | timeout | T | Duree affichage sent |

---

*Derniere mise a jour : gnss + modem + alert (B1/B2/B3) + led (bN_pres/send/sent) + config + short names operationnels*
