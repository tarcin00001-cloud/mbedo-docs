# 196 - Pico Energy Safety Contactor Logger

Build a smart power contactor that monitors current loads using an ACS712 sensor, opens safety relays during overcurrent states, and writes breach event logs to flash.

## Goal
Learn how to read current sensor signals, calculate root-mean-square (RMS) current value bounds, trigger fast safety cut-offs, and append alert timestamps to the internal flash.

## What You Will Build
An overcurrent protection contactor:
- **ACS712 Current Sensor (GP26)**: Measures current draw of the electrical load.
- **Power Relay (GP10)**: Connects or disconnects power to the load (default ON/Closed).
- **Alarm Buzzer (GP11)**: Sounds yelp alerts during overcurrent lockouts.
- **Reset Button (GP13)**: Restores power after a safety trip.
- **SSD1306 OLED (GP4, GP5)**: Displays live current, peak current, and contactor state (NORMAL, TRIP, or LOCKED).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| ACS712 Current Sensor | `potentiometer` | Yes (represented by potentiometer) | Yes (5A or 20A ACS712) |
| Relay Module | `relay` | Yes | Yes (normally closed contactor) |
| Active Buzzer | `buzzer` | Yes | Yes |
| Tactile Push Button | `button` | Yes | Yes |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| ACS712 Sensor | OUT | GP26 | Yellow | Analog current sensor input |
| ACS712 Sensor | VCC / GND | 5V / GND | Red / Black | Power lines (requires 5V) |
| Power Relay | IN | GP10 | Orange | Load contactor switch pin |
| Active Buzzer | VCC (+) | GP11 | Blue | Alarm siren pin |
| Reset Button | Terminal 1 | GP13 | White | Contactor reset switch |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared Orange/Blue | Shared I2C Bus 0 |
| All Modules | GND | GND | Black | Shared return path to ground |

> **Wiring tip:** The ACS712 current sensor must be powered by 5V (VBUS) to achieve correct calibration scaling. Connect the sensor output to GP26. The OLED uses GP4/GP5.

## Code
```python
from machine import Pin, ADC, I2C
import utime, os, ssd1306

current_adc = ADC(26)
relay = Pin(10, Pin.OUT)
buzzer = Pin(11, Pin.OUT)
btn_reset = Pin(13, Pin.IN, Pin.PULL_UP)

# Keep contactor closed (power ON) initially
relay.value(1)
buzzer.value(0)

i2c = I2C(0, sda=Pin(4), scl=Pin(5), freq=400000)
oled = ssd1306.SSD1306_I2C(128, 64, i2c)

# Settings
CURRENT_LIMIT_AMPS = 3.0
ACS_OFFSET_V = 2.5      # Zero current reference voltage
ACS_SENSITIVITY = 0.185  # 185 mV/A for 5A ACS712 module

peak_current = 0.0
system_locked = False
log_file = "overloads.txt"

def read_current_amps():
    """Reads the current sensor and converts voltage to Amps."""
    # Perform oversampling (average of 50 samples to filter AC noise)
    total_val = 0
    for _ in range(50):
        total_val += current_adc.read_u16()
        utime.sleep_us(100)
    avg_val = total_val / 50
    
    # Convert ADC (0-65535) to Voltage (0-3.3V)
    voltage = avg_val * 3.3 / 65535.0
    
    # Calculate current from offset voltage
    # Current = (V_out - Offset) / Sensitivity
    current = (voltage - ACS_OFFSET_V) / ACS_SENSITIVITY
    
    # Filter minor noise fluctuations around zero
    if abs(current) < 0.15:
        current = 0.0
    return abs(current)

def log_breach(amps):
    """Appends an overcurrent event timestamp to the flash log file."""
    timestamp = utime.ticks_ms()
    try:
        with open(log_file, "a") as f:
            f.write("{},{:.2f}\n".format(timestamp, amps))
        print("Logged overcurrent breach: {} ms, {:.2f} A".format(timestamp, amps))
    except OSError as e:
        print("Flash logging failed:", e)

oled.fill(0)
oled.text("Safety Contactor", 4, 20)
oled.text("Online", 4, 36)
oled.show()
utime.sleep(1.5)

print("Contactor safety monitor active.")

while True:
    now = utime.ticks_ms()
    
    # 1. Read current load
    current_a = read_current_amps()
    peak_current = max(peak_current, current_a)
    
    # 2. Check for overcurrent breach
    if current_a > CURRENT_LIMIT_AMPS and not system_locked:
        system_locked = True
        relay.value(0)  # Open contactor immediately (cut power)
        print("!! OVERCURRENT DETECTED: {:.2f} A !!".format(current_a))
        log_breach(current_a)
        
    # 3. Handle Locked state
    if system_locked:
        # Pulse buzzer yelp siren
        buzzer.value(now // 250 % 2)
        status_str = "LOCKED"
        
        # Check for reset button click (GP13)
        if btn_reset.value() == 0:
            # Clear lock state
            system_locked = False
            peak_current = 0.0
            buzzer.value(0)
            relay.value(1)  # Close contactor (restore power)
            print("Contactor reset. Power restored.")
            oled.fill(0); oled.text("Restoring Power", 4, 28); oled.show()
            utime.sleep(1.0)
    else:
        buzzer.value(0)
        status_str = "NORMAL"
        
    # 4. Update OLED Display
    oled.fill(0)
    oled.rect(0, 0, 128, 14, 1)
    oled.text("POWER CONTACTOR", 4, 3, 0)
    
    oled.text("Status : {}".format(status_str), 10, 20, 1)
    oled.text("Current: {:.2f} A".format(current_a), 10, 32, 1)
    oled.text("Peak   : {:.2f} A".format(peak_current), 10, 44, 1)
    oled.text("Limit  : {:.2f} A".format(CURRENT_LIMIT_AMPS), 10, 56, 1)
    
    oled.rect(0, 0, 128, 64, 1)
    oled.show()
    
    utime.sleep_ms(100)
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **ACS712 Sensor** (potentiometer), **Relay**, **Buzzer**, **Push Button**, and **SSD1306 OLED** onto the canvas.
2. Connect ACS712 to **GP26**, Relay to **GP10**, Buzzer to **GP11**, Button to **GP13**, and OLED to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Slide the current potentiometer control. Slide it to maximum to exceed 3.0A. Verify that the relay opens immediately (turns OFF) and the buzzer sounds.
5. Slide the current back to 0A. Verify that the system remains locked out until you click the reset button on GP13.

## Expected Output
Terminal:
```
Contactor safety monitor active.
!! OVERCURRENT DETECTED: 3.42 A !!
Logged overcurrent breach: 5840 ms, 3.42 A
Contactor reset. Power restored.
```

## Expected Canvas Behavior
* Normal state: Relay is ON. OLED reads `Status: NORMAL`.
* Slide Current potentiometer high: Relay turns OFF immediately. Buzzer sounds. OLED reads `Status: LOCKED`.
* Slide potentiometer low: Status remains `LOCKED` until Reset Button (GP13) is pressed.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `avg_val * 3.3 / 65535.0` | Converts the averaged 16-bit ADC value to equivalent analog voltage (0 to 3.3V). |
| `(voltage - ACS_OFFSET_V) / ACS_SENSITIVITY` | Maps voltage deviation from zero-offset (2.5V) to current in Amps using ACS712 sensitivity. |

## Hardware & Safety Concept: Overcurrent Lockout Controls
Standard circuit breakers automatically trip when current limits are breached, but they require manual mechanical reset. Similarly, electronic safety contactors use **software latching lockout**: if an overcurrent occurs, power is cut immediately, and it is strictly forbidden for the software to automatically restore power once the current drops to zero. A physical operator must inspect the load and press a manual reset button to verify that the fault is cleared before closing the contactor again.

## Try This! (Challenges)
1. **Dynamic Limit Adjustment**: Connect a potentiometer on GP27 and use it to set the current limit threshold dynamically from 1.0A to 5.0A.
2. **Clear Log Command**: Monitor UART inputs. If the user types the string `"CLEAR"`, delete the `overloads.txt` log file from the flash filesystem.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Current readings fluctuate wildly around zero | Electrical noise | Increase the oversampling loop count (e.g. from 50 to 100 samples) or increase the noise filtering threshold (`abs(current) < 0.20`). |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**.

## Related Projects
- [125 - Pico Energy Safety Monitor ACS712 Relay LCD](../advanced/125-pico-energy-safety-monitor-acs712-relay-lcd.md)
- [157 - Pico PWM Fan Controller Tachometer LCD](../advanced/157-pico-pwm-fan-controller-tachometer-lcd.md)
