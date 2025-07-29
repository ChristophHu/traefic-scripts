# Grafana

## System

## Prepare
1. install ssh,
2. install docker and docker-compose

### SSH


### Docker


## Script
```bash
#!/bin/bash

set -e

# =========================
# === Einstellungen ===
# =========================

DOMAIN="meinedomain.de"
EMAIL="admin@$DOMAIN"
MQTT_USER="mqttuser"
MQTT_PASS="mqttpassword"

PROJECT_DIR=traefik-stack
mkdir -p $PROJECT_DIR
cd $PROJECT_DIR

# Ordnerstruktur
mkdir -p traefik/{certs,conf,dynamic}
mkdir -p prometheus/data
mkdir -p grafana
mkdir -p mosquitto/config
mkdir -p mosquitto/data
mkdir -p mosquitto/log
mkdir -p crowdsec
mkdir -p mqtt2prometheus

# Traefik ACME Datei
touch traefik/acme.json
chmod 600 traefik/acme.json

# Mosquitto Passwortdatei
docker run --rm -v "$(pwd)/mosquitto/config:/mosquitto/config" eclipse-mosquitto \
  mosquitto_passwd -b /mosquitto/config/passwordfile "$MQTT_USER" "$MQTT_PASS"

# =========================
# === Docker Compose ===
# =========================

cat > docker-compose.yml <<EOF
version: "3.8"

services:

  traefik:
    image: traefik:v3.0
    command:
      - "--api.dashboard=true"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.myresolver.acme.httpchallenge=true"
      - "--certificatesresolvers.myresolver.acme.httpchallenge.entrypoint=web"
      - "--certificatesresolvers.myresolver.acme.email=$EMAIL"
      - "--certificatesresolvers.myresolver.acme.storage=/acme.json"
      - "--providers.docker=true"
      - "--providers.file.directory=/etc/traefik/dynamic"
      - "--providers.file.watch=true"
      - "--log.level=INFO"
    ports:
      - "80:80"
      - "443:443"
      - "8080:8080"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
      - "./traefik/dynamic:/etc/traefik/dynamic"
      - "./traefik/acme.json:/acme.json"
    restart: unless-stopped

  grafana:
    image: grafana/grafana
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
    ports:
      - "3000:3000"
    volumes:
      - ./grafana:/var/lib/grafana
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.grafana.rule=Host(\`grafana.$DOMAIN\`)"
      - "traefik.http.routers.grafana.entrypoints=websecure"
      - "traefik.http.routers.grafana.tls.certresolver=myresolver"
      - "traefik.http.services.grafana.loadbalancer.server.port=3000"
      - "traefik.http.routers.grafana.middlewares=secure-headers@file"

  mosquitto:
    image: eclipse-mosquitto
    ports:
      - "1883:1883"
    volumes:
      - ./mosquitto/config:/mosquitto/config
      - ./mosquitto/data:/mosquitto/data
      - ./mosquitto/log:/mosquitto/log
    labels:
      - "traefik.enable=true"
      - "traefik.tcp.routers.mqtt.rule=HostSNI(\`mqtt.$DOMAIN\`)"
      - "traefik.tcp.routers.mqtt.entrypoints=websecure"
      - "traefik.tcp.routers.mqtt.tls=true"
      - "traefik.tcp.routers.mqtt.tls.certresolver=myresolver"
      - "traefik.tcp.services.mqtt.loadbalancer.server.port=1883"

  prometheus:
    image: prom/prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./prometheus/data:/prometheus
    restart: unless-stopped

  mqtt2prometheus:
    image: hannesaandresen/mqtt2prometheus:latest
    ports:
      - "9641:9641"
    volumes:
      - ./mqtt2prometheus/config.yaml:/config/config.yaml
    restart: unless-stopped

  crowdsec:
    image: crowdsecurity/crowdsec
    container_name: crowdsec
    volumes:
      - /var/log:/var/log:ro
      - /etc/crowdsec:/etc/crowdsec
      - /var/lib/crowdsec:/var/lib/crowdsec
      - ./crowdsec:/etc/crowdsec/acquis.d
    restart: unless-stopped
EOF

# =========================
# === Traefik Middleware ===
# =========================

cat > traefik/dynamic/middlewares.yml <<EOF
http:
  middlewares:
    secure-headers:
      headers:
        frameDeny: true
        sslRedirect: true
        stsIncludeSubdomains: true
        stsPreload: true
        contentTypeNosniff: true
EOF

# =========================
# === Prometheus Config ===
# =========================

cat > prometheus/prometheus.yml <<EOF
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
  - job_name: 'mqtt2prometheus'
    static_configs:
      - targets: ['mqtt2prometheus:9641']
EOF

# =========================
# === Mosquitto Config ===
# =========================

cat > mosquitto/config/mosquitto.conf <<EOF
allow_anonymous false
password_file /mosquitto/config/passwordfile
listener 1883
EOF

# =========================
# === mqtt2prometheus Config ===
# =========================

cat > mqtt2prometheus/config.yaml <<EOF
mqtt:
  server: tcp://mosquitto:1883
  topic_path: shellies/#
  client_id: mqtt2prometheus
  user: $MQTT_USER
  password: $MQTT_PASS

exporter:
  listen_port: 9641

metrics:
  - name: shelly_temperature
    help: "Temperature from Shelly"
    type: gauge
    mqtt_topic: shellies/shelly1/sensor/temperature
    value_template: "{{ . }}"
  - name: shelly_power
    help: "Power consumption from Shelly"
    type: gauge
    mqtt_topic: shellies/shelly1/relay/0/power
    value_template: "{{ . }}"
EOF

# =========================
# === Start Stack ===
# =========================

echo "🚀 Starte Docker-Stack..."
docker compose up -d

echo ""
echo "✅ Setup abgeschlossen!"
echo "Bitte folgende DNS-Einträge anlegen:"
echo "  grafana.$DOMAIN → <Deine öffentliche IP>"
echo "  mqtt.$DOMAIN    → <Deine öffentliche IP>"
echo ""
echo "📊 Grafana: https://grafana.$DOMAIN"
echo "🔐 MQTT über TLS: mqtt.$DOMAIN:443 (per TCP mit SNI)"
```
