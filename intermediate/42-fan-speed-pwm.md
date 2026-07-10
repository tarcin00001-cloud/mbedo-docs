# 42 - Fan Speed PWM

Regulate a cooling fan's speed proportionally based on thermistor temperature readings.

## Goal
Learn how to read an analog sensor (NTC Thermistor), translate it into a proportional PWM output duty cycle (0-255) using `map()`, and drive a DC cooling fan speed.

## What You Will Build
As the temperature rises (NTC raw value drops), the fan connected to pin D9 spins faster. When the temperature falls below a set threshold, the fan stops completely to save energy.

**Why A0 and D9?** Pin A0 measures the temperature analog voltage. Pin D9 is a PWM output pin capable of varying average output power to control fan speed.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| NTC Thermistor | `ntc_temp` | Yes | Yes |
| Fan | `fan` | Yes | Yes |
| NPN Transistor (e.g. PN2222) | `transistor` | Optional | Yes (critical driver) |
| Flyback Diode (e.g. 1N4007) | `diode` | Optional | Yes (critical protector) |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| Thermistor | VCC | 5V | Power supply (5V) |
| Thermistor | OUT | A0 | Analog signal connection |
| Thermistor | GND | GND | Ground reference |
| Fan | + (Positive) | D9 | Control pin (PWM) |
| Fan | - (Negative) | GND | Ground reference |

> On physical hardware, **never** connect a motor directly to an Arduino output pin. You must use a transistor (like a PN2222 or TIP120) as a electronic switch to handle the current load, along with a flyback diode in parallel with the motor.

## Code
```cpp
const int NTC_PIN = A0;
const int FAN_PIN = 9; // Must be a PWM pin (3, 5, 6, 9, 10, 11)

// Temperature limits (NTC scale: lower = hotter)
const int COLD_TEMP = 600; // Fan off threshold (e.g., 20C)
const int HOT_TEMP  = 300; // Fan max speed threshold (e.g., 40C)

void setup() {
  pinMode(FAN_PIN, OUTPUT);
  Serial.begin(9600);
  Serial.println("Smart Fan Controller Ready");
}

void loop() {
  int tempRaw = analogRead(NTC_PIN);
  
  // Calculate proportional fan speed (higher speed when hotter/lower raw value)
  int fanSpeed = map(tempRaw, COLD_TEMP, HOT_TEMP, 0, 255);
  
  // Keep speed within valid PWM bounds (0 to 255)
  fanSpeed = constrain(fanSpeed, 0, 255);
  
  analogWrite(FAN_PIN, fanSpeed);
  
  Serial.print("Temp Raw: ");
  Serial.print(tempRaw);
  Serial.print(" | Fan PWM Speed: ");
  Serial.println(fanSpeed);
  
  delay(200);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **NTC Thermistor**, and **Fan** onto the canvas.
2. Connect Thermistor **VCC** to Arduino **5V**, **OUT** to Arduino **A0**, and **GND** to Arduino **GND**.
3. Connect Fan **+** to Arduino **D9** (a PWM pin) and **-** to Arduino **GND**.
4. Paste the code into the editor.
5. Use the default Arduino interpreted runtime.
6. Click **Run**.
7. Open the **Terminal** tab in the bottom dock.
8. Double-click the Thermistor, adjust the temperature slider, and watch the Fan speed change.

## Expected Output

Terminal:
```
Smart Fan Controller Ready
Temp Raw: 650 | Fan PWM Speed: 0
Temp Raw: 450 | Fan PWM Speed: 127
Temp Raw: 250 | Fan PWM Speed: 255
...
```

### Expected Canvas Behavior

| Temperature State | Pin A0 Reading | mapped PWM Duty Cycle | Fan Speed / Spin |
| --- | --- | --- | --- |
| Cold | > 600 | 0 | Stopped |
| Warm | 450 | 127 | Medium Speed |
| Hot | < 300 | 255 | Full Speed |

The fan blades spin faster or slower depending on the temperature slider.

## Code Walkthrough

| Line / Expression | What It Does |
| --- | --- |
| `map(tempRaw, COLD_TEMP, HOT_TEMP, 0, 255)` | Scales the temperature raw input range down to the 0-255 PWM duty cycle. Notice the inversion: the input range goes from cold (600) to hot (300), mapping to 0 (off) and 255 (max speed). |
| `constrain(fanSpeed, 0, 255)` | Restricts the speed to valid PWM output values, preventing overflow issues if the reading goes outside our expected scale limits. |

## Hardware & Safety Concept: Transistor Switched Loads
DC motors and fans pull considerable current (often 200 mA to 1A+).
- Attempting to power a motor directly from an Arduino I/O pin (which has a limit of **40 mA**) will overheat the internal output circuitry and permanently damage the microchip.
- An **NPN transistor** acts as a current amplifier. A tiny control current from pin D9 turns the transistor ON, allowing a much larger current to flow from the power supply through the fan to Ground.

## Try This! (Challenges)
1. **Change Response Curve**: Change the `COLD_TEMP` and `HOT_TEMP` bounds to make the fan start spinning only under extreme heat.
2. **Audible Alert**: Add a Buzzer to D8 and trigger a warning tone if the temperature raw value falls below `200` (max heat warning).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Fan stays fully off or on | Non-PWM pin used | Verify that the Fan **+** pin is wired to D9 (a PWM pin). |
| Fan makes humming sound but won't spin | PWM value too low | Low duty cycles (e.g. < 50) don't supply enough voltage to overcome motor friction. You can add code to enforce a minimum startup speed (e.g., if speed > 0 && speed < 80, speed = 80). |

## Mode Notes
These patterns (analog sensor mapping, limits checking with `constrain`, and PWM analogWrite speed control) are supported by MbedO interpreted mode.

## Related Projects
- [20 - Thermistor Raw Print](../beginner/20-thermistor-raw-print.md)
- [41 - Thermostat Relay](41-thermostat-relay.md)
