# 🎮 Playstation Trophies ESP Display

Display your Playstation trophies on an SSD1306 OLED using an ESP32-C3 and Home Assistant.

---

## Step 0: What You Need

| Item | Notes |
|------|-------|
| ESP32-C3 DevKit | Development board with USB-C |
| SSD1306 OLED 128x64 | I2C display |
| USB-C cable | For flashing |
| Computer with Home Assistant | ESPHome integration required |
| Playstation Network account | For trophy data |

---

## Step 1: Wiring the OLED to ESP32-C3



| ESP32-C3  | → | SSD1306 OLED |
|------|-------|-------|
|------|-------|-------|
|GPIO 8|→|SDA|
|GPIO 9|→|SCL|
|GND|→|GND|
|3.3V|→|VCC|



---

## Step 2: Prepare Home Assistant

- Ensure [**Playstation integration**](https://www.home-assistant.io/integrations/playstation_network/) is installed.
- Confirm trophy sensors exist, e.g.:


  - sensor.USERNAME_platinum_trophies
  - sensor.USERNAME_gold_trophies
  - sensor.USERNAME_silver_trophies
  - sensor.USERNAME_bronze_trophies
  
- Make a note of your **USERNAME**.


---

## Step 3: Add Wi-Fi Secrets in Home Assistant
Documentation: https://www.home-assistant.io/docs/configuration/secrets/


Edit `/config/secrets.yaml`:

```yaml
wifi_ssid: "YOUR_WIFI_SSID"
wifi_password: "YOUR_WIFI_PASSWORD"
```

Using secrets avoids storing sensitive info directly in the YAML.


## Step 4: Download Arial Font

1. Download Arial.ttf.
2. Place it in /config/esphome/fonts/ in Home Assistant.
3. This allows ESPHome to render the text on the OLED.
