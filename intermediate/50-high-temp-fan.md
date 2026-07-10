# 50 - High Temp Fan

Control cooling fan speed proportionally based on temperature readings from a DHT22 sensor.

## Goal
Learn how to read digital temperature float values, convert them into an integer scale, and drive a proportional PWM fan output.

## What You Will Build
As the temperature measured by the DHT22 sensor rises above 25C, the fan connected to D9 starts spinning. The fan speed increases proportionally until it reaches maximum speed at 35C.

**Why D2 and D9?** Pin D2 reads the digital sensor packets. Pin D9 is a PWM channel capable of varying fan motor speed.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| DHT22 Sensor | `dht22` | Yes | Yes |
| Fan | `fan` | Yes | Yes |
| NPN Transistor (e.g. PN2222) | `transistor` | Optional | Yes (critical driver) |
| Flyback Diode (e.g. 1N4007) | `diode` | Optional | Yes (critical protector) |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| DHT22 Sensor | VCC | 5V | Power supply (5V) |
| DHT22 Sensor | SDA | D2 | Data connection |
| DHT22 Sensor | GND | GND | Ground reference |
| Fan | + | D9 | Control pin (PWM) |
| Fan | - | GND | Ground reference |

> On physical hardware, a **transistor driver** is required to power the motor to prevent drawing too much current from the Arduino I/O pin.

## Code
```cpp
#include <DHT.h>

const int DHT_PIN = 2;
const int DHT_TYPE = DHT22;
const int FAN_PIN = 9; // Must be a PWM pin (3, 5, 6, 9, 10, 11)

// Temperature speed bounds
const int TEMP_MIN = 25; // Fan starts spinning (25C)
const int TEMP_MAX = 35; // Fan runs at max speed (35C)

DHT dht(DHT_PIN, DHT_TYPE);

void setup() {
  pinMode(FAN_PIN, OUTPUT);
  Serial.begin(9600);
  Serial.println("DHT22 Smart Fan Controller Ready");
  
  dht.begin();
}

void loop() {
  float tempC = dht.readTemperature();
  
  if (isnan(tempC)) {
    Serial.println("Error: Failed to read temperature!");
  } else {
    // Cast to integer to use map()
    int currentTemp = (int)tempC;
    
    // Scale temperature directly to PWM speed (0 to 255)
    int fanSpeed = map(currentTemp, TEMP_MIN, TEMP_MAX, 0, 255);
    
    // Constrain bounds to prevent math overflow
    fanSpeed = constrain(fanSpeed, 0, 255);
    
    analogWrite(FAN_PIN, fanSpeed);
    
    Serial.print("Temp: ");
    Serial.print(tempC);
    Serial.print("C | Fan PWM: ");
    Serial.println(fanSpeed);
  }
  
  delay(1500); // Poll rate delay
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **DHT22 Sensor**, and **Fan** onto the canvas.
2. Connect DHT22 **VCC** to Arduino **5V**, **SDA** to Arduino **D2**, and **GND** to Arduino **GND**.
3. Connect Fan **+** to Arduino **D9** (a PWM pin) and **-** to Arduino **GND**.
4. Paste the code into the editor.
5. Use the default Arduino interpreted runtime.
6. Click **Run**.
7. Open the **Terminal** tab in the bottom dock.
8. Double-click the DHT22 sensor, adjust the temperature slider between 25C and 35C, and watch the Fan speed change.

## Expected Output

Terminal:
```
DHT22 Smart Fan Controller Ready
Temp: 24.20C | Fan PWM: 0
Temp: 30.00C | Fan PWM: 127
Temp: 36.50C | Fan PWM: 255
...
```

### Expected Canvas Behavior

| Temperature State | Pin A0 Reading | mapped PWM Duty Cycle | Fan Speed / Spin |
| --- | --- | --- | --- |
| Cool (< 25C) | 24C | 0 | Stopped |
| Warm | 30C | 127 | Medium Speed |
| Hot (> 35C) | 36C | 255 | Full Speed |

The fan blades spin faster or slower based on the temperature slider.

## Code Walkthrough

| Line / Expression | What It Does |
| --- | --- |
| `int currentTemp = (int)tempC;` | Casts the decimal float value into a whole integer. The standard `map()` function only accepts integer parameters. |
| `map(currentTemp, TEMP_MIN, TEMP_MAX, 0, 255)` | Scales the temperature range directly to the 0-255 PWM duty cycle range. |

## Hardware & Safety Concept: Fan Control Applications
Varying the speed of cooling fans dynamically is key in modern PC coolers, automotive radiators, and home HVAC systems. Regulating speed proportionally based on thermal load reduces noise, cuts power usage, and minimizes wear on fan motor bearings compared to simple, continuous ON/OFF cycles.

## Try This! (Challenges)
1. **Minimum Startup Speed**: Many fans will hum but not spin below a PWM speed of 80 due to static friction. Add an `if` block that forces the speed to `80` if it is calculated to be between 1 and 80.
2. **Alert tone**: Add a Buzzer to D8 and sound a wailing alert if the temperature exceeds 40C (critical overheat warning).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Fan stays fully off or on | Non-PWM pin used | Verify that the Fan **+** pin connects to D9, not D13 or other non-PWM pin. |
| Speed doesn't update | Sensor reading failed | Check the raw float temperature printed in the Terminal to confirm it is not returning `NAN`. |

## Mode Notes
These patterns (DHT sensor float reads, float-to-int casting, map math, and PWM speed drive) are supported by MbedO interpreted mode.

## Related Projects
- [42 - Fan Speed PWM](42-fan-speed-pwm.md)
- [47 - Temp Humidity Serial](47-temp-humidity-serial.md)
- [48 - Temp Alarm](48-temp-alarm.md)
