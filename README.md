# SignalK to Home Assistant — Helm Display

Replicates analogue engine and tank gauges in Home Assistant, fed live from SignalK via MQTT. Optimised for a Samsung Galaxy Tab Active 3 (SM-T575) running Fully Kiosk Browser in landscape.

## System Overview

```
Instruments / GPS / AIS / ECU (Kvaser)
        │
   rpi5sk (192.168.1.30)          ← PRIMARY SignalK server
   signalk-mqtt-gw plugin
        │
        ▼
   MQTT broker (192.168.1.40:1883)   ← Mosquitto on HA
        │
        ▼
   Home Assistant (192.168.1.40:8123)
   MQTT sensors → Lovelace canvas-gauge-card dashboard
```

### Failover

rpi5sk (192.168.9.244) is a backup SignalK server that mirrors the primary and publishes identical MQTT topics to the same broker. Switching to backup is transparent — the dashboard continues working without any changes.

---

## Dashboard Layout

3-row layout, all gauges filling the full screen width:

| Row | Left → Right |
|-----|-------------|
| 1 | PORT RPM \| STBD RPM |
| 2 | P Water C \| P Oil bar \| S Water C \| S Oil bar |
| 3 | P Fuel L/h \| P Tank L \| S Fuel L/h \| S Tank L |

**Colour coding:** PORT gauge titles navy blue (`#1a237e`), STBD teal (`#004d40`), needles red (`#c62828`), white plates.

Sized for Samsung Galaxy Tab Active 3 (960×600 CSS viewport in landscape): RPM 200px, small gauges 170px, total height ~570px.

---

## Gauge Reference

| Gauge | HA Entity | Source SK Path | Raw Unit | Display | Conversion |
|-------|-----------|----------------|----------|---------|------------|
| Port RPM | `sensor.helm_port_rpm` | `propulsion.port.revolutions` | Hz | RPM | ×60 |
| Port Water Temp | `sensor.sk_port_water_temp` | `sensors.port.watertemp.celsius` | °C | °C | — |
| Port Oil Pressure | `sensor.sk_port_oil_pressure` | `sensors.port.oilpressure.bar` | bar | bar | — |
| Port Fuel Flow | `sensor.helm_port_fuel_lph` | `propulsion.port.fuel.rate` | m³/s | L/h | ×3,600,000 |
| Port Tank | `sensor.sk_port_tank_l` | `tanks.fuel.port.remaining` | L | L | — |
| Stbd RPM | `sensor.helm_starboard_rpm` | `propulsion.starboard.revolutions` | Hz | RPM | ×60 |
| Stbd Water Temp | `sensor.sk_starboard_water_temp` | `sensors.starboard.watertemp.celsius` | °C | °C | — |
| Stbd Oil Pressure | `sensor.sk_starboard_oil_pressure` | `sensors.starboard.oilpressure.bar` | bar | bar | — |
| Stbd Fuel Flow | `sensor.helm_starboard_fuel_lph` | `propulsion.starboard.fuel.rate` | m³/s | L/h | ×3,600,000 |
| Stbd Tank | `sensor.sk_starboard_tank_l` | `tanks.fuel.starboard.remaining` | L | L | — |

### Gauge Zones

| Gauge | Red | Amber | Green | Amber | Red |
|-------|-----|-------|-------|-------|-----|
| RPM | — | 2800–3600 | 0–2800 | — | 3600–4400 |
| Water C | — | 40–70 (warm-up) | 70–90 | 90–100 | 100–120 |
| Oil bar | 0–2 | 2–3 | 3–8 | 8–10 | — |
| Fuel L/h | — | 14–18 | 0–14 | — | 18–24 |
| Tank L | 0–50 | 50–100 | 100–400 | — | — |

---

## Prerequisites

- **Home Assistant** with:
  - Mosquitto broker addon
  - HACS installed
  - **canvas-gauge-card** installed via HACS → Frontend
  - **kiosk-mode** installed via HACS → Frontend
  - **card-mod** installed via HACS → Frontend
- **SignalK** (rpi5sk) with **signalk-mqtt-gw** plugin configured (see Step 1)
- **fuel-tracker** SK plugin running (publishes `tanks.fuel.port.remaining` / `tanks.fuel.starboard.remaining`)
- Python 3 with `websockets`, `pyyaml`, `paho-mqtt` on your Mac

---

## Step 1 — Configure signalk-mqtt-gw

SSH into the SK server and ensure `/home/pi/.signalk/plugin-config-data/signalk-mqtt-gw.json` contains:

```json
{
  "configuration": {
    "runLocalServer": false,
    "port": 1883,
    "sendToRemote": true,
    "remoteHost": "mqtt://192.168.1.40",
    "username": "pi",
    "password": "raspberry",
    "rejectUnauthorized": false,
    "selectedOption": "1) vessels.self",
    "paths": [
      {
        "path": "[\"propulsion.port.revolutions\", \"propulsion.port.fuel.rate\", \"propulsion.starboard.revolutions\", \"propulsion.starboard.fuel.rate\", \"sensors.port.oilpressure.*\", \"sensors.port.watertemp.*\", \"sensors.starboard.oilpressure.*\", \"sensors.starboard.watertemp.*\", \"tanks.fuel.port.remaining\", \"tanks.fuel.starboard.remaining\", \"navigation.headingMagnetic\", \"navigation.speedOverGround\", \"environment.wind.*\", \"environment.depth.*\", \"sensors.engines.total.fuel.litersPerNm\"]",
        "interval": 1
      }
    ]
  },
  "enabled": true
}
```

Then restart SignalK:
```bash
sudo systemctl restart signalk
```

---

## Step 2 — Create HA MQTT sensors

Publish auto-discovery messages to the MQTT broker. Run from your Mac:

```bash
pip install paho-mqtt pyyaml websockets
python3 create_sensors.py
```

### Raw sensors (no conversion)
| unique_id | state_topic | unit |
|-----------|-------------|------|
| `sk_port_water_temp` | `vessels/self/sensors/port/watertemp/celsius` | °C |
| `sk_port_oil_pressure` | `vessels/self/sensors/port/oilpressure/bar` | bar |
| `sk_starboard_water_temp` | `vessels/self/sensors/starboard/watertemp/celsius` | °C |
| `sk_starboard_oil_pressure` | `vessels/self/sensors/starboard/oilpressure/bar` | bar |
| `sk_port_tank_l` | `vessels/self/tanks/fuel/port/remaining` | L |
| `sk_starboard_tank_l` | `vessels/self/tanks/fuel/starboard/remaining` | L |

### Converted sensors
| unique_id | state_topic | value_template | unit |
|-----------|-------------|----------------|------|
| `helm_port_rpm` | `vessels/self/propulsion/port/revolutions` | `{{ (value\|float(0) * 60)\|round(0)\|int }}` | RPM |
| `helm_port_fuel_lph` | `vessels/self/propulsion/port/fuel/rate` | `{{ (value\|float(0) * 3600000)\|round(1) }}` | L/h |
| `helm_starboard_rpm` | `vessels/self/propulsion/starboard/revolutions` | `{{ (value\|float(0) * 60)\|round(0)\|int }}` | RPM |
| `helm_starboard_fuel_lph` | `vessels/self/propulsion/starboard/fuel/rate` | `{{ (value\|float(0) * 3600000)\|round(1) }}` | L/h |
| `helm_sog` | `vessels/self/navigation/speedOverGround` | `{{ (value\|float(0) * 1.94384)\|round(1) }}` | kn |
| `helm_heading` | `vessels/self/navigation/headingMagnetic` | `{{ (value\|float(0) * 57.2958)\|round(0)\|int }}` | ° |

---

## Step 3 — Install HACS cards

1. HA → **HACS** → **Frontend** → **＋ Explore & Download**
2. Install **canvas-gauge-card**
3. Install **kiosk-mode**
4. Install **card-mod**

---

## Step 4 — Create the Helm Display dashboard

HA → **Settings → Dashboards → ＋ Add Dashboard**

| Field | Value |
|-------|-------|
| Title | `Helm Display` |
| URL path | `helm-display` |

---

## Step 5 — Push the dashboard config

```python
import asyncio, json, yaml, websockets

TOKEN = "your-ha-long-lived-token"   # HA → Profile → Security → Long-lived access tokens

async def push():
    config = yaml.safe_load(open("helm_dashboard.yaml").read())
    async with websockets.connect("ws://192.168.1.40:8123/api/websocket") as ws:
        await ws.recv()
        await ws.send(json.dumps({"type": "auth", "access_token": TOKEN}))
        await ws.recv()
        await ws.send(json.dumps({"id": 1, "type": "lovelace/config/save",
                                   "url_path": "helm-display", "config": config}))
        print(await ws.recv())

asyncio.run(push())
```

---

## Step 6 — Set up Fully Kiosk Browser on the tablet

1. Install **Fully Kiosk Browser** from the Play Store
2. Open it → three-dot menu → **Settings**
3. **Start URL**: `http://192.168.1.40:8123/helm-display/engines?kiosk`
   - The `?kiosk` parameter activates kiosk-mode, hiding the HA header and sidebar
4. **Web Content** → enable **Kiosk Mode** (hides browser chrome)
5. **Device Management** → enable **Keep Screen On**

---

## Customising

Edit `helm_dashboard.yaml` then push using the script above, or:
**HA → Helm Display → Edit (pencil) → Raw configuration editor**

| Property | What it does |
|----------|-------------|
| `gauge.width` / `gauge.height` | Canvas diameter in px — set both the same |
| `card_height` | HA card slot height in px — set ~10px above `height` |
| `highlights` | Colour zones — `from`/`to` are gauge values |
| `minValue` / `maxValue` | Scale range |
| `majorTicks` | Tick label positions (must be strings) |
| `colorTitle` / `colorUnits` | Label colours — navy `#1a237e` = PORT, teal `#004d40` = STBD |

**Sizing note:** Galaxy Tab Active 3 (SM-T575) has a 960×600 CSS viewport in landscape. Total `card_height` of all rows must stay under ~580px to avoid clipping.

---

## Network Reference

| Device | IP | Role |
|--------|----|------|
| rpi5sk | 192.168.1.30 | PRIMARY SignalK |
| Home Assistant | 192.168.1.40 | HA + MQTT broker |

MQTT credentials: `pi` / `raspberry`
SSH credentials: `pi` / `raspberry`
