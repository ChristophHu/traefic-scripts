# Grafana

## System

## Prepare
1. install ssh,
2. install docker and docker-compose

### SSH


### Docker

## Structure
| Dienst | Beschreibung | Erreichbar unter |
| --- | --- | --- |
| Mosquitto | MQTT Broker (intern) | mqtt://localhost:1883 |
| InfluxDB | Zeitreihen-Datenbank | https://influx.yourdomain |
| Grafana | Dashboard-Visualisierung | https://grafana.yourdomain |
| Node-RED | Flow-basierte Automatisierung | https://nodered.yourdomain |
| Traefik | Reverse Proxy mit SSL & Routing | HTTP/HTTPS |
| CrowdSec | Schutz vor Brute-Force & Bot-Angriffen | über Traefik integriert |

```plaintext
grafana-stack/
├── docker-compose.yml
├── traefik/
│   ├── traefik.yml
│   └── dynamic/
│       └── conf.yml
├── crowdsec/
│   └── config.yaml (wird automatisch erzeugt)
```

## Script
```bash
#!/bin/bash

set -e

INFLUXDB_DB="shelly"
INFLUXDB_ADMIN_USER="admin"
INFLUXDB_ADMIN_PASSWORD="adminpass"

mkdir -p grafana-stack/mosquitto grafana-stack/nodered grafana-stack/influxdb grafana-stack/grafana
sudo chown -R 1000:1000 grafana-stack/nodered

cd grafana-stack

# Mosquitto Konfiguration
cat > mosquitto/mosquitto.conf <<EOF
listener 1883
allow_anonymous true
EOF

# Docker Compose
cat > docker-compose.yml <<EOF
services:

  mosquitto:
    image: eclipse-mosquitto:2.0
    volumes:
      - ./mosquitto/mosquitto.conf:/mosquitto/config/mosquitto.conf
    ports:
      - "1883:1883"
    restart: always

  influxdb:
    image: influxdb:1.8
    environment:
      - INFLUXDB_DB=$INFLUXDB_DB
      - INFLUXDB_ADMIN_USER=$INFLUXDB_ADMIN_USER
      - INFLUXDB_ADMIN_PASSWORD=$INFLUXDB_ADMIN_PASSWORD
      - INFLUXDB_HTTP_AUTH_ENABLED=true
    volumes:
      - influxdb_data:/var/lib/influxdb
    ports:
      - "8086:8086"
    restart: always

  nodered:
    image: nodered/node-red:latest
    ports:
      - "1880:1880"
    volumes:
      - ./nodered:/data
    depends_on:
      - mosquitto
      - influxdb
    restart: always

  grafana:
    image: grafana/grafana-oss:latest
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
    restart: always

volumes:
  influxdb_data:
  grafana_data:
EOF

echo "✅ Setup abgeschlossen."
echo "📡 MQTT: mqtt://localhost:1883"
echo "📈 InfluxDB: http://localhost:8086 ($INFLUXDB_ADMIN_USER/$INFLUXDB_ADMIN_PASSWORD)"
echo "🔧 Node-RED: http://localhost:1880"
echo "📊 Grafana: http://localhost:3000 (admin/admin)"

echo
echo "🚀 Starte jetzt mit:"
echo "   cd grafana-stack"
echo "   docker-compose up -d"
```

Node-RED öffnen (http://localhost:1880)
Palette öffnen → Plugin installieren: node-red-contrib-influxdb
MQTT-Node erstellen (mqtt in) → z. B. Topic shellies/#
InfluxDB-Node erstellen → Datenbank shelly, Auth admin/adminpass
Funktion zwischenschalten, falls nötig
Grafana öffnen (http://localhost:3000)
Login: admin / admin
Datenquelle InfluxDB einrichten → http://influxdb:8086, DB shelly