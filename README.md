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

<img width="539" height="709" alt="SchematicForTrophy" src="https://github.com/user-attachments/assets/7b5961bb-f3f5-489b-9dac-5ca03852029c" />

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

---

## Step 4: Download Arial Font

1. Download [**Arial.ttf.**](https://github.com/talving/Playstation-Trophies-ESP-Display/blob/main/ARIAL.TTF) 
2. Place it in /config/esphome/fonts/ in Home Assistant.
3. This allows ESPHome to render the text on the OLED.



---

## Step 5: First Flash the ESP32-C3 with Default Config

1. In Home Assistant, Open ESPHome → + NEW DEVICE → ESP32-C3 DevKit.
2. Click INSTALL → Plug into USB and flash.
3. This generates:
  -  API key for Home Assistant integration
  -  OTA password for wireless updates
4. Write down the API key and OTA password


---

## Step 6: Full YAML Code for Copy-Paste

Copy this code into ESPHome.  
Replace **`USERNAME`**, **`DISPLAYNAME`**, **`YOUR_API_KEY`**, and **`YOUR_OTA_PASSWORD`** before flashing.


```yaml
substitutions:
  username: USERNAME 		## Set To HomeAssistant Playstation Online ID. Example: tomtechie

globals:
  - id: display_layout
    type: bool
    initial_value: 'true'
  - id: Username
    type: std::string
    initial_value: '"DISPLAYNAME"'     ## Set To The Username You Would Like Shown On The Display. Example: "Tomtechie"

interval:
  - interval: 20s              ## Set how long each screen should be shown. Example: 20s
    then:
      - lambda: 'id(display_layout) = !id(display_layout);'

esphome:
  name: playstationtrophiesesp
  friendly_name: PlaystationTrophiesESP

esp32:
  board: esp32-c3-devkitm-1
  framework:
    type: arduino

logger:

api:
  encryption:
    key: "xxxxxxxxxxxxxxxxxxxxxxxx"   ## Replace with API Key from ESPHome 

ota:
  - platform: esphome
    password: "xxxxxxxxxxxxxxxxxxx" ## Replace with password from ESPHome 

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

#  manual_ip:                      ##If you are having issues with Wifi connection, remove the comments here and set the Static IP.
#    static_ip: 192.168.1.236 
#    gateway: 192.168.1.1
#    subnet: 255.255.255.0
#    dns1: 192.168.1.1
#    dns2: 8.8.8.8

  ap:
    ssid: "Playstationtrophiesesp"
    password: "mzfAMXnmEblF"

captive_portal:

i2c:
  sda: 8
  scl: 9
  scan: true

font:
  - id: oled_font
    file: "fonts/ARIAL.TTF"    ## Download font and add it to /homeassistant/esphome/fonts/ in HomeAssistant
    size: 12

sensor:
  - platform: homeassistant
    id: plat_trophies
    entity_id: sensor.${username}_platinum_trophies
  - platform: homeassistant
    id: gold_trophies
    entity_id: sensor.${username}_gold_trophies
  - platform: homeassistant
    id: silver_trophies
    entity_id: sensor.${username}_silver_trophies
  - platform: homeassistant
    id: bronze_trophies
    entity_id: sensor.${username}_bronze_trophies


display:
  - platform: ssd1306_i2c
    model: "SSD1306_128x64"
    id: oled
    address: 0x3C
    update_interval: 1s
    lambda: |-
      int screen_width = 128;
      int screen_height = 64;
      int line_height = 16;  // vertical spacing

      // Helper lambda to measure text width using get_text_bounds
      auto centered_x = [&](std::string text) {
        int x1, y1, w, h;
        it.get_text_bounds(0, 0, text.c_str(), id(oled_font), TextAlign::TOP_LEFT, &x1, &y1, &w, &h);
        return (screen_width - w) / 2;
      };

      if (id(display_layout)) {
        // Screen One: Username + Platinum
        float plat_val = id(plat_trophies).state;
        if (isnan(plat_val)) plat_val = 0;

        std::string user_text = id(Username);
        std::string plat_text = "Platinum";
        std::string plat_number = std::to_string((int)plat_val);

        int total_height = line_height * 3;  // 3 lines
        int start_y = (screen_height - total_height) / 2;

        it.printf(centered_x(user_text),  start_y + line_height * 0, id(oled_font), "%s", user_text.c_str());
        it.printf(centered_x(plat_text),  start_y + line_height * 1, id(oled_font), "%s", plat_text.c_str());
        it.printf(centered_x(plat_number), start_y + line_height * 2, id(oled_font), "%s", plat_number.c_str());
      } else {
        // Screen Two: Gold, Silver, Bronze
        float gold_val = id(gold_trophies).state;
        if (isnan(gold_val)) gold_val = 0;
        float silver_val = id(silver_trophies).state;
        if (isnan(silver_val)) silver_val = 0;
        float bronze_val = id(bronze_trophies).state;
        if (isnan(bronze_val)) bronze_val = 0;

        std::string gold_text = "Gold:" + std::to_string((int)gold_val);
        std::string silver_text = "Silver:" + std::to_string((int)silver_val);
        std::string bronze_text = "Bronze:" + std::to_string((int)bronze_val);

        int total_height = line_height * 3;  // 3 lines
        int start_y = (screen_height - total_height) / 2;

        it.printf(centered_x(gold_text),   start_y + line_height * 0, id(oled_font), "%s", gold_text.c_str());
        it.printf(centered_x(silver_text), start_y + line_height * 1, id(oled_font), "%s", silver_text.c_str());
        it.printf(centered_x(bronze_text), start_y + line_height * 2, id(oled_font), "%s", bronze_text.c_str());
      }


```



---

## Step 7: Flash the Full Code


1. Click INSTALL → Plug into USB
2. After Wi-Fi connection, OLED will rotate between:
  -  Username + Platinum trophies
  -  Gold / Silver / Bronze trophies


---

## Step 8: Optional Customizations


- Change screen rotation interval (default: 20s)



---

## Having Wi-Fi issues?: Try static IP

Remove the **`#`** from the **`manual IP`** and set the settings according to your network


```yaml
  manual_ip:                      
    static_ip: 192.168.1.236 
    gateway: 192.168.1.1
    subnet: 255.255.255.0
    dns1: 192.168.1.1
    dns2: 8.8.8.8
```

