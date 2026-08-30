# TONEX Pedal Controller

Single-page web controller for the IK Multimedia TONEX Pedal. Manages presets via USB MIDI and reads names/configurations directly from the pedal via the USB CDC serial interface.

Computer:
![Interface PC](captures/tnx1.png)

Android smartphone:
![Interface Android1](captures/android1.png)
![Interface Android2](captures/android2.png)


Computer demo video:
[![Video PC](https://img.youtube.com/vi/ZrpM73ms7fk/0.jpg)](https://www.youtube.com/watch?v=ZrpM73ms7fk)

Android demo vido:
[![Video Android](https://img.youtube.com/vi/XhKJ70A9dGQ/0.jpg)](https://www.youtube.com/watch?v=XhKJ70A9dGQ)

## Features

- **3×3 Grid** of assignable presets with names and AMP/CAB badges
- **Full library** of 150 presets (50 banks × 3 slots A/B/C)
- **USB Sync** — reads all names and AMP/CAB flags directly from the pedal
- **MIDI Control** — sends Bank Select + Program Change to change presets
- **Drag & drop** — assign a preset to a button, swap between buttons, or delete via trash
- **Editing** — double-click to rename a preset and toggle AMP/CAB
- **Search** — filtering in the library
- **Persistence** — configuration saved in localStorage
- **Responsive** — adaptive text via `container-type: inline-size` + `cqi` units
- **Library toggle** — discreet chevron to minimize/expand the preset library
- **Export/Import JSON** — export preset names to a file, import them on another setup
- **Android support** — works on Android Chrome via WebUSB fallback (Web Serial not available on Android)

## Prerequisites

| Component | Required version |
|-----------|-----------------|
| Browser | Chrome 89+ or Edge 89+ (Web MIDI + Web Serial API) |
| OS | Windows 10/11, Android (via WebUSB) |
| Pedal | IK Multimedia TONEX Pedal (full size) |
| Cable | USB-C connected to the pedal's USB port |

> **Note**: On Android, Web Serial is not available — the app falls back to WebUSB for USB CDC communication. MIDI is not available on Android (no Web MIDI API).

## Installation

### Option 1 — Local web server (recommended)

Copy the `tonexpedal/` folder to your web server root, then access via:
```
https://your-server/tonexpedal/
```

### Option 2 — localhost with a simple server

```bash
# From the tonexpedal/ folder
npx serve -s . -l 3000
# or
python -m http.server 3000
```

Then open `http://localhost:3000`.

### Option 3 — Static file (no server required)

Simply double-click `index.html` or open it via `file:///` in your browser.

## Usage

### MIDI Connection

1. Connect the TONEX Pedal via USB
2. Open the app in Chrome/Edge
3. Select the MIDI device in the **Device** dropdown
4. Choose the MIDI channel (default: Ch 1)
5. Status changes to **Connected** (green dot)

### USB Sync (reading presets)

1. Click **Sync USB**
2. Select the TONEX Pedal serial port in the dialog
3. Progress shows: Hello → State → Reading 150 presets
4. Names and AMP/CAB badges fill in automatically
5. Button shows **Done! X/150 presets read**

### Export / Import JSON

- Click **⬇ JSON** to download all preset names as `tonex-presets.json`
- Click **⬆ JSON** to import a previously exported file (validates format, replaces names)

JSON format:
```json
{
  "0_A": "Trooper - 80s Pack",
  "0_B": "80s Lead - 80s Pack",
  "1_A": "Final Countdown - 80s Pack"
}
```

### 3×3 Grid

- **Single click** on a button → sends Bank Select + Program Change to the pedal
- **Drag** a preset from the library → assigns to the button
- **Drag** a button to another → swaps positions
- **Drag** a button to the trash → clears the button
- **Double-click** → opens edit modal (name, AMP, CAB)

### Library

- **Single click** → sends MIDI to audition the preset
- **Double-click** → edits name and AMP/CAB flags
- **Search** → filters by name or bank/slot number
- **Drag** to grid → assigns the preset
- **Chevron toggle** (▶/◀) on the panel border → minimizes/expands the library

## Technical Architecture

### Files

```
tonexpedal/
├── index.html          # Single-file application (HTML + CSS + JS)
├── favicon.svg         # SVG icon
├── README.md           # This documentation
├── docs/
│   └── index.html      # Web documentation page (FR/EN)
├── captures/
│   └── tnx1.png        # Interface screenshot
└── V1.0/
    └── index.html      # Version 1.0 archive
```

### MIDI Protocol

The TONEX Pedal uses 50 banks × 3 slots (A/B/C) = 150 presets.

| Preset # | Bank Select (CC#0) | Program Change |
|----------|-------------------|----------------|
| 0–127    | CC#0 = 0          | PC = preset#   |
| 128–149  | CC#0 = 1          | PC = preset# − 128 |

```
Bank Select:    [0xB0 + channel, 0x00, value]
Program Change: [0xC0 + channel, PC]
```

### USB CDC Serial Protocol (HDLC)

The pedal exposes two USB interfaces:
- **USB-MIDI** — for Bank Select / Program Change
- **USB CDC** — for serial communication (reading presets, parameters)

#### HDLC Frame

```
[0x7E] [payload stuffed] [CRC_lo stuffed] [CRC_hi stuffed] [0x7E]
```

- **Delimiter**: `0x7E`
- **Byte stuffing**: `0x7E` → `0x7D 0x5E`, `0x7D` → `0x7D 0x5D`
- **CRC-CCITT**: polynomial `0x8408`, init `0xFFFF`, inverted result (`~crc & 0xFFFF`)

#### Commands

| Command | Payload | Description |
|---------|---------|-------------|
| Hello | `b9 03 00 82 04 00 80 10 01 b9 02 02 10` | Connection init |
| Request State | `b9 03 00 82 06 00 80 10 03 b9 02 81 01 02 10` | Request current state |
| Request Preset (0–127) | `b9 03 81 00 02 82 06 00 80 10 03 b9 04 10 01 [index] 00` | Request preset (17 bytes) |
| Request Preset (128+) | `b9 03 81 00 02 82 06 00 80 10 03 b9 04 10 01 80 [index] 00` | Request preset (18 bytes, escape `0x80`) |

#### Preset Response — Structure

```
[header] [B9 04 B9 02 BC 21] [name 33 bytes] [parameters...]
                                          ↑ NAME_MARKER
```

The parameters section starts with marker `BA 03 BA 6D` (`PARAM_MARKER`), followed by encoded floats `0x88` + 4 bytes (little-endian):

| Parameter index | Byte offset (×5) | Description |
|----------------|-------------------|-------------|
| 17 | 85 | **AMP Enable** — 0.0 = off, >0.5 = on |
| 22 | 110 | **CAB Type** — 0.0 = off, 1.0 = VIR, 2.0 = Tone Model |

### Device ID

- **TONEX Pedal (full size)**: `0x10`
- TONEX One: `0x0B` (not supported)

### Transport Abstraction (Android Support)

The app uses a transport abstraction layer to support both **Web Serial** (desktop) and **WebUSB** (Android):

```
transportSend(frame)      → Serial.write() or USB.bulkTransferOut()
transportStartRead()      → Serial reader loop or USB.bulkTransferIn() loop
transportIsOpen()         → serialPort.opened or usbDevice.opened
transportDisconnect()     → serialPort.close() or usbDevice.close()
```

**Connection flow:**
1. Try **Web Serial** first (desktop Chrome/Edge)
2. If unavailable or fails, fallback to **WebUSB** (Android Chrome)
3. WebUSB shows the device picker filtered by VID `0x1963` (IK Multimedia)

**WebUSB CDC setup:**
- Find CDC Communication interface (class `0x02`) → control transfers (SET_LINE_CODING, SET_CONTROL_LINE_STATE)
- Find CDC Data interface (class `0x0A`) → bulk endpoints for HDLC data
- If class `0x0A` not found, fallback to any interface with bulk endpoints

### Persistence

Everything is saved in `localStorage` under the key `tonex-state`:

```json
{
  "buttons": {
    "0": { "bank": 0, "slot": "A" },
    "4": { "bank": 1, "slot": "B" }
  },
  "midi": { "device": "ToneX MIDI Out", "channel": 0 },
  "presets": {
    "0_A": { "name": "Trooper - 80s Pack", "amp": true, "cab": false },
    "0_B": { "name": "80s Lead - 80s Pack", "amp": true, "cab": true }
  }
}
```

## Troubleshooting

| Problem | Solution |
|---------|----------|
| No MIDI device | Check USB connection. Chrome → `chrome://midi-devices` |
| Web Serial unavailable | Use Chrome 89+ or Edge 89+. Check HTTPS/localhost |
| USB Sync fails | Close IK Tonex and any app using the serial port |
| Names don't appear | Re-run Sync USB. Check console (F12) for errors |
| AMP/CAB always grey | Check console for correct float32 values (log for first 3 presets) |
| Blank page after load | Reload the page, localStorage may be corrupted |
| Android: Sync doesn't read data | WebUSB fallback should auto-activate. Check console for interface/endpoint logs |

## Credits

- USB CDC protocol: reverse-engineered from [Builty/TonexOneController](https://github.com/Builty/TonexOneController)
- Protocol documentation: [vit3k/tonex_controller](https://github.com/vit3k/tonex_controller)
- Interface: IK Multimedia TONEX Pedal Controller v1.1

## License

Personal project — non-commercial use.
