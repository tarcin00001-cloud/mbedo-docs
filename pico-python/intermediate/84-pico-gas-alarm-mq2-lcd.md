# 84 - Pico Gas Alarm MQ2 LCD

Build a gas leak detector panel that displays smoke/gas concentration percentages on an I2C LCD and sounds a warning buzzer when safety thresholds are exceeded.

## Goal
Learn how to read analog inputs from gas sensors, calculate gas concentration percentages, implement latching alarm states, output status text to an I2C character LCD, and control warning sirens in MicroPython.

## What You Will Build
A gas safety control panel:
- **MQ-2 Gas Sensor (GP26)**: Measures gas/smoke concentrations.
- **Active Buzzer (GP15)**: Sounds a pulsing siren when gas is detected.
- **Reset Button (GP13)**: Silences the alarm once the air is clean.
- **I2C 16x2 LCD (GP4, GP5)**: Displays the calculated gas percentage and active alarm state ("GAS ALERT!" or "AIR: CLEAN").

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| MQ-2 Gas Sensor Module | `potentiometer` | Yes (represented by potentiometer slider) | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Tactile Push Button | `button` | Yes (reset switch) | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MQ-2 Sensor Board | VCC | 5V (VBUS) | Red | MQ sensors require 5V heater power |
| MQ-2 Sensor Board | GND | GND | Black | Ground reference |
| MQ-2 Sensor Board | AO (Analog Out) | GP26 | Yellow | Analog input signal |
| Reset Button | Terminal 1 | GP13 | White | Reset input (reads LOW on press) |
| Reset Button | Terminal 2 | GND | Black | Shorts GP13 to GND |
| Active Buzzer | VCC (+) | GP15 | Red | Alarm output |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C screen bus |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** Connect the MQ-2 sensor's VCC to the Pico's 5V VBUS pin. Connect the analog output to GP26, and the reset button to GP13. Connect the active buzzer to GP15 and GND. All grounds are shared.

## Code
```python
from machine import Pin, ADC, I2C
import utime
from machine_lcd import I2cLcd

mq2_analog = ADC(26) # GP26 = ADC channel 0
btn_reset  = Pin(13, Pin.IN, Pin.PULL_UP)
buzzer     = Pin(15, Pin.OUT)

# Initialize I2C Bus 0 on GP4/GP5
i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
lcd = I2cLcd(i2c, 0x27, 2, 16)

# Calibration bounds (raw ADC scale 0 - 65535)
SENSOR_MIN = 8000
SENSOR_MAX = 50000

# Control limits
GAS_LIMIT = 22000 # Trigger alarm if gas level > 22000

alarm_state = False
buzzer.value(0)

lcd.clear()
lcd.putstr("Gas Safety Panel")
lcd.move_to(0, 1)
lcd.putstr("Sensor Warmup...")
utime.sleep(2) # Initial warmup delay

while True:
    raw = mq2_analog.read_u16()
    reset_pressed = (btn_reset.value() == 0)
    
    # Calculate concentration percentage
    percent = (raw - SENSOR_MIN) * 100 // (SENSOR_MAX - SENSOR_MIN)
    if percent < 0:
        percent = 0
    elif percent > 100:
        percent = 100
        
    # Check for trigger (Set)
    if raw > GAS_LIMIT and not alarm_state:
        alarm_state = True
        print(">> WARNING: Gas leak detected! Alarm active.")
        
    # Check for disarm (Reset)
    if reset_pressed and alarm_state:
        alarm_state = False
        buzzer.value(0) # Silence immediately
        print(">> Gas alarm reset.")
        utime.sleep(1) # Lockout delay
        
    # Update LCD Display
    lcd.clear()
    lcd.move_to(0, 0)
    if alarm_state:
        lcd.putstr("!! GAS ALERT !!")
        lcd.move_to(0, 1)
        lcd.putstr("Gas Level: " + str(percent) + " %")
    else:
        lcd.putstr("Air: CLEAN")
        lcd.move_to(0, 1)
        lcd.putstr("Gas Lvl: " + str(percent) + " %")
        
    # Pulsing safety siren
    if alarm_state:
        buzzer.value(1)
        utime.sleep_ms(150)
        buzzer.value(0)
        utime.sleep_ms(150)
    else:
        utime.sleep_ms(200) # Normal update speed
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Potentiometer** (representing gas sensor), **Push Button** (representing reset), **Active Buzzer**, and **I2C LCD** onto the canvas.
2. Connect Gas Sensor to **GP26**, Reset Button to **GP13**, Buzzer to **GP15**, and LCD to **GP4/GP5**. Connect all grounds.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Slide the gas sensor to maximum to trigger the alarm. Click Reset to disarm and clear the display.

## Expected Output
```
>> WARNING: Gas leak detected! Alarm active.
>> Gas alarm reset.
```
(On screen: "Air: CLEAN / Gas Lvl: X %" changing to "!! GAS ALERT !! / Gas Level: Y %" when triggered.)

## Expected Canvas Behavior
- The buzzer component on the canvas pulses active and the LCD screen displays a warning message when the gas sensor slider is moved past 22000.
- Clicking the Reset Button returns the LCD screen to the clean message.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `(raw - SENSOR_MIN) * 100 // ...` | Scales the raw analog value into a concentration percentage based on the sensor calibration limits. |
| `lcd.putstr("!! GAS ALERT !!")` | Renders a high-priority warning message on the LCD display buffer. |

## Hardware & Safety Concept: Industrial Gas Leak Detectors
Gas leak detectors are deployed in harsh industrial environments where they are exposed to varying temperatures and humidity. Since environmental changes can cause the sensor's baseline reading to drift, industrial systems implement automatic baseline tracking algorithms that slowly adjust the calibration bounds over time, preventing false alarms.

## Try This! (Challenges)
1. **Exhaust Fan Relay**: Connect an LED/Relay on GP14 that turns ON (simulating a ventilation fan) when the alarm is active.
2. **Visual strobe**: Connect a Red LED on GP12 to flash in sync with the buzzer siren.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm triggers constantly on boot | Sensor heating phase | Allow the sensor to run for a few minutes to heat up and stabilize. |
| Reset button does not disarm | Button pin mismatch | Verify the reset button is wired to GP13 as configured in the code. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [51 - Pico LCD Hello World](51-pico-lcd-hello-world.md)
- [73 - Pico Gas Alarm MQ2 Buzzer](73-pico-gas-alarm-mq2-buzzer.md)
- [83 - Pico Flame Alarm LCD](83-pico-flame-alarm-lcd.md)
