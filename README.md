# 🎮 Playstation Trophies ESP Display


<img width="4284" height="3213" alt="TrophyMainPic" src="https://github.com/user-attachments/assets/9f1f2633-c41e-4688-9b73-ae9e0469ad37" />


## [**3D model files can be found on MakerWorld**](https://makerworld.com/en/@tomtechie) 

### This project shows your PlayStation trophy stats on a small ESP32-C3 with an SSD1306 OLED display.

There are currently two versions of the project:



Home Assistant version → Uses an API and switches between two screens of information.  
More stable, recommended.



Arduino IDE version → Runs standalone, shows all info on a single screen.  
Eligibility Requirement: PSN Trophy Leaders does not include every PSN user. To be listed on their site, your profile must meet at least one of these conditions:  
- Be level 30 or higher  
- Have earned at least 1 platinum trophy  
- Have earned at least 100 trophies

If you do not meet these criteria, you must register manually on their site before your account can be added.

 
---
 


The Home Assistant version cycles through two clean, minimalistic screens showing your stats.
<img width="2780" height="908" alt="Untitled-1" src="https://github.com/user-attachments/assets/b8deaa27-dc2c-4930-b2fd-f9685d3b7a70" />


The Arduino IDE version condenses everything into one screen for quick viewing.
<img width="1468" height="1043" alt="IMG_5114" src="https://github.com/user-attachments/assets/2ea13a59-7e0d-451f-8021-481aa72e979a" />




---

## Installation


## Arduino IDE Steps
<details>
  <summary>🟢 Arduino IDE Installation</summary>


  ## Step 0: What You Need

| Item | Notes |
|------|-------|
| ESP32-C3 DevKit | Development board with USB-C |
| SSD1306 OLED 128x64 | I2C display |
| USB-C cable | For flashing |
| Computer with Arduino IDE installed | Arduino IDE Programming|
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

## Step 2: Arduino IDE Installation

- Download and install [**Arduino IDE**](https://www.arduino.cc/en/software/)


- In Arduino IDE, go to File → Preferences → add this URL to Additional Board Manager URLs:
```url
https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json
```
  Then open Tools → Board → Board Manager, search for ESP32, and click Install.

  Also Make sure you have the following libaries installed by going to "Manage Libaries" and searching for them: 
  - Adafruit GFX Libary
  - Adafruit SSD1306

-------------------------------------------------------


- Plug your ESP32-C3 into your computer with a USB cable.
- Then in Arduino IDE, go to Tools → Board and select ESP32C3 Dev Module.
- Copy & paste the following code into a new sketch in Arduino IDE:


```ino
/* PSNTrophyLeaders scraper on ESP32-C3 (ssd1306 display)
   - Fetches https://psntrophyleaders.com/user/view/<user>#games
   - Extracts total + platinum/gold/silver/bronze
   - Displays on 0.96" SSD1306
   - Sets hostname to PlaystationTrophiesESP
   - Shows "WiFi!" in top-right if not connected
   NOTE: Uses WiFiClientSecure.setInsecure() for TLS.
*/

#include <WiFi.h>
#include <WiFiClientSecure.h>
#include <HTTPClient.h>
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

// === CONFIG ===
const char* WIFI_SSID = "YOUR_WIFI";                     // CHange to your Wi-Fi name
const char* WIFI_PASS = "YOUR_PASSWORD";                 // Change to your Wi-Fi password
const char* PSNPROFILE_USER = "YOUR_PSN_USERNAME";       // change to your PSN username
const unsigned long FETCH_INTERVAL_MS = 10UL * 60UL * 1000UL; // 10 minutes

#define I2C_SDA_PIN 8
#define I2C_SCL_PIN 9

// ===== Helpers to parse HTML =====

// Extract the nth <big>…</big> value (0 = platinum, 1 = gold, 2 = silver, 3 = bronze)
String getBigValue(String& data, int nth) {
  int idx = -1;
  for (int i = 0; i <= nth; i++) {
    idx = data.indexOf("<big>", idx + 1);
    if (idx == -1) return "--";
  }
  int start = idx + 5;
  int end = data.indexOf("</big>", start);
  if (end == -1) return "--";
  String val = data.substring(start, end);
  val.trim();
  return val;
}

// Extract total from "Trophies (5615)"
long extractTotal(const String& html) {
  int pos = html.indexOf("Trophies (");
  if (pos < 0) return -1;
  int start = pos + 10;
  int end = html.indexOf(")", start);
  if (end < 0) return -1;
  String num = html.substring(start, end);
  num.replace(",", "");
  return num.toInt();
}

// fetch page via HTTPS (insecure TLS) and return HTML string
String fetchProfileHtml(const char* user) {
  if (WiFi.status() != WL_CONNECTED) {
    Serial.println("No WiFi, skipping fetch");
    return String();
  }
  String host = "psntrophyleaders.com";
  String url = String("/user/view/") + user + "#games";
  WiFiClientSecure client;
  client.setInsecure(); // Accept any cert
  HTTPClient https;
  String full = String("https://") + host + url;
  if (!https.begin(client, full)) {
    Serial.println("HTTPS begin failed");
    return String();
  }
  https.setUserAgent("esp-trophy-counter/1.0");
  int code = https.GET();
  String payload = "";
  if (code == HTTP_CODE_OK) {
    payload = https.getString();
  } else {
    Serial.printf("HTTP error: %d\n", code);
  }
  https.end();
  return payload;
}

// Helper: format large numbers with "K" (used only for G/S/B)
String formatK(String val) {
  val.replace(",", "");       // remove commas
  long n = val.toInt();
  if (n >= 10000) {
    long k = (n + 500) / 1000; // round to nearest K
    return String(k) + "K";
  }
  return val;
}

void drawDisplay(long total, String plat, String gold, String silver, String bronze, bool wifiOK) {
  display.clearDisplay();
  display.setTextColor(SSD1306_WHITE);

  // === Header ===
  display.setTextSize(1);
  display.setCursor(0, 0);
  display.print("PSN: ");
  display.println(PSNPROFILE_USER);

  // WiFi indicator
  if (!wifiOK) {
    display.setCursor(SCREEN_WIDTH - 30, 0);
    display.print("WiFi!");
  }

  // Full-width separator line
  display.drawLine(0, 10, SCREEN_WIDTH, 10, SSD1306_WHITE);

  // === Total trophies (big number, full number) ===
  String sTot = (total >= 0) ? String(total) : String("--");
  int size = 3;
  if (sTot.length() >= 5) size = 2;  // shrink if too long
  display.setTextSize(size);
  int16_t x1, y1;
  uint16_t w, h;
  display.getTextBounds(sTot, 0, 0, &x1, &y1, &w, &h);
  display.setCursor((SCREEN_WIDTH - w) / 2, 16);
  display.print(sTot);

  // === Platinum centered below total (full number) ===
  display.setTextSize(1);
  String platText = "Plat:" + plat;
  display.getTextBounds(platText, 0, 0, &x1, &y1, &w, &h);
  display.setCursor((SCREEN_WIDTH - w) / 2, 40);
  display.print(platText);

  // === Gold / Silver / Bronze all in one centered line (K format if >10k) ===
  String row = "G:" + formatK(gold) + " S:" + formatK(silver) + " B:" + formatK(bronze);
  display.getTextBounds(row, 0, 0, &x1, &y1, &w, &h);
  display.setCursor((SCREEN_WIDTH - w) / 2, 54);
  display.print(row);

  display.display();
}

void connectWiFi() {
  if (WiFi.status() == WL_CONNECTED) return;
  Serial.printf("Connecting to %s", WIFI_SSID);
  WiFi.begin(WIFI_SSID, WIFI_PASS);
  unsigned long start = millis();
  while (WiFi.status() != WL_CONNECTED && millis() - start < 10000) {
    delay(300);
    Serial.print(".");
  }
  if (WiFi.status() == WL_CONNECTED) {
    Serial.printf("\nConnected, IP: %s, Hostname: %s\n",
                  WiFi.localIP().toString().c_str(),
                  WiFi.getHostname());
  } else {
    Serial.println("\nWiFi connect failed.");
  }
}

void setup() {
  Serial.begin(115200);
  Wire.begin(I2C_SDA_PIN, I2C_SCL_PIN);
  display.begin(SSD1306_SWITCHCAPVCC, 0x3C);
  display.clearDisplay();
  display.setTextColor(SSD1306_WHITE);
  display.display();

  // Set custom hostname BEFORE WiFi.begin()
  WiFi.config(INADDR_NONE, INADDR_NONE, INADDR_NONE);
  WiFi.setHostname("PlaystationTrophiesESP");

  connectWiFi();
  drawDisplay(-1, "--", "--", "--", "--", WiFi.status() == WL_CONNECTED);
}

unsigned long lastFetch = 0;
long cachedTotal = -1;
String cachedPlat = "--", cachedGold = "--", cachedSilver = "--", cachedBronze = "--";

void loop() {
  connectWiFi();

  unsigned long now = millis();
  if (now - lastFetch >= FETCH_INTERVAL_MS || lastFetch == 0) {
    lastFetch = now;
    if (WiFi.status() == WL_CONNECTED) {
      Serial.println("Fetching PSNTrophyLeaders...");
      String html = fetchProfileHtml(PSNPROFILE_USER);
      if (html.length() > 0) {
        cachedTotal = extractTotal(html);
        cachedPlat = getBigValue(html, 0);
        cachedGold = getBigValue(html, 1);
        cachedSilver = getBigValue(html, 2);
        cachedBronze = getBigValue(html, 3);

        Serial.printf("Parsed: total=%ld plat=%s gold=%s silver=%s bronze=%s\n",
                      cachedTotal, cachedPlat.c_str(), cachedGold.c_str(),
                      cachedSilver.c_str(), cachedBronze.c_str());
      } else {
        Serial.println("Fetch failed or empty HTML - keeping old values");
      }
    }
  }

  drawDisplay(cachedTotal, cachedPlat, cachedGold, cachedSilver, cachedBronze,
              WiFi.status() == WL_CONNECTED);

  delay(500);
}

```

- Replace the following placeholders:
  - **`YOUR_WIFI`**
  - **`YOUR_PASSWORD`**
  - **`YOUR_PSN_USERNAME`**

- Upload the code to your ESP32C3
- Done


### Having issues?

Try the following:

- Is [**PSNTrophyLeaders**](https://psntrophyleaders.com) up?
- Try another username, for example try the top players username.
- Create a issue or send me a message on MakerWorld.

</details>


## Home Assistant Steps
<details>
  <summary>🔵 Home Assistant Installation</summary>



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


</details>



