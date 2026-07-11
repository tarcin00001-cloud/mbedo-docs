# 145 - GPS Logger

Connect a NEO-6M GPS module to the ARIES v3 board's second hardware UART and print raw NMEA sentence telemetry to the Serial Monitor in real time.

## Goal

Learn how to configure a second hardware serial port (`Serial2`) on the ARIES v3 to receive NMEA 0183 sentences from a u-blox NEO-6M GPS module and echo them to the USB Serial Monitor so they can be inspected, logged, or piped into a parser on a host PC.

## What You Will Build

A NEO-6M GPS module is wired to ARIES GPIO 16 (TX2) and GPIO 17 (RX2). The firmware opens `Serial2` at 9600 baud — the NEO-6M default — and opens USB `Serial` at 115200 baud. Every character arriving on `Serial2` from the GPS is immediately forwarded to `Serial`, producing a live stream of NMEA sentences such as `$GPGGA`, `$GPRMC`, and `$GPVTG` in the Serial Monitor.

## Parts Needed

| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| u-blox NEO-6M GPS Module | `neo6m` | Yes | Yes |
| Active GPS Antenna (SMA) | — | No | Yes |
| Jumper Wires | — | No | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| NEO-6M | VCC | 3V3 | Red | NEO-6M operates at 3.3 V; confirm your module's regulator |
| NEO-6M | GND | GND | Black | Common ground |
| NEO-6M | TX | GPIO 17 (UART2 RX) | Green | GPS transmits → ARIES receives |
| NEO-6M | RX | GPIO 16 (UART2 TX) | Blue | ARIES transmits → GPS receives (optional for this project) |

> **Wiring tip:** On the NEO-6M breakout, **TX** on the GPS goes to **RX2** on ARIES (GPIO 17) and **RX** on the GPS goes to **TX2** on ARIES (GPIO 16). Cross the TX/RX lines — do not connect TX to TX. For outdoor use, attach an active GPS patch antenna to the SMA connector. In a covered room, signal fix may take 1–5 minutes or may not be acquired; the raw NMEA sentences will still stream even before a fix (the fix-quality field will show `0`).

## Code

```cpp
// 145 - GPS Logger
// NEO-6M GPS NMEA sentences forwarded from Serial2 -> Serial Monitor

#define GPS_BAUD    9600
#define SERIAL_BAUD 115200

// State variable: 1 = system banner printed, 0 = not yet
int bannerPrinted = 0;

// Character accumulation for one NMEA sentence (no array buffer)
// We use a String to collect each sentence
String nmeaLine = "";

// Single character read variable
char gpsCh = 0;

void setup() {
  Serial.begin(SERIAL_BAUD);
  delay(200);
  Serial.println("=== GPS Logger ===");
  Serial.println("Waiting for NEO-6M GPS data on Serial2 (GPIO 16/17)...");
  Serial.println("---------------------------------------------------");

  // Open the second hardware UART at 9600 baud (NEO-6M default)
  Serial2.begin(GPS_BAUD);

  bannerPrinted = 1;
}

void loop() {
  // Forward every available byte from GPS UART to Serial Monitor
  if (Serial2.available() > 0) {
    gpsCh = (char)Serial2.read();

    // Echo raw character immediately for transparent bridge mode
    Serial.print(gpsCh);

    // Also accumulate into a line so sentence boundaries are visible
    if (gpsCh == '\n') {
      // A complete NMEA sentence has been received - already printed above
      nmeaLine = "";
    } else if (gpsCh != '\r') {
      nmeaLine += gpsCh;
    }
  }
}
```

## What to Click in MbedO

1. Drag the **VEGA ARIES v3** and **NEO-6M GPS** components onto the canvas.
2. Wire the NEO-6M: **TX → GPIO 17 (UART2 RX)**, **RX → GPIO 16 (UART2 TX)**, **VCC → 3V3**, **GND → GND**.
3. Paste the code into the editor and select **Interpreted Mode**.
4. Click **Run**.
5. In the MbedO canvas, the NEO-6M widget will start streaming simulated NMEA sentences automatically.
6. Observe the raw NMEA stream in the **Serial Monitor** pane.

## Expected Output

Serial Monitor:
```
=== GPS Logger ===
Waiting for NEO-6M GPS data on Serial2 (GPIO 16/17)...
---------------------------------------------------
$GPGGA,105234.00,1234.5678,N,07712.3456,E,1,08,0.9,216.5,M,-43.6,M,,*67
$GPRMC,105234.00,A,1234.5678,N,07712.3456,E,0.10,0.00,110726,,,A*65
$GPVTG,0.00,T,,M,0.10,N,0.19,K,A*30
$GPGSA,A,3,01,03,08,10,17,21,22,27,,,,,1.4,0.9,1.1*32
$GPGSV,3,1,10,01,65,285,40,03,18,043,35,08,72,130,42,10,23,311,38*72
```

## Expected Canvas Behavior

* Raw NMEA sentences scroll continuously in the Serial Monitor at the GPS update rate (1 Hz by default on NEO-6M).
* Each sentence starts with `$GP` (GPS-only) or `$GN` (GNSS multi-constellation on newer firmware).
* Before a satellite fix is acquired, `$GPGGA` will show `0` in the fix-quality field and latitude/longitude will be empty.
* The system outputs every byte transparently — no parsing or filtering is applied.

## Code Walkthrough

| Line | Check / Action |
| --- | --- |
| `Serial2.begin(GPS_BAUD)` | Opens UART2 on GPIO 16 (TX) / GPIO 17 (RX) at 9600 baud to match NEO-6M factory default. |
| `Serial2.available() > 0` | Non-blocking check — returns the number of bytes waiting in the UART2 receive buffer. |
| `Serial2.read()` | Reads one byte from the UART2 buffer and advances the internal read pointer. |
| `Serial.print(gpsCh)` | Forwards the character immediately to the USB Serial Monitor (transparent bridge). |
| `nmeaLine += gpsCh` | Accumulates characters for optional per-sentence processing in future extensions. |
| `gpsCh == '\n'` | NMEA sentences are terminated with CR+LF (`\r\n`); the newline marks a complete sentence. |

## Hardware & Safety Concept

* **UART Transparent Bridge**: This firmware acts as a byte-for-byte bridge between UART2 (GPS) and USB Serial. Because NMEA is a line-oriented ASCII protocol, each `$GP...` sentence arrives terminated by `\r\n`. The NEO-6M sends all configured sentences once per second at 9600 baud, occupying less than 10 ms of bus time per cycle — well within the ARIES loop execution budget.
* **Cold vs. Warm Start**: After power-on without a previous almanac, the NEO-6M performs a cold start and may take 30–120 seconds outdoors to achieve a first fix (TTFF — Time To First Fix). The raw serial stream is active immediately; only the coordinate and fix-quality fields change once satellites are locked. An active antenna with a clear sky view significantly reduces TTFF.
* **Baud Rate Compatibility**: The NEO-6M factory baud rate is 9600. If the module has been reconfigured via u-center to a higher rate (38400, 115200), `Serial2.begin()` must match. A baud mismatch produces garbled ASCII output.

## Try This! (Challenges)

1. **GPRMC Filter**: Add a global `String sentence = ""` accumulator. In `loop()`, instead of printing every character, only print the full `nmeaLine` string when it starts with `"$GPRMC"` — this shows only the recommended minimum navigation data sentence.
2. **Sentence Counter**: Add a global `int sentenceCount = 0` variable. Increment it each time `gpsCh == '\n'` and print the running count alongside each sentence so you can verify the GPS update rate.

## Troubleshooting

| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| No output in Serial Monitor | TX/RX crossed incorrectly | NEO-6M TX must go to ARIES GPIO 17 (RX2), not GPIO 16 (TX2). |
| Garbled characters (`����`) | Baud rate mismatch | Confirm `Serial2.begin(9600)` matches NEO-6M factory setting; check module firmware with u-center. |
| Stream stops after a few seconds | UART2 buffer overflow | The loop is fast enough; if using other blocking code, ensure `loop()` cycles quickly to drain the buffer. |
| Only `$GPGSV` sentences appear | Module has no fix yet | Move the antenna outdoors or near a window; wait for the fix LED on the module to blink at 1 Hz. |

## Mode Notes

This project runs in MbedO **interpreted C++ mode**.

## Related Projects

- [146 - GPS Location Display](146-gps-location-display.md)
- [147 - GPS Velocity Logger](147-gps-velocity-logger.md)
- [144 - RFID Access Logger](144-rfid-access-logger.md)
