# 194 - Pico Greenhouse Automation Hub

Build a fully automated greenhouse environmental station that monitors climate and water reservoirs, controls heating and irrigation relays, and alerts on low-water states.

## Goal
Learn how to pool analog level and digital climate sensors, coordinate dual high-load relays with hysteresis-like safety triggers, and construct multi-page LCD dashboards in MicroPython.

## What You Will Build
An automated greenhouse terminal:
- **DHT22 Climate Sensor (GP14)**: Measures ambient temperature and humidity.
- **Water Level Sensor (GP26)**: Monitors the volume of the water reservoir (0 to 100%).
- **Relay 1 (Pump Control, GP10)**: Triggers an irrigation pump if soil is dry and water is available.
- **Relay 2 (Heater Control, GP11)**: Activates a heating lamp if temperature drops.
- **Active Buzzer (GP12)**: Sounds a pulsing alarm if the water reservoir is empty (< 20%).
- **Manual Pump Button (GP13)**: Overrides automation to run the pump manually.
- **16x2 I2C LCD (GP4, GP5)**: Displays temperature, humidity, water levels, and active relay outputs.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| DHT22 Sensor Module | `dht22` | Yes | Yes |
| Water Level Sensor | `potentiometer` | Yes (represented by potentiometer) | Yes |
| Relay Modules | `relay` | Yes (two relays) | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Tactile Push Button | `button` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |
| 10k-ohm Resistors | `resistor` | Optional | Yes (voltage divider) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 | DATA | GP14 | Yellow | Climate sensor data |
| DHT22 | VCC / GND | 3.3V / GND | Red / Black | Power lines |
| Water Sensor | Signal OUT | GP26 | Green | Reservoir depth analog input |
| Water Sensor | VCC / GND | 3.3V / GND | Red / Black | Power lines |
| Relay 1 (Pump) | IN | GP10 | Orange | Water pump control |
| Relay 2 (Heater) | IN | GP11 | Blue | Heating lamp control |
| Active Buzzer | VCC (+) | GP12 | Purple | Dry reservoir warning buzzer |
| Manual Button | Terminal 1 | GP13 | White | Manual pump override |
| Manual Button | Terminal 2 | GND | Black | Ground return |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C Bus 0 |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Both relays and the buzzer connect to GP10, GP11, and GP12. The manual button uses GP13 with an internal pull-up. The LCD uses GP4/GP5.

## Code
```python
from machine import Pin, ADC, I2C
import utime, dht
from machine_lcd import I2cLcd

# Sensors
dht_sensor = dht.DHT22(Pin(14))
water_adc = ADC(26)

# Actuators
relay_pump = Pin(10, Pin.OUT)
relay_heat = Pin(11, Pin.OUT)
buzzer = Pin(12, Pin.OUT)

relay_pump.value(0)
relay_heat.value(0)
buzzer.value(0)

btn_manual = Pin(13, Pin.IN, Pin.PULL_UP)

i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

# Threshold limits
TEMP_LOW_LIMIT = 20.0     # Turn ON heater if temp < 20°C
HUMID_LOW_LIMIT = 40.0    # Turn ON pump if humidity < 40% (simulating dry soil)
WATER_EMPTY_LIMIT = 20.0  # Alert if reservoir water < 20%

manual_override = False
last_btn = 0

lcd.clear()
lcd.putstr("Greenhouse Hub\nOnline & Active")
utime.sleep(1.5)

print("Greenhouse controller active.")

while True:
    now = utime.ticks_ms()
    
    # 1. Read DHT22
    try:
        dht_sensor.measure()
        temp = dht_sensor.temperature()
        humid = dht_sensor.humidity()
        sensor_ok = True
    except OSError:
        temp, humid = 0.0, 0.0
        sensor_ok = False
        
    # 2. Read Water Level (convert 0-65535 to 0-100%)
    raw_water = water_adc.read_u16()
    water_pct = raw_water * 100 // 65536
    
    # 3. Handle Manual Override Button (GP13)
    if btn_manual.value() == 0 and utime.ticks_diff(now, last_btn) > 400:
        manual_override = not manual_override
        last_btn = now
        print("Manual Pump Override:", manual_override)
        
    # 4. Automation Control Logic
    # Dry reservoir alarm: sounds buzzer, overrides pump OFF for safety
    if water_pct < WATER_EMPTY_LIMIT:
        # Pulse buzzer
        buzzer.value(1)
        utime.sleep_ms(80)
        buzzer.value(0)
        
        # Force pump OFF to prevent dry-running damage
        relay_pump.value(0)
        pump_state = "ERR:DRY"
    else:
        buzzer.value(0)
        # Pump control
        if manual_override:
            relay_pump.value(1)
            pump_state = "FORCE"
        else:
            if sensor_ok and humid < HUMID_LOW_LIMIT:
                relay_pump.value(1)
                pump_state = "AUTO:ON"
            else:
                relay_pump.value(0)
                pump_state = "AUTO:OFF"
                
    # Heater control
    if sensor_ok and temp < TEMP_LOW_LIMIT:
        relay_heat.value(1)
        heat_state = "ON "
    else:
        relay_heat.value(0)
        heat_state = "OFF"
        
    # 5. Update LCD Display
    lcd.clear()
    if sensor_ok:
        lcd.putstr("T:{:.1f}C H:{:.0f}%\n".format(temp, humid))
    else:
        lcd.putstr("SENSOR ERROR\n")
        
    lcd.putstr("W:{}% P:{} H:{}".format(water_pct, pump_state[:5], heat_state))
    
    print("T:{:.1f}C H:{:.0f}% | Water:{}% | Pump:{} | Heater:{}".format(
        temp, humid, water_pct, pump_state, heat_state
    ))
    
    utime.sleep_ms(300)
```

## What to Click in MbedO
1. Drag **Pico**, **DHT22**, **Water Level Sensor** (potentiometer), **two Relays**, **Buzzer**, **Button**, and **I2C LCD** onto the canvas.
2. Connect DHT22 to **GP14**, Water Level to **GP26**, Relays to **GP10/GP11**, Buzzer to **GP12**, Button to **GP13**, and LCD to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Slide the DHT22 temperature slider below 20°C. Observe the heater relay close.
5. Slide the water reservoir potentiometer below 20%. Verify that the buzzer beeps and the pump shuts off.

## Expected Output
Terminal:
```
Greenhouse controller active.
T:24.5C H:50% | Water:85% | Pump:AUTO:OFF | Heater:OFF
T:18.2C H:50% | Water:85% | Pump:AUTO:OFF | Heater:ON
T:18.2C H:35% | Water:15% | Pump:ERR:DRY | Heater:ON
```

## Expected Canvas Behavior
* Normal state (temp > 20°C, humid > 40%, water > 20%): Relays are OFF.
* Temperature falls below 20.0°C: Relay 2 (GP11) turns ON. LCD shows `H:ON`.
* Humidity falls below 40.0°C: Relay 1 (GP10) turns ON. LCD shows `P:AUTO:ON`.
* Water level falls below 20%: Buzzer beeps, Relay 1 turns OFF. LCD shows `P:ERR:D`.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `raw_water * 100 // 65536` | Converts the 16-bit analog voltage reading to a percentage (0 to 100%). |
| `relay_pump.value(0)` | Safety shutoff: cuts power to the pump relay immediately to prevent dry-running damage. |

## Hardware & Safety Concept: Dry-Run Pump Protection
Water pumps rely on the flowing liquid to cool and lubricate their internal impellers and seals. If a pump runs dry (operating without liquid inside the chamber), the mechanical friction causes rapid heat buildup, melting plastic components and destroying the motor windings within seconds. High-reliability irrigation controllers always monitor reservoir levels and hardcode a **low-level cutoff** to isolate the pump relay during empty states.

## Try This! (Challenges)
1. **Dynamic Fan Ventilation**: Add a third relay on GP15 and turn ON an exhaust fan if the greenhouse temperature exceeds 35°C (overheat cooling).
2. **Alert Log**: Log every low-water event to the Pico's internal flash memory as a CSV timestamp log.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Pump oscillates ON and OFF repeatedly | Water level hovering near 20% | Add a hysteresis band to the water check: trigger alarm below 20%, but only clear it once the water rises back above 25%. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [122 - Pico Greenhouse Controller DHT22 Relay LCD](../advanced/122-pico-greenhouse-controller-dht22-relay-lcd.md)
- [146 - Pico Soil Moisture Auto Irrigator OLED](../advanced/146-pico-soil-moisture-auto-irrigator-oled.md)
