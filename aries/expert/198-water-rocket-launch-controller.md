# 198 - Water Rocket Launch Controller

Design a safe, automated model launch controller for water-pressure rockets used in educational science demonstrations. The system features a 16x2 I2C LCD status screen, buzzer warning cadences, a key-arming safety indicator, and a servo-driven release latch that triggers at the end of a 5-second countdown.

## Goal
Learn how to create multi-stage state machines with conditional transitions, synchronize visual status readouts with audio feedback patterns, and drive servo actuators safely at the end of structured time schedules, without using loops or blocking delays.

## What You Will Build
An educational launch box. On boot, the system is in a `DISARMED` state. Pressing the safety button on GPIO 16 once arms the system, lighting up the warning LED on GPIO 15. Pressing the button again initiates a 5-second visual and audible countdown on the LCD. At T-0, the servo motor on GPIO 13 rotates to 90° to trigger the pressure valve release, and the buzzer sounds a continuous tone. After 3 seconds, the servo returns to 0° and the system automatically disarms.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |
| Servo Motor | `servo` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Warning LED | `led` | Yes | Yes |
| Push Button | `button` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| I2C LCD | VCC | 5V | Red | LCD power supply |
| I2C LCD | GND | GND | Black | Ground reference |
| I2C LCD | SDA | SDA0 (GP17) | Blue | I2C Data line |
| I2C LCD | SCL | SCL0 (GP16) | Yellow | I2C Clock line |
| Servo Motor | Signal | GPIO 13 | Yellow | Servo control line |
| Servo Motor | VCC | 5V | Red | Servo power supply |
| Servo Motor | GND | GND | Black | Ground reference |
| Active Buzzer | + | GPIO 14 | Orange | Warning buzzer |
| Active Buzzer | - | GND | Black | Ground reference |
| Warning LED | Anode | GPIO 15 | Orange | Arming safety indicator |
| Warning LED | Cathode | GND (via 220 Ω) | Black | Current-limiting resistor |
| Push Button | Pin 1 | GPIO 16 | Yellow | Launch trigger button |
| Push Button | Pin 2 | GND | Black | Ground reference |

> **Wiring tip:** Water rockets utilize high water pressure (typically 40-60 PSI inside a plastic bottle). Ensure the servo horn is mechanically connected to a physical latch pins (e.g. quick-release coupler) that can hold the launch pressures without slipping.

## Code
```cpp
// 198 - Water Rocket Launch Controller
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <Servo.h>

LiquidCrystal_I2C lcd(0x27, 16, 2);
Servo releaseServo;

const int BUZZER_PIN = 14;
const int WARN_LED_PIN = 15;
const int BUTTON_PIN = 16;
const int SERVO_PIN = 13;

// System states: 0 = DISARMED, 1 = ARMED, 2 = COUNTDOWN, 3 = LAUNCH, 4 = COOLDOWN
int systemState = 0;
int countdownVal = 5;
int stateTimer = 0; // Tracks ticks (1 tick = 100 ms)
int lastButtonState = HIGH;

void setup() {
  Serial.begin(115200);
  Wire.begin();

  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("LAUNCH CONTROL");
  lcd.setCursor(0, 1);
  lcd.print("SYSTEM ONLINE");

  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(WARN_LED_PIN, OUTPUT);
  pinMode(BUTTON_PIN, INPUT_PULLUP);

  digitalWrite(BUZZER_PIN, LOW);
  digitalWrite(WARN_LED_PIN, LOW);

  releaseServo.attach(SERVO_PIN);
  releaseServo.write(0); // Latch closed

  delay(1500);
  lcd.clear();
  Serial.println("Launch Controller Initialised.");
}

void loop() {
  // Read button transition (Active LOW)
  int currentButtonState = digitalRead(BUTTON_PIN);
  int buttonPressed = (currentButtonState == LOW && lastButtonState == HIGH);
  lastButtonState = currentButtonState;

  // --- STATE MACHINE ---
  if (systemState == 0) {
    // DISARMED STATE
    digitalWrite(WARN_LED_PIN, LOW);
    digitalWrite(BUZZER_PIN, LOW);
    releaseServo.write(0);

    // Refresh LCD static text
    stateTimer++;
    if (stateTimer >= 10) {
      stateTimer = 0;
      lcd.setCursor(0, 0);
      lcd.print("STATUS: DISARMED");
      lcd.setCursor(0, 1);
      lcd.print("PRESS BTN TO ARM");
    }

    if (buttonPressed) {
      systemState = 1;
      stateTimer = 0;
      lcd.clear();
      Serial.println("System ARMED. Standby for launch command.");
      // Short authorization beep
      digitalWrite(BUZZER_PIN, HIGH);
      delay(100);
      digitalWrite(BUZZER_PIN, LOW);
    }
  } 
  else if (systemState == 1) {
    // ARMED STATE
    digitalWrite(WARN_LED_PIN, HIGH); // Warning indicator solid ON
    
    stateTimer++;
    if (stateTimer >= 10) {
      stateTimer = 0;
      lcd.setCursor(0, 0);
      lcd.print("STATUS: ARMED   ");
      lcd.setCursor(0, 1);
      lcd.print("PRESS TO LAUNCH ");
    }

    if (buttonPressed) {
      systemState = 2;
      countdownVal = 5;
      stateTimer = 0;
      lcd.clear();
      Serial.println("Launch Countdown Initiated.");
    }
  } 
  else if (systemState == 2) {
    // COUNTDOWN STATE
    digitalWrite(WARN_LED_PIN, HIGH);

    // Run countdown update every 1 second (10 ticks * 100 ms)
    stateTimer++;
    
    // Chirp warning buzzer (100 ms pulse at start of every second)
    if (stateTimer == 1) {
      digitalWrite(BUZZER_PIN, HIGH);
      lcd.setCursor(0, 0);
      lcd.print("LAUNCH COUNTDOWN");
      lcd.setCursor(0, 1);
      lcd.print("T-MINUS: ");
      lcd.print(countdownVal);
      lcd.print(" SEC  ");
      
      Serial.print("T-Minus: ");
      Serial.print(countdownVal);
      Serial.println("s");
    }
    
    if (stateTimer == 2) {
      digitalWrite(BUZZER_PIN, LOW);
    }

    if (stateTimer >= 10) {
      stateTimer = 0;
      countdownVal--;
      if (countdownVal < 0) {
        systemState = 3; // Trigger launch
        lcd.clear();
      }
    }
  } 
  else if (systemState == 3) {
    // LAUNCH TRIGGER STATE (Hold open for 3 seconds)
    digitalWrite(WARN_LED_PIN, HIGH);
    digitalWrite(BUZZER_PIN, HIGH); // Continuous siren
    releaseServo.write(90);        // Pull release latch

    lcd.setCursor(0, 0);
    lcd.print("!!! LAUNCH !!!  ");
    lcd.setCursor(0, 1);
    lcd.print("VALVE OPENED    ");

    stateTimer++;
    if (stateTimer >= 30) { // 3 seconds elapsed
      systemState = 4;       // Transition to cooldown
      stateTimer = 0;
      digitalWrite(BUZZER_PIN, LOW);
      lcd.clear();
      Serial.println("Launch complete. Resetting valve.");
    }
  } 
  else if (systemState == 4) {
    // COOLDOWN / RESET STATE
    digitalWrite(WARN_LED_PIN, LOW);
    digitalWrite(BUZZER_PIN, LOW);
    releaseServo.write(0); // Return latch to closed

    lcd.setCursor(0, 0);
    lcd.print("VALVE RESETTING ");
    lcd.setCursor(0, 1);
    lcd.print("STANDBY...      ");

    stateTimer++;
    if (stateTimer >= 20) { // 2 seconds cooldown
      systemState = 0; // Return to disarmed
      stateTimer = 0;
      lcd.clear();
      Serial.println("System disarmed. Ready for next cycle.");
    }
  }

  delay(100); // Base clock tick
}
```

## What to Click in MbedO
1. Drag **VEGA ARIES v3**, **I2C LCD**, **Servo Motor**, **Active Buzzer**, **Warning LED**, and **Push Button** onto the canvas.
2. Wire I2C LCD: **SDA → SDA0 (GP17)**, **SCL → SCL0 (GP16)**.
3. Wire Servo: **Signal → GPIO 13**, **VCC → 5V**, **GND → GND**.
4. Wire Buzzer: **+ → GPIO 14**; Warning LED: **Anode → GPIO 15**, **Cathode → GND** (via 220 Ω).
5. Wire Button: **Pin 1 → GPIO 16**, **Pin 2 → GND**.
6. Paste code, select **Interpreted Mode**, and click **Run**.
7. Press the button once to transition to ARMED (warning LED lights up). Press again to start the countdown. Watch the LCD updates, buzzer beeps, and final servo sweep to 90°.

## Expected Output
Serial Monitor:
```
Launch Controller Initialised.
System ARMED. Standby for launch command.
Launch Countdown Initiated.
T-Minus: 5s
T-Minus: 4s
T-Minus: 3s
T-Minus: 2s
T-Minus: 1s
T-Minus: 0s
Launch complete. Resetting valve.
System disarmed. Ready for next cycle.
```

## Expected Canvas Behavior
* Boot: LCD shows `LAUNCH CONTROL SYSTEM ONLINE`. The servo is at 0°.
* Armed: Warning LED turns solid green/red. LCD updates to `STATUS: ARMED`.
* Countdown: LCD counts down from 5 to 0. Buzzer chirps once every second.
* Launch: The servo sweeps to 90° and holds for 3 seconds. The buzzer sounds continuously.
* Cooldown: The servo sweeps back to 0°. The LCD shows resetting status, then reverts to `STATUS: DISARMED`.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `releaseServo.attach(13)` | Attaches the physical valve release servo to GPIO pin 13. |
| `buttonPressed` | Latches button transition from HIGH to LOW to avoid double-triggers. |
| `systemState == 0` | Evaluates if system is currently disarmed and safe. |
| `systemState = 2` | Moves the state machine to countdown status, triggering safety timers. |
| `countdownVal--` | Decrements the time remaining before launch. |
| `releaseServo.write(90)` | Open the pressure valve by rotating the latch to 90°. |

## Hardware & Safety Concept
* **Water Rocket Mechanics:** Water rockets function under the principles of Newton's third law of motion. Compressed air inside the plastic bottle stores mechanical energy. When the latch releases, the air pushes the water down, generating high thrust. Since it is water and air, there are no chemical reaction hazards, but mechanical latches must withstand holding forces.
* **Armed Interlock Logic:** Launching model rockets requires strict safety procedures. The system utilizes double button keying: the first press is a safety bypass that arms the node. The second command executes the timer. This prevents accidental launches if the button is bumped.

## Try This! (Challenges)
1. **Pressure Switch Safety:** Connect an analog pressure sensor to ADC0 (GP26). Only allow arming if the pressure inside the chamber is above 30 PSI, and trigger an emergency abort (valve release) if pressure exceeds 70 PSI.
2. **Launch Abort Key:** Implement an abort sequence. If the button on GPIO 16 is held down for more than 1 second during the countdown, cancel the launch, sound a rapid warning chirp, and return to disarmed.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LCD does not print numbers | Contrast potentiometer or address mismatch | Adjust the blue contrast potentiometer on the back of the LCD backpack. |
| Servo does not pull the release latch | Underpowered motor | Ensure the servo VCC is connected to the 5V line, as 3.3V does not provide sufficient torque. |
| Countdown skips numbers | Loop delay incorrect | Verify the base tick loop delay is set to exactly 100 ms. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [51 - Servo Motor 180 Degree Sweep](../intermediate/51-servo-motor-180-degree-sweep.md)
- [58 - 16x2 I2C LCD Hello World](../intermediate/58-16x2-i2c-lcd-hello-world.md)
- [113 - DS3231 RTC Alarm System](../intermediate/113-ds3231-rtc-alarm-system.md)
