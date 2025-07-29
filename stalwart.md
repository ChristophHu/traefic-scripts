# Stalwart

## Script
```bash
#!/bin/bash

set -e

PROJECT_DIR="traefik-mail"
DOMAIN="deinedomain.de"
EMAIL="deine@mailadresse.de"
CROWDSEC_API_KEY="traefik123456"

echo "[1/6] Erstelle Verzeichnisse..."
mkdir -p $PROJECT_DIR/{traefik/dynamic,stalwart,crowdsec}

echo "[2/6] Erstelle acme.json und setze Rechte..."
touch $PROJECT_DIR/traefik/acme.json
chmod 600 $PROJECT_DIR/traefik/acme.json

echo "[3/6] Erstelle docker-compose.yml..."
cat > $PROJECT_DIR/docker-compose.yml <<EOF
version: "3.9"

services:
  traefik:
    image: traefik:latest
    container_name: traefik
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./traefik/traefik.yml:/etc/traefik/traefik.yml
      - ./traefik/acme.json:/acme.json
      - /var/run/docker.sock:/var/run/docker.sock
      - ./traefik/dynamic:/etc/traefik/dynamic
    labels:
      - "traefik.enable=true"
    networks:
      - proxy

  crowdsec:
    image: crowdsecurity/crowdsec:latest
    container_name: crowdsec
    volumes:
      - /var/log:/var/log:ro
      - crowdsec-config:/etc/crowdsec/
      - crowdsec-db:/var/lib/crowdsec/data/
    restart: always
    networks:
      - proxy

  crowdsec-bouncer:
    image: fbonalair/traefik-crowdsec-bouncer:latest
    container_name: crowdsec-bouncer
    environment:
      - CROWDSEC_BOUNCER_API_KEY=$CROWDSEC_API_KEY
      - CROWDSEC_AGENT_HOST=crowdsec:8080
    restart: always
    networks:
      - proxy

  stalwart:
    image: stalwartlabs/mail-server:latest
    container_name: stalwart
    volumes:
      - ./stalwart/config.toml:/etc/stalwart-mail/config.toml
    restart: always
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.mail.rule=Host(\`mail.$DOMAIN\`)"
      - "traefik.http.routers.mail.entrypoints=websecure"
      - "traefik.http.routers.mail.tls.certresolver=letsencrypt"
      - "traefik.http.routers.mail.middlewares=crowdsec@file"
    networks:
      - proxy

volumes:
  crowdsec-config:
  crowdsec-db:

networks:
  proxy:
    external: true
EOF

echo "[4/6] Erstelle traefik/traefik.yml..."
cat > $PROJECT_DIR/traefik/traefik.yml <<EOF
entryPoints:
  web:
    address: ":80"
  websecure:
    address: ":443"

providers:
  docker:
    exposedByDefault: false
  file:
    directory: /etc/traefik/dynamic
    watch: true

certificatesResolvers:
  letsencrypt:
    acme:
      email: $EMAIL
      storage: /acme.json
      httpChallenge:
        entryPoint: web
EOF

echo "[5/6] Erstelle Middleware-Konfiguration für CrowdSec..."
cat > $PROJECT_DIR/traefik/dynamic/crowdsec-bouncer.yml <<EOF
http:
  middlewares:
    crowdsec:
      forwardAuth:
        address: http://crowdsec-bouncer:8080/api/v1/forwardAuth
        trustForwardHeader: true
        authResponseHeaders:
          - X-Crowdsec-Decision
EOF

echo "[6/6] Erstelle einfache Stalwart-Konfiguration..."
cat > $PROJECT_DIR/stalwart/config.toml <<EOF
[general]
hostname = "mail.$DOMAIN"

[listener.smtp]
bind = "0.0.0.0:25"
protocol = "smtp"

[listener.imap]
bind = "0.0.0.0:143"
protocol = "imap"

[logging]
level = "info"
EOF

cd $PROJECT_DIR

echo "Prüfe, ob Docker-Netzwerk 'proxy' existiert..."
docker network inspect proxy >/dev/null 2>&1 || docker network create proxy

echo "Starte Docker Compose..."
docker compose up -d

echo "✅ Setup abgeschlossen! Zugriff z. B. unter: https://mail.$DOMAIN"
```