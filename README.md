# Camper-Stereo-Pi4-Snapcast-DAC-EQ

**CPE Kodiak Expedition Rig — Raspberry Pi 4 Audio Server**

A fully configured Raspberry Pi 4 audio server for camper van / expedition vehicle use. Provides multi-source audio via Snapcast (Music Assistant / Home Assistant integration) and AirPlay (shairport-sync), with dual EQ chains, BossDAC hardware output, and Home Assistant automation for amplifier power control.

---

## Hardware

| Component | Details |
|---|---|
| **SBC** | Raspberry Pi 4 |
| **DAC** | BossDAC (PCM5122 chip) — I2S HAT |
| **Amplifier** | Skar Audio SK-M9005D — 5-channel Class D, 85W×4 + 450W×1 mono |
| **Satellite speakers** | 4× Pure Resonance Audio MC2.5B Dual 2.5" Cube (8Ω, 200Hz–20kHz, 60W RMS) |
| **Tactile transducers** | 4× Dayton Audio TT25-8 Bass Shaker Puck (8Ω, 20–80Hz, 15W RMS, Fs=40Hz) |
| **Amp power control** | KinCony KC868-A16v3 MOSFET output16 → amp remote wire (12V switched) |
| **HA Integration** | Music Assistant + Snapcast integration + KC868 MQTT |

---

## Audio Architecture

```
┌─────────────────────────────────────────────────────┐
│                  Raspberry Pi 4                      │
│                                                      │
│  Music Assistant ──→ Snapserver ──→ Snapclient       │
│                                         │            │
│  AirPlay ──→ Shairport-sync             │            │
│                    │                    │            │
│                    ▼                    ▼            │
│              ALSA equal EQ        hw:BossDAC         │
│              (10-band)            (direct)           │
│                    │                    │            │
│                    └──────────┬─────────┘            │
│                               ▼                      │
│                          BossDAC HAT                 │
│                         (PCM5122)                    │
└───────────────────────────────┼─────────────────────┘
                                │ RCA out
                                ▼
                     Skar SK-M9005D Amp
                    ┌──────────────────┐
                    │ CH1+2  CH3+4  CH5│
                    └──┬───────┬────┬──┘
                       │       │    │
                  L pair  R pair  Pucks
                 (2×MC2.5B) (2×MC2.5B) (4×TT25-8)
```

### Signal Path Notes
- **Snapcast path:** Snapclient → `hw:BossDAC` direct (bypasses PipeWire for lowest latency and full headroom)
- **AirPlay path:** Shairport-sync → ALSA default device → ALSA equal EQ plugin → BossDAC
- **PipeWire EQ:** Active as a filter-chain sink (`effect_input.eq10`) — used by PipeWire audio consumers
- Both EQ chains are tuned for the MC2.5B satellite speakers in a small vehicle cabin environment

---

## RCA Signal Splitting

The BossDAC has a single stereo RCA output. The SK-M9005D has three RCA input pairs (CH1-2, CH3-4, CH5). Splitting chain:

```
BossDAC RCA out
       │
  Splitter #1 (1→2)
  ┌────┴────┐
  │         │
Splitter #2  Splitter #3
  ┌──┴──┐    ┌──┴──┐
CH1-2  CH3-4  CH5  (unused)
```

**Required:** 3× RCA 1-to-2 splitter cables total.

---

## Speaker Wiring

### CH1-2 and CH3-4 — MC2.5B Satellite Speakers
Each channel drives 2× MC2.5B in parallel:
- 2× 8Ω in parallel = **4Ω per channel**
- Above the 2Ω minimum load — safe
- 85W per channel at 4Ω

```
CH1+ ──┬── Speaker A +
       └── Speaker B +

CH1- ──┬── Speaker A -
       └── Speaker B -
```

### CH5 — Dayton TT25-8 Tactile Transducers (Mono)
4× pucks at 8Ω each, wired all in parallel:
- 4× 8Ω in parallel = **2Ω mono**
- Hits CH5 minimum rated load exactly
- CH5 delivers 450W — pucks rated 15W RMS — **keep gain conservative**

```
CH5+ ──┬── Puck 1 +
       ├── Puck 2 +
       ├── Puck 3 +
       └── Puck 4 +

CH5- ──┬── Puck 1 -
       ├── Puck 2 -
       ├── Puck 3 -
       └── Puck 4 -
```

**Verify with multimeter before connecting:** should read ~1.6–1.9Ω (nominal 2Ω).

---

## Amp Crossover Settings (SK-M9005D)

### Crossover Pot Behavior — Log Taper
All crossover pots are **logarithmic**, not linear. Equal rotation = equal frequency ratio, not equal Hz. The lower half of rotation covers a narrow Hz range with fine control; the upper half sweeps rapidly.

**CH1-4 HPF pot (20Hz–500Hz) approximate positions:**

| Clock position | Approx frequency |
|---|---|
| Full CCW / 7 o'clock | 20Hz |
| 9 o'clock | 40Hz |
| 11 o'clock | 80Hz |
| 12 o'clock | ~100Hz |
| 1 o'clock | 120–150Hz |
| 2 o'clock | 200–250Hz |
| Full CW / 5 o'clock | 500Hz |

> ⚠️ In the upper quarter of travel, small rotations produce large Hz changes. Adjust slowly.

### Recommended Settings

| Channel | Filter | Setting | Rationale |
|---|---|---|---|
| CH1-2 | HPF | **150Hz (~1 o'clock)** | MC2.5B floor is 200Hz; 12dB/oct slope gives usable output to ~120Hz where pucks still contribute |
| CH3-4 | HPF | **150Hz (~1 o'clock)** | Same as CH1-2; mode switch must be set to HPF |
| CH5 | LPF | **80Hz (~9-10 o'clock on 50–500Hz range)** | Puck usable ceiling; above 80Hz pucks are tactile not acoustic |
| CH5 | Subsonic | **~30Hz** | Protects puck voice coils below Fs of 40Hz |
| CH5 | Bass Boost | **0dB** | Start here; add slowly — pucks clip early |

### Frequency Coverage
```
20Hz ──── 80Hz ──── 150Hz ──────────────── 20kHz
  │         │          │
  Pucks    Overlap   Cubes only
(TT25-8)  (both)   (MC2.5B)
```

The 80–150Hz overlap zone covers kick drum punch and bass guitar body — critical for van acoustics.

---

## Gain Settings

| Channel | Gain pot direction | Rationale |
|---|---|---|
| CH1-4 | Toward 0.2V end (CCW) | Set by ear at comfortable listening level |
| CH5 | Toward 8V end (CW) — nearly full CW | 450W amp vs 15W puck rating; ~1/30th of full power is target |

---

## Software Stack

| Service | Type | Enabled |
|---|---|---|
| `snapserver` | system | ✓ |
| `snapclient` | system | ✓ |
| `shairport-sync` | system | ✓ |
| `pipewire` | user (admin) | ✓ |
| `pipewire-pulse` | user (admin) | ✓ |
| `wireplumber` | user (admin) | ✓ |

### Boot Stability — loginctl linger
User services (PipeWire) must start at boot without a login session:
```bash
sudo loginctl enable-linger admin
systemctl --user enable pipewire pipewire-pulse wireplumber
```

---

## snapclient Service Override

Snapclient runs as a system service but needs access to the user audio session (PipeWire). The override bridges this gap.

**`/etc/systemd/system/snapclient.service.d/override.conf`**
```ini
[Unit]
After=snapserver.service sound.target
Wants=snapserver.service

[Service]
User=admin
Group=admin
Environment=XDG_RUNTIME_DIR=/run/user/1000
ExecStartPre=/bin/sleep 5
ExecStart=
ExecStart=/usr/bin/snapclient --logsink=system --player alsa --soundcard hw:BossDAC tcp://192.168.50.100
Restart=on-failure
RestartSec=5
StartLimitIntervalSec=60
StartLimitBurst=5
```

**Key points:**
- `XDG_RUNTIME_DIR=/run/user/1000` — bridges system service to user audio session
- `ExecStart=` (blank line) — required to clear the inherited base unit command before overriding
- `--soundcard hw:BossDAC` — direct hardware path, bypasses PipeWire for better headroom
- `sleep 5` — allows PipeWire user session to initialize before snapclient opens the audio device

Apply changes:
```bash
sudo systemctl daemon-reload
sudo systemctl restart snapclient
```

---

## BossDAC (PCM5122) ALSA Settings

### Analogue Playback Boost — +6dB Hardware Gain
The PCM5122 has a hardware analog boost stage. Enable it:
```bash
amixer -D hw:BossDAC sset 'Analogue Playback Boost' 1
```

Verify all levels are at maximum:
```bash
amixer -D hw:BossDAC sget 'Digital'    # Should be 207/207 [100%] [0.00dB]
amixer -D hw:BossDAC sget 'Analogue'   # Should be 1/1 [100%] [0.00dB]
```

### Persist ALSA Settings Across Reboots
```bash
sudo alsactl store
```
Settings saved to `/var/lib/alsa/asound.state` and restored automatically by `alsa-restore` service on boot.

---

## EQ Configuration

Two independent EQ chains are active — one per audio path.

### 1. PipeWire Filter-Chain EQ (Snapcast / Music Assistant path)

**`~/.config/pipewire/filter-chain.conf.d/eq10-rig.conf`**

10-band biquad EQ tuned for MC2.5B satellites in a small vehicle cabin. Boosts low end to compensate for cabin acoustics and small driver roll-off; cuts mid-presence hump; restores air in the highs.

```
context.modules = [
    { name = libpipewire-module-filter-chain
        args = {
            node.description = "Equalizer Sink"
            media.name       = "Equalizer Sink"
            filter.graph = {
                nodes = [
                    {
                        type  = builtin
                        name  = eq_band_1
                        label = bq_lowshelf
                        control = { "Freq" = 31.0 "Q" = 1.0 "Gain" = 9.5 }
                    }
                    {
                        type  = builtin
                        name  = eq_band_2
                        label = bq_peaking
                        control = { "Freq" = 63.0 "Q" = 1.0 "Gain" = 6.5 }
                    }
                    {
                        type  = builtin
                        name  = eq_band_3
                        label = bq_peaking
                        control = { "Freq" = 125.0 "Q" = 1.0 "Gain" = 6.0 }
                    }
                    {
                        type  = builtin
                        name  = eq_band_4
                        label = bq_peaking
                        control = { "Freq" = 250.0 "Q" = 1.0 "Gain" = 1.0 }
                    }
                    {
                        type  = builtin
                        name  = eq_band_5
                        label = bq_peaking
                        control = { "Freq" = 500.0 "Q" = 1.0 "Gain" = -4.0 }
                    }
                    {
                        type  = builtin
                        name  = eq_band_6
                        label = bq_peaking
                        control = { "Freq" = 1000.0 "Q" = 1.0 "Gain" = -4.0 }
                    }
                    {
                        type  = builtin
                        name  = eq_band_7
                        label = bq_peaking
                        control = { "Freq" = 2000.0 "Q" = 1.0 "Gain" = 3.0 }
                    }
                    {
                        type  = builtin
                        name  = eq_band_8
                        label = bq_peaking
                        control = { "Freq" = 4000.0 "Q" = 1.0 "Gain" = 2.5 }
                    }
                    {
                        type  = builtin
                        name  = eq_band_9
                        label = bq_peaking
                        control = { "Freq" = 8000.0 "Q" = 1.0 "Gain" = 4.0 }
                    }
                    {
                        type  = builtin
                        name  = eq_band_10
                        label = bq_highshelf
                        control = { "Freq" = 16000.0 "Q" = 1.0 "Gain" = 3.0 }
                    }
                ]
                links = [
                    { output = "eq_band_1:Out"  input = "eq_band_2:In" }
                    { output = "eq_band_2:Out"  input = "eq_band_3:In" }
                    { output = "eq_band_3:Out"  input = "eq_band_4:In" }
                    { output = "eq_band_4:Out"  input = "eq_band_5:In" }
                    { output = "eq_band_5:Out"  input = "eq_band_6:In" }
                    { output = "eq_band_6:Out"  input = "eq_band_7:In" }
                    { output = "eq_band_7:Out"  input = "eq_band_8:In" }
                    { output = "eq_band_8:Out"  input = "eq_band_9:In" }
                    { output = "eq_band_9:Out"  input = "eq_band_10:In" }
                ]
            }
            audio.channels = 2
            audio.position = [ FL FR ]
            capture.props = {
                node.name   = "effect_input.eq10"
                media.class = Audio/Sink
            }
            playback.props = {
                node.name    = "effect_output.eq10"
                node.passive = true
            }
        }
    }
]
```

**EQ curve summary:**

| Band | Freq | Type | Gain | Purpose |
|---|---|---|---|---|
| 1 | 31Hz | Low shelf | +9.5dB | Sub-bass boost for tactile reinforcement |
| 2 | 63Hz | Peaking | +6.5dB | Bass weight / cabin compensation |
| 3 | 125Hz | Peaking | +6.0dB | Upper bass body |
| 4 | 250Hz | Peaking | +1.0dB | Warmth |
| 5 | 500Hz | Peaking | -4.0dB | Cut mid presence hump |
| 6 | 1kHz | Peaking | -4.0dB | Cut nasal mid |
| 7 | 2kHz | Peaking | +3.0dB | Presence / intelligibility |
| 8 | 4kHz | Peaking | +2.5dB | Attack / detail |
| 9 | 8kHz | Peaking | +4.0dB | Air / clarity |
| 10 | 16kHz | High shelf | +3.0dB | Top-end extension |

### 2. ALSA Equal EQ (AirPlay / shairport-sync path)

**`/etc/asound.conf`** — Routes ALSA default device through the `equal` plugin before BossDAC:

```
pcm_type.equal {
    lib "/usr/lib/aarch64-linux-gnu/alsa-lib/libasound_module_pcm_equal.so"
}
ctl_type.equal {
    lib "/usr/lib/aarch64-linux-gnu/alsa-lib/libasound_module_ctl_equal.so"
}
ctl.equal {
    type equal;
}
pcm.eq {
    type equal;
    slave.pcm "plughw:CARD=BossDAC,DEV=0";
}
pcm.!default {
    type plug;
    slave.pcm "eq";
}
```

**Current ALSA equal band values** (0–100 scale, 50 = flat):

| Band | Freq | Value | Approx dB |
|---|---|---|---|
| 00 | 31Hz | 85 | +6dB |
| 01 | 63Hz | 79 | +4dB |
| 02 | 125Hz | 78 | +4dB |
| 03 | 250Hz | 68 | +1dB |
| 04 | 500Hz | 58 | -2dB |
| 05 | 1kHz | 58 | -2dB |
| 06 | 2kHz | 72 | +3dB |
| 07 | 4kHz | 71 | +2.5dB |
| 08 | 8kHz | 74 | +3dB |
| 09 | 16kHz | 72 | +3dB |

Adjust via:
```bash
alsamixer -D equal
```
Persist via:
```bash
sudo alsactl store
```

---

## Home Assistant Integration

### Entities
| Entity | Description |
|---|---|
| `media_player.snapcast` | Active Snapcast player — Music Assistant output |
| `media_player.snapcast_2` | Second Snapcast client (not currently connected) |
| `switch.mosfet_control_board_output16` | KC868 output16 — amp remote wire 12V |
| `media_player.jbl_charge_3_...` | JBL outdoor speaker — separate MA player, independent |

### Amp Power Automation

Amp turns on when Snapcast is selected as Music Assistant output or begins playing. Turns off after idle/paused timeout.

```yaml
alias: Stereo Amp Power Control
description: Powers Skar amp on/off via KC868 output16 based on Snapcast player state
mode: single

triggers:
  - platform: state
    entity_id: media_player.snapcast
    to: playing
    id: amp_on

  - platform: state
    entity_id: media_player.snapcast
    to: idle
    from:
      - unavailable
      - "off"
    id: amp_on

  - platform: state
    entity_id: media_player.snapcast
    to: paused
    for:
      minutes: 5
    id: amp_off

  - platform: state
    entity_id: media_player.snapcast
    to: idle
    for:
      minutes: 10
    id: amp_off

actions:
  - choose:
      - conditions:
          - condition: trigger
            id: amp_on
        sequence:
          - action: switch.turn_on
            target:
              entity_id: switch.mosfet_control_board_output16

      - conditions:
          - condition: trigger
            id: amp_off
        sequence:
          - action: switch.turn_off
            target:
              entity_id: switch.mosfet_control_board_output16
```

### EQ Web UI — Dashboard iframe Card

The Pi EQ web interface (served at port 8080) can be embedded directly into a Home Assistant dashboard for in-cabin EQ adjustment without SSH.

Add this card to any dashboard in YAML edit mode:

```yaml
type: iframe
url: http://192.168.50.214:8080/
aspect_ratio: 100%
title: Music EQ
grid_options:
  columns: 36
  rows: 8
```

**Notes:**
- Pi must be reachable at `192.168.50.214` on the camper network
- The iframe card requires HA to be accessed over HTTP (not HTTPS) or the Pi EQ service must also be served over HTTPS to avoid mixed-content browser blocking
- `columns: 36` spans the full dashboard width; `rows: 8` provides enough height for the EQ controls to be usable
- If the EQ UI doesn't load, verify the web service is running on the Pi: `sudo systemctl status <eq-web-service>`

### KC868-A16v3 MQTT
- Device at `192.168.50.41`
- MQTT broker: Mosquitto on HA host `192.168.50.100`
- MQTT password must match exactly between KC868 web UI and Mosquitto credentials
- All 16 MOSFET entities publish via MQTT autodiscovery

---

## Troubleshooting

### Snapcast player unavailable in HA after reboot
1. SSH to Pi: `sudo systemctl status snapserver snapclient`
2. If snapserver inactive: `sudo systemctl start snapserver` (was disabled — now enabled)
3. If snapclient failed: check `sudo journalctl -u snapclient -n 50 --no-pager`
4. Common cause: PipeWire not ready when snapclient starts — the `sleep 5` in override.conf handles this
5. If HA still shows unavailable: Settings → Devices & Services → Snapcast → Reload

### KC868 MQTT entities unavailable
- Verify KC868 web UI MQTT broker IP = `192.168.50.100`
- Verify MQTT password matches Mosquitto credentials exactly
- Check Mosquitto broker status in HA: Settings → System → Logs

### Low volume
- Check `amixer -D hw:BossDAC sget 'Analogue Playback Boost'` — should be `1`
- Check `amixer -D hw:BossDAC sget 'Digital'` — should be `207/207 [0.00dB]`
- Check `wpctl get-volume @DEFAULT_AUDIO_SINK@` — should be `1.0`
- Amp gain trim pots — CH1-4 toward 0.2V end; set by ear

---

## Quick Reference — Boot Service Check
```bash
# System services
sudo systemctl status snapserver snapclient shairport-sync

# User services
systemctl --user status pipewire pipewire-pulse wireplumber

# Verify ALSA boost persisted
amixer -D hw:BossDAC sget 'Analogue Playback Boost'

# Verify snapclient using correct device
sudo journalctl -u snapclient -n 20 --no-pager | grep Player
# Should show: device: hw:BossDAC
```

---

## Project Context
Part of the **CPE Kodiak Expedition Rig** — a full-featured camper van build with Home Assistant automation, Victron power management, GPS tracking, water monitoring, and multi-zone audio.
