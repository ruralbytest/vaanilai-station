# Vaanilai Station

**Open hardware + firmware for a decision-oriented agricultural weather network for smallholder farmers**

> *வானிலை* (Vaanilai) — Tamil for "weather"

---

## Status note

This repository holds the specification and is **not actively maintained** by its author. If you want to build on this, **please fork it** — that is the intended path. Open issues and discussions are welcome for community conversation, but pull requests may not be reviewed promptly.

---

## What is this?

Vaanilai Station is an open-source project to build affordable, repairable weather stations that give smallholder farmers **decision-ready information** — not raw data — at a cost an order of magnitude below commercial agricultural weather stations.

A complete village network consists of:

- **One Hub per village** — atmospheric sensing (temperature, humidity, pressure, wind, lightning), LoRa concentrator, 4G uplink, local audible/visual safety alarm
- **One Node per farm** — soil moisture, soil temperature, and local rainfall, reporting over LoRa to the Hub

The Hub runs all derived-metric computation (ET₀, VPD, disease pressure, spray windows, heat index) so useful alerts reach farmers even during cloud outages.

**Reference deployment region:** Tamil Nadu, India — designed to generalize to any tropical smallholder context.

---

## Project status

| Milestone | Status |
|-----------|--------|
| M0 — Spec freeze | **Now open for community review** |
| M1 — Solo prototype | Not started |
| M2 — Hub + 3 Nodes pilot | Not started |
| M3 — v1.0 release | Not started |

We are at **M0**. The primary deliverable right now is [SPEC.md](SPEC.md) — please read it and open issues or discussions with feedback.

---

## Farmer decisions the hardware serves

| # | Decision |
|---|----------|
| D1 | When/whether to irrigate |
| D2 | Safe spray window (pesticide/fungicide) |
| D3 | Fungal disease pressure |
| D4 | Storm preparation |
| D5 | Lightning safety (local siren + app alert) |
| D6 | Sowing timing |
| D7 | Heat stress — crop and worker |

Every sensor in the BOM must map to one of these decisions. See §3 of [SPEC.md](SPEC.md) for the full decision matrix.

---

## Repository layout

```
vaanilai-station/
├── SPEC.md                  ← normative specification (start here)
├── hardware/
│   ├── hub/                 # KiCad schematics, PCB, BOM
│   ├── node/
│   └── mechanical/          # parametric CAD + STEP + 3MF
├── firmware/
│   ├── hub/                 # ESP-IDF / PlatformIO project
│   ├── node/
│   └── shared/              # sensor drivers, LoRa protocol, rules engine
├── calibration/             # provisioning tools + soil curve library
├── backend/                 # reference MQTT ingest + API
├── docs/
│   └── i18n/ta/             # Tamil translations (first-class)
└── validation/              # bench test scripts, season-log analysis notebooks
```

---

## Licences

| Layer | Licence |
|-------|---------|
| Hardware (schematics, PCB, mechanical) | [CERN-OHL-S v2](https://ohwr.org/cern_ohl_s_v2.txt) — strongly reciprocal |
| Firmware | [Apache-2.0](LICENSE-FIRMWARE) |
| Documentation | [CC-BY-SA 4.0](LICENSE-DOCS) |

The hardware licence keeps derivatives open; Apache-2.0 lets the firmware be embedded in commercial farmer-facing apps (e.g. Vayal) without friction; CC-BY-SA keeps docs and translations shared.

---

## Contributing

This project is community-maintained. All spec changes go through a public RFC process; changes to the decision matrix (§3) always require an RFC.

- Read [SPEC.md](SPEC.md) — the design principles in §1.2 are normative and binding
- Open an issue to start a discussion or propose an RFC
- Tamil translations of all documentation are a first-class priority, not an afterthought
- Code of conduct: [Contributor Covenant](https://www.contributor-covenant.org/)

*Maintained by the Rural Bytes Tamil community.*
