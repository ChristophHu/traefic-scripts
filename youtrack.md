# YouTrack

## Script
```bash
#!/bin/bash
set -e

# === BENUTZERKONFIGURATION ===
DOMAIN="youtrack.meinedomain.de"
EMAIL="admin@meinedomain.de"
BOUNCER_API_KEY="CHANGE_ME"  # <-- später ersetzen

# === VERZEICHNISERSTELLUNG ===
mkdir -p youtrack-stack/{traefik,crowdsec}
cd youtrack-stack

# === .env erstellen ===
cat <<EOF > .env
DOMAIN=$DOMAIN
EMAIL=$EMAIL
BOUNCER_API_KEY=$BOUNCER_API_KEY
EOF

# === docker-compose.yml ===
cat <<EOF > docker-compose.yml
version: "3.8"

services:

  traefik:
    image: traefik:v2.11
    command:
      - --api.dashboard=true
      - --providers.docker=true
      - --providers.file.directory=/etc/traefik
      - --entrypoints.web.address=:80
      - --entrypoints.websecure.address=:443
      - --certificatesresolvers.leresolver.acme.httpchallenge=true
      - --certificatesresolvers.leresolver.acme.httpchallenge.entrypoint=web
      - --certificatesresolvers.leresolver.acme.email=\${EMAIL}
      - --certificatesresolvers.leresolver.acme.storage=/letsencrypt/acme.json
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./traefik/traefik.yml:/etc/traefik/traefik.yml:ro
      - ./traefik/dynamic_conf.yml:/etc/traefik/dynamic_conf.yml:ro
      - ./traefik/acme.json:/letsencrypt/acme.json
      - /var/run/docker.sock:/var/run/docker.sock
    networks:
      - proxy

  youtrack:
    image: jetbrains/youtrack:latest
    restart: unless-stopped
    volumes:
      - youtrack_data:/opt/youtrack/data
      - youtrack_logs:/opt/youtrack/logs
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.youtrack.rule=Host(\${DOMAIN})"
      - "traefik.http.routers.youtrack.entrypoints=websecure"
      - "traefik.http.routers.youtrack.tls.certresolver=leresolver"
      - "traefik.http.routers.youtrack.middlewares=crowdsec-auth@file"
    networks:
      - proxy

  crowdsec:
    image: crowdsecurity/crowdsec
    container_name: crowdsec
    volumes:
      - /var/log:/var/log:ro
      - /var/lib/crowdsec:/var/lib/crowdsec
      - /etc/crowdsec:/etc/crowdsec
      - /var/run/docker.sock:/var/run/docker.sock
    restart: unless-stopped
    networks:
      - internal

  bouncer-traefik:
    image: fbonalair/traefik-crowdsec-bouncer
    environment:
      - CROWDSEC_BOUNCER_API_KEY=\${BOUNCER_API_KEY}
      - CROWDSEC_AGENT_HOST=crowdsec:8080
    depends_on:
      - crowdsec
    networks:
      - proxy

volumes:
  youtrack_data:
  youtrack_logs:

networks:
  proxy:
    external: false
  internal:
    external: false
EOF

# === traefik.yml ===
cat <<EOF > traefik/traefik.yml
log:
  level: DEBUG

api:
  dashboard: true

entryPoints:
  web:
    address: ":80"
  websecure:
    address: ":443"

providers:
  docker:
    exposedByDefault: false
  file:
    directory: /etc/traefik
EOF

# === dynamic_conf.yml mit CrowdSec Middleware ===
cat <<EOF > traefik/dynamic_conf.yml
http:
  middlewares:
    crowdsec-auth:
      forwardAuth:
        address: http://bouncer-traefik:8080/api/v1/forwardAuth
        trustForwardHeader: true
        authResponseHeaders:
          - X-Crowdsec-Decision
EOF

# === acme.json vorbereiten ===
touch traefik/acme.json
chmod 600 traefik/acme.json

# === Hinweis zur Einrichtung ===
echo
echo "[✓] Setup abgeschlossen."
echo
echo "⚠️ Noch nicht ganz fertig:"
echo "  ➤ 1. API-Key für CrowdSec Bouncer erzeugen:"
echo "       docker exec -it crowdsec cscli bouncers add traefik-bouncer"
echo
echo "  ➤ 2. Den generierten Key in .env eintragen (BOUNCER_API_KEY)!"
echo
echo "  ➤ 3. Starte den Stack:"
echo "       cd youtrack-stack && docker-compose up -d"
echo
echo "🌐 YouTrack ist dann erreichbar unter: https://$DOMAIN"
echo "📌 Achte darauf, dass DNS → Server-IP zeigt & Ports 80/443 offen sind."
```

## Use
```bash
chmod +x setup_youtrack.sh
./setup_youtrack.sh
```
Dann:
```bash
cd youtrack-stack
docker-compose up -d
```
Und CrowdSec Bouncer registrieren:
```bash
docker exec -it crowdsec cscli bouncers add traefik-bouncer
# Kopiere den Key und trage ihn in .env ein (BOUNCER_API_KEY)
```
