# Setup
- #homeassistant is required and #esphome
- you need working wifi signal next to your gas meter 
- gasmeter, i have an G4 RF1 
- some sort of ESP8266, i am using a Wemos D1 mini
- hall sensor and breakout board, i used a KY-024  (pay attention to the voltage, it needs to output 3,3v not 5v)

# Detailed Video in German: 
https://youtu.be/yp7ZDe6VlVM

Example Code i used to create a smart gasmeter for a typical german Gas Meter
check out 00-gasmeter.yaml

# Files
- `00-gasmeter.yaml` – analog variant (hall sensor on A0)
- `gasmeter-v1-simple.yaml` – digital pulse on D1 only
- `gasmeter-v2.yaml` – analog (A0) and digital (D1) in parallel, needs a 49E hall sensor
- `configuration.yaml` – Home Assistant excerpt (costs in €, utility meters)

# Install
- create a new board / instance in ESPhome in your Homeassistant
- add the code from one of the yaml files
- add these keys to your ESPHome `secrets.yaml`:
  ```yaml
  wifi_ssid: "..."
  wifi_password: "..."
  api_key_gasmeter: "..."   # generate one at https://esphome.io/components/api.html
  ota_gasmeter: "..."
  ```
- check the analog input voltages and adjust `threshold_upper` / `threshold_lower` under `substitutions`
- adjust `imp_ratio` if your gas meter is not 1 imp = 0,01 m³

- it can take quite some testing until you figured out the right values for the upper and lower limit

# Zählerstand und kWh einstellen
After flashing, open the device in Home Assistant. Under "Configuration" you find:
- **Zählerstand setzen** – enter the current reading of your gas meter in m³. The counter continues from there.
- **Zustandszahl** and **Brennwert** – both values are on your gas bill. kWh = m³ × Zustandszahl × Brennwert.

The counter is stored in flash and survives reboots and power loss. To protect the flash it is written at most every 10 minutes (`flash_write_interval`), so after a power loss up to 10 minutes of pulses can be missing – compare with the meter now and then and correct the reading.

After setting the reading for the first time the values jump once. Reset your utility meters with the action `utility_meter.calibrate` (value 0).

The kWh sensor uses `device_class: energy`, so it can be selected in the Home Assistant energy dashboard as gas consumption.

# Hardware:

IMPORTANT! If you want to use variant 2, you need to have an 49E hall Sensor. 49E will give an output based on the magnetic field, not only if there is a field or not. 
Some boards are labeled as 49E but have other Hall sensors. 
Variante 1: 
Hall Sensor https://amzn.to/3H5S6zG *

Variante 2:
49E lineare Hall Sensoren: https://amzn.to/4100lq5 * 

ESP8266 https://amzn.to/3kBZawl * 
Netzteil https://amzn.to/3XDVOaI *

Sämtliche mit „*“ markierten Links hängen mit Amazon Partnerprogrammen zusammen. Sie dienen dir als potentiellen Käufer als Orientierung und verweisen explizit auf bestimmte Produkte. Sofern diese Links genutzt werden, kann im Falle einer Kaufentscheidung eine Provision an mich ausgeschüttet werden. Bei einem Kauf über diese Links fallen natürlich keine Mehrkosten an. Danke!!

All links marked with "*" are related to Amazon affiliate programs. They serve you as a potential buyer as an orientation and refer explicitly to certain products. If these links are used, a commission can be paid to me in case of a purchase decision. In case of a purchase via these links, there are of course no additional costs. Thank you!!


