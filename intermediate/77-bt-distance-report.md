# 77 - BT Distance Report

Transmit HC-SR04 ultrasonic distance measurements wirelessly over Bluetooth when queried.

## Goal
Learn how to wait for a specific request command character over Bluetooth, trigger a distance measurement, and transmit the result back.

## What You Will Build
The Arduino waits for command requests. When the character `'D'` (Distance) is received over Bluetooth, the Arduino triggers the HC-SR04 sensor, calculates the distance, and transmits the result (e.g. "Dist: 45 cm") back over Bluetooth.

**Why D2, D3, D4, and D5?** Pins D2/D3 manage the Bluetooth communication. Pins D4/D5 trigger and read the ultrasonic sensor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| HC-05 Bluetooth Module | `hc05_bluetooth` | Yes | Yes |
| HC-SR04 Ultrasonic | `hc_sr04` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| HC-05 Module | TXD | D2 | Software RX |
| HC-05 Module | RXD | D3 | Software TX (Use divider on hardware) |
| HC-05 Module | VCC | 5V | Power supply |
| HC-05 Module | GND | GND | Ground reference |
| HC-SR04 Sensor | VCC | 5V | Power supply |
| HC-SR04 Sensor | TRIG | D4 | Trigger pin |
| HC-SR04 Sensor | ECHO | D5 | Echo pin |
| HC-SR04 Sensor | GND | GND | Ground reference |

## Code
```cpp
#include <SoftwareSerial.h>

SoftwareSerial BT(2, 3); // RX = D2, TX = D3

const int TRIG_PIN = 4;
const int ECHO_PIN = 5;

void setup() {
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  
  Serial.begin(9600);
  BT.begin(9600);
  
  Serial.println("Bluetooth Distance Station Ready");
}

void loop() {
  // Check if query command is received
  if (BT.available() > 0) {
    char command = BT.read();
    
    Serial.print("Received: ");
    Serial.println(command);
    
    if (command == 'D' || command == 'd') {
      // Trigger ultrasonic pulse
      digitalWrite(TRIG_PIN, LOW);
      delayMicroseconds(2);
      digitalWrite(TRIG_PIN, HIGH);
      delayMicroseconds(10);
      digitalWrite(TRIG_PIN, LOW);
      
      long duration = pulseIn(ECHO_PIN, HIGH);
      long distance = duration * 0.034 / 2;
      
      Serial.print("Query Response: ");
      Serial.print(distance);
      Serial.println(" cm");
      
      // Transmit response over Bluetooth
      BT.print("Dist: ");
      BT.print(distance);
      BT.println(" cm");
    }
  }
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **HC-05 Bluetooth Module**, and **HC-SR04 Sensor** onto the canvas.
2. Connect HC-05: **TXD** to **D2**, **RXD** to **D3**, **VCC** to **5V**, and **GND** to **GND**.
3. Connect HC-SR04: **VCC** to **5V**, **TRIG** to **D4**, **ECHO** to **D5**, and **GND** to **GND**.
4. Paste the code into the editor.
5. Select the **Compiled Arduino Uno** runtime.
6. Click **Build and Run** (or **Run**).
7. Open the **Terminal** tab in the bottom dock.
8. Send character `'D'` over the simulated Bluetooth port, and watch the distance values return.

## Expected Output

Terminal:
```
Bluetooth Distance Station Ready
Received: D
Query Response: 120 cm
...
```
The virtual Bluetooth device receives:
```
Dist: 120 cm
...
```

### Expected Canvas Behavior

| Sent Bluetooth Command | Sensor Slider State | Transmitted BT Output |
| --- | --- | --- |
| 'D' | 50 cm | "Dist: 50 cm" |
| 'D' | 215 cm | "Dist: 215 cm" |
| Other character | 50 cm | None (Ignored) |

The distance is reported back wirelessly only when requested by the host command.

## Code Walkthrough

| Line / Expression | What It Does |
| --- | --- |
| `if (command == 'D' || command == 'd')` | Conditional check to identify the query character code. |
| `BT.print(distance)` | Converts the long integer distance into characters and writes them to the serial output stream buffer. |

## Hardware & Safety Concept: Command-Response Telemetry
Continuous data transmission (streaming) over wireless links uses considerable bandwidth and power. 
- In battery-powered remote sensor pods (IoT nodes), the controller remains in a low-power **sleep mode** most of the time.
- It only wakes up when it receives an external wireless **query command** (interrupt trigger), takes the sensor reading, transmits the packet, and immediately returns to sleep, extending battery life from days to years.

## Try This! (Challenges)
1. **Inch Switch**: Add support for command `'I'` that returns the distance in inches instead of centimeters.
2. **Min-Max Alarm**: If the distance measured is less than 15 cm, append a warning tag: "Dist: 12 cm [WARNING - CLOSE]" to the response.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Command ignored | Incorrect runtime selected | Verify that you have selected the **Compiled Arduino Uno** runtime. |
| Returned distance is constantly 0 | Sensor pins miswired | Verify TRIG connects to D4 and ECHO connects to D5. |

## Mode Notes
This project **requires the Compiled Arduino Uno** runtime because custom byte parsing and nested sensor trigger checks are not processed by the interpreted mode runner.

## Related Projects
- [54 - Distance Serial](54-distance-serial.md)
- [72 - BT Serial Bridge](72-bt-serial-bridge.md)
- [76 - BT Sensor Report](76-bt-sensor-report.md)
