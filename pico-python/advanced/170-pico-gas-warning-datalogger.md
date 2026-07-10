# 170 - Pico Gas Warning Datalogger

Build an industrial gas and fire safety console that monitors gas levels and infrared flame indicators, shuts off safety valves, and streams CSV logs to Serial.

## Goal
Learn how to monitor multiple analog inputs, update warning summaries on I2C LCDs, and stream structured CSV logs to the Serial Monitor in MicroPython.

## What You Will Build
An industrial gas warning monitor:
- **MQ-2 Gas Sensor (GP26)**: Measures gas concentrations.
- **LDR Flame Sensor (GP27)**: Detects fire infrared light.
- **Relay Module (GP10)**: Closes an emergency solenoid valve during alerts.
- **Active Buzzer (GP14)**: Sounds a warning siren.
- **16x2 I2C LCD (GP4, GP5)**: Displays live gas indices and safety status.
- **Serial Datalogger**: Streams CSV log lines (e.g. `Gas,Fire,Valve_state`) every 3 seconds.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| MQ-2 Gas Sensor | `gas_sensor` | Yes | Yes |
| LDR Photoresistor | `ldr` | Yes | Yes (configured as flame IR receiver) |
| Relay Module | `relay` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MQ-2 Sensor | AO | GP26 | Yellow | Gas concentration input |
| MQ-2 Sensor | VCC / GND | 5V / GND | Red / Black | Gas sensor power (requires 5V) |
| LDR Sensor | AO | GP27 | Green | Flame index input |
| LDR Sensor | VCC / GND | 3.3V / GND | Red / Black | LDR power |
| Relay Module | IN | GP10 | Orange | Gas supply solenoid valve |
| Relay Module | VCC / GND | 5V / GND | Red / Black | Relay power |
| Active Buzzer | VCC (+) | GP14 | Blue | Alarm siren pin |
| Active Buzzer | GND | GND | Black | Ground reference |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** The MQ-2 gas sensor and Relay Module require 5V (VBUS). LDR sensor runs on 3.3V. Active buzzer connects to GP14. The LCD shares GP4/GP5. All grounds must be tied together.

## Code
```python
from machine import Pin, ADC, I2C
import utime
from machine_lcd import I2cLcd

mq2_pin = ADC(26)
flame_pin = ADC(27)
relay_pin = Pin(10, Pin.OUT)
buzzer_pin = Pin(14, Pin.OUT)

# Keep gas valve open at startup (relay LOW)
relay_pin.value(0)
buzzer_pin.value(0)

i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

# Thresholds (16-bit ADC)
GAS_LIMIT   = 28800  # Gas leak raw threshold
FLAME_LIMIT = 24000  # Flame IR threshold (drops low near fire)

lcd.clear()
lcd.putstr("Gas Logger\nSystem Active")
print("Gas_idx,Flame_idx,Valve_shutdown")
utime.sleep(1.5)

while True:
    gas_val = mq2_pin.read_u16()
    flame_val = flame_pin.read_u16()
    
    gas_leak = (gas_val > GAS_LIMIT)
    fire_detect = (flame_val < FLAME_LIMIT)
    
    lcd.clear()
    
    if gas_leak or fire_detect:
        # EMERGENCY SHUTDOWN ACTIVE
        relay_pin.value(1)  # Shut off gas supply solenoid valve
        
        if gas_leak and fire_detect:
            lcd.putstr("!! FIRE & GAS !!\n")
        elif gas_leak:
            lcd.putstr("!! GAS LEAK !!\n")
        else:
            lcd.putstr("!! FIRE ALERT !!\n")
            
        lcd.putstr("VALVE: SHUTDOWN")
        
        # Pulse siren (non-blocking beep pattern during loop sleep)
        for _ in range(5):
            buzzer_pin.value(1)
            utime.sleep_ms(100)
            buzzer_pin.value(0)
            utime.sleep_ms(100)
    else:
        # Normal Secure operation
        relay_pin.value(0)  # Keep gas supply open
        buzzer_pin.value(0)
        
        lcd.putstr("Gas: {} index\n".format(gas_val // 655))
        lcd.putstr("Valve: OPEN")
        
    # Stream CSV logs to Serial
    shutdown_state = 1 if (gas_leak or fire_detect) else 0
    print("{},{},{}".format(gas_val, flame_val, shutdown_state))
    
    utime.sleep(3.0)  # Log once every 3 seconds
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **MQ-2 Sensor**, **LDR**, **Relay**, **Active Buzzer**, and **I2C LCD** onto the canvas.
2. Connect MQ-2 to **GP26**, LDR to **GP27**, Relay to **GP10**, Buzzer to **GP14**, and LCD to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Adjust the Gas PPM slider or light intensity to trigger the emergency shutoff valve, and watch the CSV logs stream.

## Expected Output
Terminal:
```
Gas_idx,Flame_idx,Valve_shutdown
8000,65535,0
32000,65535,1
8000,12000,1
```

## Expected Canvas Behavior
* Normal state: LCD reads `Gas: 12 index` / `Valve: OPEN`. Relay is OFF. CSV prints `8000,65535,0`.
* Gas leak (> 28800): LCD reads `!! GAS LEAK !!` / `VALVE: SHUTDOWN`. Relay turns ON, buzzer beeps. CSV prints `32000,65535,1`.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `gas_leak or fire_detect` | Logical OR check that triggers the emergency shutoff valve if either hazard is detected. |
| `relay_pin.value(1)` | Activates the relay output to trigger the emergency shutdown valve. |

## Hardware & Safety Concept: Fail-Safe Valve Controls
In gas line safety systems, solenoid valves are designed to be **Normally Closed (NC)**. This means they require continuous electrical power to stay OPEN. If power is cut off during a fire or gas leak, the valve automatically snaps closed by spring pressure, preventing gas from leaking. Storing warning logs locally or streaming them over serial provides diagnostic data to identify when and why shutdowns occurred.

## Try This! (Challenges)
1. **Latching Lockout**: If an alarm is triggered, lock the valve in the SHUTDOWN state until a physical reset button (GP16) is pressed.
2. **Alert Indicator**: Connect a warning LED on GP15 and flash it in sync with the buzzer alarm.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Relay oscillates ON/OFF rapidly | Sensor noise | Add a small hysteretic delay or check averaging loops to prevent minor sensor noise from toggling the valve. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [131 - Pico Multi-Zone Fire System Flame MQ-2 Relay LCD](131-pico-multi-zone-fire-system-flame-mq2-relay-lcd.md)
- [132 - Pico BT Environment Monitor HC-05 DHT22 MQ-2 LCD](132-pico-bt-environment-monitor-hc05-dht22-mq2-lcd.md)
