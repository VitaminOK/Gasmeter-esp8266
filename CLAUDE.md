# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Projekt

ESPHome-Konfigurationen für einen smarten Gaszähler (typischer deutscher Balgengaszähler, z. B. G4 RF1) mit ESP8266 (Wemos D1 mini) und Hall-Sensor. Die Daten gehen über die ESPHome-API an Home Assistant. Es gibt keinen eigenen C++-Code, kein Build-System und keine Tests. Das Repo besteht nur aus YAML-Dateien, die in ESPHome bzw. Home Assistant eingefügt werden. Kommentare und Sensornamen sind überwiegend deutsch.

## Befehle

Kompiliert und geflasht wird mit ESPHome (lokal: `pip install esphome`, oder im ESPHome-Add-on von Home Assistant). Die Dateien ohne `.yaml`-Endung müssen für die CLI vorher als `.yaml` kopiert werden. Außerdem wird eine `secrets.yaml` mit `wifi_ssid`, `wifi_password` und (für V1/V2) `ota_gasmester` gebraucht.

```bash
esphome config 00-gasmeter.yaml    # Nur validieren (geht ohne Hardware, ersetzt "Tests")
esphome compile 00-gasmeter.yaml   # Firmware bauen
esphome run 00-gasmeter.yaml       # Bauen, flashen (USB/OTA) und Logs anzeigen
esphome logs 00-gasmeter.yaml      # Live-Logs, z. B. um ADC-Spannungen für die Schwellwerte abzulesen
```

## Aufbau: drei Varianten derselben Gerätekonfiguration

Alle drei Dateien beschreiben dasselbe Gerät (`name: 00-gasmeter`), nur mit unterschiedlicher Pulserkennung. Es wird immer nur eine davon geflasht:

| Datei | Erkennung | Board |
|---|---|---|
| `gas meter Version 1 simple` | Digital: KY-024-Digitalausgang an `D1` (`gpio`-Binary-Sensor mit `delayed_on`-Entprellung, 50 ms–1,5 s) | `d1_mini` |
| `00-gasmeter.yaml` (laut README die Hauptdatei) | Analog: linearer 49E-Hall-Sensor an `A0`, `analog_threshold` mit Hysterese (`upper`/`lower` in Volt, je nach Aufbau einstellen) | `d1_mini` |
| `00_gasmeter Version 2` | Beides parallel (analog an `A0` → `total_pulses2`, digital an `D1` → `total_pulses1`), um die Methoden zu vergleichen | `d1_mini` |

So läuft die Datenkette in jeder Variante:
1. Ein Binary-Sensor erkennt den Magnetimpuls am Zählwerk, und eine `on_press`-Lambda erhöht einen `globals`-Zähler (`restore_value: false`, nach einem Neustart also wieder 0).
2. `template`-Sensoren rechnen um: m³ = Pulse × `imp_ratio` (0,01 m³/Impuls), kWh = m³ × 10. Sie sind `state_class: total_increasing`, deshalb kommt HA mit dem Zurücksetzen auf 0 zurecht.
3. Home Assistant (`configuration.yaml`, nur ein Auszug, keine ESPHome-Datei) bildet daraus `gaskosten_euro` (kWh × Preis/kWh) und `utility_meter`-Zyklen (täglich, täglich ab 06:00, wöchentlich, monatlich, jährlich).

Was voneinander abhängt:
- Die HA-Entity-IDs entstehen aus den ESPHome-Sensornamen (`GasverbrauchAd` → `sensor.gasverbrauchad`, `Gasverbrauch-kWh` → `sensor.gasverbrauch_kwh`). Wird ein `name:` umbenannt, gehen die Verweise in `configuration.yaml` kaputt.
- `utility_meter` hängt an `sensor.gasverbrauchad`, also am **analogen** Zähler. Mit Version 1 (nur `GasverbrauchPuls`) muss die Quelle angepasst werden.
- `gastherme_brennerstart_counter` in `configuration.yaml` wird von keiner Datei hier erzeugt (er stammt aus einer anderen Integration).

## Eigenheiten

- In `00-gasmeter.yaml` ist der Faktor direkt in die Lambda geschrieben (`* 0.01` bzw. `* 0.1`), in V1/V2 steht er im Global `imp_ratio`. Der Kommentar dort spricht von „0.001“, gilt aber für 0.01.
- `ESP_LOGD` muss in den Lambdas vor dem `return` stehen, sonst wird es nie ausgeführt.
- V1 und V2 nutzen Pin `D1` doppelt (`gpio`-Binary-Sensor und `pulse_counter`). Deshalb brauchen beide Pin-Blöcke `allow_other_uses: true` und denselben `mode: INPUT_PULLUP`, sonst bricht ESPHome mit „Pin is used in multiple places“ ab.
- `ota:` ist noch in der alten Schreibweise (ohne `- platform: esphome`). Aktuelle ESPHome-Versionen (seit 2024.6) lehnen das ab.
- `gas meter Version 1 simple` enthält einen echten API-Encryption-Key im Klartext. Die anderen Dateien nutzen Platzhalter (`USEYOURWONKEY`, `"YOUR KEY"`). Neue Keys bitte nicht einchecken.
- Hardware: Der KY-024 muss mit 3,3 V betrieben werden (nicht 5 V). Für die Analog-Variante braucht man einen echten linearen 49E-Sensor, weil manche Boards nur so beschriftet sind. Video-Anleitung (deutsch): https://youtu.be/yp7ZDe6VlVM
