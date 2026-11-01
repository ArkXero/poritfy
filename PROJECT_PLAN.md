# Pocket Spotify Player — Component List & Project Plan

**Concept:** a pocket-sized, battery-powered Spotify Connect player built on a Raspberry Pi Zero 2 W. It has a small on-device screen and buttons for now-playing and transport controls. You pick music from the Spotify app on your phone; the device plays it.

_Prices are approximate (USD, Nov 2026) and should be re-checked before buying. Items marked "(est.)" were not verified against a live listing._

---

## 1. Scope

### MVP (must work)
- Appears as a Spotify Connect device on the user's Premium account and plays audio out of a 3.5 mm jack.
- On-device screen shows track, artist, album art and progress.
- Physical buttons: previous, play/pause, next, volume, and a long-press for safe shutdown.
- Runs ≥ 4 h on battery (target; measured in Phase 4).
- Rechargeable over USB, with an on/off switch, in a 3D-printed enclosure.

### Stretch
- Bluetooth output to wireless earbuds.
- Battery-percentage gauge.
- On-device playlist picker via the Spotify Web API.
- Touchscreen variant.

### Hard constraints
- **Spotify Premium is required** for all librespot-based clients.
- **No offline playback.** Open-source Connect clients stream only. The device needs Wi-Fi: home, or a phone hotspot.
- **Unofficial.** librespot, spotifyd and Raspotify reimplement the Connect protocol. They are not Spotify products, their own disclaimers say use may violate Spotify's terms, and they can break when Spotify changes things.
- **Zero 2 W has 2.4 GHz Wi-Fi only** (and 512 MB RAM). A phone hotspot must offer 2.4 GHz; on iPhone that means "Maximize Compatibility".

---

## 2. Architecture

```
 Phone (Spotify app) ──► Spotify cloud / LAN discovery ──► Pi Zero 2 W
                                                         ┌─────────────────────────────┐
                                                         │ spotifyd (Connect client)   │
                                                         │   ├─ audio ─► ALSA/I²S ─► PCM5102A DAC ─► 3.5 mm
                                                         │   └─ MPRIS (D-Bus) metadata + control
                                                         │ pocketui.py (Python)        │
                                                         │   ├─ reads MPRIS ─► draws 240×240 ST7789 LCD
                                                         │   └─ GPIO buttons ─► MPRIS commands
                                                         └─────────────────────────────┘
 LiPo ─► PowerBoost 1000C (5 V boost + USB charge) ─► Pi micro-USB PWR port
```

**Software stack:** Raspberry Pi OS Lite (32-bit, headless) → ALSA directly (no PipeWire or PulseAudio, to save RAM) → `spotifyd` → custom Python UI as a systemd service.

**Why spotifyd over Raspotify:** it exposes MPRIS over D-Bus, so the on-device UI can read metadata and send play/pause/next without extra API plumbing. Raspotify (librespot packaged for Debian) installs more easily but has no MPRIS, only an `--onevent` hook. Phase 2 includes a spike to confirm spotifyd's current status on Pi OS and fall back to Raspotify if needed.

**Why buttons rather than touch:** Pimoroni's Pirate Audio HAT puts the screen, DAC/headphone amp and four buttons on one board that matches the Zero's footprint. A 1.3" touchscreen is hard to use in a pocket anyway. A touchscreen variant is listed in §3.

---

## 3. Component list (bill of materials)

### Build A — recommended (integrated HAT)

| # | Part | Purpose | Approx. price | Notes |
|---|------|---------|---------------|-------|
| 1 | Raspberry Pi **Zero 2 WH** (pre-soldered header) | Compute, Wi-Fi, Bluetooth | $15–25 | $15 MSRP at launch; 2026 street prices look closer to $20–25 and vary by retailer. Get the **WH** so the HAT plugs straight in. |
| 2 | Pimoroni **Pirate Audio: Headphone Amp** (PIM482) | 1.3" 240×240 IPS LCD, 4 buttons, PCM5102A DAC, 3.5 mm out | ~$25 | Adafruit lists it at $24.95. Don't confuse it with the *3W Stereo Amp* (PIM484) or *Speaker* variants. |
| 3 | microSD, 16–32 GB, A1/A2 (high-endurance preferred) | OS | ~$8 (est.) | Avoids wear from logging. |
| 4 | LiPo 3.7 V, 2000–2500 mAh, **protected**, JST-PH | Battery | ~$12 (est.) | Check physical size (~50×34×8 mm) against the enclosure. |
| 5 | Adafruit **PowerBoost 1000C** | 5 V boost (1 A) + USB charging + load-sharing | ~$20 (est.) | 23×45×10 mm. A LiPo must always be connected to absorb load spikes. |
| 6 | SPDT slide switch | Power on/off (switch the PowerBoost EN pin to GND) | ~$1 | |
| 7 | Short micro-USB cable, or wire to Pi test pads (PP1 = 5 V, PP6 = GND) | PowerBoost → Pi | ~$3 | The 40-pin header is occupied by the HAT. |
| 8 | Hookup wire, heat-shrink, Kapton tape, JST extension | Wiring and insulation | ~$5 | |
| 9 | 3D-printed enclosure | Housing | $0–15 | Print yourself or use a print service. |
| | **Total (approx.)** | | **≈ $95–115** | Plus your own wired headphones. |

### Optional / dev extras
- USB-OTG Ethernet or serial cable (~$8) for debugging if Wi-Fi misbehaves.
- INA219 current-sense breakout (~$10) to measure real power draw and battery life.
- MAX17048 fuel-gauge breakout (~$8) for a battery percentage (stretch; I²C must be tapped from the Pi's underside pads).

### Build B — touchscreen variant (more work)
Replace the Pirate Audio HAT with a 2.8" capacitive SPI touch HAT (e.g. Adafruit PiTFT 2.8" capacitive, ~$35) plus a separate I²S DAC/amp board. You then have to **verify GPIO and I²S pin conflicts yourself**, write a touch UI, and the stack gets thicker. Only choose this if touch is a hard requirement.

### Rough physical envelope (est.)
About 65 × 34 × 25 mm: Pi Zero (65×30), HAT on top, battery and PowerBoost underneath. Final size depends on the battery you pick.

---

## 4. Project plan

Assumes evenings and weekends. About **25–45 hours of work**, and elapsed time is dominated by shipping and printing (≈ 2–3 weeks).

### Phase 0 — Decisions & procurement (1–2 h + shipping)
- Confirm the decisions in §7.
- Order Build A parts (one order from Adafruit or Pimoroni covers most of them) and a few spare cables.
- **Gate:** all parts in hand.

### Phase 1 — Bring-up (1–2 h)
- Flash Raspberry Pi OS Lite (32-bit) with Raspberry Pi Imager; preset hostname, SSH, home Wi-Fi.
- Add a second Wi-Fi profile for your phone hotspot (2.4 GHz).
- Update packages; confirm `ssh` works headlessly.
- **Gate:** boots, joins Wi-Fi, SSH in, `free -m` and `vcgencmd measure_temp` look sane.

### Phase 2 — Audio + Spotify Connect (3–5 h)
- Attach the Pirate Audio HAT; follow Pimoroni's setup (enable SPI, `dtoverlay=hifiberry-dac`, enable the amp GPIO). Confirm with `aplay -l` and a test tone.
- **Spike:** install spotifyd (check the current release for ARM binaries); fall back to Raspotify if blocked. Authenticate (Connect discovery or OAuth, whichever the current version supports). Confirm the device shows in the phone's device list, including from cellular.
- Run as a systemd service; verify it recovers after Wi-Fi drops and reboots.
- **Gate:** 60 minutes of continuous playback, no dropouts, RAM and CPU acceptable (target < 50% CPU, > 150 MB free), temperature < 70 °C.

### Phase 3 — On-device UI (8–15 h)
- Hello-world on the ST7789 (Pimoroni `st7789` Python lib + Pillow).
- Button handling with gpiozero (confirm button GPIOs against Pimoroni's docs; they changed between board revisions).
- MPRIS client (D-Bus or `playerctl`): track, artist, position, play state, volume. Note spotifyd needs a D-Bus session bus when running headless; plan for this in the service unit.
- Now-playing screen: album art (fetch the cover URL, cache, resize), title/artist scrolling, progress bar, state icons (Wi-Fi, playing/paused).
- Button map (proposal): **A** prev · **B** play/pause · **X** next · **Y** short = volume mode, long-press = safe shutdown.
- Screen dim and sleep when idle, to save battery.
- Package as `pocketui.service` with auto-restart.
- **Gate:** boot to playable now-playing screen with no manual steps; all four buttons behave.

### Phase 4 — Power & enclosure (6–10 h)
- Wire LiPo → PowerBoost → Pi PWR; add the slide switch. Insulate everything.
- Measure idle and playback current (INA219 or a USB power meter) and compute real runtime. Compare with the estimate (1–1.5 W average → about 4–6 h on 2000 mAh).
- Wire the PowerBoost low-battery (LBO) output to a free GPIO and trigger a safe shutdown (confirm the pin is free with the HAT fitted; tap it from the Pi's underside pad).
- Model the enclosure in CAD: openings for 3.5 mm jack, charge port, power switch, buttons and LCD window; screw-down lid; battery bay.
- Print, fit-test, iterate.
- **Gate:** assembled unit runs from battery, charges, shuts down cleanly on low battery.

### Phase 5 — Hardening & field test (3–6 h)
- Reduce SD wear (volatile journald or log2ram; consider overlay/read-only root).
- Test hotspot behaviour: join, drop, rejoin, and phone on cellular while the device is on hotspot.
- Test runtime on a full charge. Test thermal behaviour inside the closed case (`vcgencmd get_throttled`).
- Write a one-page README: setup, pairing, button map, charging, recovery.
- **Gate:** one real-world outing (≥ 2 h) with no intervention.

### Phase 6 — Stretch (optional)
- Bluetooth audio to earbuds (adds a Bluetooth audio stack; check RAM).
- Fuel gauge for battery %.
- On-device playlist picker using the Spotify Web API (needs a developer app and OAuth).
- Touchscreen variant (Build B).

---

## 5. Risks & mitigations

| Risk | Likelihood / impact | Mitigation |
|------|---------------------|------------|
| Unofficial client breaks after a Spotify change, or violates terms | Medium / High | Pin known-good versions; keep Raspotify as an alternative; accept the terms risk or choose a different player. |
| No offline playback; needs Wi-Fi | Certain / High for commuting | Set expectation early; use a 2.4 GHz hotspot. If offline matters, a smartwatch is the better device. |
| 512 MB RAM pressure | Medium / Medium | Headless Lite OS; no browser, PipeWire or desktop; measure in Phase 2. |
| Boost-converter brownout on load spikes | Medium / Medium | Always connect the LiPo; use short, thick power wires; test boot under battery. |
| LiPo safety (short, puncture, heat) | Low / High | Protected cell; insulate all joints; ventilated, rigid enclosure; no charging unattended. |
| SD corruption from sudden power loss | Medium / Medium | Safe-shutdown button, low-battery shutdown, high-endurance SD, reduced writes. |
| Thermal throttling in the enclosure | Low / Low | Audio is a light load; verify with `get_throttled`; add vents if needed. |
| HAT consumes the header, leaving few spare GPIOs | Certain / Low | Tap needed signals from the Pi's underside pads; verify pin map before ordering extras. |
| Price or stock changes (Pi Zero 2 W prices vary widely) | Medium / Low | Check retailer stock; budget $95–115 and allow ±$15. |

---

## 6. Deliverables
- Working hardware, enclosure STL/CAD files, wiring diagram.
- `pocketui` Python source and systemd units.
- Provisioning notes, or a script that rebuilds the SD card.
- One-page user README.

---

## 7. Decisions needed from you
1. **Buttons vs touchscreen?** Recommendation: buttons (Build A).
2. **Is on-device library browsing required?** Plain Connect means you choose music on your phone. On-device browsing is a Phase 6 stretch.
3. **Target battery life?** This sets the battery size and enclosure thickness. 4 h is the default.
4. **Wired or Bluetooth headphones?** Wired is MVP; Bluetooth is a stretch.
5. **Do you have a 3D printer?** If not, budget for a print service.

---

## Sources
- [Adafruit: Pimoroni Pirate Audio Headphone Amp (PIM482)](https://www.adafruit.com/product/4454)
- [The MagPi: Pimoroni's Pirate Audio headphone amp](https://magpi.raspberrypi.com/articles/pimoronis-pirate-audio-headphone-amp)
- [Adafruit Learn: PowerBoost 1000C](https://learn.adafruit.com/adafruit-powerboost-1000c-load-share-usb-charge-boost/overview)
- [Raspotify (dtcooper) on GitHub](https://github.com/dtcooper/raspotify)
- [librespot 0.8.0 docs](https://docs.rs/crate/librespot/latest/source/src/main.rs)
- [Raspberry Pi Zero 2 W launch pricing (BetaNews)](https://betanews.com/article/raspberry-pi-zero-2-w)
- [Digi-Key: Raspberry Pi Zero 2 W product highlight](https://www.digikey.com/en/product-highlight/r/raspberry-pi/raspberry-pi-zero-2-w)
