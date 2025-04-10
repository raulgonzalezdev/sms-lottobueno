# Guía de configuración de subdominio para httpSMS

Este documento explica cómo configurar el subdominio `sms.applottobueno.com` para la aplicación httpSMS, manteniendo la aplicación existente que ya se está ejecutando en `applottobueno.com`.

## Tabla de contenidos

1. [Crear el subdominio en Hostinger](#1-crear-el-subdominio-en-hostinger)
2. [Configurar Nginx para el subdominio](#2-configurar-nginx-para-el-subdominio)
3. [Obtener certificados SSL](#3-obtener-certificados-ssl-para-el-subdominio)
4. [Configurar Docker Compose para httpSMS](#4-configurar-docker-compose-para-httpsms)
5. [Crear archivos Dockerfile](#5-crear-archivos-dockerfile-para-web-y-api)
6. [Ajustar variables de entorno](#6-ajustar-las-variables-de-entorno)
7. [Iniciar servicios](#7-iniciar-los-servicios)
8. [Crear usuario del sistema](#8-crear-el-usuario-del-sistema)
9. [Consideraciones importantes](#consideraciones-importantes)

## 1. Crear el subdominio en Hostinger

1. Accede al panel de control de Hostinger
2. Ve a la sección de dominios y selecciona `applottobueno.com`
3. Busca la opción "DNS" o "Administrar registros DNS"
4. Añade un nuevo registro A:
   - Nombre: `sms`
   - Valor: [IP de tu máquina virtual en Google Cloud]
   - TTL: 3600 (o el valor recomendado)

## 2. Configurar Nginx para el subdominio

Crea un nuevo archivo de configuración para el subdominio:

```bash
sudo nano /etc/nginx/sites-available/sms.applottobueno.com
```

Con el siguiente contenido:

```nginx
# Configuración para HTTP (Puerto 80)
server {
    listen 80;
    server_name sms.applottobueno.com;

    # Servir desafíos ACME de Let's Encrypt
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
        allow all;
    }

    # Redirigir todo lo demás a HTTPS
    location / {
        return 301 https://$host$request_uri;
    }
}

# Configuración SSL (Puerto 443)
server {
    listen 443 ssl;
    server_name sms.applottobueno.com;

    # Certificados SSL
    ssl_certificate /etc/nginx/certs/sms.fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/sms.privkey.pem;

    # Proxy para el frontend de httpSMS
    location / {
        proxy_pass http://localhost:3001;  # Usaremos el puerto 3001 para el frontend de httpSMS
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Proxy para la API de httpSMS
    location /api/ {
        proxy_pass http://localhost:8001;  # Usaremos el puerto 8001 para la API de httpSMS
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Activa la configuración:

```bash
sudo ln -s /etc/nginx/sites-available/sms.applottobueno.com /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

## 3. Obtener certificados SSL para el subdominio

```bash
sudo certbot certonly --webroot -w /var/www/certbot -d sms.applottobueno.com
```

Después de obtener los certificados, cópialos a la ubicación especificada:

```bash
sudo cp /etc/letsencrypt/live/sms.applottobueno.com/fullchain.pem /etc/nginx/certs/sms.fullchain.pem
sudo cp /etc/letsencrypt/live/sms.applottobueno.com/privkey.pem /etc/nginx/certs/sms.privkey.pem
```

## 4. Configurar Docker Compose para httpSMS

Crea un directorio separado para httpSMS:

```bash
mkdir -p /home/user/httpsms
cd /home/user/httpsms
```

Crea un archivo `docker-compose.yml` para httpSMS:

```yaml
version: '3'

services:
  postgres:
    image: postgres:latest
    environment:
      POSTGRES_USER: dbusername
      POSTGRES_PASSWORD: dbpassword
      POSTGRES_DB: httpsms
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5433:5432"  # Cambiamos el puerto para evitar conflictos
    networks:
      - httpsms-network

  redis:
    image: redis:latest
    ports:
      - "6380:6379"  # Cambiamos el puerto para evitar conflictos
    volumes:
      - redis_data:/data
    networks:
      - httpsms-network

  api:
    build: 
      context: ./api
      dockerfile: Dockerfile
    environment:
      - DATABASE_URL=postgresql://dbusername:dbpassword@postgres:5432/httpsms
      - REDIS_URL=redis://redis:6379
      - APP_PORT=8000
      - GCP_PROJECT_ID=tu-proyecto-firebase
      - FIREBASE_CREDENTIALS=${FIREBASE_CREDENTIALS}
      - SMTP_USERNAME=${SMTP_USERNAME}
      - SMTP_PASSWORD=${SMTP_PASSWORD}
      - SMTP_HOST=${SMTP_HOST}
      - SMTP_PORT=${SMTP_PORT}
      - APP_URL=https://sms.applottobueno.com
    ports:
      - "8001:8000"  # Mapeamos al puerto 8001 externamente
    depends_on:
      - postgres
      - redis
    networks:
      - httpsms-network

  web:
    build:
      context: ./web
      dockerfile: Dockerfile
    environment:
      - API_BASE_URL=https://sms.applottobueno.com/api
      - FIREBASE_API_KEY=${FIREBASE_API_KEY}
      - FIREBASE_AUTH_DOMAIN=${FIREBASE_AUTH_DOMAIN}
      - FIREBASE_PROJECT_ID=${FIREBASE_PROJECT_ID}
      - FIREBASE_STORAGE_BUCKET=${FIREBASE_STORAGE_BUCKET}
      - FIREBASE_MESSAGING_SENDER_ID=${FIREBASE_MESSAGING_SENDER_ID}
      - FIREBASE_APP_ID=${FIREBASE_APP_ID}
      - FIREBASE_MEASUREMENT_ID=${FIREBASE_MEASUREMENT_ID}
    ports:
      - "3001:3000"  # Mapeamos al puerto 3001 externamente
    networks:
      - httpsms-network

volumes:
  postgres_data:
  redis_data:

networks:
  httpsms-network:
```

## 5. Crear archivos Dockerfile para web y api

Para el directorio api, crea un Dockerfile:

```dockerfile
FROM golang:1.22 as builder

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o api-server main.go

FROM alpine:latest

RUN apk --no-cache add ca-certificates

WORKDIR /root/

COPY --from=builder /app/api-server .
COPY .env.docker .env

EXPOSE 8000

CMD ["./api-server"]
```

Para el directorio web, crea un Dockerfile:

```dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .
RUN npm run build

EXPOSE 3000

CMD ["npm", "start"]
```

## 6. Ajustar las variables de entorno

Crea archivos .env para las variables de entorno necesarias:

```bash
touch .env
```

Añade las variables necesarias para Firebase y SMTP:

```
FIREBASE_API_KEY=...
FIREBASE_AUTH_DOMAIN=...
FIREBASE_PROJECT_ID=...
FIREBASE_STORAGE_BUCKET=...
FIREBASE_MESSAGING_SENDER_ID=...
FIREBASE_APP_ID=...
FIREBASE_MEASUREMENT_ID=...
FIREBASE_CREDENTIALS='{...}'
SMTP_USERNAME=...
SMTP_PASSWORD=...
SMTP_HOST=...
SMTP_PORT=...
```

## 7. Iniciar los servicios

```bash
docker-compose up -d
```

## 8. Crear el usuario del sistema

Conéctate a la base de datos y crea el usuario del sistema:

```bash
docker-compose exec postgres psql -U dbusername -d httpsms
```

```sql
INSERT INTO users (id, email, api_key, subscription_name)
VALUES ('system-user-id', 'system@example.com', 'system-user-api-key', 'free');
```

## Consideraciones importantes

1. **Puertos**: Estamos usando puertos diferentes (8001, 3001, 5433, 6380) para evitar conflictos con tu aplicación existente.

2. **Redes Docker**: Ambas aplicaciones están en redes Docker distintas para mantenerlas aisladas.

3. **Recursos**: Asegúrate de que tu VM tenga suficientes recursos para ejecutar ambas aplicaciones.

4. **Certificados SSL**: Necesitarás obtener y renovar certificados SSL para el subdominio.

5. **Respaldos**: Configura respaldos para la base de datos de httpSMS.

6. **Variables de entorno**: Asegúrate de que todas las variables de entorno en el archivo `.env` coincidan con tu configuración de Firebase y SMTP.

7. **Ajustes de API**: Si hay problemas de comunicación entre el frontend y la API, verifica que la URL base de la API esté configurada correctamente en el frontend.

Con esta configuración, ambas aplicaciones funcionarán de forma independiente pero accesibles a través de diferentes dominios en la misma máquina. 