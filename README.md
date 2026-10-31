# Smarter Gaszähler mit ESP8266 und ESPHome

🇩🇪 Deutsch · 🇷🇺 [Русский](README.ru.md)

Liest einen typischen deutschen Balgengaszähler (z. B. G4 RF1) über einen Hall-Sensor aus und schickt Zählerstand, Verbrauch in kWh, Durchfluss und Leistung an Home Assistant.

Ausführliches Video (Deutsch): https://youtu.be/yp7ZDe6VlVM

## Voraussetzungen
- Home Assistant mit ESPHome
- WLAN-Empfang am Gaszähler
- Gaszähler mit Magnet in der letzten Ziffernrolle (z. B. G4 RF1, 1 Impuls = 0,01 m³)
- ESP8266, z. B. Wemos D1 mini
- Hall-Sensor-Board, z. B. KY-024 (auf 3,3 V achten, nicht 5 V)

## Dateien
| Datei | Variante |
|---|---|
| `00-gasmeter.yaml` | Analog: Hall-Sensor an A0, Schwellwert zählt die Impulse |
| `gasmeter-v1-simple.yaml` | Digital: Schaltausgang des Sensors an D1 |
| `gasmeter-v2.yaml` | Analog (A0) und digital (D1) parallel zum Vergleich, braucht einen linearen 49E-Hall-Sensor |
| `configuration.yaml` | Auszug für Home Assistant: Kosten in €, Tages-/Wochen-/Monatszähler |

## Verdrahtung

```
 KY-024 / 49E-Board          Wemos D1 mini
 ┌──────────────┐            ┌──────────────┐
 │ +  (VCC) ────┼────────────┼── 3V3        │
 │ G  (GND) ────┼────────────┼── G          │
 │ A0 (analog) ─┼────────────┼── A0         │   Variante analog / v2
 │ D0 (digital)─┼────────────┼── D1         │   Variante v1 / v2
 └──────────────┘            └──────────────┘
```

- Den Sensor **mit 3,3 V** versorgen, sonst liegen bis zu 5 V am Eingang des ESP.
- Einzelner 49E-Sensor (flache Seite zu dir): Pin 1 = VCC, Pin 2 = GND, Pin 3 = Ausgang → A0.
- Den Sensor in die Aussparung unter der letzten Ziffernrolle setzen, dort dreht sich der Magnet an der „0“ vorbei.

## Installation
1. In ESPHome ein neues Gerät anlegen und den Inhalt einer der YAML-Dateien einfügen.
2. In der ESPHome-Datei `secrets.yaml` diese Einträge ergänzen:
   ```yaml
   wifi_ssid: "..."
   wifi_password: "..."
   api_key_gasmeter: "..."   # neuen Schlüssel erzeugen: https://esphome.io/components/api.html
   ota_gasmeter: "..."
   ```
3. Oben unter `substitutions` prüfen:
   - `imp_ratio`: Wert pro Impuls in m³ (steht auf dem Zähler, z. B. „1 imp = 0,01 m³“)
   - `threshold_upper` / `threshold_lower`: Schaltschwellen für die analoge Variante, siehe Kalibrierung
4. Flashen. Das Gerät erscheint in Home Assistant unter *Einstellungen → Geräte & Dienste → ESPHome*.

## Kalibrierung

### Analoge Variante (`00-gasmeter.yaml`, `gasmeter-v2.yaml`)
1. Beim ADC-Sensor vorübergehend `internal: false` setzen (in `00-gasmeter.yaml` steht `true`), damit „GasMeterAnalogInput“ in Home Assistant sichtbar wird.
2. Gas laufen lassen (Heizung oder Warmwasser) und im Verlauf beobachten, wie sich die Spannung ändert, wenn die „0“ an der letzten Rolle vorbeidreht. Den kleinsten und den größten Wert notieren.
3. Schwellen berechnen:
   - Mitte M = (min + max) / 2, Hub H = max − min
   - `threshold_upper` = M + 0,1 × H
   - `threshold_lower` = M − 0,1 × H

   Beispiel: min 0,520 V, max 0,560 V → M = 0,540, H = 0,040 → upper 0,544, lower 0,536.
4. Der Abstand zwischen upper und lower ist die Hysterese. Ist er zu klein, zählt ein Impuls doppelt. Ist er zu groß, werden Impulse verpasst.
5. Danach `internal: true` setzen, wenn der Wert nicht mehr gebraucht wird.

Hinweis: Auf dem ESP8266 zeigt ESPHome die Spannung direkt am Chip-Eingang (0–1,0 V). Der D1 mini hat einen Spannungsteiler, der 3,2 V am Pin A0 auf 1,0 V herunterteilt. Deshalb sind die Werte so klein.

Werden bei hohem Durchfluss Impulse verpasst, beim ADC-Sensor `update_interval` von `2s` auf `500ms` verkleinern.

### Digitale Variante (`gasmeter-v1-simple.yaml`)
1. Am Poti des KY-024 drehen, bis die LED auf dem Board genau dann leuchtet, wenn die „0“ am Sensor vorbeikommt, und sonst aus ist.
2. `delayed_on` (zwischen 50 ms und 1,5 s) filtert Prellen. Zählt der Sensor doppelt, den Wert erhöhen.

### Kontrolle
Zählerstand setzen (siehe unten), einige m³ verbrauchen und dann mit dem mechanischen Zählwerk vergleichen.

## Einstellungen in Home Assistant
Auf der Geräteseite unter **Konfiguration**:

| Entität | Bedeutung |
|---|---|
| **Zählerstand setzen** | Aktuellen Stand vom Gaszähler in m³ eintragen. Ab dann wird von diesem Wert weitergezählt. |
| **Zustandszahl** | Von der Gasrechnung, Standard 0,95 |
| **Brennwert** | Von der Gasrechnung in kWh/m³, Standard 11,2 |
| **gasmeter Neustart** | Startet den ESP neu |

kWh = m³ × Zustandszahl × Brennwert

## Sensoren
| Entität | Einheit | Bedeutung |
|---|---|---|
| GasverbrauchAd / GasverbrauchPuls | m³ | Zählerstand, gleicher Wert wie am Gaszähler |
| Gasverbrauch-kWh | kWh | Energie, für das Energie-Dashboard |
| Gas Durchfluss | m³/h | Aktueller Durchfluss, aus dem Abstand der letzten beiden Impulse berechnet |
| Gas Leistung | kW | Aktuelle Leistung der Heizung |
| gasmeter WLAN Signal | dBm | Empfang am Zähler, besser als −75 dBm ist gut |
| gasmeter Uptime | s | Zeit seit dem letzten Neustart |
| gasmeter IP, ESPHome Version | – | Diagnose |

Zum **Durchfluss**: Bei 0,01 m³ pro Impuls kommt bei kleiner Leistung nur alle paar Minuten ein Impuls. Der Wert ist deshalb ein Mittel über den letzten Impulsabstand. Kommt der nächste Impuls später als erwartet, sinkt der Wert. Nach 5 Minuten ohne Impuls zeigt er 0 (`flow_timeout_ms`).

## Energie-Dashboard
*Einstellungen → Dashboards → Energie → Gasverbrauch hinzufügen* und entweder „GasverbrauchAd“ (m³) oder „Gasverbrauch-kWh“ auswählen.

## Speichern des Zählerstands
Der Zählerstand wird im Flash gespeichert und bleibt nach Neustart und Stromausfall erhalten. Um den Flash zu schonen, wird höchstens alle 10 Minuten geschrieben (`flash_write_interval`). Nach einem Stromausfall können also bis zu 10 Minuten an Impulsen fehlen. Den Wert ab und zu mit dem Zähler vergleichen und bei Bedarf neu setzen.

Nach dem ersten Setzen des Zählerstands springen die Werte einmalig hoch. Die Zähler aus `configuration.yaml` dann mit der Aktion `utility_meter.calibrate` (Wert 0) zurücksetzen.

## Fehlersuche
| Problem | Lösung |
|---|---|
| Es wird nichts gezählt | Spannung von „GasMeterAnalogInput“ ansehen, Schwellen neu kalibrieren. Bei der digitalen Variante das Poti einstellen. |
| Es wird doppelt gezählt | Abstand zwischen upper und lower vergrößern, bzw. `delayed_on` erhöhen |
| Impulse fehlen bei hohem Verbrauch | ADC `update_interval` verkleinern |
| Gerät ist oft offline | WLAN-Signal prüfen, eventuell einen Repeater in die Nähe stellen |
| Kompilierfehler bei `ota` oder Pin D1 | ESPHome aktualisieren, die Dateien sind für ESPHome ab 2024.6 geschrieben |

## Hardware

WICHTIG! Für Variante 2 wird ein 49E-Hall-Sensor gebraucht. Der 49E gibt eine Spannung abhängig von der Stärke des Magnetfelds aus, nicht nur „Feld da / Feld nicht da“. Manche Boards sind als 49E beschriftet, haben aber einen anderen Sensor.

Variante 1:
Hall Sensor https://amzn.to/3H5S6zG *

Variante 2:
49E lineare Hall Sensoren: https://amzn.to/4100lq5 *

ESP8266 https://amzn.to/3kBZawl *
Netzteil https://amzn.to/3XDVOaI *

Sämtliche mit „*“ markierten Links hängen mit Amazon Partnerprogrammen zusammen. Sie dienen dir als potentiellen Käufer als Orientierung und verweisen explizit auf bestimmte Produkte. Sofern diese Links genutzt werden, kann im Falle einer Kaufentscheidung eine Provision an mich ausgeschüttet werden. Bei einem Kauf über diese Links fallen natürlich keine Mehrkosten an. Danke!!

All links marked with "*" are related to Amazon affiliate programs. They serve you as a potential buyer as an orientation and refer explicitly to certain products. If these links are used, a commission can be paid to me in case of a purchase decision. In case of a purchase via these links, there are of course no additional costs. Thank you!!
