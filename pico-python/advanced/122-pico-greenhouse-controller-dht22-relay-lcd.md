# 122 - Pico Greenhouse Controller DHT22 Relay LCD

Build a smart greenhouse controller that maintains optimal temperature and humidity ranges by controlling two independent relay outputs (heating and irrigation), while displaying all readings and relay states on an I2C LCD.

## Goal
Learn how to implement dual threshold control loops using a single DHT22 sensor, control two relay outputs independently based on temperature and humidity thresholds, and display multi-channel status on an I2C character LCD in MicroPython.

## What You Will Build
A dual-output greenhouse controller:
- **DHT22 Sensor (GP16)**: Reads temperature (°C) and humidity (%).
- **Relay 1 — Heater (GP10)**: Activates when temperature drops below the cold threshold.
- **Relay 2 — Irrigation (GP11)**: Activates when humidity drops below the dry threshold.
- **Buzzer (GP15)**: Sounds an alert when either threshold is in critical violation.
- **I2C 16x2 LCD (GP4, GP5)**: Displays temperature, humidity, and both relay states.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| DHT22 Sensor | `dht22` | Yes | Yes |
| Relay Module × 2 | `relay` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 | VCC (+) | 3.3V (3V3) | Red | Power line |
| DHT22 | GND (−) | GND | Black | Ground reference |
| DHT22 | DATA | GP16 | Yellow | 1-Wire data signal |
| Relay 1 (Heater) | IN | GP10 | Orange | Heating relay control |
| Relay 2 (Irrigation) | IN | GP11 | Blue | Irrigation relay control |
| Active Buzzer | VCC (+) | GP15 | Red | Critical alert output |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect DHT22 DATA to GP16. Connect Relay 1 to GP10 and Relay 2 to GP11. Connect the buzzer to GP15. Connect the LCD to GP4/GP5. All grounds are shared.

## Code
```python
from machine import Pin, I2C
import utime
import dht
from machine_lcd import I2cLcd

# Sensors and outputs
sensor    = dht.DHT22(Pin(16))
relay_heat = Pin(10, Pin.OUT)
relay_irr  = Pin(11, Pin.OUT)
buzzer     = Pin(15, Pin.OUT)

# Initialize I2C Bus 0 on GP4/GP5
i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

# Control thresholds
TEMP_COLD    = 18.0  # °C — activate heater below this
TEMP_HOT     = 30.0  # °C — critical high temperature
HUMID_DRY    = 40.0  # % — activate irrigation below this
HUMID_CRIT   = 20.0  # % — critical low humidity

relay_heat.value(0)
relay_irr.value(0)
buzzer.value(0)

lcd.clear()
lcd.move_to(0, 0)
lcd.putstr("Greenhouse Ctrl")
lcd.move_to(0, 1)
lcd.putstr("Initializing...")
utime.sleep(2)

print("Greenhouse controller active.")

while True:
    try:
        sensor.measure()
        temp_c = sensor.temperature()
        humid  = sensor.humidity()
        
        # --- Heater control (temperature) ---
        heat_on = (temp_c < TEMP_COLD)
        relay_heat.value(1 if heat_on else 0)
        
        # --- Irrigation control (humidity) ---
        irr_on = (humid < HUMID_DRY)
        relay_irr.value(1 if irr_on else 0)
        
        # --- Critical alarm ---
        critical = (temp_c > TEMP_HOT) or (humid < HUMID_CRIT)
        buzzer.value(1 if critical else 0)
        
        # --- LCD Update ---
        lcd.clear()
        lcd.move_to(0, 0)
        lcd.putstr("T:{:.1f}C H:{:.0f}%".format(temp_c, humid))
        lcd.move_to(0, 1)
        lcd.putstr("Heat:{} Irr:{}{}".format(
            "ON " if heat_on else "OFF",
            "ON " if irr_on else "OFF",
            "!!" if critical else "  "))
            
        print("T:{:.1f}C H:{:.0f}% | Heat:{} Irr:{} Crit:{}".format(
            temp_c, humid,
            "ON" if heat_on else "OFF",
            "ON" if irr_on else "OFF",
            critical))
            
    except OSError as e:
        lcd.clear()
        lcd.putstr("Sensor Error!")
        print(">> Read error:", e)
        
    utime.sleep(2)
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **DHT22**, **two Relays**, **Active Buzzer**, and **I2C LCD** onto the canvas.
2. Connect DHT22 to **GP16**, Relay 1 to **GP10**, Relay 2 to **GP11**, Buzzer to **GP15**, and LCD to **GP4/GP5**. Connect power and GND.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Slide the DHT22 temperature below 18°C to activate the heater relay. Slide humidity below 40% to activate irrigation.

## Expected Output
```
Greenhouse controller active.
T:16.0C H:38% | Heat:ON Irr:ON Crit:False
T:22.0C H:50% | Heat:OFF Irr:OFF Crit:False
T:32.0C H:15% | Heat:OFF Irr:ON Crit:True
```

## Expected Canvas Behavior
- The Relay 1 component closes when the temperature slider drops below 18°C. The Relay 2 component closes when the humidity slider drops below 40%. The buzzer sounds if temperature exceeds 30°C or humidity drops below 20%.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `heat_on = (temp_c < TEMP_COLD)` | Boolean threshold check: heater ON when temperature falls below the cold threshold. |
| `critical = (temp_c > TEMP_HOT) or (humid < HUMID_CRIT)` | OR gate: buzzer triggers if either critical condition is met. |

## Hardware & Safety Concept: Dual-Output Threshold Control
Each relay in this system is controlled by a single independent threshold comparison. This is equivalent to two parallel proportional bang-bang controllers operating on the same sensor data but different variables (temperature vs humidity). Adding **hysteresis** (separate ON and OFF thresholds) prevents relay chatter near the setpoint.

## Try This! (Challenges)
1. **Hysteresis Bands**: Change the heater logic to turn ON below 18°C and turn OFF above 20°C using two variables (`heater_on` state), preventing rapid ON/OFF cycling near the threshold.
2. **LCD Trend Arrows**: Compare current and previous readings and display a ↑/↓ arrow next to each value.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Relays chatter rapidly | Sensor readings oscillating at the threshold | Implement the hysteresis band improvement in the challenge above. |
| Buzzer sounds continuously | Critical thresholds set incorrectly | Adjust `TEMP_HOT` and `HUMID_CRIT` to values appropriate for your environment. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [114 - Pico Smart Fan DHT22 DC Motor LCD](114-pico-smart-fan-dht22-dc-motor-lcd.md)
- [103 - Pico Temperature Alarm HVAC](103-pico-temperature-alarm-hvac.md)
- [120 - Pico Multi-Sensor Alarm System LCD](120-pico-multi-sensor-alarm-system-lcd.md)
