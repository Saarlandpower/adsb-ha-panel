# ✈️ ADS-B Dashboard für Home Assistant

Ein modernes, dunkles ADS-B Radar-Dashboard als Home Assistant Custom Panel — mit Live-Karte, Flugzeugfotos, Squawk-Alarmen und Push-Benachrichtigungen.

![ADS-B Dashboard Screenshot](screenshot.png)

> **Live:** 35+ Flugzeuge in Echtzeit · Flugzeugfotos · Squawk-Alerts via Telegram & Handy-Push

---

## Features

- **Live-Karte** mit Leaflet.js auf dunklem CartoDB-Hintergrund
- **Echtzeit-Flugzeugdaten** direkt vom `adsb.im` / tar1090-Feed (alle 5 s)
- **Flugzeugfotos** via planespotters.net (automatisch beim Anklicken)
- **Squawk-Alarm System** — visuelle Warnung + Push bei 7700 / 7600 / 7500 / 7777 / Militär
- **Telegram + Handy-Push** bei kritischen Squawk-Codes (Anti-Spam, einmal pro Flugzeug)
- **Radar-Sweep Animation** im Header mit Live-Status-Dot
- **Detail-Panel** mit ICAO, Squawk, Höhe, Speed, RSSI, Vertikalrate, FR24/ADSBx-Links
- **Höhenverteilung** (> 30.000 ft / 10.000–30.000 ft / < 10.000 ft)
- **Reichweitenringe** (50 / 100 / 200 / 300 km, toggle)
- **Flugpfade** (trail der letzten 40 Positionen, toggle)
- **Farbcodierung** nach Höhe: 🟡 Boden · 🟢 niedrig · 🔵 mittel · 🔴 hoch

---

## Voraussetzungen

| Komponente | Info |
|---|---|
| Home Assistant OS | 2023.x oder neuer |
| adsb.im Image auf Pi | aktuell (tar1090 / readsb JSON API) |
| File Editor Add-on | zum Bearbeiten der Konfigurationsdateien |
| HA Companion App | für Handy-Push (optional) |
| Telegram Bot | für Telegram-Benachrichtigungen (optional) |

---

## Installation

### Schritt 1 — ADS-B REST Sensor

In `/config/configuration.yaml` einfügen (IP und Port anpassen):

```yaml
rest:
  - resource: http://192.168.178.130:1090/aircraft.json
    scan_interval: 10
    sensor:
      - name: "ADS-B Aircraft JSON"
        value_template: "{{ value_json.now | default(0) }}"
        json_attributes:
          - aircraft
          - messages
          - now
```

Den Endpunkt prüfen: `http://<pi-ip>:<port>/data/aircraft.json` im Browser öffnen — erscheint JSON, stimmt die URL.

---

### Schritt 2 — CORS-Proxy auf dem Pi

Da HA über HTTPS läuft, aber der Pi nur HTTP anbietet, wird ein CORS-Proxy benötigt. Per SSH auf dem Pi ausführen:

```bash
sudo tee /usr/local/bin/adsb-cors-proxy.py << 'EOF'
#!/usr/bin/env python3
import http.server, urllib.request

class CORSProxy(http.server.BaseHTTPRequestHandler):
    def do_GET(self):
        try:
            with urllib.request.urlopen(f'http://127.0.0.1:1090{self.path}') as r:
                data = r.read()
            self.send_response(200)
            self.send_header('Content-Type', 'application/json')
            self.send_header('Access-Control-Allow-Origin', '*')
            self.end_headers()
            self.wfile.write(data)
        except Exception as e:
            self.send_error(502, str(e))
    def log_message(self, *a): pass

http.server.HTTPServer(('0.0.0.0', 1091), CORSProxy).serve_forever()
EOF

sudo chmod +x /usr/local/bin/adsb-cors-proxy.py

sudo tee /etc/systemd/system/adsb-cors-proxy.service << 'EOF'
[Unit]
Description=ADS-B CORS Proxy
After=network.target

[Service]
ExecStart=/usr/bin/python3 /usr/local/bin/adsb-cors-proxy.py
Restart=always
User=nobody

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now adsb-cors-proxy
```

Proxy läuft jetzt auf **Port 1091**, startet automatisch nach Reboot. Status prüfen:
```bash
sudo systemctl status adsb-cors-proxy
```

---

### Schritt 3 — Dashboard-Datei ablegen

Im File Editor Ordner anlegen und Datei hineinkopieren:
```
/config/www/adsb-panel/adsb-panel.html
```

---

### Schritt 4 — Konfiguration anpassen

In `adsb-panel.html` den `CONFIG`-Block oben im `<script>`-Bereich anpassen:

```javascript
const CONFIG = {
  adsbUrl:    'http://192.168.178.130:1091',  // IP deines Pi + CORS-Proxy Port
  refreshMs:  5000,                            // Aktualisierung in ms
  homeLat:    49.23,                           // Dein Breitengrad
  homeLon:    7.00,                            // Dein Längengrad
  maxRangeKm: 400,
  mapZoom:    9,
};
```

---

### Schritt 5 — Panel in HA registrieren

**Einstellungen → Dashboards → ⋮ → Dashboard hinzufügen → Webseite**

| Feld | Wert |
|---|---|
| Titel | ADS-B Radar |
| Icon | `mdi:radar` |
| URL | `/local/adsb-panel/adsb-panel.html?v=1` |

Das `?v=1` verhindert Browser-Caching. Bei Updates auf `?v=2` erhöhen.

---

### Schritt 6 — Squawk-Alarm System

Das Alarm-System wird in drei bestehende HA-Dateien eingefügt.

#### 6a — `configuration.yaml` (ganz unten einfügen)

```yaml
input_text:
  adsb_seen_squawks:
    name: "ADS-B Gesehene Squawk-Alarme"
    max: 255
    initial: ""
```

#### 6b — `template.yaml` (ans Ende einfügen)

```yaml
- sensor:
    - name: "ADS-B Aktive Squawk Alarme"
      unique_id: adsb_active_squawk_alerts
      state: >
        {% set aircraft = state_attr('sensor.ads_b_aircraft_json', 'aircraft') %}
        {% if aircraft %}
          {% set watch = ['7700','7600','7500','7777','7001','7002','7003','7004','7005','7006','7007'] %}
          {% set found = aircraft | selectattr('squawk', 'defined')
                                  | selectattr('squawk', 'in', watch)
                                  | list %}
          {{ found | count }}
        {% else %}
          0
        {% endif %}
```

#### 6c — Automationen anlegen

**Einstellungen → Automationen → + Automation → ⋮ → Als YAML bearbeiten**

**Automation 1 — Squawk Alarm:**

```yaml
id: adsb_squawk_alert
alias: "ADS-B Squawk Alarm"
mode: single
max_exceeded: silent
trigger:
  - platform: time_pattern
    seconds: "/15"
action:
  - variables:
      aircraft: "{{ state_attr('sensor.ads_b_aircraft_json', 'aircraft') }}"
      critical_squawks:
        "7700": "🚨 NOTFALL"
        "7600": "📻 FUNKAUSFALL"
        "7500": "🔫 ENTFÜHRUNG"
        "7777": "⚔️ MILITÄR ABFANG"
        "7001": "🪖 Militär"
        "7002": "🪖 Militär"
        "7003": "🪖 Militär"
        "7004": "🪖 Militär"
        "7005": "🪖 Militär"
        "7006": "🪖 Militär"
        "7007": "🪖 Militär"
      seen: "{{ states('input_text.adsb_seen_squawks') }}"
  - condition: template
    value_template: "{{ aircraft is not none and aircraft | count > 0 }}"
  - repeat:
      for_each: >
        {{ aircraft | selectattr('squawk', 'defined')
                    | selectattr('squawk', 'in', critical_squawks.keys() | list)
                    | list }}
      sequence:
        - variables:
            ac: "{{ repeat.item }}"
            squawk: "{{ ac.squawk }}"
            callsign: "{{ ac.flight | default(ac.hex) | trim }}"
            alarm_key: "{{ callsign }}_{{ squawk }}"
            squawk_label: "{{ critical_squawks[squawk] | default('⚠️ Unbekannt') }}"
            alt_ft: "{{ ac.alt_baro | default(0) | int }}"
            speed: "{{ ac.gs | default(0) | int }}"
            lat: "{{ ac.lat | default('?') }}"
            lon: "{{ ac.lon | default('?') }}"
        - condition: template
          value_template: "{{ alarm_key not in seen }}"
        - service: notify.DEIN_TELEGRAM_SERVICE
          data:
            message: >
              ✈️ ADS-B ALARM

              {{ squawk_label }}
              Squawk: {{ squawk }}
              Callsign: {{ callsign }}
              Höhe: {{ alt_ft }} ft · Speed: {{ speed }} kts
              Position: {{ lat }}, {{ lon }}

              🔗 https://www.flightradar24.com/{{ callsign }}
        - service: notify.mobile_app_DEIN_HANDY
          continue_on_error: true
          data:
            title: "✈️ {{ squawk_label }} – Squawk {{ squawk }}"
            message: "{{ callsign }} · {{ alt_ft }}ft · {{ speed }}kts"
            data:
              url: "https://www.flightradar24.com/{{ callsign }}"
              push:
                sound: default
        - service: input_text.set_value
          target:
            entity_id: input_text.adsb_seen_squawks
          data:
            value: >
              {% set current = states('input_text.adsb_seen_squawks') %}
              {% set keys = current.split(',') | reject('eq','') | list %}
              {% set keys = (keys + [alarm_key]) | unique | list %}
              {{ keys[-20:] | join(',') }}
```

**Automation 2 — Cache Reset (alle 2 Stunden):**

```yaml
id: adsb_squawk_reset
alias: "ADS-B Squawk Cache Reset"
trigger:
  - platform: time_pattern
    hours: "/2"
action:
  - service: input_text.set_value
    target:
      entity_id: input_text.adsb_seen_squawks
    data:
      value: ""
```

#### 6d — Notify-Services anpassen

In den Automationen ersetzen:
- `notify.DEIN_TELEGRAM_SERVICE` → z.B. `notify.telegram` oder `notify.Mathias`
- `notify.mobile_app_DEIN_HANDY` → deinen Handy-Service aus **Einstellungen → Geräte → Companion App**

---

### Schritt 7 — HA neu starten

**Einstellungen → System → Neu starten**

Nach dem Neustart erscheint **ADS-B Radar** im linken Menü.

---

## Squawk-Referenz

| Squawk | Bedeutung | Darstellung |
|---|---|---|
| 7700 | 🚨 Notfall (General Emergency) | Rot blinkend |
| 7500 | 🔫 Entführung (Hijack) | Rot blinkend |
| 7600 | 📻 Funkausfall (Radio Failure) | Gelb blinkend |
| 7777 | ⚔️ Militärischer Abfang | Lila |
| 7001–7007 | 🪖 Militär | Lila |

Eigene Codes in `adsb-panel.html` im `SQUAWK_ALERTS`-Objekt ergänzen.

---

## Bedienung

| Aktion | Funktion |
|---|---|
| Klick auf Flugzeugicon | Detail-Panel + Foto öffnen |
| Klick auf Listeneintrag | Dasselbe, Karte zentrieren |
| Klick auf Squawk-Alert-Banner | Direkt zum Flugzeug springen |
| `◎` Button | Reichweitenringe ein/aus |
| `∿` Button | Flugpfade ein/aus |
| `⌖` Button | Ansicht zurücksetzen |
| Filter-Eingabe | Suche nach Callsign oder ICAO |
| ALT / SPD | Sortierung umschalten |

---

## Updates einspielen

1. Neue `adsb-panel.html` im File Editor speichern
2. URL-Version im Dashboard erhöhen: `?v=2`, `?v=3` usw.

---

## Lizenz

MIT — frei nutzbar, veränderbar, teilbar.

---

## Credits

- [Leaflet.js](https://leafletjs.com/) — Karten
- [CartoDB](https://carto.com/) — Dark Tile Layer
- [planespotters.net](https://www.planespotters.net/) — Flugzeugfotos API
- [tar1090](https://github.com/wiedehopf/tar1090) — ADS-B JSON API
- [adsb.im](https://adsb.im/) — Pi Image

---

*Gebaut mit ♥ für Home Assistant · Saarbrücken, Deutschland*
