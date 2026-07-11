# 13 - ESP32 MAC Address Resolver

Build a hardware diagnostics utility on the ESP32 that queries the wireless controller to extract and print the unique factory-burned MAC addresses for both the Station (STA) and Access Point (AP) network interfaces.

## Goal
Learn how to read hardware MAC addresses, query raw MAC byte arrays, and format hex strings.

## What You Will Build
An ESP32 DevKitC queries its internal network driver, extracts the physical MAC addresses for both the client (STA) and host (AP) interfaces, and logs them in standard colon-separated hexadecimal formats to the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |

## Wiring
No external wiring is required. The ESP32 uses its built-in RF controller registers.

## Code
```cpp
// MAC Address resolver (Read hardware MAC addresses)
#include <WiFi.h>

void printHardwareMACs() {
  Serial.println("\n--- ESP32 Hardware MAC Address Audit ---");
  
  // 1. Read Station (STA) MAC address as a string
  String staMac = WiFi.macAddress();
  Serial.print("Station (STA) MAC:     ");
  Serial.println(staMac);
  
  // 2. Read Access Point (AP) MAC address as a string
  String apMac = WiFi.softAPmacAddress();
  Serial.print("Access Point (AP) MAC: ");
  Serial.println(apMac);
  
  // 3. Read raw MAC address bytes from registers
  uint8_t rawMac[6];
  WiFi.macAddress(rawMac);
  
  Serial.print("Raw Bytes (Hex):       ");
  for (int i = 0; i < 6; i++) {
    Serial.printf("%02X", rawMac[i]);
    if (i < 5) Serial.print(":");
  }
  Serial.println();
  
  // 4. Read raw AP MAC address bytes
  uint8_t rawApMac[6];
  WiFi.softAPmacAddress(rawApMac);
  
  Serial.print("Raw AP Bytes (Hex):    ");
  for (int i = 0; i < 6; i++) {
    Serial.printf("%02X", rawApMac[i]);
    if (i < 5) Serial.print(":");
  }
  Serial.println("\n----------------------------------------\n");
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  Serial.println("Initializing MAC Resolver...");
  
  // Initialize WiFi stack to load MAC registers
  WiFi.mode(WIFI_MODE_APSTA); // Enable both interfaces
  
  printHardwareMACs();
}

void loop() {
  // Idle
  delay(10000);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** onto the canvas.
2. Paste the code and click **Run**.
3. Inspect the Serial Monitor. Verify that the unique STA and AP MAC addresses print.

## Expected Output
Serial Monitor:
```
Initializing MAC Resolver...

--- ESP32 Hardware MAC Address Audit ---
Station (STA) MAC:     24:0A:C4:00:00:10
Access Point (AP) MAC: 24:0A:C4:00:00:11
Raw Bytes (Hex):       24:0A:C4:00:00:10
Raw AP Bytes (Hex):    24:0A:C4:00:00:11
----------------------------------------
```

## Expected Canvas Behavior
* The Serial Monitor logs the hardware MAC configurations on boot.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `WiFi.macAddress()` | Retrieves the client (STA) network interface MAC address. |
| `WiFi.softAPmacAddress()` | Retrieves the host (AP) network interface MAC address. |
| `WiFi.macAddress(rawMac)` | Populates a 6-byte array with the raw hardware MAC address. |

## Hardware & Safety Concept: Understanding MAC Addresses
Every network interface has a unique 48-bit **MAC (Media Access Control) address** burned into its hardware during manufacturing. The address is split into two halves:
1. **OUI (Organizationally Unique Identifier)**: The first 3 bytes (e.g. `24:0A:C4` for Espressif), which identifies the manufacturer.
2. **UAA (Universally Administered Address)**: The last 3 bytes, which uniquely identifies the individual chip.
The ESP32 has separate MAC addresses for its client and host interfaces (usually offset by 1). MAC addresses are used for security configuration (MAC filtering) on routers.

## Try This! (Challenges)
1. **OLED MAC Dashboard**: Add an OLED screen (Project 60) and display both MAC addresses on the screen.
2. **MAC-based Security Key**: Generate a custom password hash using the ESP32's unique MAC address as a seed.
3. **MAC Register verification**: Use low-level ESP-IDF system calls (`esp_read_mac`) to read the factory MAC directly from the EFUSE registers.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| MAC addresses print as all zeros | WiFi stack offline | Ensure `WiFi.mode()` is called before reading the MAC addresses to initialize the registers |
| AP MAC address is not visible | AP interface disabled | Set the WiFi mode to `WIFI_MODE_APSTA` or `WIFI_AP` to initialize the AP interface |
| MAC address changes on reboot | Spoofing active | Use the default hardware configuration or verify any custom spoofing settings |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [03 - ESP32 WiFi IP Address Logger](03-esp32-wifi-ip-address-logger.md)
- [08 - ESP32 Access Point mode creation](08-esp32-access-point-mode-creation.md)
- [11 - ESP32 Static IP Address configuration](11-esp32-static-ip-address-configuration.md)
