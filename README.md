# SignalK to Home Assistant — Helm Display

Replicates OpenCPN DashboardSK-style analogue gauges in Home Assistant, fed live from SignalK via MQTT.

## System Overview

```
Instruments / GPS / AIS
        │
   rpi4b (192.168.1.30)          ← PRIMARY SignalK server
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

rpi5sk (192.168.9.244) is a backup SignalK server that mirrors rpi4b and publishes identical MQTT topics to the same broker. If you switch to the backup, the dashboard continues working without any changes.

---

## Gauge Reference

| Gauge | HA Entity | Source SK Path | Unit | Conversion |
|-------|-----------|----------------|------|------------|
| Port RPM | `sensor.helm_port_rpm` | `propulsion.port.revolutions` | Hz→RPM | ×60 |
| Port Water Temp | `sensor.sk_port_water_temp` | `sensors.port.watertemp.celsius` | °C | — |
| Port Oil Pressure | `sensor.sk_port_oil_pressure` | `sensors.port.oilpressure.bar` | bar | — |
| Port Fuel Flow | `sensor.helm_port_fuel_lph` | `propulsion.port.fuel.rate` | m³/s→L/h | ×3,600,000 |
| Stbd RPM | `sensor.helm_starboard_rpm` | `propulsion.starboard.revolutions` | Hz→RPM | ×60 |
| Stbd Water Temp | `sensor.sk_starboard_water_temp` | `sensors.starboard.watertemp.celsius` | °C | — |
| Stbd Oil Pressure | `sensor.sk_starboard_oil_pressure` | `sensors.starboard.oilpressure.bar` | bar | — |
| Stbd Fuel Flow | `sensor.helm_starboard_fuel_lph` | `propulsion.starboard.fuel.rate` | m³/s→L/h | ×3,600,000 |
| Depth | `sensor.sk_depth` | `environment.depth.belowTransducer` | m | — |
| Apparent Wind Speed | `sensor.sk_wind_speed` | `environment.wind.speedApparentKnots` | kn | — |
| Apparent Wind Angle | `sensor.sk_wind_angle` | `environment.wind.angleApparentDegrees` | ° | — |
| SOG | `sensor.helm_sog` | `navigation.speedOverGround` | m/s→kn | ×1.94384 |
| Heading | `sensor.helm_heading` | `navigation.headingMagnetic` | rad→° | ×57.2958 |
| Fuel Economy | `sensor.sk_fuel_per_nm` | `sensors.engines.total.fuel.litersPerNm` | L/NM | — |

---

## Prerequisites

- **Home Assistant** with:
  - Mosquitto broker addon installed and running
  - HACS installed
  - **canvas-gauge-card** installed via HACS → Frontend
- **SignalK** (rpi4b) with **signalk-mqtt-gw** plugin enabled and configured (see below)
- Python 3 with `websockets` and `pyyaml` on your Mac (for pushing the dashboard)

---

## Step 1 — Configure signalk-mqtt-gw on rpi4b

SSH into rpi4b and ensure `/home/pi/.signalk/plugin-config-data/signalk-mqtt-gw.json` contains:

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
        "path": "[\"environment.wind.*\", \"environment.depth.*\", \"propulsion.port.revolutions\", \"propulsion.port.fuel.rate\", \"propulsion.port.temperature\", \"propulsion.starboard.revolutions\", \"propulsion.starboard.fuel.rate\", \"propulsion.starboard.temperature\", \"sensors.port.oilpressure.*\", \"sensors.port.watertemp.*\", \"sensors.starboard.oilpressure.*\", \"sensors.starboard.watertemp.*\", \"navigation.headingMagnetic\", \"navigation.speedOverGround\", \"sensors.engines.total.fuel.litersPerNm\"]",
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

> **For rpi5sk (backup):** copy the identical file across and restart SK there too, so failover keeps publishing the same topics.

---

## Step 2 — Create HA sensors via MQTT auto-discovery

The sensors are created by publishing auto-discovery messages to the MQTT broker. Run this Python script from your Mac:

```bash
pip install paho-mqtt   # if not already installed
python3 create_sensors.py
```

Or publish manually with mosquitto_pub. The key sensors and their discovery topics:

### Raw MQTT sensors (no conversion needed)
| unique_id | state_topic | unit |
|-----------|-------------|------|
| `sk_port_water_temp` | `vessels/self/sensors/port/watertemp/celsius` | °C |
| `sk_port_oil_pressure` | `vessels/self/sensors/port/oilpressure/bar` | bar |
| `sk_starboard_water_temp` | `vessels/self/sensors/starboard/watertemp/celsius` | °C |
| `sk_starboard_oil_pressure` | `vessels/self/sensors/starboard/oilpressure/bar` | bar |
| `sk_depth` | `vessels/self/environment/depth/belowTransducer` | m |
| `sk_wind_speed` | `vessels/self/environment/wind/speedApparentKnots` | kn |
| `sk_wind_angle` | `vessels/self/environment/wind/angleApparentDegrees` | ° |
| `sk_fuel_per_nm` | `vessels/self/sensors/engines/total/fuel/litersPerNm` | L/NM |

### Converted sensors (value_template does the unit conversion)
| unique_id | state_topic | value_template | unit |
|-----------|-------------|----------------|------|
| `helm_port_rpm` | `vessels/self/propulsion/port/revolutions` | `{{ (value\|float(0) * 60)\|round(0)\|int }}` | RPM |
| `helm_port_fuel_lph` | `vessels/self/propulsion/port/fuel/rate` | `{{ (value\|float(0) * 3600000)\|round(1) }}` | L/h |
| `helm_starboard_rpm` | `vessels/self/propulsion/starboard/revolutions` | `{{ (value\|float(0) * 60)\|round(0)\|int }}` | RPM |
| `helm_starboard_fuel_lph` | `vessels/self/propulsion/starboard/fuel/rate` | `{{ (value\|float(0) * 3600000)\|round(1) }}` | L/h |
| `helm_sog` | `vessels/self/navigation/speedOverGround` | `{{ (value\|float(0) * 1.94384)\|round(1) }}` | kn |
| `helm_heading` | `vessels/self/navigation/headingMagnetic` | `{{ (value\|float(0) * 57.2958)\|round(0)\|int }}` | ° |

Publish each as a retained message to `homeassistant/sensor/<unique_id>/config` with the JSON payload. Example for Port RPM:

```bash
mosquitto_pub -h 192.168.1.40 -p 1883 -u pi -P raspberry -r \
  -t "homeassistant/sensor/helm_port_rpm/config" \
  -m '{"unique_id":"helm_port_rpm","name":"Helm Port RPM","state_topic":"vessels/self/propulsion/port/revolutions","unit_of_measurement":"RPM","state_class":"measurement","value_template":"{{ (value | float(0) * 60) | round(0) | int }}","icon":"mdi:engine"}'
```

---

## Step 3 — Install canvas-gauge-card

1. HA → **HACS** → **Frontend** → **＋ Explore & Download**
2. Search **canvas-gauge-card** → Download
3. Accept the prompt to add a Lovelace resource

---

## Step 4 — Push the dashboard

Run from your Mac (requires Python `websockets` and `pyyaml`):

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

The dashboard will appear at **http://192.168.1.40:8123/helm-display**.

> If the `helm-display` dashboard doesn't exist yet, create it first:  
> HA → Settings → Dashboards → **＋ Add Dashboard** → Title: `Helm Display`, URL path: `helm-display`

---

## Step 5 — Connect SK on HA to rpi4b (optional)

This makes the SK instance running on the HA server (port 3000) a live replica of rpi4b — useful for other SK consumers on the network but not required for the MQTT gauges.

Open **http://192.168.1.40:3000/admin** → **Server → Data Connections → ＋ Add**

| Field | Value |
|-------|-------|
| Data type | Signal K over WebSocket |
| Host | 192.168.1.30 |
| Port | 3000 |
| Path | /signalk/v1/stream |
| selfHandling | manualSelf |
| remoteSelf | `vessels.urn:mrn:imo:mmsi:235032206` |

Save → Restart SK on HA.

---

## Customising the dashboard

Edit `helm_dashboard.yaml` then push using the script above, or edit directly in HA:

**HA → Helm Display → Edit (pencil) → Raw configuration editor**

### Key values to adjust

| Property | What it does |
|----------|-------------|
| `gauge.width` / `gauge.height` | Rendered diameter of the canvas in px |
| `card_height` | Height of the HA card slot in px |
| `highlights` | Colour zones — `from`/`to` are gauge values, `color` is rgba |
| `colorNeedle` | Needle colour |
| `colorPlate` | Gauge background colour |
| `minValue` / `maxValue` | Scale range |
| `majorTicks` | Tick label positions |

---

## Network reference

| Device | IP | Role |
|--------|----|------|
| rpi4b | 192.168.1.30 | PRIMARY SignalK — all instruments connected |
| rpi5sk | 192.168.9.244 | BACKUP SignalK — mirrors rpi4b, same MQTT output |
| Home Assistant | 192.168.1.40 | HA + MQTT broker + SK instance |

MQTT broker credentials: `pi` / `raspberry`  
SSH credentials (rpi4b & rpi5sk): `pi` / `raspberry`
