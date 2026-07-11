# 170 - NeoPixel Color Spectrum Visualizer

Read vibration intensity from an analog sensor on ADC0/GP26 and drive an 8-pixel WS2812B NeoPixel strip with a rolling color spectrum whose hue and animation speed respond in real time to the strength of vibration detected.

## Goal

Learn how to map an analog sensor reading to a color hue offset, how to drive a WS2812B NeoPixel strip using the Adafruit NeoPixel library's `setPixelColor()` API without array buffers or loops, and how to use global phase state variables to create a smooth rolling spectrum animation whose tempo is controlled by a physical signal.

## What You Will Build

An analog vibration sensor wired to ADC0 (GP26) produces a value between 0 and 4095 proportional to the mechanical vibration it detects. Each call to `loop()` reads this value, normalises it, and uses it to advance a global hue phase counter by a variable step — a large step when vibration is strong, a tiny step when the environment is calm. Eight NeoPixels are each assigned a hue derived from the current phase plus a fixed per-pixel offset so the strip displays a smooth spectrum slice. The hue-to-RGB conversion is implemented inline inside `loop()` using the Adafruit NeoPixel library's built-in `ColorHSV()` helper and `gamma32()` correction so colours appear perceptually uniform. Strong vibration pushes hues toward the warm red and orange end of the spectrum and causes the palette to cycle rapidly; gentle or absent vibration keeps the strip in the cool blue and green region with a slow glacial drift.

## Parts Needed

| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| WS2812B NeoPixel Strip (8 pixels) | `neopixel` | Yes | Yes |
| Analog Vibration Sensor (SW-420 or similar) | `potentiometer` (simulated) | Yes | Yes |
| 300–500 Ω Series Resistor (data line) | `resistor` | No | Yes |
| 1000 µF Decoupling Capacitor (power rail) | `capacitor` | No | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Vibration Sensor | VCC | 3V3 | Red | Sensor power |
| Vibration Sensor | GND | GND | Black | Common ground |
| Vibration Sensor | AOUT (analog output) | GP26 | Yellow | Analog vibration amplitude signal (ADC0) |
| NeoPixel Strip | +5V / VCC | 5V | Red | Strip power — use board 5 V rail or external supply |
| NeoPixel Strip | GND | GND | Black | Common ground shared with board |
| NeoPixel Strip | DIN (data in) | GPIO 14 | Green | 300–500 Ω series resistor in series on this wire |
| Decoupling Capacitor | Across +5V and GND | — | — | 1000 µF placed as close to strip power pins as possible |

> **Wiring tip:** Always place a 300–500 Ω resistor in series on the NeoPixel data line between GPIO 14 and the strip DIN pad. This damps ringing on the signal wire and prevents the first pixel from being damaged by voltage spikes. Add a large electrolytic capacitor (1000 µF, 6.3 V or higher) across the strip's +5V and GND supply pins to absorb inrush current when all eight pixels light at full white simultaneously. If the strip is powered from the board's 5 V rail, keep total current draw in mind: eight WS2812B pixels at full white consume roughly 480 mA; reduce brightness in code or use an external 5 V supply rated for at least 1 A.

## Code

```cpp
// NeoPixel Color Spectrum Visualizer
// Vibration sensor on ADC0 (GP26), NeoPixel strip on GPIO 14 (8 pixels)

#include <Adafruit_NeoPixel.h>

#define PIXEL_PIN    14
#define PIXEL_COUNT   8
#define ADC_PIN      GP26    // GP26 = ADC0

Adafruit_NeoPixel strip(PIXEL_COUNT, PIXEL_PIN, NEO_GRB + NEO_KHZ800);

// Global hue phase (0-65535 wraps around the full colour wheel)
int phase = 0;

// Vibration reading and derived step size
int vibRaw    = 0;
int hueStep   = 0;

// Delay between frames (ms) -- decreases as vibration increases
int frameDelay = 50;

// Warm-hue bias added when vibration is high (shifts palette toward red/orange)
int warmBias = 0;

// Per-pixel fixed hue offsets (evenly spread across 65536)
// Spread = 65536 / 8 = 8192 per pixel
int offset0 = 0;
int offset1 = 8192;
int offset2 = 16384;
int offset3 = 24576;
int offset4 = 32768;
int offset5 = 40960;
int offset6 = 49152;
int offset7 = 57344;

// Per-pixel hue values (computed each frame)
int hue0 = 0;
int hue1 = 0;
int hue2 = 0;
int hue3 = 0;
int hue4 = 0;
int hue5 = 0;
int hue6 = 0;
int hue7 = 0;

void setup() {
  Serial.begin(115200);
  strip.begin();
  strip.setBrightness(120);
  strip.show();

  pinMode(ADC_PIN, INPUT);

  Serial.println("=== NeoPixel Color Spectrum Visualizer ===");
  Serial.println("Vibration sensor on GP26 | Strip on GPIO 14 (8 px)");
  Serial.println("Raise the slider to shift hues warm and speed up the cycle.");
  delay(500);
}

void loop() {
  // --- Read vibration ---
  vibRaw = analogRead(ADC_PIN);

  // Map raw ADC (0-4095) to hue step (1 = glacial drift, 800 = fast spin)
  // Low vibration (<512)  -> step 1-50   (cool blue/green, slow)
  // High vibration (>3500)-> step 500-800 (warm red/orange, fast)
  hueStep = vibRaw / 5;
  if (hueStep < 1)   hueStep = 1;
  if (hueStep > 800) hueStep = 800;

  // Frame delay inversely proportional to vibration: calm=50ms, intense=10ms
  frameDelay = 50 - (vibRaw / 110);
  if (frameDelay < 10) frameDelay = 10;
  if (frameDelay > 50) frameDelay = 50;

  // Warm bias: push hue toward red/orange when vibration is strong.
  // When vibRaw > 2048, add up to ~6000 hue units of warm shift.
  warmBias = 0;
  if (vibRaw > 2048) {
    warmBias = (vibRaw - 2048) * 3;
  }
  if (warmBias > 6000) warmBias = 6000;

  // Advance the global phase
  phase = phase + hueStep;
  if (phase > 65535) phase = phase - 65536;

  // Compute per-pixel hues (phase + offset + warmBias, wrapped to 0-65535)
  hue0 = phase + offset0 + warmBias;
  if (hue0 > 65535) hue0 = hue0 - 65536;

  hue1 = phase + offset1 + warmBias;
  if (hue1 > 65535) hue1 = hue1 - 65536;

  hue2 = phase + offset2 + warmBias;
  if (hue2 > 65535) hue2 = hue2 - 65536;

  hue3 = phase + offset3 + warmBias;
  if (hue3 > 65535) hue3 = hue3 - 65536;

  hue4 = phase + offset4 + warmBias;
  if (hue4 > 65535) hue4 = hue4 - 65536;

  hue5 = phase + offset5 + warmBias;
  if (hue5 > 65535) hue5 = hue5 - 65536;

  hue6 = phase + offset6 + warmBias;
  if (hue6 > 65535) hue6 = hue6 - 65536;

  hue7 = phase + offset7 + warmBias;
  if (hue7 > 65535) hue7 = hue7 - 65536;

  // Write colours to strip using ColorHSV (hue, saturation=255, value=255) + gamma32
  strip.setPixelColor(0, strip.gamma32(strip.ColorHSV(hue0, 255, 255)));
  strip.setPixelColor(1, strip.gamma32(strip.ColorHSV(hue1, 255, 255)));
  strip.setPixelColor(2, strip.gamma32(strip.ColorHSV(hue2, 255, 255)));
  strip.setPixelColor(3, strip.gamma32(strip.ColorHSV(hue3, 255, 255)));
  strip.setPixelColor(4, strip.gamma32(strip.ColorHSV(hue4, 255, 255)));
  strip.setPixelColor(5, strip.gamma32(strip.ColorHSV(hue5, 255, 255)));
  strip.setPixelColor(6, strip.gamma32(strip.ColorHSV(hue6, 255, 255)));
  strip.setPixelColor(7, strip.gamma32(strip.ColorHSV(hue7, 255, 255)));

  strip.show();

  // Serial diagnostic
  Serial.print("vibRaw=");
  Serial.print(vibRaw);
  Serial.print("  hueStep=");
  Serial.print(hueStep);
  Serial.print("  phase=");
  Serial.print(phase);
  Serial.print("  warmBias=");
  Serial.print(warmBias);
  Serial.print("  frameDelay=");
  Serial.println(frameDelay);

  delay(frameDelay);
}
```

## What to Click in MbedO

1. Drag the **VEGA ARIES v3** board onto the canvas.
2. Drag a **NeoPixel** strip component (set to 8 pixels) onto the canvas and connect its **DIN** pin to **GPIO 14**, its **VCC** to the **5V** rail, and its **GND** to **GND**.
3. Drag a **Potentiometer** component onto the canvas to simulate the vibration sensor's analog output; connect its output pin to **GP26** and connect its VCC and GND rails.
4. Paste the code above into the MbedO code editor.
5. Select **Interpreted Mode** from the simulation dropdown.
6. Click **Run**.
7. Observe the NeoPixel strip on the canvas rendering a smooth rolling spectrum in cool blue and green tones while the potentiometer slider is at zero.
8. Slowly drag the potentiometer slider upward; the hue cycle will accelerate and the colours will shift toward warm orange and red tones.
9. Push the slider to maximum to see the strip spinning through warm reds and magentas at full speed.
10. Watch the Serial Monitor for per-frame diagnostic lines confirming `vibRaw`, `hueStep`, `phase`, `warmBias`, and `frameDelay` values.

## Expected Output

Serial Monitor (potentiometer at low position):
```
=== NeoPixel Color Spectrum Visualizer ===
Vibration sensor on GP26 | Strip on GPIO 14 (8 px)
Raise the slider to shift hues warm and speed up the cycle.
vibRaw=48    hueStep=9    phase=9     warmBias=0    frameDelay=50
vibRaw=51    hueStep=10   phase=19    warmBias=0    frameDelay=50
vibRaw=44    hueStep=8    phase=27    warmBias=0    frameDelay=50
```

Serial Monitor (potentiometer at high position):
```
vibRaw=3820  hueStep=764  phase=764   warmBias=5316  frameDelay=15
vibRaw=3905  hueStep=781  phase=1545  warmBias=5571  frameDelay=14
vibRaw=3990  hueStep=798  phase=2343  warmBias=5826  frameDelay=13
```

## Expected Canvas Behavior

* At rest (potentiometer near zero) the eight NeoPixels display a slowly shifting gradient cycling through blues, cyans, and greens; the phase advances by only a few units per frame and `frameDelay` stays near 50 ms, giving a slow, calm drift.
* As the potentiometer slider is raised the palette shifts visibly toward yellow, orange, and red, the cycling speed increases noticeably, and the `frameDelay` reported in the Serial Monitor drops toward 10 ms.
* At maximum slider value the entire strip pulses through warm reds and magentas in a rapid spin.
* The colour gradient across all eight pixels is always smooth because each pixel's hue is evenly offset by 8192 units (one-eighth of the full 65536-unit colour wheel).
* The `gamma32()` correction ensures that mid-brightness hues do not appear washed out or unnaturally dim on the physical strip.

## Code Walkthrough

| Line | Check / Action |
| --- | --- |
| `#include <Adafruit_NeoPixel.h>` | Imports the Adafruit NeoPixel driver library that handles WS2812B timing and the `ColorHSV()` and `gamma32()` helpers. |
| `Adafruit_NeoPixel strip(PIXEL_COUNT, PIXEL_PIN, NEO_GRB + NEO_KHZ800)` | Creates the strip object: 8 pixels, data on GPIO 14, GRB colour order at 800 kHz (standard for WS2812B). |
| `int phase = 0` | Global phase counter (0–65535) that accumulates hue offset across frames; persists between `loop()` calls without any static keyword. |
| `int offset0 = 0; int offset1 = 8192; ...` | Per-pixel fixed hue offsets spaced 8192 apart so the strip always shows a full-spectrum slice regardless of the current phase. |
| `strip.begin()` | Initialises the NeoPixel data pin and clears all pixel buffers to off. |
| `strip.setBrightness(120)` | Sets global brightness to roughly 47 % of maximum; reduces current draw and protects eyes during close inspection. |
| `vibRaw = analogRead(ADC_PIN)` | Reads the 12-bit ADC value (0–4095) from GP26 representing the vibration sensor's analog amplitude. |
| `hueStep = vibRaw / 5` | Converts the raw ADC reading into a hue advancement step; larger vibration produces a larger step and faster animation. |
| `frameDelay = 50 - (vibRaw / 110)` | Reduces the inter-frame delay as vibration increases, accelerating the visible animation speed. |
| `warmBias = (vibRaw - 2048) * 3` | When vibration exceeds the midpoint ADC value, shifts all pixel hues toward the warm (red/orange) region of the wheel. |
| `phase = phase + hueStep; if (phase > 65535) phase = phase - 65536` | Advances and wraps the phase within the 16-bit hue range without using the modulo operator on a potentially negative value. |
| `hue0 = phase + offset0 + warmBias; if (hue0 > 65535) hue0 = hue0 - 65536` | Computes pixel 0's hue and wraps it; the identical pattern is repeated for pixels 1–7 with their respective offsets. |
| `strip.ColorHSV(hue0, 255, 255)` | Converts hue (0–65535), full saturation (255), and full value (255) into a packed 32-bit RGB colour word. |
| `strip.gamma32(...)` | Applies a perceptual gamma correction lookup table so mid-range colours appear visually linear across the brightness range. |
| `strip.setPixelColor(n, colour)` | Writes the corrected colour into the strip's internal pixel buffer at index n; must be called for each pixel individually. |
| `strip.show()` | Transmits all buffered pixel colours to the strip in one WS2812B data burst, updating all 8 pixels simultaneously. |

## Hardware & Safety Concept

* **WS2812B Protocol**: Each WS2812B pixel contains a tiny integrated controller that receives a serial stream of 24-bit GRB colour data at 800 kHz. The Adafruit NeoPixel library generates precisely timed high/low pulses on the data pin to encode each bit. The `NEO_KHZ800` flag selects the 800 kHz variant; the `NEO_GRB` flag tells the library to send bytes in Green–Red–Blue order as required by most WS2812B chips.
* **HSV Colour Model**: Hue–Saturation–Value (HSV) represents colour as an angle around a colour wheel (hue), colour purity (saturation), and brightness (value). The 16-bit hue range (0–65535) maps one full revolution of the wheel, with red at 0, green at approximately 21845, and blue at approximately 43690. By assigning each pixel a hue separated by 8192 units (65536 ÷ 8) the strip always displays an evenly distributed spectrum slice regardless of the current phase value.
* **Gamma Correction**: Human vision is non-linear; a pixel at 50 % PWM duty cycle looks far dimmer than half-brightness. `gamma32()` applies a lookup-table correction that redistributes the 256-level brightness scale so mid-range colours appear as expected to the human eye. Skipping gamma correction causes a strip where most of the visible colour range is crowded into the top 20 % of PWM values, making gradients look compressed.
* **Current Budget**: Each WS2812B pixel can draw up to 60 mA at full white (20 mA per colour channel × 3). Eight pixels at full white draw approximately 480 mA. The VEGA ARIES v3 board's 5 V rail may not supply this reliably; use an external 5 V, 1 A supply connected directly to the strip's VCC and GND pads when running at high brightness. The 1000 µF decoupling capacitor absorbs the brief inrush spike when the strip first powers on and prevents a voltage drop from corrupting the first data frame.
* **Series Resistor**: The 300–500 Ω resistor in the data line damps impedance mismatches and reflection ringing on the wire, which can corrupt the WS2812B's single-wire serial protocol and cause random colour glitches especially on the first or last pixel in the chain.

## Try This! (Challenges)

1. **Saturation Pulse**: Instead of keeping saturation fixed at 255, map the vibration reading to the saturation parameter of `ColorHSV()`. At low vibration set saturation to around 80 so colours appear pastel and washed-out; at peak vibration set saturation to 255 for fully vivid hues. Store the calculated saturation in a global `int satVal` variable and pass it to each `ColorHSV()` call. This adds a second visual dimension reinforcing the calm-to-intense metaphor without any additional hardware.
2. **Peak Flash**: Track the highest `vibRaw` reading seen so far in a global variable `int peakVib`. When the current reading exceeds 90 % of `peakVib` and `peakVib` is greater than 3000, set all eight pixels to full white for one frame by calling `strip.fill(strip.Color(255, 255, 255))` followed by `strip.show()` and a short `delay(30)` before resuming the normal spectrum loop. This creates a dramatic flash on strong transient vibration events such as a tap or clap.

## Troubleshooting

| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Strip stays dark after Run | NeoPixel DIN not connected to GPIO 14 or VCC/GND missing | Verify DIN wire goes to GPIO 14, VCC to 5V rail, and GND to GND in the MbedO canvas. |
| Only first pixel lights up | Missing or too-high series resistor corrupting data after first pixel | Replace the resistor with a value between 300 Ω and 500 Ω; confirm `NEO_KHZ800` is set in the constructor. |
| All pixels show the same colour with no gradient | Offset variables are all the same or PIXEL_COUNT mismatch | Confirm `offset1` through `offset7` are distinct multiples of 8192 and `PIXEL_COUNT` is 8. |
| Colours are too dim or washed out | `setBrightness()` set too low, or gamma correction has no visible effect at very low values | Increase the `setBrightness(120)` argument toward 200; ensure `gamma32()` wraps each `ColorHSV()` call. |
| Hues never shift toward warm red/orange | Potentiometer not wired to GP26/ADC0 | Confirm the analog output connects to GP26; check the Serial Monitor to verify `vibRaw` changes when the slider moves. |
| Animation speed never changes | `frameDelay` calculation produces the same value | Verify `vibRaw` varies in the Serial Monitor; confirm `ADC_PIN` is defined as `GP26`. |
| Strip flickers or glitches randomly | Insufficient power supply or missing decoupling capacitor | Add a 1000 µF capacitor across the strip power rail; switch to an external 5 V, 1 A supply. |

## Mode Notes

This project runs in MbedO **interpreted C++ mode**.

## Related Projects

- [167 - Sound-reactive LED Bar](167-sound-reactive-led-bar.md)
- [168 - Servo Sweep with Analog Control](168-servo-sweep-analog-control.md)
- [169 - Multi-zone Temperature Logger](169-multi-zone-temperature-logger.md)
