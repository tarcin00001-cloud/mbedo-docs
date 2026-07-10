# 147 - Pico Gas OLED Alarm

Build an industrial gas leak safety monitor that displays real-time gas levels on an OLED screen, sounds an active buzzer, and shuts off a solenoid valve relay.

## Goal
Learn how to interface analog gas sensors (MQ-2), display real-time warning logs and bar graphs on SSD1306 OLED screens, and activate safety shut-off relays.

## What You Will Build
An industrial gas warning monitor:
- **MQ-2 Gas Sensor (GP26)**: Measures gas concentration index.
- **SSD1306 OLED (GP4, GP5)**: Displays the raw gas index, a visual progress bar, and status alert messages.
- **Relay Module (GP10)**: Actuates an emergency gas shut-off valve when gas levels exceed the threshold.
- **Active Buzzer (GP14)**: Sounds a pulsing alert beep during gas alarms.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| MQ-2 Gas Sensor | `gas_sensor` | Yes | Yes |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| MQ-2 Sensor | AO | GP26 | Gas index signal |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| Relay Module | IN | GP10 | Gas supply solenoid valve |
| Active Buzzer | VCC (+) | GP14 | Alarm buzzer |
| All Modules | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

const int MQ2_PIN    = 26;
const int RELAY_PIN  = 10;
const int BUZZER_PIN = 14;

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

// Safety gas threshold limit
const int GAS_ALARM_LIMIT = 1800;

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  pinMode(MQ2_PIN, INPUT);
  pinMode(RELAY_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);

  // Keep gas valve open on startup (relay LOW)
  digitalWrite(RELAY_PIN, LOW);
  digitalWrite(BUZZER_PIN, LOW);

  if (display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    display.clearDisplay();
    display.display();
  }
}

void loop() {
  int gasVal = analogRead(MQ2_PIN);

  // Map 12-bit input (0-4095) to screen width (0-128 pixels)
  int barWidth = gasVal * 128 / 4095;

  display.clearDisplay();

  // Draw header block
  display.fillRect(0, 0, 128, 14, SSD1306_WHITE);
  display.setTextColor(SSD1306_BLACK);
  display.setCursor(15, 3);
  display.print("GAS SAFETY MONITOR");

  display.setTextColor(SSD1306_WHITE);
  display.setTextSize(1);
  display.setCursor(10, 20);
  display.print("Gas Index: ");
  display.print(gasVal);

  display.setCursor(10, 32);

  if (gasVal > GAS_ALARM_LIMIT) {
    // GAS LEAK DETECTED - SHUTDOWN
    digitalWrite(RELAY_PIN, HIGH); // Shut off solenoid valve
    display.print("VALVE: !!! SHUTDOWN !!!");

    display.drawRect(0, 46, 128, 14, SSD1306_WHITE);
    display.fillRect(2, 48, barWidth - 4, 10, SSD1306_WHITE);
    display.drawRect(0, 0, 128, 64, SSD1306_WHITE);
    display.display();

    // Pulse siren
    digitalWrite(BUZZER_PIN, HIGH);
    delay(100);
    digitalWrite(BUZZER_PIN, LOW);
    delay(100);
  } else {
    // SYSTEM SECURE
    digitalWrite(RELAY_PIN, LOW); // Keep gas supply open
    digitalWrite(BUZZER_PIN, LOW);
    display.print("VALVE: SECURE/OPEN");

    display.drawRect(0, 46, 128, 14, SSD1306_WHITE);
    if (barWidth > 0) {
      display.fillRect(2, 48, barWidth - 4, 10, SSD1306_WHITE);
    }
    display.drawRect(0, 0, 128, 64, SSD1306_WHITE);
    display.display();

    delay(300); // 3 updates per second
  }
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **MQ-2 Gas Sensor**, **SSD1306 OLED**, **Relay**, and **Active Buzzer** onto the canvas.
2. Connect MQ-2 to **GP26**, OLED to **GP4/GP5**, Relay to **GP10**, and Buzzer to **GP14**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Adjust the gas PPM slider above 1800 to activate the emergency valve shutdown, buzzer, and OLED warning.

## Expected Output

Terminal:
```
Simulation active. Gas leak warning safety node online.
```

## Expected Canvas Behavior
* Normal state (< 1500): OLED reads `VALVE: SECURE/OPEN`. Relay is OFF.
* Gas leak (> 1800): OLED reads `VALVE: !!! SHUTDOWN !!!`. Relay turns ON, buzzer beeps.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `digitalWrite(RELAY_PIN, HIGH)` | Activates the relay to close the gas supply valve immediately during gas leaks. |

## Hardware & Safety Concept: Gas Leak Emergency Venting
In industrial plants, detecting a gas leak triggers both gas line shutdown valves and secondary safety actions (such as starting explosion-proof ventilation fans to dilute the gas). To prevent sparks that could ignite the gas, exhaust fans are driven by brushless DC motors with sparkless solid-state relays.

## Try This! (Challenges)
1. **Latching Override Reset**: Connect a button on GP16 that locks the valve shut until pressed.
2. **CO2 Warning**: Add an LDR flame sensor on GP27 and trigger the alarm if fire or gas is detected.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm triggers constantly | Sensor warm-up drift | MQ sensors require a brief warm-up period (1-2 minutes) to reach stable operating temperatures. Ignore values for the first 30 seconds after startup. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [98 - Pico Gas Leak](../intermediate/98-pico-gas-leak.md)
- [117 - Pico Gas OLED](../intermediate/117-pico-gas-oled.md)
- [132 - Pico Gas Warning](132-pico-gas-warning.md)
