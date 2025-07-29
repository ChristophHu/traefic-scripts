# OpenCloud

## Script
```bash
#!/bin/bash

set -e

# === KONFIGURATION ===
DOMAIN="cloud.example.com"
EMAIL="admin@example.com"
CROWDSEC_BOUNCER_KEY="CHANGE_THIS"

# === VERZEICHNISERSTELLUNG ===
echo "[+] Verzeichnisstruktur wird erstellt..."
mkdir -p opencloud-stack/{traefik,crowdsec}
cd opencloud-stack

# === .env DATEI ===
cat <<EOF > .env
DOMAIN=$DOMAIN
EMAIL=$EMAIL
EOF

# === docker-compose.yml ===
cat <<EOF > docker-compose.yml
version: '3.8'

services:

  traefik:
    image: traefik:v2.11
    command:
      - --api.dashboard=true
      - --providers.docker=true
      - --providers.file.filename=/etc/traefik/dynamic_conf.yml
      - --entrypoints.web.address=:80
      - --entrypoints.websecure.address=:443
      - --certificatesresolvers.leresolver.acme.httpchallenge=true
      - --certificatesresolvers.leresolver.acme.httpchallenge.entrypoint=web
      - --certificatesresolvers.leresolver.acme.email=\${EMAIL}
      - --certificatesresolvers.leresolver.acme.storage=/letsencrypt/acme.json
    ports:
      - 80:80
      - 443:443
    volumes:
      - ./traefik/traefik.yml:/etc/traefik/traefik.yml:ro
      - ./traefik/dynamic_conf.yml:/etc/traefik/dynamic_conf.yml:ro
      - ./traefik/acme.json:/letsencrypt/acme.json
      - /var/run/docker.sock:/var/run/docker.sock
    labels:
      - "traefik.enable=true"
    networks:
      - proxy

  nextcloud:
    image: nextcloud
    environment:
      - MYSQL_PASSWORD=nextcloud
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
      - MYSQL_HOST=db
    volumes:
      - nextcloud:/var/www/html
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.nextcloud.rule=Host(\${DOMAIN})"
      - "traefik.http.routers.nextcloud.entrypoints=websecure"
      - "traefik.http.routers.nextcloud.tls.certresolver=leresolver"
      - "traefik.http.routers.nextcloud.middlewares=crowdsec-bouncer@file"
    depends_on:
      - db
    networks:
      - proxy
      - internal

  db:
    image: mariadb
    restart: always
    environment:
      - MYSQL_ROOT_PASSWORD=secret
      - MYSQL_PASSWORD=nextcloud
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
    volumes:
      - db:/var/lib/mysql
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
    restart: always
    networks:
      - internal

  bouncer-traefik:
    image: fbonalair/traefik-crowdsec-bouncer
    environment:
      - CROWDSEC_BOUNCER_API_KEY=${CROWDSEC_BOUNCER_KEY}
      - CROWDSEC_AGENT_HOST=crowdsec:8080
    networks:
      - proxy
    depends_on:
      - crowdsec

volumes:
  nextcloud:
  db:

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
    filename: "/etc/traefik/dynamic_conf.yml"
EOF

# === dynamic_conf.yml ===
cat <<EOF > traefik/dynamic_conf.yml
http:
  middlewares:
    crowdsec-bouncer:
      forwardauth:
        address: http://bouncer-traefik:8080/api/v1/forwardAuth
        trustForwardHeader: true
        authResponseHeaders:
          - X-Crowdsec-Decision
EOF

# === acme.json vorbereiten ===
touch traefik/acme.json
chmod 600 traefik/acme.json

# === Hinweis für Bouncer-API-Key ===
echo
echo "[!] Vergiss nicht, deinen echten CrowdSec Bouncer-API-Key einzutragen!"
echo "    Beispiel: docker exec -it crowdsec cscli bouncers add traefik-bouncer"
echo "    Und dann in setup.sh bzw. docker-compose.yml eintragen."
echo

# === Startanweisung ===
echo "[✓] Setup fertig. Starte mit:"
echo "    cd opencloud-stack && docker-compose up -d"
```
