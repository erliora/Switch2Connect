# Switch 2 Controller — BLE Input Report Byte Layout

**Purpose:** Full mapping table for the 64-byte BLE input report, extracted from the
Switch2Connect codebase. This is the Phase 3.3 deliverable of
[`ROADMAP_USB_HID_PASSTHROUGH.md`](../ROADMAP_USB_HID_PASSTHROUGH.md) — the ground truth
for the firmware's BLE→USB HID conversion layer.

**Extraction sources (all offsets verified in code):**
- `src/controller.py` — `ControllerInputData.__init__` (lines ~901-914: standard layout;
  ~797-899: NSO GameCube layout), GATT UUIDs and command constants (lines ~642-678)
- `src/config.py` — `SWITCH_BUTTONS` bitmask table (lines 44-76)
- `src/utils.py` — `get_stick_xy()` 12-bit stick unpacking, `decodeu`/`decodes` (little-endian)
- `src/cemuhook_udp.py` — IMU unit scaling (lines ~367-390)
- `src/usb_hid_controller.py` — USB report 0x05/0x09 correlation (lines ~680-747)
- `ESP32-S3 firmware/esp32s3_usb_bridge/main/main.c` — relay wrapper, UUIDs (lines 36-61)

The BLE notification payload is parsed **without any slicing or prefix removal**
(`controller.py:2395` passes the notify `data` directly into `ControllerInputData`), so
every offset below is an absolute offset into the BLE notification payload.

All multi-byte fields are **little-endian**. `u16`/`u32` = unsigned, `s16` = signed.

---

## 1. Transport

| Item | Value |
|------|-------|
| Input report characteristic (post-init) | `ab7de9be-89fe-49ad-828f-118f09df7fd2` (notify) |
| Input report characteristic (pre-init/legacy) | `7492866c-ec3e-4619-8258-32755ffcc0f8` (notify) |
| Command write | `649d4ac9-8eb7-4e6c-af44-1ea54fe5f005` |
| Command response (ACK) | `c765a961-d9d8-4d36-a20a-5315b111836a` (notify) |
| Rumble write — Pro Controller 2 | `cc483f51-9258-427d-a939-630c31f72b05` |
| Rumble write — Joy-Con 2 (R) | `fa19b0fb-cd1f-46a7-84a1-bbb09e00c149` |
| Rumble write — Joy-Con 2 (L) | `289326cb-a471-485d-a8f4-240c14f18241` |
| BLE advertisement manufacturer/company ID | `0x0553` (Nintendo) |
| Input report size | 64 bytes |

### Device identification (BLE + USB)

| Device | VID | PID |
|--------|-----|-----|
| Joy-Con 2 (Right) | `0x057E` | `0x2066` |
| Joy-Con 2 (Left) | `0x057E` | `0x2067` |
| Pro Controller 2 | `0x057E` | `0x2069` |
| NSO GameCube Controller | `0x057E` | `0x2073` |
| (Switch 1 Pro Controller, for reference) | `0x057E` | `0x2009` |

VID/PID are read from controller flash at `0x13000` (controller info block: serial at
`+2..15`, VID at `+18:20`, PID at `+20:22`, body/button colors at `+25..36`).

---

## 2. Standard input report — Joy-Con 2 / Pro Controller 2 (64 bytes)

Applies to all Switch 2 controllers **except** the NSO GameCube controller (§4).

| Offset | Size | Type | Field | Notes |
|--------|------|------|-------|-------|
| 0-3 | 4 | u32 | Packet counter / timestamp | Monotonic; used for input-rate and gyro timing |
| 4-7 | 4 | u32 | **Buttons** | 32-bit bitmask, see §3 |
| 8-9 | 2 | — | *Unknown* | Not parsed by the app |
| 10-12 | 3 | packed | **Left stick** X/Y | 12+12 bit: `X = v & 0xFFF`, `Y = v >> 12` of u24; 0-4095, center ≈2048, **Y increases upward** |
| 13-15 | 3 | packed | **Right stick** X/Y | Same encoding |
| 16-17 | 2 | u16 | Mouse X | Optical mouse sensor (Joy-Con 2 rail); requires feature enable `0x10` |
| 18-19 | 2 | u16 | Mouse Y | |
| 20-21 | 2 | u16 | Mouse surface roughness | |
| 22-23 | 2 | u16 | Mouse distance (lift) | |
| 24 | 1 | — | *Unknown* | |
| 25-26 | 2 | s16 | Magnetometer X | Requires feature enable `0x80` |
| 27-28 | 2 | s16 | Magnetometer Y | |
| 29-30 | 2 | s16 | Magnetometer Z | |
| 31-32 | 2 | u16 | Battery voltage | millivolts (`/1000.0` → V) |
| 33-34 | 2 | u16 | Battery current | `/100.0` → mA (app-side scale) |
| 35-45 | 11 | — | *Unknown* | |
| 46-47 | 2 | u16 | Temperature | `25 + raw/127.0` → °C |
| 48-49 | 2 | s16 | **Accelerometer X** | `raw / 4096.0` → G. Requires feature enable `0x04` |
| 50-51 | 2 | s16 | **Accelerometer Y** | |
| 52-53 | 2 | s16 | **Accelerometer Z** | |
| 54-55 | 2 | s16 | **Gyroscope X** (pitch) | `raw × 0.061` → deg/s (Pro 2); `× 0.0535` (Joy-Con 2) — empirically calibrated app-side |
| 56-57 | 2 | s16 | **Gyroscope Y** (yaw) | |
| 58-59 | 2 | s16 | **Gyroscope Z** (roll) | |
| 60-63 | 4 | — | *Unknown* | |

Motion/mouse/magnetometer fields are only populated after the corresponding features are
enabled via command `0x0C` (see §6); before that they read as zero.

---

## 3. Button bitmask (offset 4, u32, little-endian)

Source: `SWITCH_BUTTONS` in `src/config.py`. Bits `0x03FFFFFF` are the physical-button
mask used by the app (`controller.py:2404`); bits above that are app-internal synthetics.

| Bit | Mask | Button | Notes |
|-----|------|--------|-------|
| 0 | `0x00000001` | Y | |
| 1 | `0x00000002` | X | |
| 2 | `0x00000004` | B | |
| 3 | `0x00000008` | A | |
| 4 | `0x00000010` | SR (right Joy-Con) | |
| 5 | `0x00000020` | SL (right Joy-Con) | |
| 6 | `0x00000040` | R | |
| 7 | `0x00000080` | ZR | Digital on Pro 2 / Joy-Con 2 |
| 8 | `0x00000100` | Minus | |
| 9 | `0x00000200` | Plus | |
| 10 | `0x00000400` | Right stick click | |
| 11 | `0x00000800` | Left stick click | |
| 12 | `0x00001000` | Home | |
| 13 | `0x00002000` | Capture | |
| 14 | `0x00004000` | C (chat) | Switch 2 addition |
| 15 | — | *(unused)* | |
| 16 | `0x00010000` | D-pad Down | D-pad is 4 independent bits, **not** a hat value |
| 17 | `0x00020000` | D-pad Up | |
| 18 | `0x00040000` | D-pad Right | |
| 19 | `0x00080000` | D-pad Left | |
| 20 | `0x00100000` | SR (left Joy-Con) | |
| 21 | `0x00200000` | SL (left Joy-Con) | |
| 22 | `0x00400000` | L | |
| 23 | `0x00800000` | ZL | Digital on Pro 2 / Joy-Con 2 |
| 24 | `0x01000000` | GR (rear grip right) | Pro Controller 2 only |
| 25 | `0x02000000` | GL (rear grip left) | Pro Controller 2 only |
| 26-31 | `0xFC000000` | *(app-internal synthetics)* | PS touch/click emulation, GC digital trigger clicks — never set by the controller itself |

---

## 4. Variant layout — NSO GameCube Controller (PID 0x2073)

The GameCube controller uses a **different** report layout (`controller.py:797-899`):

| Offset | Size | Field | Notes |
|--------|------|-------|-------|
| 0 | 1 | Packet counter | Single byte, unlike the u32 in the standard layout |
| 2 | 1 | Buttons group 1 | `0x01` B, `0x02` A, `0x04` Y, `0x08` X, `0x10` R-click, `0x20` Z, `0x40` Start |
| 3 | 1 | Buttons group 2 | `0x01` D-Down, `0x02` D-Right, `0x04` D-Left, `0x08` D-Up, `0x10` L-click, `0x20` ZL |
| 4 | 1 | Buttons group 3 | `0x01` Home, `0x02` Capture, `0x10` C |
| 5-7 | 3 | Left stick | Same 12+12-bit packing |
| 8-10 | 3 | Right stick (C-stick) | |
| 12 | 1 | **Left trigger analog** | u8; typical calibration min/bump/max ≈ 36/190/240 |
| 13 | 1 | **Right trigger analog** | u8 |
| 34-39 | 6 | Accelerometer X/Y/Z | s16 ×3 — note the different offset vs. standard layout |
| 40-45 | 6 | Gyroscope X/Y/Z | s16 ×3 |

The GC controller is the only Switch 2 device with analog triggers. It lacks Minus, L3
and R3.

---

## 5. Stick calibration

Read from controller flash via command `0x02` / subcommand `0x04` (memory read, §6).
User calibration is preferred; `0xFFFFFF` in the first 3 bytes means "unset, use factory".

| Block | Address | Applies to |
|-------|---------|-----------|
| Factory, stick 1 | `0x0130A8` | Left stick (Pro 2) / the single stick (Joy-Con L) |
| Factory, stick 2 | `0x0130E8` | Right stick (Pro 2) |
| User, stick 1 | `0x1FC042` | |
| User, stick 2 | `0x1FC062` | |

Calibration blob format (9+ bytes, read length `0x0B`):

| Offset | Size | Field |
|--------|------|-------|
| 0-2 | 3 | Center X/Y (12+12-bit packed) |
| 3-5 | 3 | Min offset X/Y (subtract from center) |
| 6-8 | 3 | Max offset X/Y (add to center) |

Sanity limits used by the app: center within 1024-3072 on both axes, offsets positive and
within range; otherwise fall back to center=(2048,2048), ±1500. Joy-Con sticks get a 1.05
gain after calibration; Pro 2 sticks 1.0.

---

## 6. Command protocol (over `649d4ac9…f005`, ACK on `c765a961…836a`)

| Command | Subcommand | Purpose |
|---------|-----------|---------|
| `0x02` | `0x04` | Memory read: payload = `len(1) 7E 00 00 addr(4,LE)`, max 0x4F bytes; response echoes len at [0] and addr at [4:8], data from [8] |
| `0x03` | `0x0A` | Haptics init/enable |
| `0x09` | `0x07` | Set player LEDs (patterns: 1→`0x01`, 2→`0x03`, 3→`0x07`, 4→`0x0F`, 5→`0x09`, 6→`0x05`, 7→`0x0D`, 8→`0x06`) |
| `0x0A` | `0x02` | Play vibration preset |
| `0x0C` | `0x02` | Feature init |
| `0x0C` | `0x04` | Feature enable — bitmask: `0x04` motion (IMU), `0x10` mouse sensor, `0x80` magnetometer |
| `0x15` | `0x01`/`0x04`/`0x02`/`0x03` | Pairing: set MAC / LTK1 / LTK2 / finish |

**Rumble frame** (written to the per-device rumble UUID, 5 bytes LE-packed, 40 bits):

| Bits | Field |
|------|-------|
| 0-8 | Low-freq band frequency (9 bit) |
| 9 | Low-freq tone enable |
| 10-19 | Low-freq amplitude (10 bit) |
| 20-28 | High-freq band frequency (9 bit) |
| 29 | High-freq tone enable |
| 30-39 | High-freq amplitude (10 bit) |

---

## 7. Correlation with USB HID reports (wired Pro Controller 2)

For reference when the same parsing must handle a USB-attached pad
(`src/usb_hid_controller.py`):

- **USB input report `0x05`**: payload after the report-ID byte is **byte-identical to
  the BLE layout above** — one parser serves both transports.
- **USB input report `0x09`** (default streaming mode): compact layout — counter at [1],
  power-info at [2] (bits 2-5 = level 0-9), buttons as 3 bytes at [3:6] ("Button Format
  3", different bit order — see `_pro2_buttons_to_u32()`), sticks at [6:9]/[9:12]; motion
  is packed in an undocumented format. Not relevant for the BLE firmware path.

---

## 8. BLE → USB HID gamepad mapping (for the conversion layer)

Recommended mapping for the Phase 3 generic-HID/XInput profiles:

| HID field | Source | Conversion |
|-----------|--------|------------|
| X / Y (left stick) | bytes 10-12 | unpack 12-bit, apply calibration, scale 0-4095 → logical range; **invert Y** (BLE up = +, HID down = +) |
| Z / Rz (right stick) | bytes 13-15 | same |
| Hat switch | button bits 16-19 | 4 direction bits → 8-way hat value (handle diagonals; 0x0F = neutral when no bits set) |
| Buttons 1-14 | button bits 0-14, 22-25 | direct bit remap (Y/X/B/A/SR/SL/R/ZR/−/+/R3/L3/Home/Capture/C/L/ZL/GR/GL as needed per profile) |
| Trigger axes (LT/RT) | button bits 23 (ZL) / 7 (ZR) | digital → 0/255 (Pro 2 triggers are switches, not analog); GC variant: bytes 12/13 directly |
| Motion (optional, vendor page) | bytes 48-59 | pass through s16 ×6; document scales: accel = raw/4096 G, gyro = raw×0.061 °/s |
| Rumble (OUT report) | — | host rumble → 5-byte frame per §6 → rumble UUID (reuse firmware `rumble_drive_channel()`) |

---

## 9. Open questions / TODO

- [ ] Bytes 8-9, 24, 35-45, 60-63 of the standard report are unparsed — capture raw
  dumps to identify (candidates: charging/connection status flags, higher-rate IMU
  samples, mouse click bits).
- [ ] The **legacy/pre-init notify** characteristic (`7492866c…`) report format is not
  parsed by the app (the firmware only relays it) — verify whether its layout matches
  before using it in the passthrough firmware.
- [ ] Confirm IMU axis orientation conventions (pitch/yaw/roll assignment above follows
  the cemuhook sender) against physical testing.
- [ ] Confirm the gyro scale factors (0.061 Pro / 0.0535 Joy-Con are empirical app-side
  values, not datasheet constants).
- [ ] Battery current unit (`/100.0`) — verify sign/scale under charge vs. discharge.
