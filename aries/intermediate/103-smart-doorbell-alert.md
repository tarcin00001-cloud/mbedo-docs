# 103 - Smart Doorbell Alert

Build a motion-activated smart doorbell system using a PIR sensor, an active buzzer for chimes, and an I2C LCD screen on the VEGA ARIES v3 board.

## Goal
Learn how to interface passive infrared (PIR) motion sensors, implement state-driven screen updates, and trigger output sound chimes using simple non-blocking state comparisons.

## What You Will Build
An automated security doorbell. When a visitor approaches and triggers the PIR sensor, the system sounds a double-tone "Ding-Dong" chime and updates the LCD display to alert the household of a visitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| PIR Motion Sensor | `pir` | Yes | Yes |
| Active Buzzer (Chime) | `buzzer` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| PIR Sensor | VCC | 3V3 | Red | Sensor operating voltage (3.3V) |
| PIR Sensor | GND | GND | Black | Ground reference |
| PIR Sensor | OUT | GPIO 17 | Green | Digital output signal (Active HIGH) |
| Active Buzzer | VCC | GPIO 14 | Orange | Output high triggers chime |
| Active Buzzer | GND | GND | Black | Ground reference |
| I2C LCD | VCC | 5V | Red | LCD power supply (5V) |
| I2C LCD | GND | GND | Black | Ground reference |
| I2C LCD | SDA | SDA0 (GP17) | Blue | I2C Data line |
| I2C LCD | SCL | SCL0 (GP16) | Yellow | I2C Clock line |

> **Wiring tip:** PIR sensors typically have adjustable potentiometers on the bottom of the board for sensitivity and trigger delay time. Set the delay to its minimum (turned fully counter-clockwise) for rapid sensor polling.

## Code
```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int PIR_PIN = 17;      // PIR motion sensor on GPIO 17
const int CHIME_PIN = 14;    // Active buzzer on GPIO 14

LiquidCrystal_I2C lcd(0x27, 16, 2);
int lastPirState = -1;

void setup() {
  Wire.begin();
  
  pinMode(PIR_PIN, INPUT);
  pinMode(CHIME_PIN, OUTPUT);
  digitalWrite(CHIME_PIN, LOW); // Buzzer off

  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Smart Doorbell");
  lcd.setCursor(0, 1);
  lcd.print("System Ready");
}

void loop() {
  // Read state: 1 = Motion detected, 0 = Idle
  int pirState = digitalRead(PIR_PIN);

  // Update only on state change to prevent constant display updates and loops
  if (pirState != lastPirState) {
    if (pirState == HIGH) {
      lcd.setCursor(0, 1);
      lcd.print("Visitor Alert!  ");
      
      // Play a dual-tone "Ding-Dong" chime using active buzzer control
      digitalWrite(CHIME_PIN, HIGH);
      delay(200);
      digitalWrite(CHIME_PIN, LOW);
      delay(100);
      digitalWrite(CHIME_PIN, HIGH);
      delay(350);
      digitalWrite(CHIME_PIN, LOW);
    } else {
      lcd.setCursor(0, 1);
      lcd.print("System Ready    ");
    }
    lastPirState = pirState;
  }
  delay(100); // Polling check delay
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **PIR Motion Sensor**, **Active Buzzer**, and **I2C LCD Display** components onto the canvas.
2. Wire the PIR Sensor: **VCC** to **3V3**, **GND** to **GND**, and **OUT** to **GPIO 17**.
3. Wire the Active Buzzer: **VCC** to **GPIO 14** and **GND** to **GND**.
4. Wire the LCD: **VCC** to **5V**, **GND** to **GND**, **SDA** to **SDA0 (GP17)**, and **SCL** to **SCL0 (GP16)**.
5. Paste the code into the editor.
6. Select **Interpreted Mode** in the simulation dropdown.
7. Click **Run** to execute the project.

## Expected Output
Serial Monitor:
```
System Initialized.
PIR Sensor: MOTION DETECTED
PIR Sensor: IDLE
```

## Expected Canvas Behavior
* Clicking the simulated PIR motion sensor triggers a motion alert.
* The LCD screen immediately changes row 2 to read `Visitor Alert!`.
* The active buzzer sounds a "Ding-Dong" chime.
* After the simulated motion times out, the LCD returns to showing `System Ready`.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(PIR_PIN, INPUT)` | Configures GP17 to receive digital logic levels from the PIR sensor. |
| `digitalRead(PIR_PIN)` | Samples the PIR output pin (HIGH indicates motion). |
| `pirState != lastPirState` | Evaluates if the state has changed to prevent repeating the chime. |
| `digitalWrite(CHIME_PIN, HIGH)` | Drives GPIO 14 high to sound the active buzzer. |

## Hardware & Safety Concept
* **PIR Sensor Warm-up**: Standard PIR sensors require a initialization warm-up period of 30 to 60 seconds after power-on. During this phase, the sensor calibrates to the ambient infrared environment and may trigger false alerts.
* **Buzzer Flyback Spikes**: Large active buzzers contain miniature electromagnetic coils. When switched off, they can generate high-voltage reverse spikes. For larger buzzers, it is recommended to drive them using a transistor (such as a 2N3904) with a protection diode.

## Try This! (Challenges)
1. **Intrusion LED**: Connect a warning LED to GPIO 15. If motion is detected, turn the LED ON and keep it lit for 5 seconds after motion stops.
2. **Double Doorbell Chime**: Modify the logic to play the "Ding-Dong" chime twice in succession with a 500 ms pause in between.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Buzzer stays ON constantly | Incorrect wiring or logic type | Ensure the active buzzer is wired to GPIO 14, and the sensor logic is active HIGH. |
| PIR stays triggered for too long | Onboard time-delay pot set too high | Adjust the physical PIR "Tx" potentiometer counter-clockwise to reduce the active trigger time. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [101 - Automatic Barrier Gate](101-automatic-barrier-gate.md)
- [104 - Night Light Controller](104-night-light-controller.md)
