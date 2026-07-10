# 95 - BT Alarm Arm/Disarm

Build a Bluetooth-controlled security alarm system that can be armed and disarmed remotely by sending command characters, with a PIR sensor triggering the siren when armed.

## Goal
Learn how to implement a stateful security system with distinct operating modes (DISARMED, ARMED, ALARM), transition logic between modes, and remote control via Bluetooth serial commands.

## What You Will Build
The system has three modes:
- **DISARMED**: All outputs silent. LCD shows "Disarmed". Motion is ignored.
- **ARMED**: LCD shows "Armed - Watching". If PIR detects motion, transition to ALARM.
- **ALARM**: Buzzer siren sounds, Red LED flashes. LCD shows "ALARM!". Only a Bluetooth disarm command resets it.

Send `'A'` to arm, `'D'` to disarm, `'S'` to query status.

**Why D2, D3, D4, D8?** Pin D2 reads the PIR sensor. Pins D3/D4 drive Green/Red LEDs. Pin D8 drives the buzzer. Pins D5/D6 are the SoftwareSerial Bluetooth port.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| HC-05 Bluetooth Module | `hc05_bluetooth` | Yes | Yes |
| PIR Motion Sensor | `pir_sensor` | Yes | Yes |
| LED (Green) | `led` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |
| Passive Buzzer | `buzzer` | Yes | Yes |
| 16x2 I2C LCD | `lcd_i2c` | Yes | Yes |
| 220 ohm resistor | `resistor` | Optional | Yes (per LED) |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| HC-05 Module | TXD | D5 | Software RX |
| HC-05 Module | RXD | D6 | Software TX |
| HC-05 Module | VCC | 5V | Power supply |
| HC-05 Module | GND | GND | Ground reference |
| PIR Sensor | VCC | 5V | Power supply |
| PIR Sensor | OUT | D2 | Motion signal |
| PIR Sensor | GND | GND | Ground reference |
| Green LED | Anode (+) | D3 | Armed indicator |
| Green LED | Cathode (-) | GND | Via 220 ohm resistor |
| Red LED | Anode (+) | D4 | Alarm indicator |
| Red LED | Cathode (-) | GND | Via 220 ohm resistor |
| Buzzer | + | D8 | Signal pin |
| Buzzer | - | GND | Ground reference |
| 16x2 I2C LCD | VCC | 5V | Power supply |
| 16x2 I2C LCD | SDA | A4 | I2C Data bus |
| 16x2 I2C LCD | SCL | A5 | I2C Clock bus |
| 16x2 I2C LCD | GND | GND | Ground reference |

## Code
```cpp
#include <SoftwareSerial.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

SoftwareSerial BT(5, 6); // RX = D5, TX = D6

const int PIR_PIN   = 2;
const int GREEN_PIN = 3;
const int RED_PIN   = 4;
const int BUZ_PIN   = 8;

// System mode states
const int MODE_DISARMED = 0;
const int MODE_ARMED    = 1;
const int MODE_ALARM    = 2;

int systemMode = MODE_DISARMED;

LiquidCrystal_I2C lcd(0x27, 16, 2);

void updateLCD(const char* l1, const char* l2) {
  lcd.setCursor(0, 0); lcd.print(l1); lcd.print("                ");
  lcd.setCursor(0, 1); lcd.print(l2); lcd.print("                ");
}

void allOutputsOff() {
  digitalWrite(GREEN_PIN, LOW);
  digitalWrite(RED_PIN,   LOW);
  noTone(BUZ_PIN);
}

void setup() {
  Serial.begin(9600);
  BT.begin(9600);
  
  pinMode(PIR_PIN,   INPUT);
  pinMode(GREEN_PIN, OUTPUT);
  pinMode(RED_PIN,   OUTPUT);
  pinMode(BUZ_PIN,   OUTPUT);
  
  allOutputsOff();
  
  lcd.init();
  lcd.backlight();
  updateLCD("BT Alarm System", "Mode: DISARMED  ");
  
  Serial.println("BT Alarm System Ready");
  Serial.println("Commands: A=Arm  D=Disarm  S=Status");
  BT.println("BT Alarm Ready. A=Arm D=Disarm S=Status");
}

void loop() {
  // Process incoming Bluetooth command
  if (BT.available() > 0) {
    char cmd = BT.read();
    Serial.print("BT Command: "); Serial.println(cmd);
    
    if (cmd == 'A' || cmd == 'a') {
      systemMode = MODE_ARMED;
      allOutputsOff();
      digitalWrite(GREEN_PIN, HIGH);
      updateLCD("Mode: ARMED     ", "Watching...     ");
      BT.println("System ARMED");
      Serial.println("-> System ARMED");
      
    } else if (cmd == 'D' || cmd == 'd') {
      systemMode = MODE_DISARMED;
      allOutputsOff();
      updateLCD("Mode: DISARMED  ", "Send A to arm   ");
      BT.println("System DISARMED");
      Serial.println("-> System DISARMED");
      
    } else if (cmd == 'S' || cmd == 's') {
      String status = "Mode: ";
      if (systemMode == MODE_DISARMED) status += "DISARMED";
      else if (systemMode == MODE_ARMED) status += "ARMED";
      else status += "ALARM";
      BT.println(status);
      Serial.println(status);
    }
  }
  
  // PIR motion detection (only active when ARMED)
  if (systemMode == MODE_ARMED) {
    if (digitalRead(PIR_PIN) == HIGH) {
      systemMode = MODE_ALARM;
      Serial.println("INTRUSION DETECTED - ALARM TRIGGERED!");
      BT.println("!! INTRUSION DETECTED !!");
      updateLCD("!! ALARM !!     ", "SEND D TO RESET ");
    }
  }
  
  // Drive outputs based on current mode
  if (systemMode == MODE_ALARM) {
    digitalWrite(RED_PIN, HIGH);
    tone(BUZ_PIN, 1100, 300);
    delay(350);
    tone(BUZ_PIN, 700, 300);
    delay(350);
  } else {
    delay(50);
  }
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **HC-05 Module**, **PIR Sensor**, **Green LED**, **Red LED**, **Buzzer**, and **16x2 I2C LCD** onto the canvas.
2. Connect all components per the wiring table.
3. Paste the code into the editor.
4. Select the **Compiled Arduino Uno** runtime (SoftwareSerial + switch command parsing requires compiled mode).
5. Click **Run**.
6. Send `'A'` via the Bluetooth terminal to arm. Then trigger the PIR sensor on the canvas. Send `'D'` to disarm.

## Expected Output

Terminal:
```
BT Alarm System Ready
Commands: A=Arm  D=Disarm  S=Status
BT Command: A
-> System ARMED
INTRUSION DETECTED - ALARM TRIGGERED!
BT Command: D
-> System DISARMED
```

## Expected Canvas Behavior

| Action | System Mode | Green LED | Red LED | Buzzer | LCD Line 1 |
| --- | --- | --- | --- | --- | --- |
| Startup | DISARMED | OFF | OFF | Silent | `Mode: DISARMED` |
| Send 'A' | ARMED | ON | OFF | Silent | `Mode: ARMED` |
| PIR triggers | ALARM | OFF | Flashing | Siren | `!! ALARM !!` |
| Send 'D' | DISARMED | OFF | OFF | Silent | `Mode: DISARMED` |

## Code Walkthrough

| Expression | What It Does |
| --- | --- |
| `systemMode` integer states | Tracks the current operating mode as a numeric constant, making `if` comparisons readable and preventing illegal state combinations. |
| PIR check inside `if (systemMode == MODE_ARMED)` | Motion detection is only evaluated when armed, completely ignoring the PIR in DISARMED or ALARM states. |

## Hardware & Safety Concept: Finite State Machines
This alarm system is an example of a **Finite State Machine (FSM)** — a design pattern where a system can only be in one defined state at a time, and transitions between states are triggered by specific events (commands or sensor inputs).
- FSMs prevent impossible situations (e.g. being both ARMED and DISARMED simultaneously).
- They make behavior predictable, testable, and easy to extend with new states.

## Try This! (Challenges)
1. **Entry Delay**: When armed and motion is detected, start a 10-second countdown before entering ALARM mode. This allows an authorized person to reach and disarm the keypad before the siren sounds.
2. **Alert Log**: Store the timestamps (using `millis()`) of the last 5 alarm triggers in an array and report them when the `'L'` (log) command is sent.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Commands not received | Incorrect BT pin assignment | Verify HC-05 TXD connects to D5 and RXD connects to D6. |
| PIR triggers in DISARMED mode | Mode check missing | Confirm `if (systemMode == MODE_ARMED)` wraps the PIR read block. |

## Mode Notes
This project **requires the Compiled Arduino Uno** runtime due to SoftwareSerial command character parsing and multi-mode state logic.

## Related Projects
- [72 - BT Serial Bridge](../intermediate/72-bt-serial-bridge.md)
- [73 - BT LED Control](../intermediate/73-bt-led-control.md)
- [96 - PIR Zone Alert](96-pir-zone-alert.md)
