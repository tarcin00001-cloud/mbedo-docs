# 128 - Pico Bluetooth Home Controller HC-05 Relay LCD

Build a Bluetooth-controlled home automation panel that receives single-character commands from a smartphone, controls four independent relay outputs (lights, fan, appliance, alarm), and displays the relay states on an I2C LCD.

## Goal
Learn how to receive UART commands from an HC-05 Bluetooth module, use a command dispatch table to toggle multiple relay outputs independently, and display a multi-channel status panel on an I2C character LCD in MicroPython.

## What You Will Build
A four-channel Bluetooth home controller:
- **HC-05 Bluetooth Module (GP0, GP1)**: Receives toggle commands from a phone app.
- **Relay 1 — Light (GP10)**: Toggled by command `1`.
- **Relay 2 — Fan (GP11)**: Toggled by command `2`.
- **Relay 3 — Appliance (GP12)**: Toggled by command `3`.
- **Relay 4 — Alarm (GP13)**: Toggled by command `4`. Command `0` turns all OFF.
- **I2C 16x2 LCD (GP4, GP5)**: Displays all four relay states.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| HC-05 Bluetooth Module | `hc05` | Yes | Yes |
| Relay Module × 4 | `relay` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| HC-05 | TXD | GP1 (UART0 RX) | Yellow | BT data to Pico |
| HC-05 | RXD | GP0 (UART0 TX) | Orange | Pico data to BT |
| HC-05 | VCC | 5V (VBUS) | Red | BT module power |
| HC-05 | GND | GND | Black | Ground reference |
| Relay 1 (Light) | IN | GP10 | Orange | Channel 1 control |
| Relay 2 (Fan) | IN | GP11 | Yellow | Channel 2 control |
| Relay 3 (Appliance) | IN | GP12 | Green | Channel 3 control |
| Relay 4 (Alarm) | IN | GP13 | Blue | Channel 4 control |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect HC-05 TXD → GP1 (Pico UART0 RX) and HC-05 RXD → GP0 (Pico UART0 TX). Connect the four relay IN pins to GP10–GP13. Connect the LCD to GP4/GP5. All grounds are shared.

## Code
```python
from machine import Pin, UART, I2C
import utime
from machine_lcd import I2cLcd

# HC-05 on UART0
bt = UART(0, baudrate=9600, tx=Pin(0), rx=Pin(1))

# Four relay outputs
relays = [
    Pin(10, Pin.OUT),  # Channel 1 — Light
    Pin(11, Pin.OUT),  # Channel 2 — Fan
    Pin(12, Pin.OUT),  # Channel 3 — Appliance
    Pin(13, Pin.OUT),  # Channel 4 — Alarm
]

labels = ["LIGHT", "FAN  ", "APPL ", "ALARM"]

# I2C Bus 0 on GP4/GP5
i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

# Start all relays OFF
for r in relays:
    r.value(0)

def update_lcd():
    lcd.clear()
    # Line 1: channels 1 and 2
    lcd.move_to(0, 0)
    lcd.putstr("{}: {} {}: {}".format(
        labels[0], "ON " if relays[0].value() else "OFF",
        labels[1], "ON " if relays[1].value() else "OFF"))
    # Line 2: channels 3 and 4
    lcd.move_to(0, 1)
    lcd.putstr("{}: {} {}: {}".format(
        labels[2], "ON " if relays[2].value() else "OFF",
        labels[3], "ON " if relays[3].value() else "OFF"))

update_lcd()
print("BT Home Controller ready. Send 1/2/3/4 to toggle, 0 for all OFF.")

while True:
    if bt.any():
        raw = bt.read(1).decode('utf-8', 'ignore').strip()

        if raw == '0':
            # All OFF
            for r in relays:
                r.value(0)
            print("All channels OFF.")
        elif raw in ('1', '2', '3', '4'):
            idx = int(raw) - 1
            new_state = 1 - relays[idx].value()  # Toggle
            relays[idx].value(new_state)
            print("{} -> {}".format(labels[idx], "ON" if new_state else "OFF"))
        else:
            print("Unknown command:", repr(raw))

        update_lcd()

        # Echo status back to phone
        status = ",".join(["{}:{}".format(labels[i], "ON" if relays[i].value() else "OFF")
                           for i in range(4)])
        bt.write(status + "\r\n")

    utime.sleep_ms(50)
```

## What to Click in MbedO
1. Drag **Pico**, **HC-05**, **four Relays**, and **I2C LCD** onto the canvas.
2. Connect HC-05 to GP0/GP1, Relays to GP10–GP13, and LCD to GP4/GP5.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Open a Bluetooth Serial app, pair with HC-05, and send `1`, `2`, `3`, `4` to toggle each channel. Send `0` to turn all off.

## Expected Output
```
BT Home Controller ready. Send 1/2/3/4 to toggle, 0 for all OFF.
LIGHT -> ON
FAN   -> ON
LIGHT -> OFF
All channels OFF.
```
(On screen: all four relay states shown in a 2×2 grid layout.)

## Expected Canvas Behavior
- Each relay component on the canvas toggles independently when the corresponding digit is sent from the Bluetooth app. The LCD updates all four channel states after every command.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `1 - relays[idx].value()` | Toggles the relay: if currently 1 (ON), result is 0 (OFF) and vice versa. |
| `bt.write(status + "\r\n")` | Sends a formatted status string back to the smartphone app confirming the new state. |

## Hardware & Safety Concept: Relay Toggle vs Direct Set Commands
Toggle commands (`1` = flip state) are simple but can cause state mismatches if a Bluetooth packet is lost or duplicated. Professional home automation protocols (e.g. MQTT, Z-Wave) use explicit set commands (`1=ON`, `1=OFF`) to avoid ambiguity. For high-reliability systems, always confirm the current state before acting.

## Try This! (Challenges)
1. **Named Commands**: Support text commands like `"LIGHT ON"` and `"FAN OFF"` by parsing the full string instead of a single character.
2. **Scene Presets**: Add command `5` that sets a "Movie Mode" (Light OFF, Fan ON) and `6` for "Wake Mode" (all ON).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| HC-05 not pairing | Default PIN not `1234` | Check HC-05 documentation — default pairing PIN is usually `1234` or `0000`. |
| Commands arrive doubled | Phone app sends `\r\n` | The `.strip()` call removes trailing whitespace. Ensure it is present. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [118 - Pico Dual Relay Timer Controller LCD](../intermediate/118-pico-dual-relay-timer-controller-lcd.md)
- [127 - Pico Bluetooth Robot HC-05 L298N LCD](127-pico-bluetooth-robot-hc05-l298n-lcd.md)
