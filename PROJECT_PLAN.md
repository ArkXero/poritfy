# Pocket Spotify Player — Component List & Project Plan (v2)

**Concept:** an iPod-sized, battery-powered Spotify Connect player built on a Raspberry Pi Zero 2 W. It has a small colour screen and five buttons, and it plays over **Bluetooth** to wireless earbuds. You pick music from the Spotify app on your phone; the device plays it and shows what's playing.

_Prices are approximate (USD, Nov 2026) and should be re-checked before buying. "(est.)" means not verified against a live listing._

## 0. Locked decisions

| Decision | Choice |
|---|---|
| Controls | Five buttons only: **previous, next, volume up, volume down, play/pause** (play/pause is one toggle button) |
| Library browsing | None on the device. Music is chosen from the phone (Spotify Connect) |
| Audio output | **Bluetooth earbuds** (no 3.5 mm jack, no DAC) |
| Form factor | iPod-ish or slightly bigger. Target ≈ **110 × 68 × 17 mm** (est.); a classic iPod is ≈ 104 × 62 × 11–14 mm |
| Battery life | **≥ 4 h** |
| Fabrication | Own 3D printer |

**Change from v1:** Bluetooth output removes the need for the Pirate Audio HAT (DAC and headphone amp). It also only has four buttons, and you need five. The design now uses a plain SPI display plus discrete buttons.

---

## 1. Scope

### MVP (must work)
- Shows up as a Spotify Connect device on the user's Premium account and plays through paired Bluetooth earbuds.
- Screen shows track, artist, album art and progress; a hint screen when nothing is connected.
- The five buttons work, with press-and-hold repeat on volume. A long-press on the centre button does a safe shutdown.
- Reconnects to the earbuds automatically after power-on.
- ≥ 4 h on battery, USB charging, on/off switch, printed enclosure.

### Stretch
- On-device Bluetooth pairing screen.
- Battery percentage.
- Rotary-encoder volume knob (more iPod-like).
- On-device playlist picker via the Spotify Web API.

### Hard constraints
- **Spotify Premium is required** for all librespot-based clients.
- **No offline playback.** Open-source Connect clients only stream, so the device needs Wi-Fi: home, or a phone hotspot (**2.4 GHz only**; on iPhone, "Maximize Compatibility").
- **Unofficial.** librespot, spotifyd and Raspotify reimplement the Connect protocol and can break when Spotify changes things. Their own disclaimers say that use may violate Spotify's terms.
- **Wi-Fi and Bluetooth share one radio on the Zero 2 W.** Streaming Spotify over Wi-Fi while sending audio over Bluetooth is the main technical risk (see §5).

---

## 2. Architecture

```
 Phone (Spotify app) ─► Spotify cloud / LAN discovery ─► Pi Zero 2 W (Wi-Fi 2.4 GHz)
                                                        ┌───────────────────────────────────────┐
                                                        │ spotifyd  (Spotify Connect client)    │
                                                        │   ├─ audio ─► ALSA/PipeWire ─► Bluetooth A2DP ─► earbuds
                                                        │   └─ MPRIS (D-Bus): metadata + control │
                                                        │ pocketui.py (Python, systemd service) │
                                                        │   ├─ MPRIS ─► draws on 2.0" ST7789 LCD (SPI)
                                                        │   └─ 5 GPIO buttons ─► MPRIS commands  │
                                                        │ bt-reconnect (systemd): keeps earbuds linked
                                                        └───────────────────────────────────────┘
 LiPo 2500 mAh ─► PowerBoost 1000C (5 V boost + USB charge) ─► Pi micro-USB PWR (or test pads)
 PowerBoost LBO (low-battery) ─► one Pi GPIO ─► safe shutdown
```

**Software stack:** Raspberry Pi OS Lite (32-bit, headless) → `spotifyd` (MPRIS) → PipeWire with its Bluetooth module (or BlueALSA if RAM is tight) → Python UI.

**Volume:** use Spotify's own volume through MPRIS, so it stays in sync with the phone's volume slider, and leave the earbuds' Bluetooth volume near maximum. Confirm that spotifyd exposes volume over MPRIS in Phase 2; fall back to PipeWire's `wpctl set-volume` if not.

---

## 3. Component list (bill of materials)

| # | Part | Purpose | Approx. price | Notes |
|---|------|---------|---------------|-------|
| 1 | Raspberry Pi **Zero 2 W** (no header) | Compute, Wi-Fi, Bluetooth 4.2 | $15–25 | $15 MSRP at launch; 2026 street prices look closer to $20–25. Use the headerless board for a thin build and solder wires to the pads (≈ 15 joints). A WH board adds ~8 mm of header height. |
| 2 | Adafruit **2.0" 320×240 IPS TFT, ST7789** (#4311) | Display | $19.95 | 59.2 × 35.5 × 3.7 mm. SPI, 3.3 V/5 V logic. Has an 18-pin EYESPI connector; check the product page for the cable or pin options when ordering. |
| 3 | 5 × 6 mm tactile switches (or a 5-way nav switch) + small perfboard | Prev / next / vol+ / vol− / play-pause | ~$4 (est.) | Arranged as a cross, iPod style: left = back, right = next, up = vol+, down = vol−, centre = play/pause. |
| 4 | Adafruit **LiPo 3.7 V 2500 mAh** (#328, JST-PH) | Battery | ~$15 (est.) | 50 × 60 × 7.3 mm, 50 g. |
| 5 | Adafruit **PowerBoost 1000C** | 5 V boost (1 A) + USB charging + load sharing | ~$20 (est.) | 23 × 45 × 10 mm. The LiPo must always be connected to absorb load spikes. |
| 6 | microSD, 16–32 GB, A1/A2 (high endurance preferred) | OS | ~$8 (est.) | |
| 7 | SPDT slide switch | Hard power (PowerBoost EN to GND) | ~$1 | |
| 8 | Thin silicone wire (28–30 AWG), heat-shrink, Kapton tape | Wiring, insulation | ~$6 | |
| 9 | Panel-mount micro-USB extension or short cable | Charging port | ~$4 (est.) | |
| 10 | PLA/PETG filament | Enclosure | ~$2 | Own printer. |
| | **Total (approx.)** | | **≈ $95–110** | Plus your own Bluetooth earbuds. |

### Optional / fallback parts
- **USB Bluetooth adapter + micro-USB OTG adapter** (~$10 est.): fallback if onboard Bluetooth stutters (see §5).
- **INA219 current sensor** (~$10): measure real power draw and battery life.
- **MAX17048 fuel gauge** (~$8): battery percentage (stretch).
- **Rotary encoder with push (EC11)**: stretch volume knob.

### Physical envelope (est.)
- **Layout:** display at the top front; the button cross below it; the Pi (65 × 30 mm) behind the display; the battery (50 × 60 mm) behind the button area; PowerBoost beside it.
- **Size:** ≈ 110 × 68 × 17 mm. The display module is 59.2 mm wide, which leaves almost no bezel at a 62 mm width, so about 68 mm is the realistic minimum.
- **Thickness (est.):** about 12–16 mm without a gap. Leave free space around the battery and don't screw into or press on it.

---

## 4. Project plan

Assumes evenings and weekends. About **27–48 hours of work**, and about 1.5–2.5 weeks elapsed (shipping plus printing iterations).

### Phase 0 — Order in two stages (1–2 h + shipping)
- **Stage 1:** buy just the **Pi Zero 2 W**, an SD card and a power supply. Do Phases 1–2 with your earbuds before buying everything else. Bluetooth plus Spotify on a Zero 2 W is the biggest risk, so test it first for about $30.
- **Stage 2 (after the Phase 2 gate passes):** order the display, battery, PowerBoost, switches and wire.

### Phase 1 — Bring-up (1–2 h)
- Flash Raspberry Pi OS Lite (32-bit) with Raspberry Pi Imager. Preset hostname, SSH and home Wi-Fi.
- Add a second Wi-Fi profile for your phone hotspot (2.4 GHz).
- `sudo apt update && sudo apt full-upgrade`. This matters because newer firmware packages carry the Wi-Fi/Bluetooth coexistence fixes.
- **Gate G1:** boots headless, joins Wi-Fi, SSH works, `free -m` and `vcgencmd measure_temp` look sane.

### Phase 2 — Spotify Connect + Bluetooth audio (6–10 h) ← riskiest; do first
- Install the Bluetooth audio stack. Preferred: PipeWire + WirePlumber with the Bluetooth module. Lighter alternative: BlueALSA. If a connection fails with "profile unavailable", check that the A2DP **source** role is enabled.
- Pair the earbuds with `bluetoothctl` (pair, trust, connect). Confirm that a Bluetooth output sink appears.
- **Spike:** install spotifyd (check the current release for ARM binaries and audio backend features); fall back to Raspotify if blocked. Authenticate (Connect discovery or OAuth, whichever the current version supports). Point its output at the Bluetooth sink. Confirm the device appears in the phone's device list and plays to the earbuds. Test with the phone on cellular too.
- Test MPRIS: `playerctl` play/pause/next/previous and volume.
- Write the reconnect service: after boot, or when the link drops, retry `bluetoothctl connect <MAC>` every ~10 s.
- Check the earbud behaviour: they will try to connect to your phone first. Note how to make them connect to the Pi (multipoint, or switch the phone's Bluetooth off).
- **Wi-Fi + Bluetooth coexistence test:** 60 minutes of continuous playback. If dropouts occur, update firmware, then try spotifyd at 160 kbps, then try the USB Bluetooth adapter.
- **Gate G2:** 60 minutes of continuous playback with no audible dropouts, auto-reconnect works after a reboot, RAM > 150 MB free, CPU < 60%, temperature < 70 °C.

### Phase 3 — Display, buttons and UI (8–14 h)
- Wire the display to SPI0 (MOSI, SCLK, CE0, DC, RST, 3.3 V, GND, backlight). Hello-world with a Python ST7789 library and Pillow, on a breadboard.
- Wire the five buttons to GPIOs with pull-ups and debounce them with gpiozero.
- MPRIS client (D-Bus or `playerctl`): track, artist, position, play state, volume. spotifyd needs a D-Bus session bus when run headless, so plan this into the service unit.
- Now-playing screen: album art (fetch the cover URL, cache, resize), scrolling title/artist, progress bar, a small volume overlay, and tiny Wi-Fi and Bluetooth status icons. When nothing is playing, show a hint: "Choose 'Pocket Player' in Spotify."
- Button map:
  - **Left:** previous (the first press restarts the track if it's more than a few seconds in).
  - **Right:** next.
  - **Up / Down:** volume ± (hold to repeat).
  - **Centre:** play/pause; **hold 3 s** = safe shutdown.
- Screen dim after ~20 s idle, wake on any button.
- Package as `pocketui.service` with auto-restart.
- **Gate G3:** boot to a working now-playing screen with no manual steps; all five buttons behave.

### Phase 4 — Power, wiring and enclosure (8–14 h)
- Wire LiPo → PowerBoost → Pi PWR; add the slide switch. Insulate every joint.
- Measure idle and playback current and compute real runtime. Estimate: about 1.3 W average gives roughly 5–6 h on 2500 mAh, above the 4 h target.
- Wire the PowerBoost low-battery (LBO) output to a spare GPIO and trigger a safe shutdown.
- Model the enclosure in parametric CAD (OpenSCAD, CadQuery or Fusion): display window, button holes/caps, charging cutout, switch, battery bay with clearance, standoffs for the Pi and display, snap-fit or screwed lid.
- Print and test-fit in 2–3 iterations. Use PETG for any snap-fit parts.
- **Gate G4:** assembled unit runs from battery, charges while playing, shuts down cleanly on low battery.

### Phase 5 — Hardening and field test (3–6 h)
- Reduce SD wear (volatile journald or log2ram; consider an overlay/read-only root).
- Test hotspot behaviour: join, drop, rejoin, phone on cellular.
- Test full-charge runtime and thermals inside the closed case (`vcgencmd get_throttled`).
- Write a one-page README: pairing, button map, charging, recovery.
- **Gate G5:** a real 2-hour outing with no intervention.

### Phase 6 — Stretch (optional)
- On-device pairing screen (for example, hold left + right for 3 s).
- Fuel gauge for battery %.
- Rotary encoder volume knob.
- Playlist picker via the Spotify Web API (needs a developer app and OAuth).

---

## 5. Risks & mitigations

| Risk | Likelihood / impact | Mitigation |
|------|---------------------|------------|
| **Wi-Fi/Bluetooth coexistence stutter on the Zero 2 W** (shared 2.4 GHz radio; reported dropouts every few seconds on older firmware) | **Medium / High** | Update to current firmware (later images reportedly include the coexistence fix, at some Wi-Fi throughput cost). Lower the stream bitrate to 160 kbps. Fall back to a USB Bluetooth adapter. This is why Phase 2 comes first. |
| Bluetooth setup complexity (A2DP source role, PipeWire vs BlueALSA, sink selection) | Medium / Medium | Spike in Phase 2 before buying the rest. Keep both stacks as options. |
| Earbuds connect to the phone instead of the Pi | High / Medium | Document the switching steps, prefer earbuds with multipoint, and consider the on-device pairing stretch. |
| Unofficial Spotify client breaks or violates terms | Medium / High | Pin known-good versions; keep Raspotify as an alternative; accept the terms risk or choose a different player. |
| No offline playback; needs Wi-Fi | Certain / High for commuting | Set expectation early; use a 2.4 GHz hotspot. |
| 512 MB RAM pressure with PipeWire | Medium / Medium | Headless Lite OS; no desktop or browser; measure in Phase 2; BlueALSA as the lighter option. |
| Boost-converter brownout on load spikes | Medium / Medium | Always connect the LiPo; short, thick power wires; test boot on battery. |
| LiPo safety (short, puncture, pressure, heat) | Low / High | Protected cell; insulate all joints; free space around the battery; don't compress it; no unattended charging. |
| SD corruption from abrupt power loss | Medium / Medium | Safe-shutdown button, low-battery shutdown, high-endurance SD, fewer writes. |
| Thermal throttling in the closed case | Low / Low | Light load; verify with `get_throttled`; add vents if needed. |
| Soldering to Pi Zero pads (≈ 15 joints) | Medium / Low | Use the WH variant with jumper wires if you'd rather not, at the cost of ~8 mm thickness. |
| Price or stock changes | Medium / Low | Check stock; budget ±$15. |

---

## 6. Deliverables
- Working hardware, enclosure CAD and STL files, wiring diagram.
- `pocketui` Python source and systemd units (spotifyd, UI, Bluetooth reconnect).
- A provisioning script or notes that rebuild the SD card.
- One-page user README.

---

## 7. Open items
1. **Earbuds:** which model? Multipoint support helps a lot (§5).
2. **Soldering:** comfortable soldering about 15 joints to Pi Zero pads? If not, use a WH board and accept the extra thickness.
3. **Name:** a working name for the Spotify Connect device (avoid "iPod"; "Pocket Player" is the placeholder).

---

## Sources
- [Adafruit: 2.0" 320×240 IPS TFT, ST7789 (#4311)](https://www.adafruit.com/product/4311)
- [Adafruit: Lithium Ion Polymer Battery 3.7 V 2500 mAh (#328)](https://www.adafruit.com/products/328)
- [Adafruit Learn: PowerBoost 1000C](https://learn.adafruit.com/adafruit-powerboost-1000c-load-share-usb-charge-boost/overview)
- [RPi-Distro firmware-nonfree issue #33 (Wi-Fi/Bluetooth coexistence)](https://github.com/RPi-Distro/firmware-nonfree/issues/33)
- [moOde forum: Zero 2 W Bluetooth choppy audio](https://moodeaudio.org/forum/showthread.php?tid=5607)
- [Raspotify issue #459 (Bluetooth speaker via BlueALSA)](https://github.com/dtcooper/raspotify/issues/459)
- [Raspotify (dtcooper) on GitHub](https://github.com/dtcooper/raspotify)
- [librespot 0.8.0 docs](https://docs.rs/crate/librespot/latest/source/src/main.rs)
- [Raspberry Pi Zero 2 W launch pricing (BetaNews)](https://betanews.com/article/raspberry-pi-zero-2-w)
- [Digi-Key: Raspberry Pi Zero 2 W product highlight](https://www.digikey.com/en/product-highlight/r/raspberry-pi/raspberry-pi-zero-2-w)
