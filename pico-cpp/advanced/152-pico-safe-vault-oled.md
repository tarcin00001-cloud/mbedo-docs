# 152 - Pico Safe Vault OLED

Build a secure safe vault monitoring node that logs motion/vibration tamper events and keypad entry progress on an OLED screen.

## Goal
Learn how to implement latching alarm security systems using multiple inputs (PIR motion, spring vibration), display warning screens on OLEDs, and process loop-free keypad PIN validation.

## What You Will Build
A security safe alarm system:
- **PIR Motion Sensor (GP16)**: Detects movement near the safe.
- **Vibration Sensor (GP17)**: Detects safe tampering.
- **4x4 Keypad (GP2, GP3, GP6, GP7, GP8, GP9, GP20, GP21)**: Silences the alarm when the PIN `5555` is entered.
- **SSD1306 OLED (GP4, GP5)**: Displays status alerts (e.g. "VAULT BREACHED" or "SAFE SECURED").
- **Active Buzzer (GP14)**: Sounds a siren during alarms.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| PIR Motion Sensor | `pir` | Yes | Yes |
| Vibration Sensor Module | `button` | Yes (represented by switch button) | Yes |
| 4x4 Matrix Keypad | `keypad` | Yes | Yes |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| PIR Sensor | OUT | GP16 | Motion sense |
| Vibration Sensor | OUT | GP17 | Tamper sense |
| Keypad Rows | R1 - R4 | GP2 / GP3 / GP6 / GP7 | Output scan rows |
| Keypad Cols | C1 - C4 | GP8 / GP9 / GP20 / GP21 | Input columns (pull-up) |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| Active Buzzer | VCC (+) | GP14 | Alarm buzzer |
| All Modules | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

const int PIR_PIN    = 16;
const int VIB_PIN    = 17;
const int BUZZ_PIN   = 14;

// Keypad pins
const int R1 = 2; const int R2 = 3; const int R3 = 6; const int R4 = 7;
const int C1 = 8; const int C2 = 9; const int C3 = 20; const int C4 = 21;

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

bool alarmLatched = false;
int pinCount = 0;
char enteredPIN[5] = "    ";
char targetPIN[5]  = "5555";

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  pinMode(PIR_PIN, INPUT);
  pinMode(VIB_PIN, INPUT);
  pinMode(BUZZ_PIN, OUTPUT);
  digitalWrite(BUZZ_PIN, LOW);

  // Setup keypad pins
  pinMode(R1, OUTPUT); pinMode(R2, OUTPUT); pinMode(R3, OUTPUT); pinMode(R4, OUTPUT);
  pinMode(C1, INPUT_PULLUP); pinMode(C2, INPUT_PULLUP); pinMode(C3, INPUT_PULLUP); pinMode(C4, INPUT_PULLUP);

  if (display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    display.clearDisplay();
    display.display();
  }

  resetSafeState();
}

void loop() {
  int motion = digitalRead(PIR_PIN);
  int tamper = digitalRead(VIB_PIN); // Active-LOW spring check

  // Trigger alarm on motion or vibration
  if ((motion == HIGH || tamper == LOW) && !alarmLatched) {
    alarmLatched = true;
    display.clearDisplay();
    display.fillRect(0, 0, 128, 64, SSD1306_WHITE);
    display.setTextColor(SSD1306_BLACK);
    display.setTextSize(1);
    display.setCursor(10, 15);
    display.print("!! VAULT BREACH !!");
    display.setCursor(10, 35);
    display.print("Enter PIN: ");
    display.display();
    pinCount = 0;
  }

  // If alarm is active, run siren and read keypad override
  if (alarmLatched) {
    // Pulse siren
    digitalWrite(BUZZ_PIN, HIGH);
    delay(80);
    digitalWrite(BUZZ_PIN, LOW);
    
    // Read keypad key
    char key = scanKeypad();
    if (key != '\0') {
      tone(BUZZ_PIN, 800, 80);
      enteredPIN[pinCount] = key;
      pinCount = pinCount + 1;
      
      display.fillRect(0, 0, 128, 64, SSD1306_WHITE);
      display.setTextColor(SSD1306_BLACK);
      display.setCursor(10, 15);
      display.print("!! VAULT BREACH !!");
      display.setCursor(10, 35);
      display.print("Enter PIN: ");
      display.setCursor(75, 35);
      if (pinCount >= 1) display.print("*");
      if (pinCount >= 2) display.print("*");
      if (pinCount >= 3) display.print("*");
      if (pinCount >= 4) display.print("*");
      display.display();
      delay(300); // Key debounce

      if (pinCount >= 4) {
        // Check PIN
        bool match = true;
        if (enteredPIN[0] != targetPIN[0]) { match = false; }
        if (enteredPIN[1] != targetPIN[1]) { match = false; }
        if (enteredPIN[2] != targetPIN[2]) { match = false; }
        if (enteredPIN[3] != targetPIN[3]) { match = false; }

        display.clearDisplay();
        if (match) {
          display.fillRect(0, 0, 128, 64, SSD1306_WHITE);
          display.setTextColor(SSD1306_BLACK);
          display.setCursor(15, 20);
          display.print("SAFE SECURED");
          display.display();
          tone(BUZZ_PIN, 1200, 400);
          delay(2000);
          resetSafeState();
        } else {
          display.setTextColor(SSD1306_WHITE);
          display.setCursor(15, 20);
          display.print("ACCESS DENIED!");
          display.drawRect(0, 0, 128, 64, SSD1306_WHITE);
          display.display();
          tone(BUZZ_PIN, 150, 1000);
          delay(2000);
          
          display.clearDisplay();
          display.fillRect(0, 0, 128, 64, SSD1306_WHITE);
          display.setTextColor(SSD1306_BLACK);
          display.setCursor(10, 15);
          display.print("!! VAULT BREACH !!");
          display.setCursor(10, 35);
          display.print("Enter PIN: ");
          display.display();
          pinCount = 0;
        }
      }
    }
  } else {
    digitalWrite(BUZZ_PIN, LOW);
  }

  delay(20);
}

void resetSafeState() {
  alarmLatched = false;
  pinCount = 0;
  enteredPIN[0] = ' '; enteredPIN[1] = ' '; enteredPIN[2] = ' '; enteredPIN[3] = ' ';
  
  display.clearDisplay();
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(15, 15);
  display.print("SAFE VAULT SECURE");
  display.setCursor(15, 35);
  display.print("Monitoring...");
  display.drawRect(0, 0, 128, 64, SSD1306_WHITE);
  display.display();
}

char scanKeypad() {
  char pressedKey = '\0';

  // Row 1
  digitalWrite(R1, LOW); digitalWrite(R2, HIGH); digitalWrite(R3, HIGH); digitalWrite(R4, HIGH);
  if (digitalRead(C1) == LOW) { pressedKey = '1'; }
  if (digitalRead(C2) == LOW) { pressedKey = '2'; }
  if (digitalRead(C3) == LOW) { pressedKey = '3'; }
  if (digitalRead(C4) == LOW) { pressedKey = 'A'; }

  // Row 2
  digitalWrite(R1, HIGH); digitalWrite(R2, LOW); digitalWrite(R3, HIGH); digitalWrite(R4, HIGH);
  if (digitalRead(C1) == LOW) { pressedKey = '4'; }
  if (digitalRead(C2) == LOW) { pressedKey = '5'; }
  if (digitalRead(C3) == LOW) { pressedKey = '6'; }
  if (digitalRead(C4) == LOW) { pressedKey = 'B'; }

  // Row 3
  digitalWrite(R1, HIGH); digitalWrite(R2, HIGH); digitalWrite(R3, LOW); digitalWrite(R4, HIGH);
  if (digitalRead(C1) == LOW) { pressedKey = '7'; }
  if (digitalRead(C2) == LOW) { pressedKey = '8'; }
  if (digitalRead(C3) == LOW) { pressedKey = '9'; }
  if (digitalRead(C4) == LOW) { pressedKey = 'C'; }

  // Row 4
  digitalWrite(R1, HIGH); digitalWrite(R2, HIGH); digitalWrite(R3, HIGH); digitalWrite(R4, LOW);
  if (digitalRead(C1) == LOW) { pressedKey = '*'; }
  if (digitalRead(C2) == LOW) { pressedKey = '0'; }
  if (digitalRead(C3) == LOW) { pressedKey = '#'; }
  if (digitalRead(C4) == LOW) { pressedKey = 'D'; }

  return pressedKey;
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **PIR Sensor**, **Vibration Sensor** (switch button), **4x4 Keypad**, **SSD1306 OLED**, and **Active Buzzer** onto the canvas.
2. Connect sensors, keypad, and OLED control lines. Connect Buzzer to **GP14**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Simulate safe tampering by closing the vibration switch, then enter the PIN code `5555` on the keypad to silence the siren.

## Expected Output

Terminal:
```
Simulation active. Latching safe alarm with OLED HUD active.
```

## Expected Canvas Behavior
* Normal state: OLED reads `SAFE VAULT SECURE` / `Monitoring...`. Silent.
* Vibration triggered: OLED screen fills white, displaying `!! VAULT BREACH !!` / `Enter PIN: `, buzzer pulses rapidly.
* PIN `5555` entered: OLED displays `SAFE SECURED` in black on a white background, buzzer turns OFF.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `motion == HIGH \|\| tamper == LOW` | Checks if either movement or physical vibration is detected to trigger the latching security alarm. |

## Hardware & Safety Concept: Tamper-Resistant Safety Logic
Vibration sensors detect physical drilling, hammering, or sawing attempts on secure metal safes. In commercial vaults, these sensors run in a **Normally Closed (NC)** loop. If an intruder cuts the sensor wire, the loop opens (breaks), which triggers the alarm immediately. This is much safer than Open loops, where cut wires render the system useless.

## Try This! (Challenges)
1. **Wrong PIN Alarm Latch**: Lock the keypad inputs out for 5 minutes if an incorrect PIN is typed 3 times.
2. **Auto Dial Alert**: Add a Bluetooth module and send a wireless warning message when tampering is detected.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm won't turn OFF on correct PIN | Key debounce mismatch | Increase the debounce delay (`delay(300)`) after key detection to let the key contact fully settle. |

## Mode Notes
This multi-device I2C and multiplexed project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [106 - Pico Shake Alarm](../intermediate/106-pico-shake-alarm.md)
- [124 - Pico Smart Lock](124-pico-smart-lock.md)
- [127 - Pico Safe Vault](127-pico-safe-vault.md)
