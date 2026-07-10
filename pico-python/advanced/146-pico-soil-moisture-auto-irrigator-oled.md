# 146 - Pico Soil Moisture Auto Irrigator OLED

Build an automatic plant irrigation system that reads a capacitive soil moisture sensor, activates a water pump relay when the soil is dry, tracks watering history, and displays moisture level and pump status on an SSD1306 OLED.

## Goal
Learn how to read an analog soil moisture sensor, implement hysteresis-based irrigation control to prevent pump chatter, track total watering time, and display a graphical moisture bar and event log on an SSD1306 OLED in MicroPython.

## What You Will Build
An automated plant irrigator:
- **Soil Moisture Sensor (GP26)**: Reads volumetric water content as an analog voltage.
- **Water Pump Relay (GP10)**: Activates to water the plant when soil is dry.
- **LED (GP13)**: ON while pump is running.
- **SSD1306 OLED (GP4, GP5)**: Displays moisture percentage, pump state, and cumulative watering time.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Capacitive Soil Moisture Sensor | `potentiometer` | Yes (slider simulates moisture) | Yes |
| Relay Module (Pump control) | `relay` | Yes | Yes |
| LED | `led` | Yes | Yes |
| SSD1306 OLED Display (128×64) | `oled` | Yes | Yes |
| 330 Ω Resistor | `resistor` | Optional in MbedO | Yes (LED current limit) |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Soil Moisture Sensor | VCC | 3.3V (3V3) | Red | Sensor power |
| Soil Moisture Sensor | GND | GND | Black | Ground reference |
| Soil Moisture Sensor | AOUT | GP26 | Yellow | Analog moisture output |
| Relay Module | IN | GP10 | Orange | Pump relay control signal |
| LED | Anode (+, via 330 Ω) | GP13 | Green | Pump running indicator |
| LED | Cathode (−) | GND | Black | Ground return |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C Bus 0 |
| SSD1306 OLED | VCC / GND | 3.3V / GND | Red / Black | OLED power |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Capacitive soil sensors output higher voltage in dry soil and lower voltage in wet soil (inverted from resistive sensors). Connect sensor AOUT to GP26. Connect relay to GP10 and LED to GP13. Connect OLED to GP4/GP5. All grounds are shared.

## Code
```python
from machine import Pin, ADC, I2C
import utime, ssd1306

soil_adc = ADC(26)
relay    = Pin(10, Pin.OUT)
pump_led = Pin(13, Pin.OUT)

i2c  = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
oled = ssd1306.SSD1306_I2C(128, 64, i2c)

relay.value(0)
pump_led.value(0)

# Capacitive sensor: dry = high ADC, wet = low ADC
# Calibrate these values for your specific sensor:
DRY_RAW = 52000   # ADC reading in dry air
WET_RAW = 18000   # ADC reading submerged in water

# Hysteresis thresholds (moisture %)
PUMP_ON_BELOW  = 30   # Start watering when below 30%
PUMP_OFF_ABOVE = 55   # Stop watering when above 55%

# State
pump_on         = False
total_water_s   = 0       # Cumulative pump run seconds
pump_start_tick = 0
water_events    = 0

def raw_to_pct(raw):
    pct = (DRY_RAW - raw) * 100 // (DRY_RAW - WET_RAW)
    return max(0, min(100, pct))

def draw_oled(moisture_pct):
    oled.fill(0)
    oled.text("SOIL IRRIGATOR", 0, 0)
    oled.hline(0, 10, 128, 1)

    # Moisture bar
    oled.text("Moisture:", 0, 14)
    oled.text("{:3d}%".format(moisture_pct), 100, 14)
    bar = int(moisture_pct * 112 / 100)
    oled.rect(8, 26, 112, 10, 1)
    oled.fill_rect(8, 26, bar, 10, 1)

    # Pump status
    pump_label = "PUMP: ON " if pump_on else "PUMP: OFF"
    oled.text(pump_label, 0, 40)

    # Stats
    oled.text("Watered: {}s x{}".format(total_water_s, water_events), 0, 52)
    oled.show()

oled.fill(0); oled.text("Irrigator Init", 0, 28); oled.show()
utime.sleep(1)
print("Auto irrigator active.")

SAMPLE_MS = 1000
last_sample = utime.ticks_ms() - SAMPLE_MS

while True:
    now = utime.ticks_ms()

    if utime.ticks_diff(now, last_sample) >= SAMPLE_MS:
        raw      = soil_adc.read_u16()
        moisture = raw_to_pct(raw)
        last_sample = now

        # Hysteresis pump control
        if not pump_on and moisture < PUMP_ON_BELOW:
            pump_on       = True
            pump_start_tick = utime.ticks_ms()
            water_events += 1
            relay.value(1); pump_led.value(1)
            print("Pump ON. Moisture: {}%".format(moisture))

        elif pump_on and moisture > PUMP_OFF_ABOVE:
            pump_on = False
            elapsed = utime.ticks_diff(utime.ticks_ms(), pump_start_tick) // 1000
            total_water_s += elapsed
            relay.value(0); pump_led.value(0)
            print("Pump OFF. Ran {}s. Total: {}s".format(elapsed, total_water_s))

        draw_oled(moisture)
        print("Moisture: {}% | Pump: {}".format(moisture, "ON" if pump_on else "OFF"))

    utime.sleep_ms(100)
```

## What to Click in MbedO
1. Drag **Pico**, **Potentiometer** (moisture sensor), **Relay**, **LED**, and **SSD1306 OLED** onto the canvas.
2. Connect Potentiometer to **GP26**, Relay to **GP10**, LED to **GP13**, OLED to **GP4/GP5**. Connect power and GND.
3. Paste code, click **Run**. Slide the potentiometer down (dry) to activate the pump relay. Slide it up (wet) to stop.

## Expected Output
```
Auto irrigator active.
Moisture: 22% | Pump: OFF
Pump ON. Moisture: 22%
Moisture: 58% | Pump: ON
Pump OFF. Ran 5s. Total: 5s
```

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `(DRY_RAW - raw) * 100 // (DRY_RAW - WET_RAW)` | Maps raw ADC to moisture %; inverted because capacitive sensors read high voltage in dry conditions. |
| `PUMP_ON_BELOW = 30` / `PUMP_OFF_ABOVE = 55` | Hysteresis band: pump activates at 30% but only deactivates at 55%, preventing rapid on/off cycling. |

## Hardware & Safety Concept: Capacitive vs Resistive Soil Sensors
Resistive sensors pass current through the soil, causing electrolytic corrosion of the probes. Capacitive sensors measure the dielectric constant of the soil without passing current, giving a much longer lifespan. However, capacitive sensors require calibration for specific soil types as mineral content affects the dielectric constant.

## Try This! (Challenges)
1. **Watering Schedule**: Add a configurable maximum watering duration — if the pump runs for more than 60 seconds, stop automatically to prevent flooding.
2. **Moisture Log**: Store the hourly average moisture to EEPROM for a 24-hour history review.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Pump never turns ON | DRY_RAW calibration wrong | Print raw ADC values in genuinely dry soil and update `DRY_RAW`. |
| Pump cycles ON/OFF rapidly | Hysteresis gap too small | Increase the gap between `PUMP_ON_BELOW` and `PUMP_OFF_ABOVE`. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [130 - Pico Water Level Station LCD Buzzer](130-pico-water-level-station-lcd-buzzer.md)
- [122 - Pico Greenhouse Controller DHT22 Relay LCD](122-pico-greenhouse-controller-dht22-relay-lcd.md)
