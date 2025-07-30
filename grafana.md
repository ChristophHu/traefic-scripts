# Grafana

## System

## Prepare
1. install ssh,
2. install docker and docker-compose

### SSH


### Docker

## Structure
| Dienst | Beschreibung | Port | Erreichbar unter |
| --- | --- | --- | --- |
| Mosquitto | MQTT Broker (intern) | 1883 | mqtt://localhost:1883 |
| InfluxDB | Zeitreihen-Datenbank | 8086 | https://influx.yourdomain |
| Grafana | Dashboard-Visualisierung | 3000 | https://grafana.yourdomain |
| Node-RED | Flow-basierte Automatisierung | 1880 | https://nodered.yourdomain |
| Traefik | Reverse Proxy mit SSL & Routing | | HTTP/HTTPS |
| CrowdSec | Schutz vor Brute-Force & Bot-Angriffen | | über Traefik integriert |

```plaintext
grafana-stack/
├── crowdsec/
│   └── config.yaml (wird automatisch erzeugt)
├── docker-compose.yml
├── .env
├── grafana/
├── influxdb/
├── mosquitto/
│   └── mosquitto.conf
├── nodered/
├── setup.sh
└── traefik/
    ├── traefik.yml
    └── dynamic/
        └── conf.yml
```

## Script mit Traefik und CrowdSec
```bash
mkdir -p /etc/docker/grafana-stack
nano /etc/docker/grafana-stack/.env
```

```env
# General
DOMAIN=yourdomain.com
EMAIL=you@example.com

# InfluxDB
INFLUXDB_DB=shelly
INFLUXDB_ADMIN_USER=admin
INFLUXDB_ADMIN_PASSWORD=adminpass

# CrowdSec
CROWDSEC_BOUNCER_KEY=REPLACE_THIS_LATER
```

`grafana-setup.sh` unter `/etc/docker/grafana-stack/` erstellen:
```bash
#!/usr/bin/env bash

set -e

cd /etc/docker/grafana-stack

# .env laden
if [ -f .env ]; then
  export $(grep -v '^#' .env | xargs)
else
  echo "❌ .env Datei fehlt!"
  exit 1
fi

# Verzeichnisse
mkdir -p {mosquitto,nodered,influxdb,grafana,traefik/dynamic,crowdsec}
chown -R 1000:1000 nodered

# Mosquitto config
cat > mosquitto/mosquitto.conf <<EOF
listener 1883
allow_anonymous true
EOF

# Traefik static config
cat > traefik/traefik.yml <<EOF
entryPoints:
  web:
    address: ":80"
  websecure:
    address: ":443"

certificatesResolvers:
  letsencrypt:
    acme:
      email: $EMAIL
      storage: /letsencrypt/acme.json
      httpChallenge:
        entryPoint: web

providers:
  docker:
    exposedByDefault: false
  file:
    directory: /dynamic
    watch: true
EOF

mkdir -p traefik/letsencrypt
touch traefik/letsencrypt/acme.json
chmod 600 traefik/letsencrypt/acme.json

# Dynamic config (CrowdSec Middleware)
cat > traefik/dynamic/conf.yml <<EOF
http:
  middlewares:
    crowdsec-bouncer:
      forwardAuth:
        address: http://crowdsec-bouncer:8080/api/v1/forward-auth
        trustForwardHeader: true
        authResponseHeaders:
          - X-Auth-User
EOF

echo "✅ Setup-Dateien erstellt."
echo "📦 Erstelle docker-compose.yml"

cat > docker-compose.yml <<EOF
services:

  traefik:
    image: traefik:v3.0
    command:
      - --configFile=/traefik.yml
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./traefik/traefik.yml:/traefik.yml:ro
      - ./traefik/dynamic:/dynamic:ro
      - ./traefik/letsencrypt:/letsencrypt
      - /var/run/docker.sock:/var/run/docker.sock
    restart: always

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
      - INFLUXDB_DB=${INFLUXDB_DB}
      - INFLUXDB_ADMIN_USER=${INFLUXDB_ADMIN_USER}
      - INFLUXDB_ADMIN_PASSWORD=${INFLUXDB_ADMIN_PASSWORD}
      - INFLUXDB_HTTP_AUTH_ENABLED=true
    volumes:
      - influxdb_data:/var/lib/influxdb
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.influx.rule=Host('influx.${DOMAIN}')"
      - "traefik.http.routers.influx.entrypoints=websecure"
      - "traefik.http.routers.influx.tls.certresolver=letsencrypt"
      - "traefik.http.services.influx.loadbalancer.server.port=8086"
    restart: always

  nodered:
    image: nodered/node-red:latest
    volumes:
      - ./nodered:/data
    depends_on:
      - mosquitto
      - influxdb
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.nodered.rule=Host('nodered.${DOMAIN}')"
      - "traefik.http.routers.nodered.entrypoints=websecure"
      - "traefik.http.routers.nodered.tls.certresolver=letsencrypt"
      - "traefik.http.services.nodered.loadbalancer.server.port=1880"
    restart: always

  grafana:
    image: grafana/grafana-oss:latest
    volumes:
      - grafana_data:/var/lib/grafana
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.grafana.rule=Host('grafana.${DOMAIN}')"
      - "traefik.http.routers.grafana.entrypoints=websecure"
      - "traefik.http.routers.grafana.tls.certresolver=letsencrypt"
      - "traefik.http.services.grafana.loadbalancer.server.port=3000"
    restart: always

  crowdsec:
    image: crowdsecurity/crowdsec
    container_name: crowdsec
    volumes:
      - ./crowdsec:/etc/crowdsec
      - /var/log:/var/log:ro
    restart: unless-stopped

  crowdsec-bouncer:
    image: fbonalair/traefik-crowdsec-bouncer
    environment:
      CROWDSEC_BOUNCER_API_KEY: ${CROWDSEC_BOUNCER_KEY}
      CROWDSEC_AGENT_HOST: crowdsec:8080
    restart: unless-stopped

volumes:
  influxdb_data:
  grafana_data:
EOF
echo "cd /etc/docker/grafana-stack/"
echo "✅ docker-compose.yml erstellt."
```

```bash
bash /etc/docker/grafana-stack/grafana-setup.sh
```

### CrowdSec Bouncer
🔐 CrowdSec API Key
Starte das Setup mit `docker compose up -d` im Ordner `/etc/docker/grafana-stack`.

Erzeuge den Bouncer-API-Key:
```bash
docker exec -it crowdsec cscli bouncers add traefik-bouncer
# r1EqHZlsPXTVwk4BTUmB9lIAmwVRsMKbnQEnxRqmLcQ
```
Trage den erzeugten Key in .env oder direkt im docker-compose.yml unter CROWDSEC_BOUNCER_API_KEY ein
```bash
docker compose up -d crowdsec-bouncer
```

### Test
```bash
dig grafana.meinedomain.de @127.0.0.1
dig influx.meinedomain.de @127.0.0.1
dig nodered.meinedomain.de @127.0.0.1
```

## Script ohne Traefik und CrowdSec
```bash
#!/bin/bash

set -e

INFLUXDB_DB="shelly"
INFLUXDB_ADMIN_USER="admin"
INFLUXDB_ADMIN_PASSWORD="adminpass"

mkdir -p grafana-stack/mosquitto grafana-stack/nodered grafana-stack/influxdb grafana-stack/grafana
chown -R 1000:1000 grafana-stack/nodered

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

### Shelly MQTT Setup
```json
{
  "settings": {
    "mqtt": {
      "enabled": true,
      "connectionType": "No SSL",
      "MQTT Prefix": "shelly 1pm",
      "Enable MQTT Control": false,
      "Enable RPC over MQTT": true,
      "RPC status notifications over MQTT": true,
      "Generic status update over MQTT": true,
      "server": "192.168.1.150:1883",
      "clientId": "shelly 1pm",
      "username": "",
      "password": ""
    }
  }
}
```

Node-RED öffnen (http://localhost:1880)
Palette öffnen → Plugin installieren: node-red-contrib-influxdb
MQTT-Node erstellen (mqtt in) → z. B. Topic shellies/#
InfluxDB-Node erstellen → Datenbank shelly, Auth admin/adminpass
Funktion zwischenschalten, falls nötig
Grafana öffnen (http://localhost:3000)
Login: admin / admin
Datenquelle InfluxDB einrichten → http://influxdb:8086, DB shelly