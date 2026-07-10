# Expert Projects

10 capstone projects using 5–6 components with complex multi-sensor logic, all written in interpreted-mode safe patterns.

No `for` loops. No arrays. All sensor reads use aliases and `if/else if/else` chains only.

---

## Batch 4A — Expert Systems (10 projects)

| # | Project | Components |
| --- | --- | --- |
| 116 | [Multi-Zone Fire System](116-multi-zone-fire-system.md) | Flame + MQ-2 + Relay + Buzzer + LCD |
| 117 | [Smart Lock](117-smart-lock.md) | MFRC522 + Servo + PIR + LED + Buzzer + LCD |
| 118 | [Full Weather Display](118-full-weather-display.md) | DHT22 + BMP180 + OLED |
| 119 | [BT Environment Monitor](119-bt-environment-monitor.md) | HC-05 + DHT22 + MQ-2 + LCD |
| 120 | [Autonomous Guard](120-autonomous-guard.md) | HC-SR04 + PIR + Buzzer + LED + LCD |
| 121 | [Greenhouse Controller](121-greenhouse-controller.md) | DHT22 + Water Level + Relay + Buzzer + LCD |
| 122 | [Cold Chain Monitor](122-cold-chain-monitor.md) | DS18B20 + Relay + Buzzer + LCD |
| 123 | [Energy Safety Monitor](123-energy-safety-monitor.md) | ACS712 + Relay + Buzzer + LCD |
| 124 | [Tilt Security Vault](124-tilt-security-vault.md) | MPU-6050 + MFRC522 + Servo + Buzzer + LCD |
| 125 | [Full BT Robot HUD](125-full-bt-robot-hud.md) | HC-05 + HC-SR04 + L298N + DC Motor + LCD |

---

## Prerequisites

Before attempting expert projects, complete at least:
- 5 intermediate projects using libraries (DHT22, HC-SR04, RFID, or HC-05)
- 2 advanced projects with multi-component systems
- Read [simulation-modes.md](../../courses/simulation-modes.md)

---

## Code Pattern Reminder

Expert projects use only these safe patterns:

```cpp
// Sensor alias → threshold if/else — always safe
float t = dht.readTemperature();
if (t > 35) { /* action */ } else { /* action */ }

// Multi-sensor compound condition — always safe
if (gas > 400 && flame == LOW) { /* action */ }

// BT command routing — always safe
if (BT.available()) {
  char c = BT.read();
  if (c == '1') { /* action */ }
  else if (c == '2') { /* action */ }
}
```
