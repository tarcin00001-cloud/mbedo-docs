# 118 - ESP32 IR Remote LED Control

Control physical status LEDs wirelessly using a handheld infrared remote control to toggle outputs and perform global shutdowns.

## Goal
Learn how to map specific decoded infrared command codes to actuator events, toggling outputs and implementing master safety shutoffs.

## What You Will Build
An IR receiver module is connected to GPIO 15. A Red LED is connected to GPIO 5, and a Green LED to GPIO 12. Pressing button '1' on the remote toggles the Red LED; pressing button '2' toggles the Green LED. Pressing the 'POWER' button turns both LEDs off.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| IR Receiver Module (VS1838B) | `ir_receiver` | Yes | Yes |
| IR Handheld Remote Control | `ir_remote` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |
| LED (Green) | `led` | Yes | Yes |
| 330 Ω Resistors (2) | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| IR Receiver | OUT (Signal) | GPIO15 | Yellow | Demodulated pulse stream |
| IR Receiver | VCC / GND | 3V3 / GND | Red / Black | Power rails |
| Red LED | Anode (+) | GPIO5 via 330 Ω | Orange | Indicator |
| Red LED | Cathode (−) | GND | Black | Ground reference |
| Green LED | Anode (+) | GPIO12 via 330 Ω | Green | Indicator |
| Green LED | Cathode (−) | GND | Black | Ground reference |

> **Wiring tip:** Connect the Red LED to **GPIO 5** and Green LED to **GPIO 12**. The IR receiver signal is connected to **GPIO 15**.

## Code
```cpp
// IR Remote LED Control
#include <IRremote.h>

const int RECV_PIN = 15;
const int RED_LED = 5;
const int GREEN_LED = 12;

bool redState = false;
bool greenState = false;

void setup() {
  Serial.begin(115200);
  
  pinMode(RED_LED, OUTPUT);
  pinMode(GREEN_LED, OUTPUT);
  
  digitalWrite(RED_LED, LOW);
  digitalWrite(GREEN_LED, LOW);
  
  IrReceiver.begin(RECV_PIN, ENABLE_LED_FEEDBACK);
  Serial.println("IR LED Controller ready.");
}

void loop() {
  if (IrReceiver.decode()) {
    Serial.print("Received Command: 0x");
    Serial.println(IrReceiver.decodedIRData.command, HEX);
    
    switch (IrReceiver.decodedIRData.command) {
      // Button '1' -> Toggle Red LED
      case 0xC: 
        redState = !redState;
        digitalWrite(RED_LED, redState ? HIGH : LOW);
        Serial.print("Red LED toggled to: "); Serial.println(redState ? "ON" : "OFF");
        break;
        
      // Button '2' -> Toggle Green LED
      case 0x18: 
        greenState = !greenState;
        digitalWrite(GREEN_LED, greenState ? HIGH : LOW);
        Serial.print("Green LED toggled to: "); Serial.println(greenState ? "ON" : "OFF");
        break;
        
      // Button 'POWER' -> Turn both OFF
      case 0x45: 
        redState = false;
        greenState = false;
        digitalWrite(RED_LED, LOW);
        digitalWrite(GREEN_LED, LOW);
        Serial.println("POWER: Both LEDs shut down.");
        break;
        
      default:
        Serial.println("Command ignored.");
        break;
    }
    
    IrReceiver.resume(); // Prepare for next signal
  }
  delay(100);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **IR Receiver**, **IR Remote**, and two **LEDs** onto the canvas.
2. Wire the IR Receiver, Red LED to **GPIO5**, and Green LED to **GPIO12**.
3. Paste the code and click **Run**.
4. Click `1` on the remote widget to toggle the Red LED; click `2` to toggle the Green LED; click the `Power` button to shut both off.

## Expected Output
Serial Monitor:
```
IR LED Controller ready.
Received Command: 0xC
Red LED toggled to: ON
Received Command: 0x18
Green LED toggled to: ON
Received Command: 0x45
POWER: Both LEDs shut down.
```

## Expected Canvas Behavior
* Clicking button `1` on the remote widget turns the Red LED widget ON and OFF.
* Clicking button `2` turns the Green LED widget ON and OFF.
* Clicking `Power` turns both LED widgets OFF instantly.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `redState = !redState` | Inverts the Red LED state helper variable (toggle logic). |
| `case 0xC` | Evaluates if the code for remote button '1' was received. |
| `case 0x45` | Resets both states to false and drives pins LOW when the POWER button is pressed. |

## Hardware & Safety Concept: Optical Line of Sight Control
Infrared light behaves like visible light: it cannot penetrate walls, solid partitions, or metal enclosures. This makes IR remotes highly secure for local equipment operations (like operating TVs or garage gates within visual range), but means they require a clear line-of-sight path between the transmitter and the receiver module.

## Try This! (Challenges)
1. **Brightness Control**: Map button "VOL+" and "VOL-" to increase and decrease the Red LED brightness using PWM (`ledcWrite`).
2. **Sequential Flashing**: Add a button "PLAY" (code `0x44`) that starts a continuous back-and-forth flashing sequence between the two LEDs.
3. **Timed Relay Trigger**: Wire a relay module (Project 11) and toggle it ON/OFF using remote buttons.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Buttons do not match the code | Handheld remote uses a different protocol layout | Run Project 117 first to scan your remote's exact hexadecimal command codes, and update the case values |
| LEDs flash briefly but stay off | State variables not latching | Ensure `redState` and `greenState` are declared globally, outside the loop |
| No response at all | Power pin swapped | Verify VS1838B VCC is connected to 3.3V, not GND |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [117 - ESP32 IR Remote Receiver Command Decoder](117-esp32-ir-remote-receiver-command-decoder.md)
- [10 - ESP32 RGB LED Color Cycle](../beginner/10-esp32-rgb-led-cycle.md)
- [90 - ESP32 Bluetooth LED Controller](90-esp32-hc05-bluetooth-led-controller-string-parsing.md)
