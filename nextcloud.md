# Nextcloud

## Skript
```bash
#!/bin/bash

set -e

DOMAIN="nextcloud.deinedomain.de"
EMAIL="deine@email.de"

mkdir -p traefik certs crowdsec nextcloud/data

echo "🔧 Erstelle Traefik-Konfiguration…"

# Traefik config
cat > traefik/traefik.yml <<EOF
entryPoints:
  web:
    address: ":80"
  websecure:
    address: ":443"

providers:
  docker:
    exposedByDefault: false

certificatesResolvers:
  letsencrypt:
    acme:
      email: $EMAIL
      storage: /letsencrypt/acme.json
      httpChallenge:
        entryPoint: web
EOF

touch traefik/acme.json
chmod 600 traefik/acme.json

# Dynamic config
cat > traefik/dynamic.yml <<EOF
http:
  middlewares:
    crowdsec-bouncer:
      forwardauth:
        address: http://crowdsec-bouncer:8080/api/v1/forward-auth
        trustForwardHeader: true
        authResponseHeaders:
          - X-Forwarded-User

tls:
  certificates:
    - certFile: /certs/selfsigned.crt
      keyFile: /certs/selfsigned.key
EOF

# Self-signed fallback certs (nicht Let's Encrypt)
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout certs/selfsigned.key -out certs/selfsigned.crt \
  -subj "/CN=$DOMAIN"

echo "📦 Erstelle docker-compose.yml…"

cat > docker-compose.yml <<EOF
version: '3.8'

services:

  traefik:
    image: traefik:v2.11
    command:
      - --api.insecure=true
      - --providers.docker
      - --providers.file.filename=/etc/traefik/dynamic.yml
      - --entrypoints.web.address=:80
      - --entrypoints.websecure.address=:443
      - --certificatesresolvers.letsencrypt.acme.httpchallenge.entrypoint=web
      - --certificatesresolvers.letsencrypt.acme.email=$EMAIL
      - --certificatesresolvers.letsencrypt.acme.storage=/letsencrypt/acme.json
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./traefik/traefik.yml:/etc/traefik/traefik.yml
      - ./traefik/dynamic.yml:/etc/traefik/dynamic.yml
      - ./letsencrypt:/letsencrypt
      - ./certs:/certs
      - /var/run/docker.sock:/var/run/docker.sock
    labels:
      - "traefik.enable=true"

  nextcloud:
    image: nextcloud
    restart: unless-stopped
    volumes:
      - ./nextcloud/data:/var/www/html
    environment:
      - MYSQL_PASSWORD=geheim
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
      - MYSQL_HOST=nextcloud-db
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.nextcloud.rule=Host(\`$DOMAIN\`)"
      - "traefik.http.routers.nextcloud.entrypoints=websecure"
      - "traefik.http.routers.nextcloud.tls.certresolver=letsencrypt"
      - "traefik.http.routers.nextcloud.middlewares=crowdsec-bouncer@file"
    depends_on:
      - nextcloud-db

  nextcloud-db:
    image: mariadb
    restart: unless-stopped
    environment:
      - MYSQL_ROOT_PASSWORD=supergeheim
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
      - MYSQL_PASSWORD=geheim
    volumes:
      - nextcloud-db:/var/lib/mysql

  crowdsec:
    image: crowdsecurity/crowdsec
    restart: unless-stopped
    volumes:
      - /var/log:/var/log:ro
      - crowdsec-db:/var/lib/crowdsec/data

  crowdsec-bouncer:
    image: fbonalair/traefik-crowdsec-bouncer
    restart: unless-stopped
    environment:
      - CROWDSEC_BOUNCER_API_KEY=123456789abcdef
      - CROWDSEC_AGENT_HOST=crowdsec:8080
    depends_on:
      - crowdsec

volumes:
  nextcloud-db:
  crowdsec-db:
EOF

echo "🚀 Starte Docker-Container…"
docker compose up -d

echo "✅ Fertig! Rufe https://$DOMAIN im Browser auf."
```
