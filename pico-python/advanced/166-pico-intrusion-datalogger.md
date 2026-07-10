# 166 - Pico Intrusion Datalogger

Build an advanced dual-beam laser security grid that displays status on an LCD and streams real-time breach log strings over Bluetooth.

## Goal
Learn how to control light transmitters (lasers), read analog light sensors (LDRs) in parallel, and transmit real-time ASCII security warnings over a Bluetooth serial link (UART) in MicroPython.

## What You Will Build
A wireless security barrier:
- **Laser Transmitter 1 (GP16)**: Constantly ON.
- **Laser Transmitter 2 (GP17)**: Constantly ON.
- **LDR Receiver 1 (GP26)**: Aligned with Laser 1.
- **LDR Receiver 2 (GP27)**: Aligned with Laser 2.
- **HC-05 Bluetooth Module (GP0, GP1)**: Transmits breach alerts wirelessly to a smartphone or computer.
- **16x2 I2C LCD (GP4, GP5)**: Displays active barrier grid status.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Laser Diodes | `led` | Yes (represented by separate LEDs) | Yes (low-power laser pointers) |
| LDR Photoresistors | `ldr` | Yes (two LDRs) | Yes |
| HC-05 Bluetooth Module | `bluetooth` | Yes (represented by serial UART receiver) | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Laser 1 | Anode | GP16 | Red | Laser 1 power control |
| Laser 2 | Anode | GP17 | Orange | Laser 2 power control |
| LDR 1 | Signal AO | GP26 | Green | Analog receiver 1 |
| LDR 2 | Signal AO | GP27 | Blue | Analog receiver 2 |
| HC-05 | TXD | GP1 (UART0 RX) | Yellow | Bluetooth TX to Pico RX |
| HC-05 | RXD | GP0 (UART0 TX) | White | Pico TX to Bluetooth RX |
| HC-05 | VCC / GND | 3.3V / GND | Red / Black | Module power |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect Lasers to GP16 and GP17. LDR outputs connect to analog pins GP26 and GP27. Connect HC-05 TXD to Pico GP1 (RX) and RXD to Pico GP0 (TX). The LCD shares Bus 0 (GP4/GP5).

## Code
```python
from machine import Pin, ADC, UART, I2C
import utime
from machine_lcd import I2cLcd

# Lasers
laser1 = Pin(16, Pin.OUT); laser1.value(1)
laser2 = Pin(17, Pin.OUT); laser2.value(1)

# LDR Receivers
ldr1 = ADC(26)
ldr2 = ADC(27)

# Bluetooth UART0
bt = UART(0, baudrate=9600, tx=Pin(0), rx=Pin(1))

# LCD
i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

# Light threshold level (16-bit ADC limit - beam is broken when below this)
TRIP_LIMIT = 32000

alarm_latched = False

def reset_security_state():
    global alarm_latched
    alarm_latched = False
    lcd.clear()
    lcd.putstr("Laser Grid Sys\nGrid: SECURE")

# Initialize
reset_security_state()
bt.write("Laser Security Grid Active.\r\n")

while True:
    val1 = ldr1.read_u16()
    val2 = ldr2.read_u16()
    
    b1_broken = (val1 < TRIP_LIMIT)
    b2_broken = (val2 < TRIP_LIMIT)
    
    # Latch alarm if either beam is broken
    if (b1_broken or b2_broken) and not alarm_latched:
        alarm_latched = True
        lcd.clear()
        lcd.putstr("!! GRID BREACH !!\n")
        
        if b1_broken and b2_broken:
            lcd.putstr("Beams: 1 & 2 CUT")
            bt.write("ALERT: GRID BREACH - BEAMS 1 & 2 CUT\r\n")
        elif b1_broken:
            lcd.putstr("Beam: 1 CUT")
            bt.write("ALERT: GRID BREACH - BEAM 1 CUT\r\n")
        else:
            lcd.putstr("Beam: 2 CUT")
            bt.write("ALERT: GRID BREACH - BEAM 2 CUT\r\n")
            
    # Check for disarm reset command over Bluetooth ('R')
    if bt.any():
        cmd = bt.read(1)
        if cmd == b'R':
            reset_security_state()
            bt.write("SYSTEM RESET: Grid Armed & Secure\r\n")
            
    utime.sleep_ms(50)
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **two Lasers** (represented by LEDs), **two LDRs**, **HC-05**, and **I2C LCD** onto the canvas.
2. Connect Lasers to **GP16/GP17**, LDRs to **GP26/GP27**, HC-05 to **GP0/GP1**, and LCD to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Adjust either LDR slider down to break a laser beam, check if the alert prints over Bluetooth, then type `'R'` to reset the system.

## Expected Output
Terminal (Bluetooth):
```
Laser Security Grid Active.
ALERT: GRID BREACH - BEAM 1 CUT
SYSTEM RESET: Grid Armed & Secure
```

## Expected Canvas Behavior
* Normal state: LCD reads `Grid: SECURE`. Silent.
* Beam 1 broken (LDR 1 < TRIP_LIMIT): LCD reads `!! GRID BREACH !!` / `Beam: 1 CUT`. Bluetooth transmits warning.
* Type `'R'` on BT: System resets, LCD returns to `Grid: SECURE`.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `bt.write("ALERT: ...")` | Sends a real-time breach log message over the Bluetooth UART channel to the remote supervisor terminal. |
| `bt.any()` | Checks if any bytes are waiting to be read in the UART buffer from the Bluetooth link. |

## Hardware & Safety Concept: Remote Security Logging
Wireless security sensors stream alerts over Bluetooth or Wi-Fi to a remote monitoring computer. To prevent intruders from jamming the wireless signal, security consoles use **heartbeat check-ins** (e.g. sending a "SYSTEM OK" status byte every 5 seconds). If the receiver misses a heartbeat check-in, it triggers an alert immediately.

## Try This! (Challenges)
1. **Physical Reset Button**: Add a push button on GP14 to reset and re-arm the grid.
2. **Alert Siren**: Connect a buzzer on GP15 and sound an alarm beep when a breach occurs.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Grid triggers on minor ambient light changes | Threshold set too high | Laser beams are bright. Ensure the `TRIP_LIMIT` is set lower than the laser's active light reading but higher than normal room lighting. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [108 - Pico Intrusion Detector Alarm (Laser + Photoresistor + Relay)](../intermediate/108-pico-intrusion-detector-alarm-laser-photoresistor-relay.md)
- [137 - Pico RFID Locker System Servo LCD Buzzer](135-pico-rfid-locker-system-servo-lcd-buzzer.md)
