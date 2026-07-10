# 192 - Pico Energy Safety Logger OLED

Build an industrial power safety interlock console that measures load currents via an ACS712, opens circuit relays on overcurrent, displays graphs on an OLED, and logs events over Bluetooth.

## Goal
Learn how to interface analog current sensors (ACS712), implement latching overcurrent cutoff algorithms, design graphical OLED dashboards with fill bars, and stream Bluetooth audit reports.

## What You Will Build
An overcurrent safety cutoff console:
- **ACS712 Current Sensor (GP26)**: Monitors the electrical load current.
- **Relay Module (GP10)**: Actuates a simulated circuit breaker (normally closed).
- **SSD1306 OLED (GP4, GP5)**: Displays active current load, warning screens, and a visual bar graph.
- **HC-05 Bluetooth Module (GP0, GP1)**: Transmits wireless audit warning logs.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| ACS712 Current Sensor | `potentiometer` | Yes (represented by potentiometer slider) | Yes (5A ACS712 module) |
| Relay Module | `relay` | Yes | Yes |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |
| HC-05 Bluetooth Module | `bluetooth` | Yes (represented by serial UART receiver) | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| ACS712 Sensor | OUT | GP26 | Analog current load sensor |
| Relay Module | IN | GP10 | Circuit breaker relay |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| HC-05 | TXD / RXD | GP1 / GP0 | UART0 serial lines |
| All Modules | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

const int ACS_PIN    = 26;
const int RELAY_PIN  = 10;

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

// Overcurrent trip limit (milliamperes)
const float TRIP_CURRENT_mA = 3500.0;

bool breakerTripped = false;

void setup() {
  Serial1.begin(9600); // Bluetooth hardware UART

  pinMode(ACS_PIN, INPUT);
  pinMode(RELAY_PIN, OUTPUT);

  // Keep breaker closed initially (relay HIGH - power ON)
  digitalWrite(RELAY_PIN, HIGH);

  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  if (display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    display.clearDisplay();
    display.display();
  }

  Serial1.println("Power Safety Interlock Online.");
  delay(1000);
}

void loop() {
  int rawACS = analogRead(ACS_PIN);
  
  // Convert current: maps 12-bit ADC (0-4095) to 0-5000mA
  float current_mA = rawACS * 5000.0 / 4095.0;

  // Map current (0-5000mA) to screen width (0-128 pixels)
  int barWidth = current_mA * 128 / 5000;

  // Check overcurrent limit
  if (current_mA > TRIP_CURRENT_mA && !breakerTripped) {
    breakerTripped = true;
    digitalWrite(RELAY_PIN, LOW); // Trip breaker (power OFF)

    display.clearDisplay();
    display.fillRect(0, 0, 128, 64, SSD1306_WHITE);
    display.setTextColor(SSD1306_BLACK);
    display.setTextSize(2);
    display.setCursor(15, 10);
    display.print("BREAKER!");
    display.setTextSize(1);
    display.setCursor(15, 42);
    display.print("TRIP: OVERCURRENT");
    display.display();

    Serial1.print("ALERT: OVERCURRENT TRIP - ");
    Serial1.print(current_mA);
    Serial1.println(" mA");
  }

  // Check for reset command over Bluetooth
  if (Serial1.available()) {
    char cmd = Serial1.read();
    if (cmd == 'R') {
      breakerTripped = false;
      digitalWrite(RELAY_PIN, HIGH); // Re-close breaker (power ON)
      Serial1.println("SYSTEM RESET: Breaker Re-closed");
    }
  }

  // Update OLED if not tripped
  if (!breakerTripped) {
    display.clearDisplay();

    // Draw header block
    display.fillRect(0, 0, 128, 14, SSD1306_WHITE);
    display.setTextColor(SSD1306_BLACK);
    display.setCursor(18, 3);
    display.print("POWER MONITOR");

    display.setTextColor(SSD1306_WHITE);
    display.setTextSize(1);
    
    display.setCursor(10, 20);
    display.print("Load: ");
    display.print(current_mA, 0);
    display.print(" mA");

    display.setCursor(10, 32);
    display.print("State: BREAKER CLOSED");

    // Draw current bar graph
    display.drawRect(0, 46, 128, 14, SSD1306_WHITE);
    if (barWidth > 0) {
      display.fillRect(2, 48, barWidth - 4, 10, SSD1306_WHITE);
    }
    display.drawRect(0, 0, 128, 64, SSD1306_WHITE);
    display.display();

    delay(300);
  } else {
    delay(100);
  }
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **potentiometer** (for ACS712), **Relay**, **SSD1306 OLED**, and **HC-05** onto the canvas.
2. Connect ACS712 to **GP26**, Relay to **GP10**, OLED to **GP4/GP5**, and HC-05 to **GP0/GP1**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Slide the current potentiometer above 3500 mA (70% slider), verify the breaker trips, and type `'R'` on Bluetooth to reset it.

## Expected Output

Terminal (Bluetooth):
```
Power Safety Interlock Online.
ALERT: OVERCURRENT TRIP - 3800.00 mA
SYSTEM RESET: Breaker Re-closed
```

## Expected Canvas Behavior
* Normal state (< 3500 mA): OLED reads `State: BREAKER CLOSED`. Relay remains ON (GP10 HIGH).
* Overcurrent (> 3500 mA): OLED screen fills white, displaying `BREAKER!` / `TRIP: OVERCURRENT`. Relay turns OFF (GP10 LOW). Bluetooth transmits warning log.
* Reset command (`'R'`): Relay turns back ON. OLED returns to normal.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `digitalWrite(RELAY_PIN, LOW)` | Trips the breaker relay, cutting power to the load immediately during overcurrent. |

## Hardware & Safety Concept: Overcurrent Safety Interlocks
Overcurrent safety controllers protect motors and power electronics from shorts and stalls. If currents exceed safety thresholds, the controller trips the circuit breaker relay immediately. These safety controllers must run continuously to protect hardware from damage.

## Try This! (Challenges)
1. **Latching Lockout**: Disable the reset command if an overcurrent event occurs 3 times within 1 minute.
2. **Audio Warning Alarm**: Connect a buzzer on GP14 and sound a pulsing warning beep while the breaker is tripped.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Breaker trips immediately on boot | Sensor calibration drift | Hall-effect sensors are sensitive to stray magnetic fields. Add a calibration offset in code to zero the reading at startup. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [158 - Pico Solar Datalogger](../advanced/158-pico-solar-datalogger.md)
- [164 - Pico Solar Panel HUD](../advanced/164-pico-solar-panel-hud.md)
- [180 - Pico Energy Safety Datalogger](180-pico-energy-safety-datalogger.md)
