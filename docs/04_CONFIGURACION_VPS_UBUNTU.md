# Configuración del VPS Ubuntu 22.04

Esta guía detalla cómo configurar un VPS Ubuntu 22.04 para hospedar el sistema de grabación de Teams.

## 1. Requisitos del VPS

### Especificaciones Mínimas Recomendadas

```
CPU: 4 cores (8 cores recomendado para producción)
RAM: 8 GB mínimo (16 GB+ recomendado)
Storage: 500 GB SSD (depende de cuántas grabaciones almacenarás)
Network: 1 Gbps
OS: Ubuntu 22.04 LTS
```

### Especificaciones por Plan

```
Plan Basic (hasta 5 grabaciones simultáneas):
- CPU: 4 cores
- RAM: 8 GB
- Storage: 500 GB

Plan Pro (hasta 20 grabaciones simultáneas):
- CPU: 8 cores
- RAM: 16 GB
- Storage: 1 TB

Plan Enterprise (grabaciones ilimitadas):
- CPU: 16+ cores
- RAM: 32+ GB
- Storage: 2+ TB
```

## 2. Configuración Inicial del Servidor

### 2.1 Actualizar el Sistema

```bash
# Conectarse al VPS via SSH
ssh root@tu-vps-ip

# Actualizar paquetes
sudo apt update && sudo apt upgrade -y

# Instalar utilidades básicas
sudo apt install -y curl wget git build-essential software-properties-common
```

### 2.2 Crear Usuario para la Aplicación

```bash
# Crear usuario (no usar root en producción)
sudo adduser botadmin

# Agregar a grupo sudo
sudo usermod -aG sudo botadmin

# Cambiar a nuevo usuario
su - botadmin
```

### 2.3 Configurar Firewall

```bash
# Habilitar UFW (Uncomplicated Firewall)
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp    # HTTP
sudo ufw allow 443/tcp   # HTTPS
sudo ufw allow 5432/tcp  # PostgreSQL (si está en el mismo servidor)
sudo ufw enable

# Verificar estado
sudo ufw status
```

## 3. Instalación de Dependencias

### 3.1 Instalar Node.js (v18+)

```bash
# Instalar Node.js v20 (LTS)
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs

# Verificar instalación
node --version  # Debería mostrar v20.x.x
npm --version

# Instalar pnpm globalmente
sudo npm install -g pnpm
pnpm --version
```

### 3.2 Instalar Docker

```bash
# Instalar Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Agregar usuario al grupo docker
sudo usermod -aG docker $USER

# Verificar instalación (necesitarás hacer logout/login primero)
docker --version
docker compose version

# Habilitar Docker para que inicie con el sistema
sudo systemctl enable docker
sudo systemctl start docker
```

### 3.3 Instalar PostgreSQL

```bash
# Instalar PostgreSQL 15
sudo apt install -y postgresql postgresql-contrib

# Verificar que está corriendo
sudo systemctl status postgresql

# Crear base de datos y usuario
sudo -u postgres psql

# Dentro de psql:
CREATE DATABASE teamsrecording;
CREATE USER botadmin WITH ENCRYPTED PASSWORD 'tu-password-seguro';
GRANT ALL PRIVILEGES ON DATABASE teamsrecording TO botadmin;
\q

# Configurar PostgreSQL para aceptar conexiones locales
sudo nano /etc/postgresql/15/main/pg_hba.conf

# Cambiar la línea:
# local   all             all                                     peer
# Por:
# local   all             all                                     md5

# Reiniciar PostgreSQL
sudo systemctl restart postgresql
```

### 3.4 Instalar Dependencias de Puppeteer

Puppeteer necesita varias dependencias del sistema para correr:

```bash
# Instalar dependencias de Chrome/Chromium
sudo apt install -y \
  ca-certificates \
  fonts-liberation \
  libappindicator3-1 \
  libasound2 \
  libatk-bridge2.0-0 \
  libatk1.0-0 \
  libc6 \
  libcairo2 \
  libcups2 \
  libdbus-1-3 \
  libexpat1 \
  libfontconfig1 \
  libgbm1 \
  libgcc1 \
  libglib2.0-0 \
  libgtk-3-0 \
  libnspr4 \
  libnss3 \
  libpango-1.0-0 \
  libpangocairo-1.0-0 \
  libstdc++6 \
  libx11-6 \
  libx11-xcb1 \
  libxcb1 \
  libxcomposite1 \
  libxcursor1 \
  libxdamage1 \
  libxext6 \
  libxfixes3 \
  libxi6 \
  libxrandr2 \
  libxrender1 \
  libxss1 \
  libxtst6 \
  lsb-release \
  wget \
  xdg-utils
```

### 3.5 Instalar NGINX (Reverse Proxy)

```bash
# Instalar NGINX
sudo apt install -y nginx

# Habilitar NGINX
sudo systemctl enable nginx
sudo systemctl start nginx

# Verificar que está corriendo
sudo systemctl status nginx
```

### 3.6 Instalar Certbot (SSL con Let's Encrypt)

```bash
# Instalar Certbot
sudo apt install -y certbot python3-certbot-nginx

# Obtener certificado SSL (reemplaza con tu dominio)
sudo certbot --nginx -d tu-dominio.com -d www.tu-dominio.com

# Verificar renovación automática
sudo certbot renew --dry-run
```

## 4. Configuración del Proyecto

### 4.1 Clonar el Repositorio

```bash
# Crear directorio para la aplicación
sudo mkdir -p /opt/teams-recording-bot
sudo chown botadmin:botadmin /opt/teams-recording-bot

# Clonar repositorio
cd /opt/teams-recording-bot
git clone <tu-repositorio-git> .

# Instalar dependencias
pnpm install
```

### 4.2 Configurar Variables de Entorno

```bash
# Crear archivo .env en el directorio del servidor
cd /opt/teams-recording-bot/src/server
nano .env
```

Contenido del archivo `.env`:

```bash
# Database
DATABASE_URL=postgresql://botadmin:tu-password-seguro@localhost:5432/teamsrecording

# Azure AD
AZURE_CLIENT_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
AZURE_TENANT_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
AZURE_CLIENT_SECRET=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# Azure Communication Services
ACS_CONNECTION_STRING=endpoint=https://...;accesskey=...

# Azure Blob Storage
AZURE_STORAGE_ACCOUNT_NAME=stteamsrecordings
AZURE_STORAGE_ACCESS_KEY=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
AZURE_STORAGE_CONNECTION_STRING=DefaultEndpointsProtocol=https;...

# VPS Storage
VPS_STORAGE_PATH=/var/recordings
VPS_MAX_STORAGE_GB=1000

# Next.js
NEXTAUTH_SECRET=genera-un-secret-aleatorio-aqui
NEXTAUTH_URL=https://tu-dominio.com

# GitHub OAuth (para autenticación del panel)
GITHUB_CLIENT_ID=tu-github-client-id
GITHUB_CLIENT_SECRET=tu-github-client-secret

# Webhook
WEBHOOK_SECRET=genera-otro-secret-aleatorio
NEXT_PUBLIC_APP_URL=https://tu-dominio.com

# Node
NODE_ENV=production
PORT=3000
```

Generar secretos aleatorios:

```bash
# Generar NEXTAUTH_SECRET
openssl rand -base64 32

# Generar WEBHOOK_SECRET
openssl rand -base64 32
```

### 4.3 Configurar Almacenamiento VPS

```bash
# Crear directorio para grabaciones
sudo mkdir -p /var/recordings
sudo chown -R botadmin:botadmin /var/recordings
sudo chmod 755 /var/recordings

# Crear estructura de directorios
cd /var/recordings
mkdir -p users metadata logs

# Configurar rotación de logs
sudo nano /etc/logrotate.d/recordings
```

Contenido de `/etc/logrotate.d/recordings`:

```
/var/recordings/logs/*.log {
    daily
    missingok
    rotate 14
    compress
    delaycompress
    notifempty
    create 0640 botadmin botadmin
    sharedscripts
}
```

## 5. Configurar NGINX como Reverse Proxy

```bash
# Crear configuración de NGINX
sudo nano /etc/nginx/sites-available/teams-recording-bot
```

Contenido del archivo:

```nginx
server {
    listen 80;
    server_name tu-dominio.com www.tu-dominio.com;

    # Redirect HTTP to HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name tu-dominio.com www.tu-dominio.com;

    # SSL certificates (Certbot los configura automáticamente)
    ssl_certificate /etc/letsencrypt/live/tu-dominio.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/tu-dominio.com/privkey.pem;

    # SSL configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    # Client max body size (para subir grabaciones grandes)
    client_max_body_size 500M;

    # Proxy settings
    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }

    # Webhook endpoint - aumentar timeout
    location /api/webhooks {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 300s;
        proxy_connect_timeout 75s;
    }

    # API para descargar grabaciones
    location /api/recordings/download {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_buffering off;
        proxy_read_timeout 300s;
    }

    # Logs
    access_log /var/log/nginx/teams-recording-bot.access.log;
    error_log /var/log/nginx/teams-recording-bot.error.log;
}
```

Habilitar el sitio:

```bash
# Crear symlink
sudo ln -s /etc/nginx/sites-available/teams-recording-bot /etc/nginx/sites-enabled/

# Verificar configuración
sudo nginx -t

# Recargar NGINX
sudo systemctl reload nginx
```

## 6. Configurar Systemd para Auto-Inicio

### 6.1 Crear Servicio para el Servidor

```bash
sudo nano /etc/systemd/system/teams-recording-server.service
```

Contenido:

```ini
[Unit]
Description=Teams Recording Bot Server
After=network.target postgresql.service

[Service]
Type=simple
User=botadmin
WorkingDirectory=/opt/teams-recording-bot/src/server
Environment=NODE_ENV=production
ExecStart=/usr/bin/pnpm run start
Restart=always
RestartSec=10
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=teams-recording-server

[Install]
WantedBy=multi-user.target
```

### 6.2 Habilitar y Iniciar el Servicio

```bash
# Recargar systemd
sudo systemctl daemon-reload

# Habilitar servicio para auto-inicio
sudo systemctl enable teams-recording-server

# Iniciar servicio
sudo systemctl start teams-recording-server

# Verificar estado
sudo systemctl status teams-recording-server

# Ver logs
sudo journalctl -u teams-recording-server -f
```

## 7. Configuración de Monitoreo

### 7.1 Instalar PM2 (Alternativa a systemd)

```bash
# Instalar PM2 globalmente
sudo npm install -g pm2

# Iniciar aplicación con PM2
cd /opt/teams-recording-bot/src/server
pm2 start pnpm --name "teams-recording-server" -- start

# Guardar configuración de PM2
pm2 save

# Configurar PM2 para auto-inicio
pm2 startup

# Ejecutar el comando que PM2 te proporciona
```

### 7.2 Configurar Monitoreo de Disco

```bash
# Script para monitorear espacio en disco
sudo nano /usr/local/bin/check-recording-space.sh
```

Contenido:

```bash
#!/bin/bash

THRESHOLD=90
CURRENT=$(df /var/recordings | grep / | awk '{ print $5}' | sed 's/%//g')

if [ "$CURRENT" -gt "$THRESHOLD" ]; then
    echo "WARNING: Recording storage is ${CURRENT}% full"
    # Aquí podrías enviar una notificación por email o webhook
fi
```

Hacer ejecutable y agregar a cron:

```bash
sudo chmod +x /usr/local/bin/check-recording-space.sh

# Agregar a crontab (cada hora)
sudo crontab -e

# Agregar esta línea:
0 * * * * /usr/local/bin/check-recording-space.sh
```

## 8. Seguridad Adicional

### 8.1 Configurar Fail2Ban

```bash
# Instalar Fail2Ban
sudo apt install -y fail2ban

# Crear configuración local
sudo nano /etc/fail2ban/jail.local
```

Contenido:

```ini
[DEFAULT]
bantime = 3600
findtime = 600
maxretry = 5

[sshd]
enabled = true
port = ssh
logpath = /var/log/auth.log

[nginx-http-auth]
enabled = true
port = http,https
logpath = /var/log/nginx/error.log
```

Reiniciar Fail2Ban:

```bash
sudo systemctl restart fail2ban
sudo systemctl status fail2ban
```

### 8.2 Configurar Backups Automáticos

```bash
# Crear script de backup
sudo nano /usr/local/bin/backup-recordings.sh
```

Contenido:

```bash
#!/bin/bash

BACKUP_DIR=/var/backups/recordings
DATE=$(date +%Y%m%d_%H%M%S)
RECORDINGS_DIR=/var/recordings

# Crear directorio de backups si no existe
mkdir -p $BACKUP_DIR

# Backup de base de datos
PGPASSWORD=tu-password-seguro pg_dump -U botadmin -h localhost teamsrecording > $BACKUP_DIR/db_$DATE.sql

# Comprimir y guardar
gzip $BACKUP_DIR/db_$DATE.sql

# Eliminar backups más antiguos de 30 días
find $BACKUP_DIR -name "*.sql.gz" -mtime +30 -delete

echo "Backup completado: $DATE"
```

Hacer ejecutable y agregar a cron:

```bash
sudo chmod +x /usr/local/bin/backup-recordings.sh

# Agregar a crontab (diariamente a las 2 AM)
sudo crontab -e

# Agregar:
0 2 * * * /usr/local/bin/backup-recordings.sh
```

## 9. Build y Deployment

### 9.1 Build de Producción

```bash
cd /opt/teams-recording-bot

# Build del servidor
cd src/server
pnpm run build

# Ejecutar migraciones de base de datos
pnpm run db:push
```

### 9.2 Script de Deployment Automatizado

Crear `deploy.sh` en la raíz del proyecto:

```bash
#!/bin/bash

set -e

echo "🚀 Starting deployment..."

# Pull latest changes
git pull origin main

# Install dependencies
pnpm install

# Build server
cd src/server
pnpm run build

# Run database migrations
pnpm run db:push

# Restart service
sudo systemctl restart teams-recording-server

echo "✅ Deployment completed successfully!"
```

## 10. Verificación Final

```bash
# Verificar que todos los servicios están corriendo
sudo systemctl status nginx
sudo systemctl status postgresql
sudo systemctl status teams-recording-server

# Verificar logs
sudo journalctl -u teams-recording-server -n 50

# Verificar conectividad
curl https://tu-dominio.com

# Verificar webhook endpoint
curl https://tu-dominio.com/api/webhooks/graph
```

## 11. Troubleshooting Común

### Problema: Puppeteer no puede iniciar Chrome

```bash
# Instalar dependencias faltantes
sudo apt install -y libgbm1 libnss3 libxss1
```

### Problema: Sin espacio en disco

```bash
# Verificar uso de disco
df -h

# Encontrar archivos grandes
du -h /var/recordings | sort -rh | head -20

# Limpiar grabaciones antiguas (cuidado!)
find /var/recordings -name "*.webm" -mtime +90 -delete
```

### Problema: PostgreSQL no acepta conexiones

```bash
# Verificar configuración
sudo nano /etc/postgresql/15/main/postgresql.conf

# Asegurar que escucha en localhost
listen_addresses = 'localhost'

# Reiniciar
sudo systemctl restart postgresql
```

## Resumen de Configuración

Al finalizar esta guía, deberías tener:

✅ VPS Ubuntu 22.04 configurado y asegurado
✅ Node.js, Docker, PostgreSQL instalados
✅ NGINX como reverse proxy con SSL
✅ Aplicación corriendo como servicio systemd
✅ Almacenamiento local configurado en `/var/recordings`
✅ Backups automáticos configurados
✅ Monitoreo de recursos habilitado
✅ Firewall y Fail2Ban configurados

### Próximos pasos

- Configurar monitoreo avanzado (Prometheus, Grafana)
- Implementar CDN para servir grabaciones
- Configurar replicación de base de datos
- Implementar queue system (Redis) para auto-join jobs
