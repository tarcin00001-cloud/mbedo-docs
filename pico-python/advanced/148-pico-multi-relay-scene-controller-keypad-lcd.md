# 148 - Pico Multi-Relay Scene Controller Keypad LCD

Build a four-channel relay scene controller that uses a 4×4 matrix keypad to select lighting/appliance scene presets, activate individual relay toggles, and display the active scene name and relay states on an I2C LCD.

## Goal
Learn how to scan a 4×4 matrix keypad by row/column multiplexing, map key inputs to both individual relay toggles and multi-relay scene presets, and display the active configuration on an I2C character LCD in MicroPython.

## What You Will Build
A keypad-driven relay scene controller:
- **4×4 Matrix Keypad (GP0-GP3 rows, GP10-GP13 cols)**: Keys 1-4 toggle individual relays; A-D select preset scenes.
- **Relay 1 — Lights (GP4)**: Controlled by key `1` and scenes.
- **Relay 2 — Fan (GP5)**: Controlled by key `2` and scenes.
- **Relay 3 — Projector (GP6)**: Controlled by key `3` and scenes.
- **Relay 4 — Curtains (GP7)**: Controlled by key `4` and scenes.
- **I2C 16x2 LCD (GP14, GP15)**: Displays the active scene name and relay states.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| 4×4 Matrix Keypad | `keypad` | Yes | Yes |
| Relay Module × 4 | `relay` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Keypad Rows | R1–R4 | GP0, GP1, GP2, GP3 | Blue wires | Row scan output pins |
| Keypad Cols | C1–C4 | GP10, GP11, GP12, GP13 | Yellow wires | Column read input pins |
| Relay 1 (Lights) | IN | GP4 | Orange | Scene relay 1 control |
| Relay 2 (Fan) | IN | GP5 | Yellow | Scene relay 2 control |
| Relay 3 (Projector) | IN | GP6 | Green | Scene relay 3 control |
| Relay 4 (Curtains) | IN | GP7 | Blue | Scene relay 4 control |
| I2C LCD | SDA / SCL | GP14 / GP15 | Shared Orange/Blue | I2C Bus 1 |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Keypad rows use GP0-GP3 (outputs) and columns GP10-GP13 (inputs with pull-down). Relays on GP4-GP7. LCD uses I2C Bus 1 on GP14/GP15 to avoid conflicts. All grounds are shared.

## Code
```python
from machine import Pin, I2C
import utime
from machine_lcd import I2cLcd

# Keypad matrix
ROWS = [Pin(p, Pin.OUT) for p in (0, 1, 2, 3)]
COLS = [Pin(p, Pin.IN, Pin.PULL_DOWN) for p in (10, 11, 12, 13)]
KEYS = [
    ['1','2','3','A'],
    ['4','5','6','B'],
    ['7','8','9','C'],
    ['*','0','#','D'],
]

# Relays: GP4-GP7 (index 0-3)
relays = [Pin(p, Pin.OUT) for p in (4, 5, 6, 7)]
for r in relays: r.value(0)

RELAY_LABELS = ["LIGHT", "FAN  ", "PROJ ", "CURT "]

# I2C Bus 1 on GP14/GP15
i2c = I2C(1, sda=Pin(14), scl=Pin(15), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

# Scene presets: (name, [relay states list])
SCENES = {
    'A': ("MOVIE MODE  ", [0, 1, 1, 1]),   # Fan+Proj+Curtains ON; Lights OFF
    'B': ("WORK  MODE  ", [1, 1, 0, 0]),   # Lights+Fan ON
    'C': ("SLEEP MODE  ", [0, 0, 0, 1]),   # Curtains only
    'D': ("ALL OFF     ", [0, 0, 0, 0]),   # All relays OFF
}

active_scene = "MANUAL      "

def scan_keypad():
    for r, row_pin in enumerate(ROWS):
        row_pin.value(1)
        for c, col_pin in enumerate(COLS):
            if col_pin.value() == 1:
                row_pin.value(0)
                utime.sleep_ms(30)
                return KEYS[r][c]
        row_pin.value(0)
    return None

def relay_states():
    return "".join("1" if r.value() else "0" for r in relays)

def update_lcd():
    lcd.clear()
    lcd.move_to(0, 0)
    lcd.putstr(active_scene)
    lcd.move_to(0, 1)
    lcd.putstr(" ".join(
        ("{}:{}".format(RELAY_LABELS[i][:1], "O" if relays[i].value() else "."))
        for i in range(4)))

update_lcd()
print("Scene controller ready. Keys 1-4=toggle, A-D=scene.")

last_key = None
last_key_ms = 0

while True:
    key = scan_keypad()
    now = utime.ticks_ms()

    if key and key != last_key and utime.ticks_diff(now, last_key_ms) > 250:
        last_key    = key
        last_key_ms = now
        print("Key:", key)

        if key in ('1', '2', '3', '4'):
            idx = int(key) - 1
            relays[idx].value(1 - relays[idx].value())  # Toggle
            active_scene = "MANUAL      "
            print("Relay {} toggled → {}".format(idx + 1, relays[idx].value()))

        elif key in SCENES:
            name, states = SCENES[key]
            active_scene = name
            for i, s in enumerate(states):
                relays[i].value(s)
            print("Scene:", name.strip())

        update_lcd()
        print("States:", relay_states())

    elif not key:
        last_key = None

    utime.sleep_ms(20)
```

## What to Click in MbedO
1. Drag **Pico**, **4×4 Keypad**, **four Relays**, and **I2C LCD** onto the canvas.
2. Connect Keypad rows to **GP0-GP3**, cols to **GP10-GP13**. Connect Relays to **GP4-GP7**. Connect LCD to **GP14/GP15** (I2C Bus 1). Connect power and GND.
3. Paste code, click **Run**. Press keys 1–4 to toggle individual relays. Press A–D to activate scene presets.

## Expected Output
```
Scene controller ready. Keys 1-4=toggle, A-D=scene.
Key: A
Scene: MOVIE MODE
States: 0111
Key: 1
Relay 1 toggled → 1
States: 1111
Key: D
Scene: ALL OFF
States: 0000
```

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `1 - relays[idx].value()` | Toggles relay: 1 becomes 0, 0 becomes 1. |
| `SCENES['A'] = ("MOVIE MODE", [0,1,1,1])` | Scene dictionary maps a key character to a name string and a list of relay target states. |

## Hardware & Safety Concept: Scene Presets vs Individual Control
Scene presets allow complex multi-relay configurations to be activated with a single keypress, reducing errors when multiple devices must be switched simultaneously. This is the same concept used in commercial building management systems (BMS) where "scenes" (occupancy, presentation, night) are programmed once and recalled instantly.

## Try This! (Challenges)
1. **Scene Memory**: Store the current relay states to EEPROM when `*` is pressed and restore them automatically at boot.
2. **Keypad Backlight**: Connect the LCD backlight control pin to a relay channel and turn it off during SLEEP MODE.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Wrong keys register | Row/col pin mapping reversed | Swap ROWS and COLS lists to match your specific keypad pinout. |
| LCD shows garbled characters | I2C Bus 1 address wrong | Run `i2c.scan()` on Bus 1 and use the detected address (try 0x27 or 0x3F). |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [128 - Pico Bluetooth Home Controller HC-05 Relay LCD](128-pico-bluetooth-home-controller-hc05-relay-lcd.md)
- [118 - Pico Dual Relay Timer Controller LCD](../intermediate/118-pico-dual-relay-timer-controller-lcd.md)
