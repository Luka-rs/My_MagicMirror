# Smart Mirror with Motion-Activated Display

This project is a motion-activated MagicMirror² setup using a Raspberry Pi and an ultrasonic sensor to automatically turn a connected monitor on and off.

---

## 🧰 Materials

- HC-SR04 Ultrasonic Sensor
- 22" HDMI Monitor
- Raspberry Pi 3B+
- Jumper wires (male-female)

---

## 💻 MagicMirror² Setup

### 🔧 Installation

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y git curl
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs
git clone https://github.com/MichMich/MagicMirror
cd MagicMirror
npm install
```

### ⚙️ Configuration

Edit the config file:

```bash
cp config/config.js.sample config/config.js
nano config/config.js
```

Basic settings for Zürich, Switzerland with clock and OpenMeteo weather (no API key needed):

```js
language: "de",
locale: "de-CH",
timeFormat: 24,
units: "metric",
modules: [
  {
    module: "clock",
    position: "top_left"
  },
  {
    module: "weather",
    position: "top_right",
    config: {
      weatherProvider: "openmeteo",
      type: "current",
      lat: 47.3769,
      lon: 8.5417
    }
  },
  {
    module: "weather",
    position: "top_right",
    header: "Weather Forecast",
    config: {
      weatherProvider: "openmeteo",
      type: "forecast",
      lat: 47.3769,
      lon: 8.5417
    }
  }
]
```

### 🔁 Autostart with PM2

```bash
sudo npm install -g pm2
cd ~/MagicMirror
DISPLAY=:0 pm2 start npm --name "magicmirror" -- start
pm2 save
pm2 startup
# Run the suggested sudo command after 'pm2 startup'
```

---

## 🐍 Python Sensor Script

### 🔧 Installation

```bash
sudo apt install python3-rpi.gpio
```

Save the script as `monitor_control.py` in your home directory (`/home/pi/monitor_control.py`).

### 📜 Python Script

```python
import RPi.GPIO as GPIO
import time
import subprocess

TRIG = 23
ECHO = 24
HDMI_OUTPUT = "HDMI-A-1"
DISTANCE_THRESHOLD_CM = 60
HYSTERESIS_CM = 5
CHECK_INTERVAL_SEC = 1
STATE = None

GPIO.setmode(GPIO.BCM)
GPIO.setup(TRIG, GPIO.OUT)
GPIO.setup(ECHO, GPIO.IN)

def get_distance():
    GPIO.output(TRIG, False)
    time.sleep(0.05)
    GPIO.output(TRIG, True)
    time.sleep(0.00001)
    GPIO.output(TRIG, False)
    start_time = time.time()
    stop_time = time.time()
    while GPIO.input(ECHO) == 0:
        start_time = time.time()
    while GPIO.input(ECHO) == 1:
        stop_time = time.time()
    elapsed = stop_time - start_time
    return round((elapsed * 34300) / 2, 1)

def set_display(state):
    global STATE
    if STATE == state:
        return
    subprocess.run(["wlr-randr", "--output", HDMI_OUTPUT, f"--{state}"])
    STATE = state

set_display("off")

try:
    while True:
        dist = get_distance()
        if STATE != "on" and dist <= DISTANCE_THRESHOLD_CM:
            set_display("on")
        elif STATE != "off" and dist >= DISTANCE_THRESHOLD_CM + HYSTERESIS_CM:
            set_display("off")
        time.sleep(CHECK_INTERVAL_SEC)
except KeyboardInterrupt:
    GPIO.cleanup()
```

### 🔁 Autostart with systemd

```bash
mkdir -p ~/.config/systemd/user
nano ~/.config/systemd/user/monitor_control.service
```

Paste this:

```ini
[Unit]
Description=Ultrasonic Monitor Control
After=graphical-session.target

[Service]
ExecStart=/usr/bin/python3 /home/pi/monitor_control.py
Restart=on-failure
Environment=WAYLAND_DISPLAY=wayland-0
Environment=XDG_RUNTIME_DIR=/run/user/1000

[Install]
WantedBy=default.target
```

Enable the service:

```bash
loginctl enable-linger $USER
systemctl --user daemon-reload
systemctl --user enable monitor_control
systemctl --user start monitor_control
```

---

## ✅ Done!

Your MagicMirror is now motion-activated using a simple ultrasonic sensor setup. Enjoy your smart mirror! 🪞🚀