# Vaanilai Station — Open Source Village Weather Network

**An open hardware + firmware specification for a decision-oriented agricultural weather system for smallholder farmers**

| | |
|---|---|
| **Spec version** | 0.1.0 (Draft for community review) |
| **Status** | DRAFT |
| **Hardware license** | CERN-OHL-S v2 (strongly reciprocal) — proposed |
| **Firmware license** | Apache-2.0 — proposed |
| **Documentation license** | CC-BY-SA 4.0 — proposed |
| **Target region (reference deployment)** | Tamil Nadu, India (design generalizes to tropical smallholder contexts) |

The key words MUST, MUST NOT, SHOULD, SHOULD NOT, and MAY in this document are to be interpreted as described in RFC 2119.

---

## 1. Mission and Design Philosophy

### 1.1 Mission

Give smallholder farmers locally-sensed, decision-ready agricultural weather information — irrigation timing, spray windows, disease risk, storm and lightning safety — at a per-farmer cost an order of magnitude below commercial agricultural weather stations, using a design anyone can build, repair, and extend.

### 1.2 Design principles (normative)

These principles are binding on all design decisions. Pull requests that violate them require an explicit spec amendment.

1. **Decisions, not data.** Every sensor and every derived metric MUST map to a named farmer decision (see §3). Hardware that does not feed a decision MUST NOT be added to the reference BOM.
2. **Sensor density follows spatial variability.** Parameters that vary regionally (pressure, lightning, wind, air temperature) are measured once per village at the **Hub**. Parameters that vary plot-to-plot (soil moisture, soil temperature, local rainfall) are measured at cheap per-field **Nodes**. This is the core architectural decision of the project.
3. **Compute, don't sense.** If a parameter can be derived from existing measurements with acceptable accuracy (dew point, VPD, ET0, leaf-wetness proxy), the project derives it in firmware rather than adding hardware.
4. **Safety functions are local-first.** Any safety alert (lightning) MUST be raised by the device itself (sound/light) without depending on cellular connectivity, cloud round-trips, or a phone.
5. **False alarms are safety failures.** For safety features, the false-alarm rate is a tracked quality metric, because alarm fatigue destroys trust and therefore protection.
6. **Repairable in a village.** Prefer through-hole or module-level components, 3D-printable mechanical parts, and standard fasteners. Every mechanical part MUST ship as a printable model (STEP + 3MF/STL) in the repo.
7. **Calibration is data, not code.** Per-unit calibration values (lightning antenna tuning, rain gauge tip volume, soil curves, vane north offset) MUST be stored in device NVS and exportable, never hard-coded.

---

## 2. System Overview

```
┌────────────── Village ──────────────┐
│                                     │
│  Field Node 1 ──┐                   │
│  (soil, rain)   │  LoRa (433/865 MHz)
│                 ├──► Village Hub ───┼──► 4G/LTE ──► Backend ──► Farmer app
│  Field Node 2 ──┤   (air T/RH,     │              (MQTT/HTTP)   (e.g. Vayal)
│  (soil, rain)   │    pressure,     │
│                 │    wind,         │
│  Field Node N ──┘    lightning,    │
│                      local alarm)  │
└─────────────────────────────────────┘
```

Two device classes:

- **Vaanilai Hub** — one per village/cluster. Mains-independent (solar + LiFePO4). Carries atmospheric sensors, the lightning detector, the local audible/visual alarm, the LoRa concentrator, and the single cellular uplink. Runs all derived-metric computation.
- **Vaanilai Node** — one or more per farm. Ultra-low-cost, solar-trickle or long-life battery, deep-sleeps between samples, reports to the Hub over LoRa. Carries soil and local-rain sensing.

A **single-box "Solo" build profile** (Hub sensors + one soil/rain set in one enclosure) MUST remain buildable from the same firmware tree for prototyping and single-farm deployments, selected by build flag.

---

## 3. Decision Matrix (the requirements source of truth)

| # | Farmer decision | Inputs (sensed) | Inputs (derived) | Output to farmer |
|---|---|---|---|---|
| D1 | When/whether to irrigate | Soil moisture, soil temp, rainfall | ET0, VPD | "Irrigate in N days" / "Skip — rain expected & soil wet" |
| D2 | Safe spray window | Wind speed + gusts, RH, rainfall | — | "Good spray window 06:00–09:00" / "Do not spray: wind/gusts" |
| D3 | Fungal disease pressure | Air T, RH | Leaf-wetness-duration proxy, dew point | "High disease risk — scout field" |
| D4 | Storm preparation | Pressure trend, rainfall | — | "Pressure falling fast — secure harvest/equipment" |
| D5 | Lightning safety | AS3935 events | Distance tier, 30-min latch | Local siren/strobe + app push: "Take shelter" |
| D6 | Sowing timing | Soil temperature | — | "Soil ready for [crop] sowing" |
| D7 | Heat stress (crop & worker) | Air T, RH | Heat index | "Extreme heat 12:00–15:00 — avoid field work" |

Anything proposed for the BOM or the API MUST cite a row in this table.

---

## 4. Hardware Specification

### 4.1 Hub — sensing requirements

| Param | Sensor (reference) | Requirements |
|---|---|---|
| Air temp + RH | **SHT41/SHT45** (I2C) | MUST be mounted in a passive multi-plate radiation shield (§4.4). MUST be on a stub/cable ≥10 cm from main PCB to avoid self-heating. Firmware SHOULD pulse the on-chip heater when RH ≥ 99% persists (monsoon condensation recovery). DHT-series sensors MUST NOT be used (drift). |
| Barometric pressure | **BMP390** or BME280 pressure channel | Vented to ambient via hydrophobic membrane (PTFE). Trend is the product; absolute calibration NOT required. |
| Wind speed | Cup anemometer, pulse output | Sampled in ≥3 s bursts; both mean and **gust max** MUST be reported. Bearing class is a named reliability concern; mounts MUST be field-replaceable without soldering. |
| Wind direction | Vane (resistive ladder → ADC) | North offset captured at install, stored in NVS. |
| Lightning | **AS3935** (DFRobot Gravity or equivalent) | Full requirements in §6. |
| Local alarm | Piezo siren ≥ 95 dB @ 1 m + high-vis strobe | MUST operate from battery rail independent of modem state. |

Measurement height: T/RH at 1.25–2 m; wind as high and unobstructed as the mast allows (document actual height in device metadata — we do not claim WMO 10 m compliance).

### 4.2 Node — sensing requirements

| Param | Sensor (reference) | Requirements |
|---|---|---|
| Soil moisture | Capacitive probe (1–2 depths: 15 cm, optional 30 cm) | Resistive probes MUST NOT be used. Probe-to-cable junction MUST be potted/sealed. Readings MUST be temperature-compensated using soil temp. Per-soil-type calibration per §8.3; uncalibrated devices MUST report "relative" units, never % VWC. |
| Soil temp | **DS18B20** waterproof (1-Wire) | Installed at root depth (~15 cm), co-located with moisture probe. |
| Rainfall | Tipping bucket + reed switch | Interrupt-driven (wakes MCU per tip). Debounce ≥ 5 ms in firmware. Funnel MUST have an insect/debris mesh. Per-unit tip volume calibrated per §8.2. Firmware SHOULD apply a high-intensity undercount correction above ~50 mm/h. |

### 4.3 Compute, power, comms

| Subsystem | Hub | Node |
|---|---|---|
| MCU | ESP32-S3 (reference) | ESP32-C3 or C6 (cost) |
| Uplink | 4G Cat-1/Cat-M module (region-appropriate) | LoRa to Hub (865–867 MHz in India, build-time region config) |
| Power | ≥10 W solar + **LiFePO4** pack | Small solar trickle or ≥6-month primary battery |
| Battery chemistry | LiFePO4 MANDATORY for any pack in a sealed outdoor enclosure in tropical heat. Standard Li-ion/LiPo MUST NOT be used. | Same rule if rechargeable. |
| Modem power | Bulk capacitance sized for ≥2 A TX bursts; modem inrush MUST NOT brown out the MCU (separate regulation or sequencing). | n/a |

**Power-state requirements:**
- Node MUST deep-sleep between samples; wake sources: timer, rain-tip interrupt.
- Hub MAY light-sleep but the AS3935 MUST remain powered and listening at all times; its IRQ MUST be a wake source.
- Hub MUST implement modem duty-cycling (power the modem only for uplink windows) with one exception: a confirmed close-range lightning event MAY trigger an immediate out-of-cycle uplink, but only after the event has been read, latched, and the local alarm started.

### 4.4 Mechanical (all parts open, printable)

- **Radiation shield:** stacked-plate passive shield, ≥6 plates, white PETG or ASA (PLA MUST NOT be used for sun-exposed structural parts — glass transition too low for tropical sun). This part is accuracy-critical: an unshielded sensor reads 5–15 °C high in direct sun and corrupts RH with it.
- **Enclosures:** IP65 target. Printed enclosures MUST use printed gasket channels + TPU or silicone gaskets, conformal-coated PCBs, and membrane vents (pressure equalization prevents condensation pumping).
- **Mast/mounting:** standard 25 mm galvanized pipe interfaces; all printed clamps sized to it.
- Every mechanical part ships as parametric source (FreeCAD/OpenSCAD/Fusion export) + STEP + 3MF.

---

## 5. Derived Metrics (firmware-computed, no extra hardware)

These MUST be computed on the Hub and exposed in the API as first-class parameters:

| Metric | Method | Feeds |
|---|---|---|
| Dew point | Magnus formula from T, RH | D3 |
| VPD | Saturation vapor pressure (T) × (1 − RH/100) | D1 |
| ET0 (reference evapotranspiration) | **Hargreaves-Samani** from Tmin/Tmax/Tmean + latitude + DOY (no pyranometer required) | D1 |
| Leaf-wetness duration (proxy) | Accumulated hours where RH ≥ 90% or T ≤ dew point + 1 °C | D3 |
| Heat index | NOAA regression from T, RH | D7 |
| Pressure trend | 3 h rolling slope of station pressure | D4 |
| Spray window score | Rule combination: wind mean < 15 km/h AND gust < 20 km/h AND no rain in trailing 1 h AND RH in 40–90% | D2 |

A true pyranometer (Penman-Monteith ET) and a dedicated leaf-wetness sensor are explicitly **out of scope for v1** and tracked as optional expansion modules (§12).

---

## 6. Lightning Subsystem (safety-critical; normative)

The AS3935 has one root problem — unwanted ~500 kHz energy reaching its analog front end — which causes both false alarms and desensitization. Mitigation is layered; lower layers MUST be implemented before tuning registers.

### 6.1 Layer 0 — Per-unit antenna tuning (MANDATORY)
- LC tank MUST be calibrated to 500 kHz ± 1% per unit at provisioning: enable `DISP_LCO`, measure the divided resonance on IRQ with the ESP32 PCNT peripheral, sweep `TUN_CAP` (0–15), select nearest to 500 kHz, persist to NVS.
- A device with no stored tuning value MUST refuse to report lightning distance (events MAY still be logged as untrusted).

### 6.2 Layer 1 — Power integrity
- AS3935 MUST be powered from a dedicated LDO, not the switching rail feeding the modem. Local decoupling (100 nF + ≥1 µF) at VDD. Single-point ground; modem return currents MUST NOT share the sensor ground path.

### 6.3 Layer 2 — Physical layout
- AS3935 and cellular module at opposite ends of the enclosure; AS3935 antenna kept clear of inductors/SMPS; loop antenna plane oriented for minimum coupling to the cellular antenna. This is a board/enclosure layout requirement from rev A.

### 6.4 Layer 3 — TX blanking
- Firmware MUST discard AS3935 events occurring inside known modem TX windows and clear latched disturber state after each TX burst.
- Exception ordering for event-triggered uplink: read event → latch → start local alarm → only then power modem.

### 6.5 Register baseline (tune from field data, not defaults)
- Outdoor gain (`AFE_GB = 0x0E`); watchdog ≈ 2; spike rejection ≈ 2; noise floor mid-range; `MIN_NUM_LIGH = 1` (first credible strike alarms; corroboration is firmware's job, not the chip's).
- Frequent noise-high interrupts MUST be treated as a hardware defect signal (revisit Layers 1–2), not solved by raising the noise floor.

### 6.6 Alarm behavior
| Distance | Action |
|---|---|
| > 25 km | Log + app informational |
| 10–25 km | App warning: storm approaching |
| ≤ 10 km | **Local siren + strobe immediately**, app push, immediate uplink |
- Danger state MUST latch for 30 minutes after the last ≤10 km strike (standard lightning-safety rule; prevents alarm flicker).
- The local alarm MUST fire on local detection alone. Cloud lightning feeds and pressure-trend corroboration are for offline tuning and false-alarm analysis only and MUST NOT gate the live alarm.

### 6.7 Interrupt handling
On IRQ: wait ≥ 2 ms → read interrupt source → branch: noise-high → health metric; disturber → counter only (disturber *rate* is the live noise-health signal), never alarm; lightning → read distance + energy, apply §6.6.

### 6.8 Validation requirements
- Bench: piezo spark-igniter stimulus detection ≥ 90%; zero false lightning events across 100 modem TX cycles.
- Field: one full storm season of complete event logs compared against a reference lightning dataset; release builds MUST publish hit/false/miss rates. Threshold changes MUST cite this data.

---

## 7. Firmware Specification

### 7.1 Sampling cadence (defaults, remotely configurable)

| Parameter | Cadence |
|---|---|
| Air T/RH, pressure | 10 min |
| Wind | 3 s burst every 2 min; report 10-min mean + gust max |
| Soil moisture/temp | 30 min |
| Rain | Interrupt per tip + hourly/daily totals |
| Lightning | Continuous (interrupt) |
| Hub→cloud uplink | 15 min batch; immediate on lightning ≤ 10 km |
| Node→Hub LoRa | Per sample, with ACK + retry |

### 7.2 Behavioral requirements
- Store-and-forward: Hub MUST buffer ≥ 7 days of all telemetry in flash and backfill on reconnect. Cellular outage MUST NOT lose data.
- All decision logic (D1–D7) SHOULD run on the Hub so alerts degrade gracefully without cloud.
- Alert rules SHOULD be expressed in a declarative rules file (YAML) shipped with firmware and remotely updatable — thresholds are agronomy, not code.
- OTA updates MUST be supported (signed images, A/B partitions); a failed OTA MUST roll back.
- Time: NTP via cellular, RTC fallback; all telemetry timestamped UTC with timezone metadata.
- Watchdog + brownout detection mandatory; unexpected resets are telemetry.

### 7.3 Data model (wire format sketch)

```json
{
  "device_id": "vh-tn-0042",
  "fw": "1.2.0",
  "ts": "2026-06-11T06:30:00Z",
  "battery_mv": 13280,
  "obs": {
    "t_air_c": 31.4, "rh_pct": 72.1, "p_hpa": 1004.2,
    "wind_ms": 2.1, "gust_ms": 4.8, "wind_dir_deg": 210,
    "rain_mm_1h": 0.0, "rain_mm_24h": 12.5
  },
  "derived": {
    "dew_c": 25.8, "vpd_kpa": 1.29, "et0_mm_day": 5.6,
    "lwd_h_24h": 9.5, "heat_index_c": 38.0, "p_trend_hpa_3h": -1.8
  },
  "events": [
    {"type": "lightning", "dist_km": 14, "energy_rel": 21043, "ts": "..."}
  ],
  "health": {"disturber_rate_1h": 3, "noise_high_1h": 0, "rssi": -71}
}
```

Field names, units, and the event taxonomy are part of the public API and follow semver.

---

## 8. Calibration & Provisioning Procedures (shipped as docs + firmware routines)

1. **8.1 Lightning antenna tuning** — automated routine per §6.1; result stored + printed on provisioning report.
2. **8.2 Rain gauge** — pour a measured volume (e.g. 300 ml) through the funnel; firmware counts tips; tip volume stored. Re-run seasonally.
3. **8.3 Soil moisture** — minimum two-point (air-dry, saturated) per probe per soil class; project maintains a community-contributed soil-class curve library keyed by region. Devices without a curve report relative units.
4. **8.4 Wind vane north** — compass alignment at install; offset stored.
5. **8.5 Provisioning record** — every built unit exports a JSON calibration certificate committed to the deployment registry.

---

## 9. Repository & Project Structure

```
vaanilai-station/
├── SPEC.md                  # this document (normative)
├── hardware/
│   ├── hub/                 # schematics (KiCad), PCB, BOM (CSV w/ India sourcing links)
│   ├── node/
│   └── mechanical/          # parametric CAD + STEP + 3MF (shield, enclosures, mounts)
├── firmware/
│   ├── hub/                 # ESP-IDF or PlatformIO project
│   ├── node/
│   └── shared/              # sensor drivers, LoRa protocol, rules engine
├── calibration/             # provisioning tools + soil curve library
├── backend/                 # reference MQTT ingest + API (thin; app-agnostic)
├── docs/                    # build guide, install guide, farmer-facing material
│   └── i18n/ta/             # Tamil translations are first-class, not an afterthought
└── validation/              # bench test scripts, season-log analysis notebooks
```

**Licensing rationale:** CERN-OHL-S keeps hardware derivatives open (the point of the project); Apache-2.0 lets the firmware be embedded in commercial farmer-facing apps (e.g. Vayal) without friction; CC-BY-SA keeps documentation and its translations shared.

**Governance (initial):** maintainer-led with a public RFC process for spec changes; decision matrix (§3) changes require an RFC. Code of conduct: Contributor Covenant.

---

## 10. Quality & Acceptance Criteria (v1.0 release gates)

- [ ] Hub survives 30 continuous days outdoors in-region (monsoon or peak summer) with zero data loss and no enclosure water ingress.
- [ ] Node achieves ≥ 6 months on its power budget at default cadence (measured, not simulated).
- [ ] T/RH inside reference shield within ±1.5 °C of an aspirated reference in full sun.
- [ ] Rain gauge within ±5% of reference at ≤ 25 mm/h.
- [ ] Lightning: published season hit/false/miss report; false-alarm rate target < 1 false local alarm / device / month.
- [ ] Full build reproducible by a third party from the repo alone (documented independent build).
- [ ] Total reference BOM cost published, with target: Node ≤ ₹2,500; Hub ≤ ₹12,000 at qty 10 (targets to be validated and revised in the open).

## 11. Explicit Non-Goals (v1)

- WMO/IMD-grade metrological compliance (we are decision-grade, not synoptic-grade).
- Pyranometer-based Penman-Monteith ET; dedicated leaf-wetness hardware.
- Camera/imagery, pest traps, actuation (valve control) — expansion candidates only.
- Proprietary cloud lock-in: the backend is a thin reference; any MQTT consumer works.

## 12. Roadmap

| Milestone | Contents |
|---|---|
| **M0 — Spec freeze** | Community review of this document; RFC window |
| **M1 — Solo prototype** | Single-box build, Layer 0–3 lightning mitigations, derived metrics, bench validation |
| **M2 — Hub + 3 Nodes pilot** | One village, full monsoon season logging, calibration tooling |
| **M3 — v1.0** | Acceptance criteria met, build guide + Tamil docs complete, public BOM |
| **M4 — Expansion modules** | Pyranometer, leaf wetness, multi-depth soil profiles, valve actuation RFCs |

---

*Maintainer: Rural Bytes Tamil community. Contributions, regional adaptations, and soil-curve submissions welcome.*
