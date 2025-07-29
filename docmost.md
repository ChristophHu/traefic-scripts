# Docmost

## Script
```bash
#!/bin/bash
set -e

# === KONFIGURATION ===
DOMAIN="dockmost.deinedomain.de"
EMAIL="admin@deinedomain.de"
APP_SECRET=$(openssl rand -hex 32)
BOUNCER_API_KEY="CHANGE_ME"  # Bitte später durch echten API-Key ersetzen!

# === VERZEICHNISERSTELLUNG ===
mkdir -p dockmost-stack/{traefik,crowdsec}
cd dockmost-stack

# === .env DATEI ===
cat <<EOF > .env
DOMAIN=$DOMAIN
EMAIL=$EMAIL
APP_SECRET=$APP_SECRET
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

  dockmost:
    image: docmost/docmost:latest
    depends_on:
      - db
      - redis
    environment:
      - APP_URL=https://\${DOMAIN}
      - APP_SECRET=\${APP_SECRET}
      - DATABASE_URL=postgresql://docmost:docmostpw@db:5432/docmost?schema=public
      - REDIS_URL=redis://redis:6379
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.dockmost.rule=Host(\${DOMAIN})"
      - "traefik.http.routers.dockmost.entrypoints=websecure"
      - "traefik.http.routers.dockmost.tls.certresolver=leresolver"
      - "traefik.http.routers.dockmost.middlewares=crowdsec-auth@file"
    networks:
      - proxy
      - internal
    volumes:
      - dockmost_data:/app/data/storage

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: docmost
      POSTGRES_USER: docmost
      POSTGRES_PASSWORD: docmostpw
    volumes:
      - db_data:/var/lib/postgresql/data
    networks:
      - internal

  redis:
    image: redis:7.2-alpine
    volumes:
      - redis_data:/data
    networks:
      - internal

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
  dockmost_data:
  db_data:
  redis_data:

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

# === dynamic_conf.yml ===
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

# === Abschluss-Hinweis ===
echo
echo "[✓] Setup abgeschlossen!"
echo
echo "Starte nun mit:"
echo "   cd dockmost-stack && docker-compose up -d"
echo
echo "➡️  Docmost unter: https://$DOMAIN"
echo
echo "⚠️ Wichtig:"
echo "- Setze einen echten API-Key für CROWDSEC_BOUNCER_API_KEY in der .env Datei."
echo "- Führe aus:"
echo "     docker exec -it crowdsec cscli bouncers add traefik-bouncer"
echo "     # → und kopiere den Key in .env / docker-compose.yml"
echo "- DNS muss auf deinen Server zeigen. Ports 80/443 müssen offen sein."

```
