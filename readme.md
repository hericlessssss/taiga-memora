Guia Completo de Configuração e Manutenção do Taiga via Docker + NGINX

Este documento cobre toda a configuração e manutenção de um ambiente Taiga com Docker Compose, usando NGINX como proxy reverso e com suporte a arquivos protegidos via X-Accel-Redirect. Também trata de soluções de problemas comuns como traduções quebradas e imagens 404.

🌐 Visão Geral da Arquitetura

taiga-front: Interface web do usuário

taiga-back: API e lógica de negócio

taiga-async: Tarefas assíncronas

taiga-events: Websockets (eventos em tempo real)

taiga-protected: Gerencia o acesso a arquivos protegidos

NGINX externo: Faz proxy reverso com SSL e expõe o domínio público

🚫 Arquivos Protegidos e X-Accel-Redirect

Para que imagens protegidas (como avatars de usuários) funcionem corretamente, é necessário que o NGINX externo respeite os headers X-Accel-Redirect vindos do taiga-protected.

NGINX externo (/etc/nginx/conf.d/taiga.conf):

location /media/ {
    rewrite ^/media/(.*)$ /_protected/$1 break;
}

location /_protected/ {
    internal;
    alias /taiga/media/;
    add_header Content-Disposition "attachment";
}

NGINX dentro do container taiga-gateway já deve estar corretamente configurado com:

location /media/ {
    proxy_pass http://taiga-protected:8003/;
    proxy_set_header Host $http_host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Scheme $scheme;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_redirect off;
}

location /_protected/ {
    internal;
    alias /taiga/media/;
    add_header Content-disposition "attachment";
}

No docker-compose.yaml:

taiga-protected:
  ports:
    - "8003:8003"

🌎 SSL + Proxy Reverso

O NGINX externo também deve estar com SSL configurado corretamente:

server {
    listen 443 ssl;
    server_name taiga.memora.com.br;

    ssl_certificate /etc/letsencrypt/live/taiga.memora.com.br/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/taiga.memora.com.br/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location / {
        proxy_pass http://192.168.6.30:9000;
        ...
    }
    # demais locations: /api/, /admin/, /events, /media/
}

🔍 Resolvendo Problema com Tradução

Se a interface do frontend do Taiga estiver mostrando "códigos" ao invés de textos amigáveis (ex: sidebar.dashboard), pode ser:

Linguagem inválida no conf.json

Arquivo de tradução ausente no diretório do frontend

Solução 1: Corrigir conf.json

{
  "defaultLanguage": "pt"
}

❌ Evite pt-br se não houver suporte no build do frontend

Solução 2: Recompilar o Frontend com a Tradução

Caso você precise gerar arquivos .json com traduções:

cd taiga-front-dist
npm install  # (apenas uma vez)
./generate.sh

Após isso:

docker cp dist/i18n/pt.json taiga-docker-taiga-front-1:/usr/share/nginx/html/i18n/pt.json

🔢 Debug: Verificando arquivos protegidos

Script útil para confirmar se os arquivos estão presentes no volume:

#!/bin/bash
CONTAINER=taiga-docker-taiga-protected-1

FILES=(
  "user/d/7/2/6/abc123/lv17.png.80x80_q85_crop.png"
  "user/d/7/2/6/abc123/lv17.png.300x300_q85_crop.png"
)

echo "Verificando arquivos no container $CONTAINER..."
for file in "${FILES[@]}"; do
    echo -n "Verificando $file... "
    docker exec "$CONTAINER" test -f "/taiga/media/$file" && echo "✅ ENCONTRADO" || echo "❌ NÃO ENCONTRADO"
done

🎓 Considerações Finais

Sempre use os nomes corretos de idiomas (pt, en, etc.) no conf.json

Certifique-se de que os headers X-Accel-Redirect estejam funcionando entre taiga-protected e o nginx externo

Para qualquer erro 404 ou 500 em arquivos de imagem, verifique se:

A imagem existe dentro de /taiga/media no container do protected

O nginx externo está redirecionando corretamente para /media/

arquivo final nginx:

server {
    listen 443 ssl;
    server_name taiga.memora.com.br;
    root /usr/share/nginx/html;
    index index.html;

    ssl_certificate /etc/letsencrypt/live/taiga.memora.com.br/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/taiga.memora.com.br/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    client_max_body_size 100M;
    charset utf-8;

    # Frontend
    location / {
        proxy_pass http://192.168.6.30:9000;  # Frontend mapeado para o IP e porta do container
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # API
    location /api/ {
        proxy_pass http://192.168.6.30:8000/api/;  # Backend mapeado para o IP e porta do container
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # WebSocket Events
    location /events {
        proxy_pass http://192.168.6.30:8888/events;  # Websocket mapeado para o IP e porta do container
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_connect_timeout 7d;
        proxy_send_timeout 7d;
        proxy_read_timeout 7d;
    }

    # Static Files
    location /static/ {
        proxy_pass http://192.168.6.30:9000/static/;  # Servindo arquivos estáticos do frontend
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # Admin
    location /admin/ {
        proxy_pass http://192.168.6.30:8000/admin/;  # Área administrativa
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }


    # Protected media files (X-Accel-Redirect)
    location /_protected/ {
        internal;
        alias /taiga/media/;
        add_header Content-Disposition "attachment";
    }

    # Public media access point (maps to protected)
    location /media/ {
        proxy_pass http://192.168.6.30:9000/media/;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Scheme $scheme;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_redirect off;
    }

    # Media Exports
    location /media/exports/ {
        proxy_pass http://192.168.6.30:8003/;
        add_header Content-disposition "attachment";
    }

}

server {
    listen 80;
    server_name taiga.memora.com.br;
    return 301 https://$host$request_uri;
}


---

arquivo final docker-compose.yaml

x-environment:
  &default-back-environment
  # These environment variables will be used by taiga-back and taiga-async.
  # Database settings
  POSTGRES_DB: "taiga"
  POSTGRES_USER: "${POSTGRES_USER}"
  POSTGRES_PASSWORD: "${POSTGRES_PASSWORD}"
  POSTGRES_HOST: "taiga-db"
  # Taiga settings
  TAIGA_SECRET_KEY: "${SECRET_KEY}"
  TAIGA_SITES_SCHEME: "${TAIGA_SCHEME}"
  TAIGA_SITES_DOMAIN: "${TAIGA_DOMAIN}"
  TAIGA_SUBPATH: "${SUBPATH}"
  # Email settings.
  EMAIL_BACKEND: "django.core.mail.backends.${EMAIL_BACKEND}.EmailBackend"
  DEFAULT_FROM_EMAIL: "${EMAIL_DEFAULT_FROM}"
  EMAIL_USE_TLS: "${EMAIL_USE_TLS}"
  EMAIL_USE_SSL: "${EMAIL_USE_SSL}"
  EMAIL_HOST: "${EMAIL_HOST}"
  EMAIL_PORT: "${EMAIL_PORT}"
  EMAIL_HOST_USER: "${EMAIL_HOST_USER}"
  EMAIL_HOST_PASSWORD: "${EMAIL_HOST_PASSWORD}"
  # Rabbitmq settings
  RABBITMQ_USER: "${RABBITMQ_USER}"
  RABBITMQ_PASS: "${RABBITMQ_PASS}"
  # Telemetry settings
  ENABLE_TELEMETRY: "${ENABLE_TELEMETRY}"

x-volumes:
  &default-back-volumes
  - taiga-static-data:/taiga-back/static
  - taiga-media-data:/taiga-back/media

services:
  taiga-db:
    image: postgres:12.3
    environment:
      POSTGRES_DB: "taiga"
      POSTGRES_USER: "${POSTGRES_USER}"
      POSTGRES_PASSWORD: "${POSTGRES_PASSWORD}"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 2s
      timeout: 15s
      retries: 5
      start_period: 3s
    volumes:
      - taiga-db-data:/var/lib/postgresql/data
    networks:
      - taiga

  taiga-back:
    image: taigaio/taiga-back:latest
    environment: *default-back-environment
    volumes: *default-back-volumes
    ports:
      - "8000:8000"
    networks:
      - taiga
    depends_on:
      taiga-db:
        condition: service_healthy
      taiga-events-rabbitmq:
        condition: service_started
      taiga-async-rabbitmq:
        condition: service_started

  taiga-async:
    image: taigaio/taiga-back:latest
    entrypoint: ["/taiga-back/docker/async_entrypoint.sh"]
    environment: *default-back-environment
    volumes: *default-back-volumes
    networks:
      - taiga
    depends_on:
      taiga-db:
        condition: service_healthy
      taiga-events-rabbitmq:
        condition: service_started
      taiga-async-rabbitmq:
        condition: service_started

  taiga-async-rabbitmq:
    image: rabbitmq:3.8-management-alpine
    environment:
      RABBITMQ_ERLANG_COOKIE: "${RABBITMQ_ERLANG_COOKIE}"
      RABBITMQ_DEFAULT_USER: "${RABBITMQ_USER}"
      RABBITMQ_DEFAULT_PASS: "${RABBITMQ_PASS}"
      RABBITMQ_DEFAULT_VHOST: "${RABBITMQ_VHOST}"
    hostname: "taiga-async-rabbitmq"
    volumes:
      - taiga-async-rabbitmq-data:/var/lib/rabbitmq
    networks:
      - taiga

  taiga-front:
    image: taigaio/taiga-front:latest
    environment:
      TAIGA_URL: "http://${TAIGA_DOMAIN}"
      TAIGA_WEBSOCKETS_URL: "ws://${TAIGA_DOMAIN}/events"
      TAIGA_SUBPATH: "${SUBPATH}"
    volumes:
      - ./conf.json:/usr/share/nginx/html/conf.json
    networks:
      - taiga

  taiga-events:
    image: taigaio/taiga-events:latest
    environment:
      RABBITMQ_USER: "${RABBITMQ_USER}"
      RABBITMQ_PASS: "${RABBITMQ_PASS}"
      TAIGA_SECRET_KEY: "${SECRET_KEY}"
    ports:
      - "8888:8888"
    networks:
      - taiga
    depends_on:
      taiga-events-rabbitmq:
        condition: service_started

  taiga-events-rabbitmq:
    image: rabbitmq:3.8-management-alpine
    environment:
      RABBITMQ_ERLANG_COOKIE: "${RABBITMQ_ERLANG_COOKIE}"
      RABBITMQ_DEFAULT_USER: "${RABBITMQ_USER}"
      RABBITMQ_DEFAULT_PASS: "${RABBITMQ_PASS}"
      RABBITMQ_DEFAULT_VHOST: "${RABBITMQ_VHOST}"
    hostname: "taiga-events-rabbitmq"
    volumes:
      - taiga-events-rabbitmq-data:/var/lib/rabbitmq
    networks:
      - taiga

  taiga-protected:
    image: taigaio/taiga-protected:latest
    environment:
      MAX_AGE: "${ATTACHMENTS_MAX_AGE}"
      SECRET_KEY: "${SECRET_KEY}"
    ports:
      - "8003:8003"
    volumes:
      - taiga-media-data:/taiga/media
    networks:
      - taiga

  taiga-gateway:
    image: nginx:1.19-alpine
    ports:
      - "443:443"
      - "9000:80"
    volumes:
      - ./taiga-gateway/taiga.conf:/etc/nginx/conf.d/default.conf
      - taiga-static-data:/taiga/static
      - taiga-media-data:/taiga/media
    networks:
      - taiga
    depends_on:
      - taiga-front
      - taiga-back
      - taiga-events

volumes:
  taiga-static-data:
  taiga-media-data:
  taiga-db-data:
  taiga-async-rabbitmq-data:
  taiga-events-rabbitmq-data:

networks:
  taiga:
