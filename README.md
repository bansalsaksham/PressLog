# PressLog — Industrial IoT OEE Downtime Logger

A manufacturing-ready PCB and firmware architecture for capturing machine downtime reasons on a factory floor.

> **Origin:** The original design brief (see `Original Case Study Brief.pdf`) was given as a second-round technical case study during an internship interview process. I completed the full PCB design and firmware architecture independently and am sharing it here as my own project.

> **Status:** PCB design and firmware architecture complete. DRC/ERC clean, Gerbers and BOM generated for fabrication. Not yet physically built/soldered — this repository documents the design and engineering rationale, not a deployed product.

---

## 1. The Problem

OEE (Overall Equipment Effectiveness) is only as good as its data. But capturing downtime reasons at the machine edge is a human-interface problem as much as a software one. The factory floor kills conventional UX.

| Constraint | Root Cause | Why Standard Solutions Fail |
|---|---|---|
| **Gloved hands** | Thick Kevlar + metallic dust make capacitive screens useless | Tablet solutions require bare fingers: 15–30s to glove off/on. Fails the 5-second window. |
| **Patchy Wi-Fi** | VFDs, metal shielding, and RF interference create 2-hour dead zones | Cloud-first designs lose events silently. Operator never knows. OEE has invisible gaps. |
| **5-second window** | Operator moves to the next task seconds after a machine stops | Any UI requiring navigation or multiple confirmations is abandoned. |

## 2. Design Philosophy

- **Physical First** — Six large 30mm illuminated tactile buttons, one per reason code. No menus, no navigation, no scrolling. One press = one logged event, confirmed by an LED flash, an 85dB buzzer, and vibration.
- **Offline First** — Every press is timestamped via a DS3231 RTC and written to 8MB LittleFS flash *before* any network attempt. When Wi-Fi returns, the device silently replays the queue.
- **OEE Ready** — Each event carries `device_id`, `machine_id`, `reason_code`, `operator_uid` (NFC badge tap), UTC timestamp, and a sequence number. Sequence gaps are auto-detected downstream so any connectivity hole is visible to plant management.

## 3. Hardware — Custom 2-Layer PCB (KiCad)

Designed specifically for high-noise industrial environments.

| Engineering Decision | Why It Matters |
|---|---|
| **RC hardware debounce matrix** (10kΩ/0.1µF) on all 6 inputs | Software-only debouncing fails near VFDs and heavy machinery — this prevents ghost button events at the hardware level |
| **Isolated I2C routing for DS3231 RTC** | Routed completely clear of the 150kHz buck converter switching node — prevents inductive noise from corrupting timestamps. If the timestamp is wrong, the offline queue is worthless. |
| **DFM: thermal vias 0.2mm → 0.3mm** | Standard drill size reduces fabrication cost without sacrificing thermal performance |
| **RF-notched board edge** | Lets the ESP32 antenna hang off the fiberglass substrate, preventing signal attenuation |

### Physical Specifications

| Component | Specification |
|---|---|
| Platform | ESP32-S3, dual-core 240MHz |
| Buttons | 6× 30mm illuminated tactile (5N actuation) |
| Connectivity | Wi-Fi 802.11n + BLE 5.0 |
| Offline queue | 8MB LittleFS (~65,000 events) |
| RTC | DS3231, ±2ppm, coin-cell backup |
| Power | 24VDC input + 2000mAh LiPo backup (8h) |
| Identity | PN532 NFC module |

## 4. Firmware Architecture

**Offline-first data flow:**
```
Button Press → RTC Timestamp → Flash Queue (always written first) → Wi-Fi Check → Cloud Sync (batch HTTPS POST)
```

Defense-in-depth: the hardware RC filter kills high-frequency spikes; a 50ms software debounce is the secondary safety net.

```cpp
// Button ISR with secondary 50ms software debounce
void IRAM_ATTR isr_btn_0() {
  uint32_t now = millis();
  if (!pressed[0] && (now - last[0]) > 50) {
    pressed[0] = true;
    last[0] = now;
  }
}

// Queue write — always before network attempt
queueWrite(seq, rtc.now().unixtime(), buttonIdx); // flash first
confirmFeedback(buttonIdx);                       // LED + buzzer + vibration
if (wifiOnline) syncQueue();                       // attempt immediate sync
```

**Event payload schema:**
```json
{
  "device_id": "DL-CNC07-A3",
  "seq": 1042,
  "ts": "2026-04-08T14:22:08Z",
  "ts_source": "rtc",
  "reason_code": "MATERIAL_JAM",
  "reason_label": "Material Jam",
  "operator_uid": "a3f9d82c",
  "machine_id": "CNC-07"
}
```

## 5. Repository Structure

```
├── Hardware_Source/          # KiCad project — schematics + PCB layout
│   ├── ESP32_Core.kicad_sch
│   ├── HMI_IO.kicad_sch
│   ├── Power_Delivery.kicad_sch
│   ├── RTC_SubSystem.kicad_sch
│   └── PressLog_Edge_Node.kicad_pcb / .kicad_pro
├── Manufacturing/            # Fabrication-ready outputs
│   ├── PressLog_Gerbers/
│   └── PressLog_BOM.csv
├── Documentation/            # Renders, design rule reports, app mockups
│   ├── PressLog_Edge_Node.pdf
│   ├── DRC.rpt / ERC.rpt    # Design Rule Check / Electrical Rule Check — board is verified manufacturable
│   ├── PressLog_Front.png / _Back.png / _Side.png
│   └── App Interface Designs.pdf
└── Original Case Study Brief.pdf   # Full original case study writeup
```

## 6. What's Verified vs. Designed-Only

- ✅ Schematic capture across 4 subsystems (ESP32 core, HMI/IO, power delivery, RTC)
- ✅ PCB layout, routed and DRC/ERC clean
- ✅ Gerbers + BOM generated — ready to send to a fab house
- ⬜ Physical assembly / soldering — not yet built
- ⬜ Firmware — architecture and key patterns designed; not yet flashed to hardware

This was a design and engineering reasoning exercise, not a built product. The PCB design itself is real, complete, and manufacturable.

---

**Tools used:** KiCad, ESP32-S3, LittleFS, DS3231 RTC, PN532 NFC
