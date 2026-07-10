# 63 - Pico RFID Serial

Read card serial UIDs from an MFRC522 RFID reader over the SPI protocol and print the values to the serial monitor.

## Goal
Learn how to configure the SPI communication bus, load external RFID reader libraries, scan keycards, and log hexadecimal UID bytes in MicroPython.

## What You Will Build
An RFID card reader system:
- **MFRC522 RFID Reader**: Scans passive RFID cards.
- **Serial Output**: Prints the unique serial ID (UID) of scanned cards to the terminal in real time.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| MFRC522 RFID Reader | `mfrc522` | Yes | Yes |

## Wiring

| Component | Component Pin | Pico Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MFRC522 | 3.3V | 3.3V (3V3) | Red | Power line |
| MFRC522 | GND | GND | Black | Ground reference |
| MFRC522 | MISO | GP16 | Yellow | SPI MISO data line |
| MFRC522 | MOSI | GP19 | Green | SPI MOSI data line |
| MFRC522 | SCK | GP18 | Blue | SPI clock line |
| MFRC522 | SDA (CS/SS) | GP17 | Orange | SPI Chip Select line |
| MFRC522 | RST | GP22 | White | Reset line |

> **Wiring tip:** Connect the MFRC522 SPI pins to GP16-GP19 (SPI0 Port) and reset to GP22. Ensure the power is connected to 3.3V (PIR/MFRC522 boards can burn out on 5V).

## Code
```python
from machine import Pin, SPI
import utime
from mfrc522 import MFRC522 # Assumes driver package is present/loaded

# Initialize SPI 0 on GP16-19 pins
sck_pin  = Pin(18, Pin.OUT)
mosi_pin = Pin(19, Pin.OUT)
miso_pin = Pin(16, Pin.OUT)
sda_pin  = Pin(17, Pin.OUT)
rst_pin  = Pin(22, Pin.OUT)

spi = SPI(0, baudrate=1000000, polarity=0, phase=0,
          sck=sck_pin, mosi=mosi_pin, miso=miso_pin)

reader = MFRC522(spi, sda_pin, rst_pin)

print("RFID Scanner ready. Scan a card!")

while True:
    # Check for new cards
    (status, tag_type) = reader.request(reader.REQIDL)
    
    if status == reader.OK:
        # Read the card's UID
        (status, uid) = reader.anticoll()
        
        if status == reader.OK:
            # Format and print the UID in hexadecimal format
            uid_str = " ".join([hex(x)[2:].upper().zfill(2) for x in uid])
            print("Card Scanned! UID:", uid_str)
            
            # Simple card classification example
            if uid[0] == 0xDE and uid[1] == 0x12:
                print(">> Access Granted: authorized administrator")
            else:
                print(">> Access Denied: unknown card holder")
                
            utime.sleep(2) # Cooldown delay between scans
            
    utime.sleep_ms(100) # Scan polling interval
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and **MFRC522 RFID Reader** onto the canvas.
2. Connect SPI pins to **GP16-GP19** and RST to **GP22**. Connect power and GND.
3. Paste the code, select **MicroPython** mode, and click **Run**.
4. Click the RFID card component on the canvas and observe the terminal output.

## Expected Output
```
RFID Scanner ready. Scan a card!
Card Scanned! UID: DE 12 A4 C3
>> Access Granted: authorized administrator
Card Scanned! UID: AA BB CC DD
>> Access Denied: unknown card holder
```

## Expected Canvas Behavior
- The serial terminal prints scanned card UIDs when the card component on the canvas is clicked.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `SPI(0, sck=..., mosi=..., miso=...)` | Initializes SPI bus 0 on the Pico's hardware SPI pins. |
| `reader.request(reader.REQIDL)` | Sends an antenna command to check if any passive RFID tag is within range. |
| `reader.anticoll()` | Runs the anti-collision algorithm to retrieve the unique serial number (UID) of the tag. |

## Hardware & Safety Concept: RFID Anti-Collision
RFID cards operate passively, powering their internal chips using energy from the reader's magnetic antenna field. If multiple cards are placed near the reader at the same time, their signals overlap and interfere. RFID readers run an **anti-collision algorithm** (`anticoll`) that isolates and reads each card's unique serial number individually.

## Try This! (Challenges)
1. **Indicator LED**: Add a Red LED on GP14 and Green LED on GP15 to show Access Granted/Denied status.
2. **Door Lock Latch**: Connect a relay on GP10 that opens a door strike for 3 seconds on authorized card scans.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Cards are not detected | SPI wire disconnected | Check all SPI connections and ensure SCK, MOSI, and MISO are wired correctly. |
| Reading errors | CS/SS pin mismatch | Verify that the reader's SDA pin is connected to the CS/SS select pin GP17. |

## Mode Notes
This project runs in MbedO **MicroPython interpreted mode**. The `mfrc522` driver library is simulated internally.

## Related Projects
- [51 - Pico LCD Hello World](51-pico-lcd-hello-world.md)
- [62 - Pico Relay Button](62-pico-relay-button.md)
