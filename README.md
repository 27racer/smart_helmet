# Smart Helmet + Fall & Helmet-Removal Detection System

> ENGG1101 Project — Raspberry Pi 5

## Wiring Diagram

```
  Raspberry Pi 5 GPIO Header
  ┌─────────────────────────────────────────────┐
  │  3V3 [1]  [2]  5V                           │
  │  SDA [3]  [4]  5V     ← I2C power           │
  │  SCL [5]  [6]  GND                           │
  │      [7]  [8]                                │
  │  GND [9]  [10]                               │
  │      [11] [12] GPIO18 ← KY-012 Buzzer        │
  │      [13] [14] GND                           │
  │      [15] [16] GPIO23 ← HC-SR04 TRIG         │
  │  3V3 [17] [18] GPIO24 ← HC-SR04 ECHO *       │
  │      [19] [20] GND                           │
  │      [21] [22]                               │
  │      [23] [24]                               │
  │  GND [25] [26]                               │
  │      [27] [28]                               │
  │      [29] [30] GND                           │
  │      [31] [32]                               │
  │      [33] [34] GND                           │
  │      [35] [36]                               │
  │      [37] [38]                               │
  │  GND [39] [40] GPIO21 ← DHT22 Data           │
  │      [11] = GPIO17    ← IR Sensor Data        │
  └─────────────────────────────────────────────┘

  * HC-SR04 ECHO pin outputs 5V — use a voltage divider
    (1kΩ + 2kΩ) to bring it down to 3.3V for the Pi!
```

### Full Wiring Table

| Component        | Pin/Function | Raspberry Pi GPIO | Physical Pin | Notes                    |
|------------------|-------------|-------------------|-------------|--------------------------|
| **HC-SR04**      | VCC         | 5V                | Pin 2       |                          |
|                  | GND         | GND               | Pin 6       |                          |
|                  | TRIG        | GPIO 23           | Pin 16      |                          |
|                  | ECHO        | GPIO 24           | Pin 18      | ⚠ Use voltage divider!  |
| **DHT22**        | VCC         | 3.3V              | Pin 1       |                          |
|                  | GND         | GND               | Pin 9       |                          |
|                  | DATA        | GPIO 21           | Pin 40      | 10kΩ pull-up to 3.3V    |
| **MPU6050**      | VCC         | 3.3V              | Pin 17      |                          |
|                  | GND         | GND               | Pin 14      |                          |
|                  | SDA         | GPIO 2 (SDA)      | Pin 3       | I2C bus                  |
|                  | SCL         | GPIO 3 (SCL)      | Pin 5       | I2C bus                  |
| **VL53L0X**      | VCC         | 3.3V              | Pin 17      |                          |
|                  | GND         | GND               | Pin 14      |                          |
|                  | SDA         | GPIO 2 (SDA)      | Pin 3       | I2C bus                  |
|                  | SCL         | GPIO 3 (SCL)      | Pin 5       | I2C bus                  |
| **BH1750**       | VCC         | 3.3V              | Pin 17      |                          |
|                  | GND         | GND               | Pin 14      |                          |
|                  | SDA         | GPIO 2 (SDA)      | Pin 3       | I2C bus (addr: 0x23)     |
|                  | SCL         | GPIO 3 (SCL)      | Pin 5       | I2C bus                  |
| **SSD1306 OLED** | VCC         | 3.3V              | Pin 17      |                          |
|                  | GND         | GND               | Pin 14      |                          |
|                  | SDA         | GPIO 2 (SDA)      | Pin 3       | I2C bus (addr: 0x3C)     |
|                  | SCL         | GPIO 3 (SCL)      | Pin 5       | I2C bus                  |
| **IR Sensor**    | VCC         | 3.3V              | Pin 1       |                          |
|                  | GND         | GND               | Pin 25      |                          |
|                  | OUT         | GPIO 17           | Pin 11      | Digital output           |
| **KY-012 Buzzer**| Signal (+)  | GPIO 18           | Pin 12      |                          |
|                  | GND (-)     | GND               | Pin 34      |                          |

### I2C Bus Summary

All four I2C devices share the same SDA/SCL lines:

| Device   | Address |
|----------|---------|
| BH1750   | 0x23    |
| VL53L0X  | 0x29    |
| SSD1306  | 0x3C    |
| MPU6050  | 0x68    |

Verify with: `i2cdetect -y 1`

---

## Quick Start

```bash
# 1. Copy the project folder to your Pi
scp -r smart_helmet/ pi@<your-pi-ip>:~/

# 2. SSH into the Pi
ssh pi@<your-pi-ip>

# 3. Run the setup script (once)
cd ~/smart_helmet
chmod +x setup.sh
sudo ./setup.sh

# 4. Reboot if I2C was just enabled
sudo reboot

# 5. Run the system
cd ~/smart_helmet
source venv/bin/activate
python smart_helmet.py
```

---

## How It Works

### 1. Fall Detection (MPU6050)
The system uses a **two-phase** algorithm:
- **Phase 1 — Free-fall**: Total acceleration drops below 0.4g for ≥150ms
- **Phase 2 — Impact**: Acceleration spikes above 3.0g after free-fall
- Also detects rapid tumbling (gyro > 250°/s + high impact)

### 2. Helmet-Removal Detection (IR Sensor)
- IR sensor is mounted **inside** the helmet pointing at the wearer's head
- If no head is detected for 3+ continuous seconds → alert triggered

### 3. Proximity Warning (HC-SR04 + VL53L0X)
- **HC-SR04**: Long-range detection (2–400 cm), warns at 100 cm, danger at 40 cm
- **VL53L0X**: Precise short-range detection (up to 2m), warns at 600 mm, danger at 250 mm
- Both sensors work together for multi-range obstacle awareness

### 4. Heat Stress Monitoring (DHT22)
- Reads temperature and humidity inside the helmet
- Computes **Heat Index** (feels-like temperature) using the Rothfusz equation
- Three alert levels: Caution (27°C), Danger (32°C), Extreme (40°C)

### 5. Low Visibility Warning (BH1750)
- Ambient light below 50 lux triggers a warning

### 6. OLED Display
Shows real-time data: acceleration, distances, temperature, humidity, heat index, light level, and helmet status.

### 7. Buzzer Alert Patterns
| Pattern    | Sound                  | Meaning                  |
|------------|------------------------|--------------------------|
| Silent     | —                      | All OK                   |
| Short beep | • (every 1s)          | Warning (proximity/heat) |
| Double beep| •• (every 0.8s)       | Danger                   |
| Rapid alarm| ••••• (continuous)     | Fall detected            |
| Long beep  | ——— (every 1s)        | Helmet removed           |

---

## Adjusting Thresholds

All thresholds are defined at the top of `smart_helmet.py` in the **CONFIGURATION** section. Key values:

```python
FREEFALL_THRESHOLD_G  = 0.4    # Increase if false-positive falls
IMPACT_THRESHOLD_G    = 3.0    # Decrease for more sensitive impact detection
ULTRASONIC_WARN_CM    = 100    # Proximity warning distance
HEAT_INDEX_DANGER     = 32.0   # Heat danger threshold (°C)
LOW_LIGHT_LUX         = 50     # Low visibility threshold
```

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| `i2cdetect` shows no devices | Check SDA/SCL wiring; verify I2C is enabled in `raspi-config` |
| DHT22 returns None often | Normal — DHT22 is slow; ensure 10kΩ pull-up on data line |
| Ultrasonic always returns -1 | Check voltage divider on ECHO pin (must be 3.3V not 5V) |
| OLED is blank | Verify address with `i2cdetect -y 1` — should show 0x3C |
| Buzzer won't stop | Ctrl+C to stop the program; it will clean up GPIO |
| `lgpio` import error | Run `pip install lgpio` — required for Pi 5 GPIO |

---

## Auto-Start on Boot (Optional)

Create a systemd service:

```bash
sudo nano /etc/systemd/system/smart-helmet.service
```

Paste:

```ini
[Unit]
Description=Smart Helmet System
After=multi-user.target

[Service]
Type=simple
User=pi
WorkingDirectory=/home/pi/smart_helmet
ExecStart=/home/pi/smart_helmet/venv/bin/python smart_helmet.py
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Then enable:

```bash
sudo systemctl enable smart-helmet
sudo systemctl start smart-helmet
```
