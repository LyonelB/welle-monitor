# welle-monitor

Fork de [welle.io](https://github.com/AlbrechtL/welle.io) — améliorations de `welle-cli` pour la surveillance DAB+ en production, développées dans le cadre du projet [DAB+ Monitor](https://github.com/LyonelB/dabplus-monitor).

## Pourquoi ce fork ?

`welle-cli` est un excellent outil de démodulation DAB+, mais son usage en monitoring 24h/24 révèle plusieurs limitations :

- **Freeze HTTP** : quand le signal est très dégradé, `rx_mut` (le mutex du démodulateur) n'est jamais libéré. Les requêtes HTTP sur `/mux.json` bloquent indéfiniment au lieu de répondre avec un état dégradé.
- **Pas d'endpoint léger** : `/mux.json` fait ~20 Ko et acquiert `rx_mut`. Un watchdog ne peut pas vérifier si welle-cli est vivant sans risquer de se bloquer lui aussi.
- **Pas de timestamp serveur** : impossible de savoir si les données reçues sont fraîches sans synchronisation d'horloge entre client et serveur.
- **Pas de reset à chaud** : quand le décodeur se fige, la seule solution est `pkill -9 welle-cli`.

## Modifications apportées

### `src/welle-cli/webradiointerface.h`
- `std::mutex rx_mut` → `std::timed_mutex rx_mut`
- Déclarations de `send_status()` et `handle_reset_post()`

### `src/welle-cli/webradiointerface.cpp`

**Fix freeze HTTP (principal)**  
`send_mux_json()` utilise maintenant `unique_lock<timed_mutex>` avec un timeout de 3 secondes. Si le démodulateur est bloqué, la fonction répond immédiatement avec un JSON minimal dégradé plutôt que de freezer le thread HTTP :
```json
{"error":"receiver_busy","server_time":1234567890,"demodulator":{"snr":0,"frequencycorrection":0},"services":[]}
```

**Nouvel endpoint `GET /status`**  
Endpoint ultra-léger qui n'acquiert **jamais** `rx_mut`. Répond même quand le démodulateur est complètement bloqué. Idéal pour les watchdogs :
```json
{"alive":true,"server_time":1234567890,"snr":15,"frequencycorrection":-173,"fic_crc_errors":110}
```

**Nouvel endpoint `POST /reset`**  
Déclenche un `restart_decoder()` dans un thread détaché sans tuer le processus. Répond immédiatement :
```json
{"status":"resetting","server_time":1234567890}
```

**`server_time` dans `/mux.json`**  
Timestamp Unix (secondes) côté serveur ajouté à chaque réponse. Permet de calculer la fraîcheur des données sans synchronisation d'horloge entre client et serveur.

**`current_carousel_sid` dans `/mux.json`**  
En mode carousel (`-C N`), indique le SID du service en cours de décodage. Permet au client de savoir quel service est actif sans surveiller les `audiolevel.time` de chaque service.

### `src/welle-cli/jsonconvert.h` et `jsonconvert.cpp`
Ajout des champs `server_time` et `current_carousel_sid` dans la struct `MuxJson` et leur sérialisation JSON.

## Compilation

```bash
# Dépendances (Debian/Raspbian)
sudo apt install cmake g++ libfaad-dev libfftw3-dev \
    librtlsdr-dev libmp3lame-dev libmpg123-dev libusb-1.0-0-dev xxd

# Cloner et compiler (welle-cli uniquement, sans Qt)
git clone https://github.com/LyonelB/welle-monitor.git
cd welle-monitor
mkdir build && cd build
cmake .. -DRTLSDR=1 -DBUILD_WELLE_IO=OFF -DBUILD_WELLE_CLI=ON
make -j4

# Tester
./welle-cli -c 9A -g -1 -C 1 -w 7979
curl -s http://localhost:7979/status | python3 -m json.tool
```

## Testé sur

- Raspberry Pi 4 (4 Go) — Debian 13 (Trixie) — aarch64
- RTL-SDR Blog V4 (Rafael Micro R828D)
- Canal 9A — 202.928 MHz — La Roche-sur-Yon, Vendée (France)

## Compatibilité avec DAB+ Monitor

Ces modifications sont conçues pour fonctionner avec [DAB+ Monitor](https://github.com/LyonelB/dabplus-monitor) :

- `/status` remplace `curl --max-time` dans le watchdog Python → plus fiable
- `server_time` permet de détecter les données périmées en mode carousel
- `current_carousel_sid` permet d'afficher le service actif dans le dashboard
- `POST /reset` permet un redémarrage doux sans `pkill -9`

## Licence

GPL-2.0-or-later — identique au projet original [welle.io](https://github.com/AlbrechtL/welle.io).

## Crédits

- welle.io : Albrecht Lohofener & Matthias P. Braendli
- Modifications DAB+ Monitor : Lyonel B. — [Graffiti Radio](https://graffitiradio.fr), La Roche-sur-Yon

## Issue upstream

Ces modifications ont été signalées aux mainteneurs de welle.io :
https://github.com/AlbrechtL/welle.io/issues/911
