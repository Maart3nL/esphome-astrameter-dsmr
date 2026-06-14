# ESPHome — DSMR P1 → AstraMeter CT002 (Marstek)

ESPHome configuration that reads a Dutch **DSMR 5 (P1)** smart meter and feeds the
signed per-phase grid power into the **[AstraMeter](https://github.com/tomquist/astrameter)
`ct002` component**, emulating a Marstek CT002 grid sensor so Marstek batteries can
do zero-export / self-consumption. Runs on a **Waveshare ESP32-S3-POE-ETH** board.

```
DSMR P1 telegram ──► grid_l1 / grid_l2 / grid_l3  (Watt, signed)
                 ──► ct002:  (Marstek CT002 emulator, over UDP)
```

Sign convention: **+ = consumption / grid import**, **− = production / grid export**.

## Files

| File | Purpose |
|------|---------|
| [`astrameter-ct002-dsmr.yaml`](astrameter-ct002-dsmr.yaml) | Full device: DSMR reader + CT002 emulator + web UI + Tailscale + MQTT |
| [`common/dsmr-p1-reader.yaml`](common/dsmr-p1-reader.yaml) | Shared package: board, Ethernet, P1 UART, DSMR + all meter sensors |
| `secrets.yaml` | Your secrets (gitignored — copy from `secrets.yaml.example`) |

## Hardware & wiring

Board: **Waveshare ESP32-S3-POE-ETH** (ESP32-S3-WROOM-1 N16R8: 16 MB flash, 8 MB octal
PSRAM; W5500 Ethernet over SPI; PoE). PSRAM is **required** by the Tailscale component.

P1 port (RJ12) → board:

| P1 pin | Signal | → board |
|--------|--------|---------|
| 2 | Data Request (RTS) | GPIO17 |
| 3 | Data GND | GND |
| 5 | Data (RxD) | GPIO16 — **10 kΩ pull-up to 3V3** |
| 6 | Power GND | GND |

The P1 data line is open-collector and inverted, so it's inverted in software. Using a
ready-made P1 cable with a built-in inverter? Set `inverted: false` in the package.

## Setup

```bash
cp secrets.yaml.example secrets.yaml   # then fill in the values
esphome run astrameter-ct002-dsmr.yaml
```

Fill in `secrets.yaml`: a strong `web_password`, an `api_encryption_key`
(`openssl rand -base64 32`), an `ota_password` (`openssl rand -hex 16`), your MQTT
broker/credentials, and a Tailscale auth key.

## Features

- **Per-phase + total net grid power**, plus voltages and lifetime energy import/export
  (ready for Home Assistant's Energy dashboard).
- **CT002 emulator** with the full filter / balancer / saturation tuning pipeline,
  exposed as compile-time substitutions at the top of the config.
- **Built-in web UI** (`web_server`) listing all entities, with an OTA firmware-upload form.
- **Tailscale** — the device joins your tailnet as a real node (via
  [esphome-tailscale](https://github.com/Csontikka/esphome-tailscale)), so the web UI is
  reachable remotely with no port-forwarding.
- **MQTT over Tailscale** — connects to Home Assistant's broker on the tailnet; drives
  AstraMeter's `mqtt_insights` (HA discovery of per-battery state + Marstek-app responder).
- **Runtime + settings entities** — CT002 live state and every configured tuning value
  surfaced as (diagnostic) entities.
- **Runs fully standalone** — no Home Assistant required; `reboot_timeout: 0s` on both
  `api:` and `mqtt:` keeps it up if HA / the broker is unreachable.

## Notes & caveats

- The `esphome-tailscale` component is **third-party and experimental**, requires an
  ESP32-S3 with octal PSRAM + esp-idf, and caps tunnel throughput at ~2–5 Mbit/s. After
  first connect, **disable node-key expiry** for the device in the Tailscale admin console.
  Consider pinning its `ref:` to a commit instead of `main` for reproducible builds.
- The `ct002` tuning values are compile-time; changing one requires a re-flash.
- Verified with `esphome compile` (ESPHome 2026.5.x): firmware ≈ 1.15 MB, RAM ≈ 16% static.

## Credits

- [AstraMeter](https://github.com/tomquist/astrameter) by Tom Quist — the `ct002` emulator.
- [esphome-tailscale](https://github.com/Csontikka/esphome-tailscale) by Csontikka — Tailscale on ESP32.
