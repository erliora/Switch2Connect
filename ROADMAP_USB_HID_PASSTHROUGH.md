# Switch 2 Controller → USB HID Passthrough Roadmap

**Objective:** Rework the ESP32 firmware to eliminate the Windows desktop application by implementing direct USB HID device emulation, making Switch 2 controllers plug-and-play without software installation.

**Status:** Planning Phase  
**Last Updated:** July 20, 2026

---

## Executive Summary

The current Switch2Connect architecture requires:
1. Windows Python/C# desktop app
2. Virtual gamepad driver (WinUHid/ViGEmBus/USBIP)
3. USB CDC serial bridge from ESP32

The proposed architecture:
1. Direct USB HID emulation on ESP32
2. No desktop app required
3. No driver installation needed (Windows native HID support)
4. Comparable to DS5Dongle project for DualSense 5

---

## Phase 1: Reference & Research

### 1.1 DS5Dongle Repository
**Reference implementation for USB HID emulation (Pico 2 W / RP2350)**

- **Repository:** https://github.com/awalol/DS5Dongle
- **Key Files:**
  - [`src/usb_descriptors.cpp`](https://github.com/awalol/DS5Dongle/blob/master/src/usb_descriptors.cpp) — Device, configuration, HID report descriptors (321-437 bytes)
  - [`src/usb.cpp`](https://github.com/awalol/DS5Dongle/blob/master/src/usb.cpp) — USB device initialization and I/O
  - [`src/bt.cpp`](https://github.com/awalol/DS5Dongle/blob/master/src/bt.cpp) — Bluetooth stack integration (BTstack)
  - [`src/main.cpp`](https://github.com/awalol/DS5Dongle/blob/master/src/main.cpp) — Main event loop
  - [`tools/config_tool.py`](https://github.com/awalol/DS5Dongle/blob/master/tools/config_tool.py) — HID feature report configuration protocol

- **Build System:** CMake + Pico SDK + TinyUSB 0.21.0
- **Key Insights:**
  - Uses **TinyUSB** for HID device class (not CDC serial)
  - VID/PID spoofing (0x054C Sony) for native Windows recognition
  - Audio + HID composite device with multiple interfaces
  - Report descriptors define exact controller shape to Windows
  - Runs at **150 MHz** (no overclock needed)

### 1.2 XinHeLianSheng-Pro2-Bridge Repository ⭐ PRIMARY REFERENCE
**Closest existing implementation — same board, same controller, same architecture**

- **Repository:** https://github.com/LeonChrome/XinHeLianSheng-Pro2-Bridge
- **ESP32-S3 firmware branch:** `codex/v5.9.13-finale-dual-release`
- **Why it matters:** This project already does exactly what this roadmap proposes — an ESP32-S3 N16R8 acting as BLE central to a real Switch 2 Pro controller while enumerating as a USB game controller via TinyUSB on the native USB port. Unlike DS5Dongle (RP2350/Pico SDK), it is built on **ESP-IDF 5.4.2**, so its build system and TinyUSB integration carry over directly.

- **Key Files (branch `codex/v5.9.13-finale-dual-release`):**
  - `firmware/esp32s3_switch2_bridge/main/usb/usb_descriptors.c` — Device/config/HID report descriptors for all identity modes
  - `firmware/esp32s3_switch2_bridge/main/usb/hid_report.c` — Input report construction (neutral states, button bitmasks, stick encoding)
  - `firmware/esp32s3_switch2_bridge/main/usb/usb_switch2_vendor.c` — Switch 2 vendor bulk protocol emulation (init handshake, flash reads, calibration)
  - `firmware/esp32s3_switch2_bridge/main/usb/usb_hid_device.c` / `usb_xinput_device.c` — TinyUSB HID and XInput device drivers
  - `firmware/esp32s3_switch2_bridge/main/tusb_config.h` — TinyUSB configuration for ESP32-S3
  - `firmware/esp32s3_switch2_bridge/main/ble/`, `bridge/` — BLE central + BLE→USB conversion layer
  - `firmware/esp32s3_dualsense_identity_experiment/` — Separate firmware emulating DualSense identity (054C:0CE6) with gyro + haptic audio routing

- **Identity Modes (build profiles from one codebase):**
  | Mode | VID:PID | Interface layout | Notes |
  |------|---------|------------------|-------|
  | Nintendo Pro (experiment) | `057E:2069` | HID (vendor page `0xFF00`) + vendor bulk (EP 0x82/0x02) | bcdUSB 2.01, strings "Nintendo Co., Ltd." / "Nintendo Switch Pro Controller", 500 mA |
  | Switch Pro legacy (dual HID) | `057E` + legacy Pro PID | Two HID interfaces, both bidirectional, 1 ms polling | Classic Joy-Con/Pro report IDs (0x30 full state, 0x21 subcommand reply, etc.) |
  | XInput | `045E:028E` | Single vendor-class interface (class 0xFF), 4/8 ms polling, 32-byte packets | Xbox 360 identity — broadest game compatibility |
  | DualSense | `054C:0CE6` | HID + audio | Separate firmware tree, gyro + HD haptics |

- **Key Insights:**
  - **The Switch Pro identity is NOT a standard gamepad HID descriptor.** Nintendo mode uses vendor usage page `0xFF00` with raw 63-byte reports (input `0x05`, output `0x02`, feature `0x7F`). Windows recognizes it as HID but does **not** map it as a gamepad natively — Steam/SDL does the interpretation. This directly contradicts the "sticks/hat/buttons descriptor" assumption in Phase 3.1 and forces an identity-mode decision (see 3.1).
  - **Switch 2 init handshake must be emulated** for the Nintendo identity: the vendor bulk interface answers command `0x02` (flash reads at `0x13000` serial, `0x13080`/`0x130C0` calibration), `0x03` (calibration, stick neutral = 2048), `0x15 0x01` (MAC address), plus a "Steam init guard" that gates HID IN reports until the host sends start-output (`0x03 0x0D`).
  - **Rumble path proven:** HID output report `0x02` (subtype `0x5x`) → decoded → re-encoded as Pro 2 BLE vibration packets. Same relay concept as the existing `rumble_drive_channel()`.
  - **BLE robustness:** fast connection parameters are deferred until input stabilizes to avoid reconnect loops — worth copying.
  - Gates HID IN submissions immediately on TinyUSB bus reset (prevents stale-report lockups on replug).

### 1.3 Current Switch2Connect Architecture
**Baseline to understand and refactor**

- **Repository:** https://github.com/TommyWabg/Switch2Connect
- **Main Entry Points:**
  - [`ESP32-S3 firmware/esp32s3_usb_bridge/main/main.c`](https://github.com/TommyWabg/Switch2Connect/blob/main/ESP32-S3%20firmware/esp32s3_usb_bridge/main/main.c) (75KB) — Firmware core
    - BLE connection management
    - Input report forwarding (relay from controller → USB CDC)
    - Rumble relay engine (specialized frame-by-frame rumble handling)
    - Serial command parsing (`wr`, `wrpair`, `rs`, `conn`, `disc`)

  - [`src/virtual_controller.py`](https://github.com/TommyWabg/Switch2Connect/blob/main/src/virtual_controller.py) — Python app side
    - Button mapping logic
    - Gyro processing (1000Hz interpolation)
    - Driver abstraction (WinUHid/ViGEmBus/USBIP)

- **Switch 2 BLE Protocol:**
  - **Input characteristic UUIDs:**
    - `ab7de9be-89fe-49ad-828f-118f09df7fd2` (FD2 notify, post-init)
    - `7492866c-ec3e-4619-8258-32755ffcc0f8` (legacy notify, pre-init)
  - **Command characteristics:**
    - `649d4ac9-8eb7-4e6c-af44-1ea54fe5f005` (command UUID)
    - Rumble: `cc483f51-9258-427d-a939-630c31f72b05` (Pro), `fa19b0fb-cd1f-46a7-84a1-bbb09e00c149` (Joy-Con R), `289326cb-a471-485d-a8f4-240c14f18241` (Joy-Con L)
  - **Report size:** 64 bytes
  - **Button layout:** Reverse-engineered from existing code

### 1.4 USB HID Specification References
**Standards for descriptor definition**

- **USB HID Spec:** https://www.usb.org/sites/default/files/documents/hid1_11.pdf
- **HID Usage Tables:** https://www.usb.org/sites/default/files/documents/hut1_12v2.pdf
- **Game Pad Profile:** Generic Desktop Controls → Usage 0x05

---

## Phase 2: Architecture Design

### 2.1 System Architecture

```
┌─────────────────────────────────────────────────────┐
│                 Physical Controller                  │
│        (Switch 2 Joy-Con / Pro Controller)          │
└────────────────┬────────────────────────────────────┘
                 │ Bluetooth 5.0
                 ▼
┌─────────────────────────────────────────────────────┐
│    ESP32-S3 N16R8 Firmware (NEW ARCHITECTURE)      │
│                                                      │
│  ┌──────────────────────────────────────────────┐  │
│  │  BLE Stack (NimBLE) - Keep Existing          │  │
│  │  ├─ Scan & Connect                            │  │
│  │  ├─ GATT Discovery                            │  │
│  │  ├─ Input Report Subscription                 │  │
│  │  └─ Rumble Command Relay                      │  │
│  └──────────────────────────────────────────────┘  │
│                      │                              │
│  ┌──────────────────────────────────────────────┐  │
│  │  NEW: USB HID Device (TinyUSB)                │  │
│  │  ├─ Device Descriptor (VID/PID)               │  │
│  │  ├─ Configuration Descriptor                  │  │
│  │  ├─ HID Report Descriptor                     │  │
│  │  │   └─ Analog sticks (L/R)                   │  │
│  │  │   └─ D-pad (Hat switch)                    │  │
│  │  │   └─ Buttons (A/B/X/Y/LB/RB/etc)          │  │
│  │  │   └─ Triggers (LT/RT)                      │  │
│  │  │   └─ Motion data (gyro/accel)              │  │
│  │  ├─ Interrupt endpoints (EP IN/OUT)           │  │
│  │  └─ HID report buffer handling                │  │
│  └──────────────────────────────────────────────┘  │
│                      │                              │
│  ┌──────────────────────────────────────────────┐  │
│  │  BLE-to-HID Conversion Layer (NEW)            │  │
│  │  ├─ BLE 64-byte input → HID gamepad format    │  │
│  │  ├─ Button/stick mapping                      │  │
│  │  ├─ Motion sensor data transformation         │  │
│  │  └─ Rumble feedback back to controller        │  │
│  └──────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
                      │ USB (HID class)
                      ▼
        ┌─────────────────────────────────────┐
        │   Windows 10/11                      │
        │   (Native HID driver - built-in)    │
        │   Auto-recognized as gamepad        │
        └─────────────────────────────────────┘
```

### 2.2 Key Changes vs. Current Design

| Aspect | Current (CDC Serial) | Proposed (USB HID) |
|--------|---|---|
| **Protocol** | USB Communications Device Class | USB Human Interface Device Class |
| **Data Format** | Binary protocol + serial commands | HID report descriptors + interrupt transfers |
| **Driver Dependency** | Requires WinUHid/ViGEmBus/USBIP | Windows native HID (no install) |
| **Windows App** | Required (button mapping, gyro processing) | Not required for basic use |
| **Latency** | ~5-10ms (app processing + driver) | ~1-2ms (direct USB) |
| **Plug-and-Play** | No (requires app launch) | Yes (instant recognition) |
| **VID/PID** | TinyUSB generic (0x16C0:0x05DF) | Emulate Nintendo/Sony VID |

---

## Phase 3: Implementation Breakdown

### 3.1 HID Report Descriptor Design

**Task:** Define the HID report format for Switch 2 controller

**Inputs from existing code:**
- Switch 2 BLE report format (from [`main.c` lines 56-61](https://github.com/TommyWabg/Switch2Connect/blob/main/ESP32-S3%20firmware/esp32s3_usb_bridge/main/main.c#L56-L61))
  ```c
  typedef struct {
      uint8_t channel;
      uint8_t length;
      uint8_t is_command;   // 0 = input report, 1 = command/ack response
      uint8_t payload[NINTENDO_REPORT_SIZE];  // 64 bytes
  } controller_report_t;
  ```

**Reference template:** DS5Dongle HID descriptor
- File: [`src/usb_descriptors.cpp` lines 488-647](https://github.com/awalol/DS5Dongle/blob/master/src/usb_descriptors.cpp#L488-L647) (DualSense, 321 bytes)
- Structure:
  ```
  Usage Page: Generic Desktop (0x01)
  Usage: Game Pad (0x05)
  Collection: Application
  ├─ Report ID: 0x01
  ├─ Analog Sticks: X, Y, Z, Rz (left/right), Rx, Ry (triggers)
  ├─ D-Pad: Hat switch (4 bits)
  ├─ Buttons: 15 button bits
  ├─ Vendor-defined: Motion data, touchpad, etc. (13 bytes)
  └─ Report Count: 64 bytes total
  ```

**⚠️ Identity Mode Decision (informed by XinHeLianSheng-Pro2-Bridge):**

A real Switch (2) Pro Controller does **not** present a standard gamepad HID descriptor — it uses vendor usage page `0xFF00` with raw 63-byte reports that only Steam/SDL-aware software interprets. "Plug-and-play in Windows with no software" therefore depends on which USB identity we choose:

| Identity | Windows native? | Steam/SDL? | XInput games? | Complexity |
|----------|----------------|-----------|---------------|------------|
| **Generic HID gamepad** (standard descriptor) | ✅ DirectInput | ✅ | ❌ (no XInput) | Low — pure descriptor work |
| **XInput (045E:028E)** | ✅ | ✅ | ✅ | Medium — vendor-class interface, XInput packet format |
| **Switch Pro (057E:2069)** | HID only (no gamepad mapping) | ✅ incl. gyro | ❌ | High — vendor bulk init handshake emulation |
| **DualSense (054C:0CE6)** | Partial | ✅ incl. gyro | ❌ | High — DS5 report format + feature reports |

**Recommendation:** Start with generic HID gamepad (Phase 1 milestone: enumerate + inputs work), then add XInput as a second build profile for game compatibility. Nintendo/DualSense identities become Phase 2 options if gyro passthrough via Steam is wanted. XinHeLianSheng-Pro2-Bridge demonstrates all of these as build-time profiles from one codebase — mirror that structure.

**Deliverables:**
- [x] Reverse-engineer Switch 2 BLE report format → **[docs/SWITCH2_BLE_REPORT_LAYOUT.md](docs/SWITCH2_BLE_REPORT_LAYOUT.md)**
  - Button mapping (from existing firmware parsing)
  - Stick ranges and deadzone
  - Trigger axis ranges
  - Motion sensor byte order
- [ ] Define USB HID report descriptor (raw bytes)
  - Map BLE bytes → HID fields
  - Set logical/physical min/max ranges
  - Report size & count
- [ ] Create Python utility to validate descriptor (optional)

**Related Resources:**
- HID Descriptor Tool: https://eleccelerator.com/usbdescreqparser/
- HID Report Descriptor Format: https://www.usb.org/sites/default/files/documents/hid1_11.pdf (Chapter 6)

---

### 3.2 USB Descriptors & Device Initialization

**Task:** Implement TinyUSB device descriptors for Switch 2 controller emulation

**Files to create/modify:**

1. **`usb_descriptors.c`** (new, or rename existing)
   - Device descriptor (VID/PID, device class, manufacturer/product strings)
   - Configuration descriptor (interfaces, endpoints)
   - HID report descriptor (from Phase 3.1)
   - String descriptors

   **Reference:** [`DS5Dongle/src/usb_descriptors.cpp`](https://github.com/awalol/DS5Dongle/blob/master/src/usb_descriptors.cpp)
   - Lines 90-133: Device descriptor
   - Lines 139-482: Configuration descriptor
   - Lines 488-647: HID report descriptor

2. **`tusb_config.h`** (modify)
   - Enable HID device class (`CFG_TUD_HID=1`)
   - Disable CDC if removing serial bridge (`CFG_TUD_CDC=0`)
   - Configure endpoint counts and sizes

   **Reference:** [`DS5Dongle/src/tusb_config.h`](https://github.com/awalol/DS5Dongle/blob/master/src/tusb_config.h)

3. **`usb.c`** / **`usb.cpp`** (new or extensive refactor)
   - TinyUSB initialization
   - HID report callback (`tud_hid_report()`)
   - Periodic report transmission (~125 Hz for Switch 2)

   **Reference:** [`DS5Dongle/src/usb.cpp`](https://github.com/awalol/DS5Dongle/blob/master/src/usb.cpp)

**Key Constants (per identity mode — see decision table in 3.1):**
```c
// Generic HID gamepad (Phase 1 recommendation)
#define USB_VID         0x16C0      // Or project-specific VID
#define USB_PID         0x05DF
// XInput profile:   VID 0x045E, PID 0x028E (Xbox 360), vendor class 0xFF
// Switch Pro:       VID 0x057E, PID 0x2069, bcdUSB 0x0201, + vendor bulk itf
// DualSense:        VID 0x054C, PID 0x0CE6

#define HID_REPORT_SIZE 64          // Standard gamepad report

#define EP_HID_IN       0x81        // Endpoint for input reports
#define EP_HID_OUT      0x01        // Endpoint for output (rumble)
#define POLLING_RATE    0x01        // 1ms interval (XinHeLianSheng uses 1ms for HID modes)
```

**Deliverables:**
- [ ] Device descriptor struct with correct VID/PID
- [ ] Configuration descriptor with HID interface (no CDC)
- [ ] HID report descriptor (321 bytes, from 3.1)
- [ ] Endpoint configuration (IN/OUT interrupt)
- [ ] TinyUSB driver setup code
- [ ] USB initialization in main()

---

### 3.3 BLE-to-HID Conversion Layer

**Task:** Bridge BLE input reports from Switch 2 to USB HID format

**Current Code to Adapt:**
- BLE input handling: [`main.c` lines 1458-1532 `handle_notify_rx()`](https://github.com/TommyWabg/Switch2Connect/blob/main/ESP32-S3%20firmware/esp32s3_usb_bridge/main/main.c#L1458-L1532)
- Controller report structure: [`main.c` lines 56-61](https://github.com/TommyWabg/Switch2Connect/blob/main/ESP32-S3%20firmware/esp32s3_usb_bridge/main/main.c#L56-L61)
- Input shadow buffer: [`main.c` lines 108-110](https://github.com/TommyWabg/Switch2Connect/blob/main/ESP32-S3%20firmware/esp32s3_usb_bridge/main/main.c#L108-L110)

**Conversion Function Skeleton:**
```c
// Convert 64-byte BLE report → 64-byte HID report
void ble_to_hid_report(uint8_t *ble_data, uint8_t *hid_report) {
    // hid_report[0] = report_id (0x01);
    
    // Parse BLE bytes (from Switch2Connect docs/reverse eng)
    // hid_report[1] = left_stick_x   (from ble_data[?])
    // hid_report[2] = left_stick_y   (from ble_data[?])
    // hid_report[3] = right_stick_x  (from ble_data[?])
    // hid_report[4] = right_stick_y  (from ble_data[?])
    
    // Buttons: map BLE button bits → HID button bits
    // D-pad: map BLE d-pad → HID hat switch
    // Triggers: map BLE trigger values → HID trigger axes
    // Motion: optional (gyro/accel) in vendor-defined section
}
```

**BLE Report Format Research:**
- Current firmware has parsing logic but obfuscated in C
- Need to extract from: [`virtual_controller.py`](https://github.com/TommyWabg/Switch2Connect/blob/main/src/virtual_controller.py) Python side (clearer documentation of byte offsets)
- Reference issue: [Switch 2 controller protocol research](https://github.com/Nadeflore/switch2-controllers/issues)

**Deliverables:**
- [x] Document Switch 2 BLE report byte layout (full mapping table) → **[docs/SWITCH2_BLE_REPORT_LAYOUT.md](docs/SWITCH2_BLE_REPORT_LAYOUT.md)**
- [ ] Implement `ble_to_hid_report()` function
- [ ] Integration with BLE notify handler
- [ ] Button/stick calibration (dead zones, ranges)
- [ ] Motion sensor data mapping (if keeping gyro passthrough)

---

### 3.4 Rumble Feedback (Optional, Phase 2)

**Task:** Handle USB OUT reports for rumble/haptic feedback

**Current Rumble Implementation:**
- Rumble relay engine: [`main.c` lines 689-755 `rumble_drive_channel()`](https://github.com/TommyWabg/Switch2Connect/blob/main/ESP32-S3%20firmware/esp32s3_usb_bridge/main/main.c#L689-L755)
- Rumble shadow system: [`main.c` lines 151-161](https://github.com/TommyWabg/Switch2Connect/blob/main/ESP32-S3%20firmware/esp32s3_usb_bridge/main/main.c#L151-L161)
- Rumble packet format (Joy-Con): 3x 5-byte frames, frequency/amplitude encoded

**HID Rumble Mechanism:**
- HID report OUT (host → device) with rumble payload
- Decode rumble intensity/frequency from HID standard gamepad format
- Relay back to Switch 2 controller via BLE

**Deliverables:**
- [ ] HID OUT report handler (`tud_hid_output_report_cb()`)
- [ ] Rumble payload parsing (translate from standard gamepad rumble → Switch 2 format)
- [ ] Integration with existing rumble relay engine (reuse `rumble_drive_channel()`)
- [ ] Testing with game titles

---

### 3.5 Build System Refactor

**Task:** Integrate TinyUSB HID changes into CMake build

**Current Build System:**
- [`CMakeLists.txt`](https://github.com/TommyWabg/Switch2Connect/blob/main/CMakeLists.txt) (main project build)
- [`ESP32-S3 firmware/esp32s3_usb_bridge/CMakeLists.txt`](https://github.com/TommyWabg/Switch2Connect/blob/main/ESP32-S3%20firmware/esp32s3_usb_bridge/CMakeLists.txt)

**Changes Needed:**
- Remove CDC components (if not keeping serial fallback)
- Ensure TinyUSB HID is enabled
- Update firmware version/build name
- Add optional features (motion, rumble) as CMake toggles

**Reference:**
- DS5Dongle CMakeLists: [`CMakeLists.txt` lines ~50-150](https://github.com/awalol/DS5Dongle/blob/master/CMakeLists.txt)
- Feature toggles: `PICO_W_BUILD`, `ENABLE_WAKE_HID`, `ENABLE_SERIAL`

**Deliverables:**
- [ ] Updated CMakeLists.txt with HID-only configuration
- [ ] Optional flags for experimental features
- [ ] Build validation (compile without errors)

---

## Phase 4: Testing & Validation

### 4.1 Hardware Testing

**Prerequisites:**
- ESP32-S3 N16R8 board
- Switch 2 Joy-Con or Pro Controller
- Windows 10/11 PC with USB port

**Test Cases:**
- [ ] USB device enumeration (Device Manager shows gamepad)
- [ ] BLE pairing and connection
- [ ] Button input (all 14+ buttons)
- [ ] Analog stick movement (full range, dead zone behavior)
- [ ] D-pad input
- [ ] Trigger buttons (LT/RT)
- [ ] Motion sensor data (if implemented)
- [ ] Rumble feedback (if implemented)
- [ ] Multi-controller support (2+ controllers simultaneously)
- [ ] Reconnection after power loss
- [ ] Windows recognition without drivers

### 4.2 Firmware Validation

**Automated Testing (if desired):**
- HID descriptor validation script (Python + hidapi)
- BLE → HID conversion unit tests
- Memory usage profiling

**Reference Tool:**
- DS5Dongle config tool: [`tools/config_tool.py`](https://github.com/awalol/DS5Dongle/blob/master/tools/config_tool.py) reads HID feature reports

### 4.3 Compatibility Testing

**Game Testing:**
- Elden Ring, Dark Souls series
- Fortnite, Valorant
- Emulators (Dolphin, CEMU, Ryujinx)
- Windows settings (calibration tools)

---

## Phase 5: Documentation & Release

### 5.1 User Documentation

**Deliverables:**
- [ ] Updated README.md (no app required, direct HID)
- [ ] Flashing instructions (same as before)
- [ ] Troubleshooting guide
- [ ] Comparison chart: old vs. new architecture

### 5.2 Developer Documentation

**Deliverables:**
- [ ] Architecture document (this roadmap refined)
- [ ] BLE ↔ HID conversion specification
- [ ] USB descriptor explanation
- [ ] Contributing guide for forks

### 5.3 Release Build

**Deliverables:**
- [ ] Firmware binary (`.uf2` or `.bin`)
- [ ] Version tagging (e.g., `v2.0.0-hid-passthrough`)
- [ ] Release notes with breaking changes
- [ ] Migration guide for existing users

---

## Resource Links Summary

### Code Repositories

| Resource | URL | Purpose |
|----------|-----|---------|
| **XinHeLianSheng-Pro2-Bridge** ⭐ | https://github.com/LeonChrome/XinHeLianSheng-Pro2-Bridge | Primary reference — ESP32-S3 BLE→USB bridge for Switch 2 Pro (branch `codex/v5.9.13-finale-dual-release`) |
| **DS5Dongle** | https://github.com/awalol/DS5Dongle | Reference HID implementation (RP2350/Pico) |
| **Switch2Connect** | https://github.com/TommyWabg/Switch2Connect | Current codebase to refactor |
| **Nadeflore/switch2-controllers** | https://github.com/Nadeflore/switch2-controllers | Original Switch 2 BLE reverse engineering |

### Files to Study

#### XinHeLianSheng-Pro2-Bridge (Primary Reference — branch `codex/v5.9.13-finale-dual-release`)
| File | Purpose |
|------|---------|
| `firmware/esp32s3_switch2_bridge/main/usb/usb_descriptors.c` | Device/config/HID descriptors for Nintendo, Switch-legacy, XInput, generic modes |
| `firmware/esp32s3_switch2_bridge/main/usb/hid_report.c` | Input report layouts, neutral states, button bitmasks |
| `firmware/esp32s3_switch2_bridge/main/usb/usb_switch2_vendor.c` | Switch 2 vendor bulk init protocol (flash reads, calibration, MAC, Steam init guard) |
| `firmware/esp32s3_switch2_bridge/main/usb/usb_xinput_device.c` | XInput vendor-class device implementation |
| `firmware/esp32s3_switch2_bridge/main/tusb_config.h` | TinyUSB config for ESP32-S3 (ESP-IDF 5.4.2) |
| `firmware/esp32s3_switch2_bridge/main/ble/`, `bridge/` | BLE central + BLE→USB conversion layer |
| `firmware/esp32s3_dualsense_identity_experiment/` | DualSense identity firmware (gyro + haptic audio) |

#### DS5Dongle (Secondary Reference)
| File | Lines | Purpose |
|------|-------|---------|
| `src/usb_descriptors.cpp` | [1-133](https://github.com/awalol/DS5Dongle/blob/master/src/usb_descriptors.cpp#L1-L133) | Device & config descriptors |
| | [488-647](https://github.com/awalol/DS5Dongle/blob/master/src/usb_descriptors.cpp#L488-L647) | HID report descriptor (DualSense) |
| `src/usb.cpp` | | USB initialization & HID callbacks |
| `src/bt.cpp` | | Bluetooth stack integration |
| `src/main.cpp` | | Event loop & initialization |
| `tools/config_tool.py` | | HID feature report protocol (Python) |

#### Switch2Connect (Baseline)
| File | Lines | Purpose |
|------|-------|---------|
| `ESP32-S3 firmware/esp32s3_usb_bridge/main/main.c` | [56-61](https://github.com/TommyWabg/Switch2Connect/blob/main/ESP32-S3%20firmware/esp32s3_usb_bridge/main/main.c#L56-L61) | Report structure |
| | [1458-1532](https://github.com/TommyWabg/Switch2Connect/blob/main/ESP32-S3%20firmware/esp32s3_usb_bridge/main/main.c#L1458-L1532) | BLE input handling |
| | [689-755](https://github.com/TommyWabg/Switch2Connect/blob/main/ESP32-S3%20firmware/esp32s3_usb_bridge/main/main.c#L689-L755) | Rumble relay engine |
| `src/virtual_controller.py` | | Python app logic (for reference on byte mapping) |

### Standards & Specifications

| Standard | URL | Purpose |
|----------|-----|---------|
| **USB HID Spec 1.11** | https://www.usb.org/sites/default/files/documents/hid1_11.pdf | HID report descriptor syntax, device classes |
| **USB HID Usage Tables** | https://www.usb.org/sites/default/files/documents/hut1_12v2.pdf | Usage page definitions (game pad = 0x01:0x05) |
| **TinyUSB Documentation** | https://docs.tinyusb.org/ | USB device implementation |
| **Pico SDK Docs** | https://raspberrypi.com/documentation/microcontrollers/c_sdk.html | RP2350 GPIO, peripherals (bonus for comparison with ESP32) |

### Tools & Utilities

| Tool | URL | Purpose |
|------|-----|---------|
| **HID Descriptor Parser** | https://eleccelerator.com/usbdescreqparser/ | Validate HID descriptors |
| **USBTreeView** | https://www.usbdeview.com/usb-tree-view.html | Inspect USB device descriptors on Windows |
| **Wireshark + usbpcap** | https://www.wireshark.org/ | Capture USB traffic for debugging |
| **Python hidapi** | https://pypi.org/project/hidapi/ | Test HID reports from PC |

---

## Dependencies & Prerequisites

### Hardware
- **ESP32-S3 N16R8** (same as current Switch2Connect)
- **Switch 2 Joy-Con / Pro Controller**
- **USB cable** (for flashing and operation)

### Software
- **ESP-IDF 5.4.2+** (XinHeLianSheng-Pro2-Bridge validates this version; Pico SDK is NOT needed — that's DS5Dongle's RP2350 toolchain)
- **CMake 3.13+** (bundled with ESP-IDF)
- **Xtensa toolchain** (bundled with ESP-IDF; no arm-none-eabi-gcc)
- **Python 3.8+** (for build tools)
- **TinyUSB** (via ESP-IDF `esp_tinyusb` / managed component)
- **NimBLE** (Bluetooth stack, already in use)

### Development Tools
- **Visual Studio Code** (recommended editor)
- **Git** (version control)
- **Esptool** (flashing utility for ESP32)

---

## Milestones & Effort Estimation

| Phase | Task | Est. Hours | Priority |
|-------|------|-----------|----------|
| **1** | Research & documentation | 4 | P0 |
| **2** | Architecture & design docs | 3 | P0 |
| **3.1** | HID descriptor definition | 6 | P0 |
| **3.2** | USB descriptor implementation | 8 | P0 |
| **3.3** | BLE-to-HID conversion | 10 | P0 |
| **3.4** | Rumble feedback (optional) | 8 | P2 |
| **3.5** | Build system refactor | 4 | P1 |
| **4** | Testing & validation | 12 | P0 |
| **5** | Documentation & release | 6 | P1 |
| **Total** | | **61 hours** | |

---

## Known Challenges & Mitigation

### Challenge 1: Switch 2 BLE Protocol Reverse Engineering
- **Problem:** Byte layout in BLE reports not fully documented
- **Mitigation:** Extract from existing Python code in `virtual_controller.py`; cross-reference firmware parsing logic
- **Resources:** [Nadeflore/switch2-controllers issues](https://github.com/Nadeflore/switch2-controllers/issues)

### Challenge 2: ESP32-S3 vs. Pico 2W Hardware Differences
- **Problem:** DS5Dongle uses RP2350 (ARM Cortex M33); ESP32-S3 is dual-core Xtensa
- **Mitigation:** TinyUSB and NimBLE are portable; main work is data conversion, not HAL changes
- **Resources:** Existing Switch2Connect firmware already handles ESP32-S3 specifics

### Challenge 3: Motion Sensor Data (Gyro/Accel)
- **Problem:** Current Switch2Connect app does complex gyro processing (1000Hz interpolation)
- **Mitigation:** Phase 1 delivers basic passthrough; motion can be Phase 2 enhancement
- **Alternative:** Provide vendor-defined HID reports for raw motion data (Windows can read but won't interpret)

### Challenge 4: Rumble Feedback Complexity
- **Problem:** Switch 2 HD Rumble uses proprietary frequency encoding
- **Mitigation:** Reuse existing rumble relay engine; map standard gamepad rumble → Switch 2 format
- **Resources:** Existing `rumble_drive_channel()` function in firmware

---

## Success Criteria

- [ ] ESP32-S3 boots and enumerates as USB HID gamepad (no CDC serial)
- [ ] Windows 10/11 recognizes device without driver installation
- [ ] All 14+ buttons map correctly to standard gamepad layout
- [ ] Analog sticks and triggers respond to input
- [ ] D-pad works as hat switch
- [ ] Multi-controller support (tested with 2 controllers)
- [ ] No lag detected vs. current CDC architecture
- [ ] Firmware compiles without errors
- [ ] Basic documentation complete
- [ ] Community testing validates game compatibility (3+ titles)

---

## Next Steps (Immediate Actions)

1. **Read & Review**
   - [ ] DS5Dongle `src/usb_descriptors.cpp` (full file)
   - [ ] DS5Dongle `src/usb.cpp` (HID callbacks)
   - [ ] Switch2Connect firmware `main.c` (BLE handler + report structure)

2. **Document**
   - [ ] Extract Switch 2 BLE report byte layout (create mapping table)
   - [ ] Map to standard USB HID gamepad fields

3. **Prototype**
   - [ ] Create minimal HID descriptor for Switch 2 controller
   - [ ] Compile test with existing firmware (TinyUSB swap only)
   - [ ] Verify USB enumeration

4. **Iterate**
   - [ ] Implement BLE-to-HID conversion
   - [ ] Test with one button, then all inputs
   - [ ] Add rumble feedback

---

## Appendix: Related Projects & Forks

### Official Forks with Enhanced Features
- **[loteran/DS5Dongle](https://github.com/loteran/DS5Dongle)** — Audio-driven haptic feedback
- **[SundayMoments/DS5_Bridge](https://github.com/SundayMoments/DS5_Bridge)** — More customization
- **[MarcelineVPQ/DS5Dongle-OLED-Edition](https://github.com/MarcelineVPQ/DS5Dongle-OLED-Edition)** — OLED display add-on
- **[snipem/DS4Dongle](https://github.com/snipem/DS4Dongle)** — DualShock 4 variant

### Switch 2 Community Resources
- **[LeonChrome/XinHeLianSheng-Pro2-Bridge](https://github.com/LeonChrome/XinHeLianSheng-Pro2-Bridge)** — ESP32-S3 BLE→USB bridge with four identity modes (primary reference, see 1.2)
- **y700-switch2-pro-bridge** (mentioned in Switch2Connect README as reference)
- **Nadeflore/switch2-controllers** — Original reverse engineering work

---

**Document Version:** 1.1  
**Last Updated:** 2026-07-20  
**Author:** Development Team  
**Status:** Ready for Phase 1 - Research  
**Changelog:** v1.1 — Added XinHeLianSheng-Pro2-Bridge as primary reference (same ESP32-S3 N16R8 board, proven BLE→USB bridge); added USB identity mode decision table; corrected toolchain dependencies (ESP-IDF, not Pico SDK)