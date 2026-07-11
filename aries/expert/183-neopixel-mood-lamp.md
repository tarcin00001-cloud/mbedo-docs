# 183 - NeoPixel Mood Lamp

Build an ambient smart lamp using a WS2812B NeoPixel addressable LED strip that shifts color based on DHT22 temperature readings and adjusts brightness automatically using an LDR light sensor.

## Goal
Learn how to control addressable WS2812B LEDs, map continuous physical inputs (temperature) to color hues (RGB) using mathematical scaling, and scale output brightness using ambient light feedback without loops.

## What You Will Build
An intelligent home mood lamp. A DHT22 sensor tracks the temperature of the room. An LDR measures the ambient lighting. The 8-pixel NeoPixel strip changes color dynamically: fading from blue (cold, <15°C) to green (comfortable, ~25°C) and red (hot, >35°C). The LDR automatically dims the LEDs in a dark room to prevent glare. A potentiometer can be rotated to adjust the target hue manual offset.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| DHT22 Sensor | `dht22` | Yes | Yes |
| WS2812B NeoPixel Strip (8 LEDs)| `neopixel` | Yes | Yes |
| LDR (Photoresistor) | `ldr` | Yes | Yes |
| Potentiometer | `potentiometer` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Warning LED | `led` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| NeoPixel Strip | DIN | GPIO 13 | Yellow | NeoPixel data control |
| NeoPixel Strip | 5V | 5V | Red | Power |
| NeoPixel Strip | GND | GND | Black | Ground |
| DHT22 Sensor | DATA | GPIO 12 | Purple | Temperature input |
| DHT22 Sensor | VCC | 3V3 | Red | Power |
| DHT22 Sensor | GND | GND | Black | Ground |
| LDR | OUT (analog) | ADC1 | Blue | Ambient light input (GP27) |
| Potentiometer | Wiper | ADC0 | Orange | Manual color offset (GP26) |
| Potentiometer | VCC | 3V3 | Red | Power |
| Potentiometer | GND | GND | Black | Ground |
| Active Buzzer | + | GPIO 14 | Gray | Alarm buzzer |
| Active Buzzer | - | GND | Black | Ground |
| Warning LED | Anode | GPIO 15 | Red | Temperature warning LED |
| Warning LED | Cathode | GND | Black | Ground |

> **Wiring tip:** WS2812B NeoPixels are sensitive to signal reflection. Connecting a 330 Ohm resistor in series between the ARIES GPIO 13 and the DIN terminal of the strip prevents voltage spikes from destroying the first pixel.

## Code
```cpp
// NeoPixel Mood Lamp - VEGA ARIES v3
#include <Adafruit_NeoPixel.h>
#include <DHT.h>

const int NEO_PIN = 13;
const int DHT_PIN = 12;
const int LDR_PIN = ADC1;
const int POT_PIN = ADC0;
const int BUZZER_PIN = 14;
const int WARNING_LED = 15;

// Define a strip of 8 pixels
Adafruit_NeoPixel strip(8, NEO_PIN, NEO_GRB + NEO_KHZ800);
DHT dht(DHT_PIN, DHT22);

unsigned long lastSampleTime = 0;

void setup() {
  Serial.begin(115200);
  dht.begin();
  strip.begin();
  
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(WARNING_LED, OUTPUT);
  
  // Initialize all pixels to off
  strip.setPixelColor(0, 0, 0, 0);
  strip.setPixelColor(1, 0, 0, 0);
  strip.setPixelColor(2, 0, 0, 0);
  strip.setPixelColor(3, 0, 0, 0);
  strip.setPixelColor(4, 0, 0, 0);
  strip.setPixelColor(5, 0, 0, 0);
  strip.setPixelColor(6, 0, 0, 0);
  strip.setPixelColor(7, 0, 0, 0);
  strip.show();
  
  lastSampleTime = millis();
}

void loop() {
  unsigned long currentTime = millis();
  
  // Read sensors and update lamp every 1 second
  if (currentTime - lastSampleTime >= 1000) {
    lastSampleTime = currentTime;
    
    // Read Temperature
    float temp = dht.readTemperature();
    if (isnan(temp)) temp = 25.0; // Safe default
    
    // Read manual offset from potentiometer (-5 to +5 degrees)
    int potVal = analogRead(POT_PIN);
    float offset = map(potVal, 0, 1023, -50, 50) / 10.0;
    float adjustedTemp = temp + offset;
    
    // Read LDR value to scale brightness (0-1023)
    int lightVal = analogRead(LDR_PIN);
    float brightnessFactor = map(lightVal, 0, 1023, 10, 100) / 100.0; // scale between 10% and 100%
    
    // Calculate RGB values based on Temperature (Linear Interpolation)
    int r = 0;
    int g = 0;
    int b = 0;
    
    if (adjustedTemp <= 15.0) {
      r = 0; g = 0; b = 255; // Cold -> Blue
    } 
    else if (adjustedTemp >= 35.0) {
      r = 255; g = 0; b = 0; // Hot -> Red
      // Sound alarm if temperature exceeds 35C
      digitalWrite(BUZZER_PIN, HIGH);
      digitalWrite(WARNING_LED, HIGH);
    } 
    else {
      digitalWrite(BUZZER_PIN, LOW);
      digitalWrite(WARNING_LED, LOW);
      
      // Calculate color mix
      // Factor is 0.0 at 15C, 1.0 at 35C
      float factor = (adjustedTemp - 15.0) / 20.0;
      r = factor * 255;
      b = (1.0 - factor) * 255;
      
      // Add green peak in middle range (comfort zone ~25C)
      g = (1.0 - abs(factor - 0.5) * 2.0) * 180;
      if (g < 0) g = 0;
    }
    
    // Apply auto-brightness scaling
    int finalR = r * brightnessFactor;
    int finalG = g * brightnessFactor;
    int finalB = b * brightnessFactor;
    
    // Update all 8 LEDs manually without using loops
    strip.setPixelColor(0, finalR, finalG, finalB);
    strip.setPixelColor(1, finalR, finalG, finalB);
    strip.setPixelColor(2, finalR, finalG, finalB);
    strip.setPixelColor(3, finalR, finalG, finalB);
    strip.setPixelColor(4, finalR, finalG, finalB);
    strip.setPixelColor(5, finalR, finalG, finalB);
    strip.setPixelColor(6, finalR, finalG, finalB);
    strip.setPixelColor(7, finalR, finalG, finalB);
    strip.show();
    
    // Print telemetry to serial monitor
    Serial.print("Raw Temp: ");
    Serial.print(temp);
    Serial.print("C | Adjusted: ");
    Serial.print(adjustedTemp);
    Serial.print("C | Brightness: ");
    Serial.print(brightnessFactor * 100);
    Serial.print("% | Color R:");
    Serial.print(finalR);
    Serial.print(" G:");
    Serial.print(finalG);
    Serial.print(" B:");
    Serial.println(finalB);
  }
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **DHT22**, **NeoPixel Strip (8 pixels)**, **LDR**, **Potentiometer**, **Buzzer**, and **LED** onto the canvas.
2. Complete the wiring according to the table.
3. Paste the code into the editor and choose **Interpreted Mode**.
4. Click **Run**.
5. Adjust the DHT22 temperature widget and LDR light widget sliders to see the NeoPixel colors change and adapt brightness.

## Expected Output
Serial Monitor:
```
Raw Temp: 22.40C | Adjusted: 22.40C | Brightness: 80.00% | Color R:76 G:102 B:128
Raw Temp: 28.50C | Adjusted: 28.50C | Brightness: 45.00% | Color R:77 G:32 B:37
Raw Temp: 36.20C | Adjusted: 36.20C | Brightness: 90.00% | Color R:229 G:0 B:0
```

## Expected Canvas Behavior
* On running, the NeoPixel strip glows green-blue at normal room temperature.
* Raising the DHT22 temperature slider above 30°C shifts the NeoPixel color towards orange-red. Above 35°C, the strip turns solid red, the warning LED turns ON, and the buzzer sounds.
* Lowering the LDR slider dims the strip to a soft glow.
* Rotating the potentiometer slider offsets the color values manually.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `Adafruit_NeoPixel strip(...)` | Configures the NeoPixel driver for an 8-LED strip on control pin 13. |
| `strip.begin()` | Prepares the GPIO pin for transmitting high-frequency NZR protocols. |
| `adjustedTemp = temp + offset` | Incorporates manual potentiometer trim values into physical temperature reads. |
| `brightnessFactor` | Determines attenuation scalar based on photoresistor voltage. |
| `factor = (adjustedTemp - 15.0) / 20.0` | Maps the temperature domain (15 to 35) to a normalized factor (0.0 to 1.0). |
| `strip.setPixelColor(0, ...)` | Sequentially writes values to index buffers. No loops are declared. |

## Hardware & Safety Concept
* **High NeoPixel Current Draw**: One NeoPixel LED consumes up to 60mA at full brightness white (Red + Green + Blue). For a strip of 8 LEDs, this equals 480mA, which is safe for USB power. Larger strips must be powered by an external 5V power supply to protect the ARIES onboard LDO regulator.
* **Auto-Dimming Safety**: Reducing brightness in dark environments not only saves battery power in off-grid solar lamps, but also reduces visual fatigue.

## Try This! (Challenges)
1. **Pulse Breathing Effect**: Add code to pulse the brightness factor up and down slowly by +/- 10% using a sine wave function based on `millis()` to make the lamp "breathe".
2. **Humidity Rainbow Trigger**: When ambient humidity exceeds 70% (indicating rain or moisture), bypass the temperature mapping and display a solid blue color.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| First pixel is destroyed / dead | Voltage spikes on DIN | Add a 330 Ohm resistor in series on the data line. |
| Colors are completely wrong (e.g. Red is Green) | Wrong color order configuration | Change `NEO_GRB` to `NEO_RGB` in the NeoPixel constructor. |
| LEDs flicker | Common ground missing | Make sure the GND of the NeoPixel strip is securely connected to the ARIES board GND. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [56 - Automatic Night Light](../intermediate/56-automatic-night-light.md)
- [152 - Servo PID Angle Balancer](../advanced/152-servo-pid-angle-balancer.md)
- [180 - Automated Greenhouse](180-automated-greenhouse.md)
