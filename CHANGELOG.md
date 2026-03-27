# Changelog — welle-monitor

All notable changes to this project will be documented in this file.
This project is a fork of [welle.io](https://github.com/AlbrechtL/welle.io).

---

## [2.7.1-dabmonitor] - 2026-03-27

### Context

This fork was created to address production issues encountered when running
welle-cli as a 24/7 DAB+ monitoring backend for
[DAB+ Monitor](https://github.com/LyonelB/dabplus-monitor) on a Raspberry Pi 4
with an RTL-SDR Blog V4 dongle.

The stock welle-cli v2.7 works well under normal conditions but exhibits
several problems under degraded signal or long uptime that make it unsuitable
for unattended monitoring use.

---

### Fixed

**HTTP freeze on degraded signal (`src/welle-cli/webradiointerface.h`,
`src/welle-cli/webradiointerface.cpp`)**

`std::mutex rx_mut` → `std::timed_mutex rx_mut`.

Under very degraded signal conditions, DSP threads hold `rx_mut` indefinitely.
Any HTTP request to `/mux.json` would then block forever, hanging the HTTP
thread and making the entire web interface unresponsive.

`send_mux_json()` now uses `unique_lock<timed_mutex>` with a 3-second timeout.
If the lock cannot be acquired, a degraded JSON response is returned immediately
instead of freezing:

```json
{"error":"receiver_busy","server_time":1234567890,
 "demodulator":{"snr":0,"frequencycorrection":0},"services":[]}
```

All other usages of `rx_mut` (previously `lock_guard<mutex>` and
`unique_lock<mutex>`) were updated to use the corresponding timed variants.

---

### Added

**`GET /status` — lightweight health endpoint
(`src/welle-cli/webradiointerface.cpp`)**

A new endpoint that never acquires `rx_mut`. It responds even when the
demodulator is completely frozen, making it safe to use in watchdog loops:

```json
{
  "alive": true,
  "server_time": 1234567890,
  "snr": 15,
  "frequencycorrection": -165,
  "fic_crc_errors": 132
}
```

**`POST /reset` — soft decoder restart
(`src/welle-cli/webradiointerface.cpp`)**

Triggers `restart_decoder()` in a detached thread without killing the welle-cli
process. Useful when the RTL-SDR dongle freezes internally. The Icecast audio
stream is not interrupted. Responds immediately:

```json
{"status":"resetting","server_time":1234567890}
```

**`POST /carousel/pin/<sid>` — immediate carousel service selection
(`src/welle-cli/webradiointerface.cpp`)**

Forces the carousel decoder to switch immediately to the requested service
(identified by hex SID, e.g. `0xfa41`) without waiting for the natural
rotation interval. Moves the requested service to the front of
`carousel_services_active` and calls `check_decoders_required()`.

This reduces the audio player startup delay in DAB+ Monitor from ~10s to ~2s
when switching between services.

Returns:
```json
{"status":"pinned","sid":"0xfa41","server_time":1234567890}
```

Returns HTTP 404 if the SID is not found in the current ensemble, HTTP 503 if
`rx_mut` cannot be acquired within 2 seconds.

**`server_time` field in `/mux.json`
(`src/welle-cli/jsonconvert.h`, `src/welle-cli/jsonconvert.cpp`)**

Unix timestamp (seconds) from the server side, added to every `mux.json`
response. Allows monitoring clients to calculate data freshness without
clock synchronisation between client and server.

**`current_carousel_sid` field in `/mux.json`
(`src/welle-cli/jsonconvert.h`, `src/welle-cli/jsonconvert.cpp`)**

When operating in carousel mode (`-C N`), indicates the SID (hex string) of
the service currently being decoded. Empty string when using `-D` (decode all)
or when no service is active.

---

### Tested on

- Raspberry Pi 4 (4 GB RAM) — Debian 13 Trixie — aarch64 — GCC 14.2
- RTL-SDR Blog V4 (Rafael Micro R828D)
- Channel 9A — 202.928 MHz — LA ROCHE SUR YON ensemble (France)
- welle-cli compiled with `-DRTLSDR=1 -DBUILD_WELLE_IO=OFF -DBUILD_WELLE_CLI=ON`

---

### Build

```bash
sudo apt install cmake g++ libfaad-dev libfftw3-dev \
    librtlsdr-dev libmp3lame-dev libmpg123-dev libusb-1.0-0-dev xxd

git clone https://github.com/LyonelB/welle-monitor.git
cd welle-monitor
mkdir build && cd build
cmake .. -DRTLSDR=1 -DBUILD_WELLE_IO=OFF -DBUILD_WELLE_CLI=ON
make -j4
sudo cp welle-cli /usr/bin/welle-cli
```

---

### Upstream

Base: [AlbrechtL/welle.io](https://github.com/AlbrechtL/welle.io) — tag `v2.7`
License: GPL-2.0-or-later
