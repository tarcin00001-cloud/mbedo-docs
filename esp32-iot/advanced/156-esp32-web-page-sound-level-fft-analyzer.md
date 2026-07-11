# 156 - Web Page Sound Level FFT analyzer

Build a real-time audio spectrum analyzer on the ESP32 that samples an analog microphone on GPIO 34, computes a Fast Fourier Transform (FFT) algorithm in C++ to extract frequency components, and hosts a web dashboard with a live-scrolling HTML5 Canvas bar chart of the audio spectrum.

## Goal
Learn how to implement Fast Fourier Transform (FFT) mathematics in C++, configure high-frequency analog sampling rates, map frequency bins, and render audio spectrum charts.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. An analog microphone is on GPIO 34. The ESP32 collects 64 audio samples at a 4 kHz sampling frequency and computes a Cooley-Tukey radix-2 FFT to split the audio into 32 frequency bins. Navigating to the ESP32's IP address displays a webpage with a live bar graph of the frequency spectrum (from 0 Hz to 2 kHz) updating in real time.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Analog Sound Sensor / Microphone | `potentiometer` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **MbedO Note:** In simulation, use a potentiometer to simulate the audio amplitude. Turning it up and down at different speeds simulates different audio frequencies.

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Sound Module | OUT (Analog) | GPIO34 | Yellow | Audio amplitude input |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Power the microphone module from the 3.3V pin to minimize digital power supply noise.

## Code
```cpp
// Web Page Sound Level FFT analyzer (Microphone sampler + Radix-2 FFT math + Canvas Spectrum)
#include <WiFi.h>
#include <WebServer.h>
#include <math.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int MIC_PIN = 34;

WebServer server(80);

// FFT Configuration (Power of 2 for Radix-2 FFT)
const int SAMPLES = 64;
const double SAMPLING_FREQUENCY = 4000.0; // 4 kHz sampling rate -> 2 kHz Max frequency (Nyquist)

double vReal[SAMPLES];
double vImag[SAMPLES];

// Global spectrum magnitudes for API sharing
float magnitudes[SAMPLES / 2];

// Simple Cooley-Tukey Radix-2 FFT C++ implementation
// Avoids external library dependencies to ensure instant compilation
void computeFFT(double *vR, double *vI, uint16_t samples) {
  // 1. Bit-reversal permutation
  uint16_t j = 0;
  for (uint16_t i = 0; i < samples; i++) {
    if (i < j) {
      double tempR = vR[i];
      double tempI = vI[i];
      vR[i] = vR[j];
      vI[i] = vI[j];
      vR[j] = tempR;
      vI[j] = tempI;
    }
    uint16_t m = samples >> 1;
    while (m >= 1 && j >= m) {
      j -= m;
      m >>= 1;
    }
    j += m;
  }
  
  // 2. Compute butterfly stages
  for (uint16_t len = 2; len <= samples; len <<= 1) {
    double angle = -2.0 * PI / len;
    double wlenR = cos(angle);
    double wlenI = sin(angle);
    for (uint16_t i = 0; i < samples; i += len) {
      double wR = 1.0;
      double wI = 0.0;
      for (uint16_t k = 0; k < len / 2; k++) {
        uint16_t u = i + k;
        uint16_t v = u + len / 2;
        double tR = wR * vR[v] - wI * vI[v];
        double tI = wR * vI[v] + wI * vR[v];
        vR[v] = vR[u] - tR;
        vI[v] = vI[u] - tI;
        vR[u] += tR;
        vI[u] += tI;
        double nextWR = wR * wlenR - wI * wlenI;
        wI = wR * wlenI + wI * wlenR;
        wR = nextWR;
      }
    }
  }
}

// Samples the microphone at 4 kHz and computes the FFT spectrum
void runAudioFFT() {
  unsigned long microsecondsStep = 1000000 / SAMPLING_FREQUENCY;
  
  // 1. Collect samples
  for (int i = 0; i < SAMPLES; i++) {
    unsigned long start = micros();
    vReal[i] = (double)analogRead(MIC_PIN);
    vImag[i] = 0.0; // Imaginary part is 0 for raw input
    
    // Wait for the next sampling step
    while (micros() - start < microsecondsStep);
  }
  
  // Remove DC Offset (Subtract average value to center signal)
  double sum = 0;
  for (int i = 0; i < SAMPLES; i++) sum += vReal[i];
  double mean = sum / SAMPLES;
  for (int i = 0; i < SAMPLES; i++) vReal[i] -= mean;
  
  // 2. Run local FFT
  computeFFT(vReal, vImag, SAMPLES);
  
  // 3. Compute bin magnitudes
  for (int i = 0; i < SAMPLES / 2; i++) {
    magnitudes[i] = sqrt(vReal[i] * vReal[i] + vImag[i] * vImag[i]);
  }
}

// HTTP API endpoint returning JSON data
void handleGetFFT() {
  String json = "{\"magnitudes\":[";
  for (int i = 0; i < SAMPLES / 2; i++) {
    json += String(magnitudes[i], 1);
    if (i < (SAMPLES / 2) - 1) json += ",";
  }
  json += "]}";
  server.send(200, "application/json", json);
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Audio FFT Analyzer</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 520px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  canvas { background-color: #0f172a; border-radius: 8px; border: 1px solid #334155; margin-top: 20px; width: 100%; aspect-ratio: 2/1; }\n";
  html += "  .badge { display: inline-block; padding: 6px 12px; border-radius: 12px; font-size: 11px; font-weight: bold; background-color: #065f46; color: #a7f3d0; margin-bottom: 15px; }\n";
  html += "  .footer { text-align: center; margin-top: 25px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Audio FFT Spectrum</h1>\n";
  html += "  <div class=\"badge\">REAL-TIME FFT ACTIVE</div>\n";
  
  html += "  <canvas id=\"fftCanvas\" width=\"400\" height=\"200\"></canvas>\n";
  
  html += "  <p class=\"footer\">Frequency Range: 0 Hz to 2000 Hz | 32 Bins (62.5 Hz/Bin)</p>\n";
  html += "</div>\n";
  
  // JavaScript canvas bar chart drawing script
  html += "<script>\n";
  html += "  const canvas = document.getElementById('fftCanvas');\n";
  html += "  const ctx = canvas.getContext('2d');\n";
  
  html += "  function drawSpectrum(magnitudes) {\n";
  html += "    ctx.clearRect(0, 0, canvas.width, canvas.height);\n";
  
  // Draw grid lines
  html += "    ctx.strokeStyle = '#334155';\n";
  html += "    ctx.lineWidth = 1;\n";
  html += "    for(let y = 50; y < 200; y += 50) {\n";
  html += "      ctx.beginPath();\n";
  html += "      ctx.moveTo(0, y);\n";
  html += "      ctx.lineTo(canvas.width, y);\n";
  html += "      ctx.stroke();\n";
  html += "    }\n";
  
  // Draw spectrum bars
  html += "    const barWidth = canvas.width / magnitudes.length;\n";
  html += "    magnitudes.forEach((val, i) => {\n";
  html += "      const x = i * barWidth;\n";
  
  // Scale magnitude values to fit canvas height
  html += "      const height = Math.min(val * 0.15, canvas.height);\n";
  html += "      const y = canvas.height - height;\n";
  
  // Gradient fill color for bars
  html += "      const grad = ctx.createLinearGradient(x, y, x, canvas.height);\n";
  html += "      grad.addColorStop(0, '#38bdf8'); // sky blue top\n";
  html += "      grad.addColorStop(1, '#065f46'); // dark green bottom\n";
  
  html += "      ctx.fillStyle = grad;\n";
  html += "      ctx.fillRect(x + 2, y, barWidth - 4, height);\n";
  html += "    });\n";
  html += "  }\n";
  
  html += "  function pollFFT() {\n";
  html += "    fetch('/api/fft')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        drawSpectrum(data.magnitudes);\n";
  html += "      });\n";
  html += "  }\n";
  
  html += "  window.onload = function() {\n";
  html += "    pollFFT();\n";
  html += "    setInterval(pollFFT, 100); // Poll 10 times per second\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(MIC_PIN, INPUT);
  
  Serial.println("\nESP32 Audio FFT Node starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/api/fft", handleGetFFT);
  
  server.begin();
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
}

void loop() {
  server.handleClient();
  
  // Continuously sample audio and update FFT calculations
  runAudioFFT();
  
  delay(20);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and a **Potentiometer** (to simulate the microphone) onto the canvas.
2. Wire the potentiometer wiper to **GPIO34**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. In normal state, keep the potentiometer steady. Verify that the webpage bars are flat.
6. Slide the potentiometer slider up and down quickly. Verify that frequency bars appear on the webpage bar chart.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
```

Browser JSON Console (`/api/fft`):
```json
{"magnitudes":[120.4,15.2,5.1,0.5,0.0,0.0,0.0,0.0]}
```

## Expected Canvas Behavior
* Varying the simulated microphone input potentiometer generates dynamic audio frequency spectrum bars on the webpage.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `computeFFT(...)` | Executes the Cooley-Tukey bit-reversal and butterfly math calculation. |
| `1000000 / SAMPLING_FREQUENCY` | Calculates the microseconds delay between samples to match the sample rate. |
| `vReal[i] -= mean` | Removes the DC bias offset to isolate the AC audio signal. |
| `sqrt(vReal[i]*vReal[i] + vImag[i]*vImag[i])` | Calculates the absolute magnitude of each frequency bin. |

## Hardware & Safety Concept: Nyquist Frequency and High-Frequency ADC Sampling
* **Nyquist Frequency**: The Nyquist theorem states that to reconstruct a signal, you must sample at more than twice the highest frequency component (`Fs > 2 * Fmax`). With a sampling frequency of 4000 Hz, the maximum frequency we can analyze is 2000 Hz. Any audio above 2 kHz will fold back and cause distortion (aliasing). Solder an RC low-pass filter with a cutoff frequency of 2 kHz in series with the microphone output to act as an anti-aliasing filter.
* **FFT Scaling**: The Cooley-Tukey radix-2 FFT requires the number of samples to be a power of 2 (e.g. 32, 64, 128, 256). Increasing the samples increases frequency resolution but requires more memory and calculation time.

## Try This! (Challenges)
1. **OLED Spectrum Display**: Add an OLED screen (Project 60) and display a 16-bar spectrum analyzer graph locally.
2. **Frequency Trigger Relay**: Add a relay (GPIO 12) to turn ON if a high magnitude is detected in a specific frequency bin (e.g., detecting a 1000 Hz whistle).
3. **Double Sampling Rate**: Increase the sampling rate to 8000 Hz and SAMPLES to 128 to analyze audio up to 4 kHz.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The bars are all stuck at maximum height | DC offset not removed | Ensure the DC offset subtraction logic (`vReal[i] -= mean`) is running before the FFT calculations |
| Webpage updates are very slow | Sampling blocks server | The sampling loop blocks execution during the SAMPLES acquisition (64 samples @ 4kHz = 16ms). Keep the SAMPLES count small to prevent choking the web server |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [96 - ESP32 Web-controlled Sound Level OLED DB meter](96-esp32-web-controlled-sound-level-oled-db-meter.md)
- [154 - ESP32 Web Page Kalman Filter Graph](154-esp32-web-page-kalman-filter-graph.md)
- [157 - Web Page MPU-6050 3D Orientation renderer](157-esp32-web-page-mpu-6050-3d-orientation-renderer.md) (Next project)
