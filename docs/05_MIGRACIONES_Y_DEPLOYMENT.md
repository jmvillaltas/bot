# Migraciones de Base de Datos y Deployment

Este documento contiene scripts para migraciones de base de datos y deployment automatizado.

## 1. Migraciones de Base de Datos

### 1.1 Script de Migración SQL

**Archivo**: `src/server/migrations/001_teams_recording_tables.sql` (NUEVO)

```sql
-- ========================================
-- MIGRATION: Teams Recording Bot Tables
-- Versión: 001
-- Fecha: 2024-12-XX
-- ========================================

-- Tabla de planes
CREATE TABLE IF NOT EXISTS plans (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    max_simultaneous_recordings INTEGER NOT NULL,
    price_monthly INTEGER NOT NULL, -- en centavos
    storage_type VARCHAR(50) NOT NULL DEFAULT 'azure',
    features JSONB NOT NULL DEFAULT '[]'::jsonb,
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Tabla de planes de usuario
CREATE TABLE IF NOT EXISTS user_plans (
    id SERIAL PRIMARY KEY,
    user_id UUID NOT NULL REFERENCES "user"(id) ON DELETE CASCADE,
    plan_id INTEGER NOT NULL REFERENCES plans(id),
    start_date TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    end_date TIMESTAMP,
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Tabla de usuarios para grabar
CREATE TABLE IF NOT EXISTS recording_targets (
    id SERIAL PRIMARY KEY,
    tenant_admin_id UUID NOT NULL REFERENCES "user"(id) ON DELETE CASCADE,
    target_email VARCHAR(255) NOT NULL,
    target_user_id VARCHAR(255),
    display_name VARCHAR(255),
    is_active BOOLEAN NOT NULL DEFAULT true,
    recording_enabled BOOLEAN NOT NULL DEFAULT true,
    has_accepted_disclaimer BOOLEAN NOT NULL DEFAULT false,
    disclaimer_accepted_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(tenant_admin_id, target_email)
);

-- Tabla de criterios de exclusión
CREATE TABLE IF NOT EXISTS exclusion_criteria (
    id SERIAL PRIMARY KEY,
    recording_target_id INTEGER NOT NULL REFERENCES recording_targets(id) ON DELETE CASCADE,
    criteria_type VARCHAR(100) NOT NULL,
    criteria_value JSONB NOT NULL,
    description TEXT,
    is_active BOOLEAN NOT NULL DEFAULT true,
    priority INTEGER NOT NULL DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Índice para búsqueda rápida de criterios
CREATE INDEX IF NOT EXISTS idx_exclusion_criteria_target
ON exclusion_criteria(recording_target_id, is_active);

-- Tabla de suscripciones de Graph API
CREATE TABLE IF NOT EXISTS graph_subscriptions (
    id SERIAL PRIMARY KEY,
    subscription_id VARCHAR(255) NOT NULL UNIQUE,
    user_id UUID NOT NULL REFERENCES "user"(id) ON DELETE CASCADE,
    target_email VARCHAR(255) NOT NULL,
    resource VARCHAR(500) NOT NULL,
    change_type VARCHAR(100) NOT NULL,
    expiration_date_time TIMESTAMP NOT NULL,
    client_state VARCHAR(255) NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Índice para verificar suscripciones expiradas
CREATE INDEX IF NOT EXISTS idx_graph_subscriptions_expiration
ON graph_subscriptions(expiration_date_time, is_active);

-- Tabla de logs de auto-join
CREATE TABLE IF NOT EXISTS auto_join_logs (
    id SERIAL PRIMARY KEY,
    recording_target_id INTEGER REFERENCES recording_targets(id) ON DELETE SET NULL,
    meeting_id VARCHAR(255) NOT NULL,
    meeting_subject VARCHAR(500),
    meeting_start_time TIMESTAMP NOT NULL,
    decision VARCHAR(50) NOT NULL,
    reason TEXT,
    exclusion_criteria_matched JSONB,
    bot_id INTEGER REFERENCES bots(id) ON DELETE SET NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Índice para análisis de decisiones
CREATE INDEX IF NOT EXISTS idx_auto_join_logs_decision
ON auto_join_logs(decision, created_at);

-- Tabla de configuración de almacenamiento
CREATE TABLE IF NOT EXISTS storage_config (
    id SERIAL PRIMARY KEY,
    user_id UUID NOT NULL UNIQUE REFERENCES "user"(id) ON DELETE CASCADE,
    storage_type VARCHAR(50) NOT NULL DEFAULT 'azure',
    azure_config JSONB,
    vps_config JSONB,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Tabla de metadata de grabaciones
CREATE TABLE IF NOT EXISTS recording_metadata (
    id SERIAL PRIMARY KEY,
    bot_id INTEGER NOT NULL REFERENCES bots(id) ON DELETE CASCADE,
    recording_target_id INTEGER REFERENCES recording_targets(id) ON DELETE SET NULL,
    meeting_id VARCHAR(255),
    meeting_subject VARCHAR(500),
    organizer VARCHAR(255),
    participants JSONB,
    start_time TIMESTAMP NOT NULL,
    end_time TIMESTAMP NOT NULL,
    duration_minutes INTEGER,
    storage_location VARCHAR(1000),
    storage_type VARCHAR(50),
    file_size INTEGER,
    transcription_available BOOLEAN DEFAULT false,
    transcription_path VARCHAR(1000),
    tags JSONB,
    is_deleted BOOLEAN DEFAULT false,
    deleted_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Índices para búsquedas comunes
CREATE INDEX IF NOT EXISTS idx_recording_metadata_bot
ON recording_metadata(bot_id);

CREATE INDEX IF NOT EXISTS idx_recording_metadata_target
ON recording_metadata(recording_target_id);

CREATE INDEX IF NOT EXISTS idx_recording_metadata_dates
ON recording_metadata(start_time, end_time);

-- Insertar planes por defecto
INSERT INTO plans (name, description, max_simultaneous_recordings, price_monthly, storage_type, features) VALUES
('Basic', 'Plan básico para equipos pequeños', 5, 4900, 'azure', '["Auto-join a reuniones", "Almacenamiento en Azure", "Criterios de exclusión básicos"]'::jsonb),
('Pro', 'Plan profesional para equipos medianos', 20, 14900, 'both', '["Auto-join a reuniones", "Almacenamiento Azure + VPS", "Criterios de exclusión avanzados", "Transcripción automática", "Soporte prioritario"]'::jsonb),
('Enterprise', 'Plan empresarial para organizaciones grandes', 0, 49900, 'both', '["Grabaciones simultáneas ilimitadas", "Almacenamiento híbrido", "Criterios de exclusión personalizados", "Transcripción y análisis", "API dedicada", "Soporte 24/7", "SLA garantizado"]'::jsonb)
ON CONFLICT DO NOTHING;

-- Trigger para actualizar updated_at automáticamente
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ language 'plpgsql';

-- Aplicar trigger a tablas relevantes
CREATE TRIGGER update_recording_targets_updated_at BEFORE UPDATE ON recording_targets
FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_exclusion_criteria_updated_at BEFORE UPDATE ON exclusion_criteria
FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_storage_config_updated_at BEFORE UPDATE ON storage_config
FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_plans_updated_at BEFORE UPDATE ON plans
FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

-- Comentarios para documentación
COMMENT ON TABLE plans IS 'Planes de suscripción disponibles para usuarios';
COMMENT ON TABLE user_plans IS 'Planes asignados a usuarios específicos';
COMMENT ON TABLE recording_targets IS 'Usuarios de Teams que serán grabados automáticamente';
COMMENT ON TABLE exclusion_criteria IS 'Criterios para excluir reuniones de la grabación';
COMMENT ON TABLE graph_subscriptions IS 'Suscripciones activas de Microsoft Graph API';
COMMENT ON TABLE auto_join_logs IS 'Registro de decisiones de auto-join para auditoría';
COMMENT ON TABLE storage_config IS 'Configuración de almacenamiento por usuario';
COMMENT ON TABLE recording_metadata IS 'Metadata extendida de grabaciones';

-- Completado
SELECT 'Migration 001 completed successfully' AS status;
```

### 1.2 Script de Rollback

**Archivo**: `src/server/migrations/001_teams_recording_tables_rollback.sql` (NUEVO)

```sql
-- ========================================
-- ROLLBACK: Teams Recording Bot Tables
-- Versión: 001
-- ========================================

-- Eliminar triggers
DROP TRIGGER IF EXISTS update_recording_targets_updated_at ON recording_targets;
DROP TRIGGER IF EXISTS update_exclusion_criteria_updated_at ON exclusion_criteria;
DROP TRIGGER IF EXISTS update_storage_config_updated_at ON storage_config;
DROP TRIGGER IF EXISTS update_plans_updated_at ON plans;

-- Eliminar función de trigger
DROP FUNCTION IF EXISTS update_updated_at_column();

-- Eliminar tablas en orden inverso (respetando foreign keys)
DROP TABLE IF EXISTS recording_metadata CASCADE;
DROP TABLE IF EXISTS auto_join_logs CASCADE;
DROP TABLE IF EXISTS graph_subscriptions CASCADE;
DROP TABLE IF EXISTS storage_config CASCADE;
DROP TABLE IF EXISTS exclusion_criteria CASCADE;
DROP TABLE IF EXISTS recording_targets CASCADE;
DROP TABLE IF EXISTS user_plans CASCADE;
DROP TABLE IF EXISTS plans CASCADE;

SELECT 'Rollback 001 completed successfully' AS status;
```

## 2. Scripts de Deployment

### 2.1 Script de Pre-deployment (Verificación)

**Archivo**: `scripts/pre-deploy.sh` (NUEVO)

```bash
#!/bin/bash

set -e

echo "🔍 Running pre-deployment checks..."

# Verificar Node.js
if ! command -v node &> /dev/null; then
    echo "❌ Node.js is not installed"
    exit 1
fi

NODE_VERSION=$(node -v | cut -d'v' -f2 | cut -d'.' -f1)
if [ "$NODE_VERSION" -lt 18 ]; then
    echo "❌ Node.js version must be 18 or higher (current: $NODE_VERSION)"
    exit 1
fi
echo "✅ Node.js version: $(node -v)"

# Verificar pnpm
if ! command -v pnpm &> /dev/null; then
    echo "❌ pnpm is not installed"
    exit 1
fi
echo "✅ pnpm version: $(pnpm -v)"

# Verificar PostgreSQL
if ! command -v psql &> /dev/null; then
    echo "⚠️  PostgreSQL client not found"
else
    echo "✅ PostgreSQL client version: $(psql --version)"
fi

# Verificar variables de entorno críticas
REQUIRED_VARS=(
    "DATABASE_URL"
    "AZURE_CLIENT_ID"
    "AZURE_TENANT_ID"
    "AZURE_CLIENT_SECRET"
    "NEXTAUTH_SECRET"
    "NEXTAUTH_URL"
)

MISSING_VARS=()
for var in "${REQUIRED_VARS[@]}"; do
    if [ -z "${!var}" ]; then
        MISSING_VARS+=("$var")
    fi
done

if [ ${#MISSING_VARS[@]} -gt 0 ]; then
    echo "❌ Missing required environment variables:"
    printf '   - %s\n' "${MISSING_VARS[@]}"
    exit 1
fi
echo "✅ All required environment variables are set"

# Verificar conectividad a la base de datos
echo "🔄 Testing database connection..."
if psql "$DATABASE_URL" -c "SELECT 1" &> /dev/null; then
    echo "✅ Database connection successful"
else
    echo "❌ Database connection failed"
    exit 1
fi

# Verificar espacio en disco
AVAILABLE_SPACE=$(df -BG /var/recordings 2>/dev/null | tail -1 | awk '{print $4}' | sed 's/G//')
if [ -z "$AVAILABLE_SPACE" ]; then
    echo "⚠️  Could not check disk space for /var/recordings"
elif [ "$AVAILABLE_SPACE" -lt 50 ]; then
    echo "⚠️  Warning: Less than 50GB available in /var/recordings ($AVAILABLE_SPACE GB free)"
else
    echo "✅ Sufficient disk space available ($AVAILABLE_SPACE GB free)"
fi

echo "✅ Pre-deployment checks passed!"
```

### 2.2 Script de Deployment Principal

**Archivo**: `scripts/deploy.sh` (NUEVO)

```bash
#!/bin/bash

set -e

# Colores para output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

echo -e "${GREEN}🚀 Starting deployment...${NC}"

# Verificar que estamos en el directorio correcto
if [ ! -f "package.json" ]; then
    echo -e "${RED}❌ Error: package.json not found. Run this script from the project root.${NC}"
    exit 1
fi

# Ejecutar pre-deployment checks
echo -e "${YELLOW}🔍 Running pre-deployment checks...${NC}"
bash scripts/pre-deploy.sh

# Cargar variables de entorno
if [ -f ".env" ]; then
    export $(cat .env | grep -v '^#' | xargs)
fi

# Git pull (si es deployment desde repo)
if [ "$1" == "--pull" ]; then
    echo -e "${YELLOW}📥 Pulling latest changes from git...${NC}"
    git pull origin main
fi

# Install/update dependencies
echo -e "${YELLOW}📦 Installing dependencies...${NC}"
pnpm install --frozen-lockfile

# Ejecutar migraciones
echo -e "${YELLOW}🗄️  Running database migrations...${NC}"
cd src/server

# Opción 1: Si usas Drizzle Kit
pnpm run db:push

# Opción 2: Si usas SQL directo
# psql "$DATABASE_URL" -f ../../migrations/001_teams_recording_tables.sql

# Build del proyecto
echo -e "${YELLOW}🔨 Building project...${NC}"
pnpm run build

cd ../..

# Backup de la base de datos antes de deployment
echo -e "${YELLOW}💾 Creating database backup...${NC}"
BACKUP_DIR="/var/backups/recordings"
mkdir -p "$BACKUP_DIR"
DATE=$(date +%Y%m%d_%H%M%S)

if [ ! -z "$DATABASE_URL" ]; then
    pg_dump "$DATABASE_URL" | gzip > "$BACKUP_DIR/pre_deploy_$DATE.sql.gz"
    echo -e "${GREEN}✅ Database backup created: pre_deploy_$DATE.sql.gz${NC}"
fi

# Reiniciar servicios
echo -e "${YELLOW}🔄 Restarting services...${NC}"

if command -v systemctl &> /dev/null; then
    # Usando systemd
    sudo systemctl restart teams-recording-server
    echo -e "${GREEN}✅ Service restarted via systemd${NC}"
elif command -v pm2 &> /dev/null; then
    # Usando PM2
    pm2 restart teams-recording-server
    echo -e "${GREEN}✅ Service restarted via PM2${NC}"
else
    echo -e "${YELLOW}⚠️  No service manager found. Please restart manually.${NC}"
fi

# Verificar que el servicio está corriendo
echo -e "${YELLOW}🔍 Verifying deployment...${NC}"
sleep 5

# Health check
if curl -f http://localhost:3000/api/health &> /dev/null; then
    echo -e "${GREEN}✅ Application is running and healthy${NC}"
else
    echo -e "${RED}❌ Application health check failed${NC}"
    exit 1
fi

# Limpiar archivos temporales
echo -e "${YELLOW}🧹 Cleaning up...${NC}"
find . -name "*.log" -mtime +7 -delete 2>/dev/null || true

echo -e "${GREEN}✅ Deployment completed successfully!${NC}"
echo ""
echo "📊 Deployment Summary:"
echo "   - Date: $(date)"
echo "   - Node version: $(node -v)"
echo "   - Database backup: $BACKUP_DIR/pre_deploy_$DATE.sql.gz"
echo ""
echo "Next steps:"
echo "   - Check logs: journalctl -u teams-recording-server -f"
echo "   - Monitor application: https://your-domain.com"
```

### 2.3 Script de Health Check

**Archivo**: `scripts/health-check.sh` (NUEVO)

```bash
#!/bin/bash

# Script de health check para monitoreo

# Verificar API
if curl -f -s http://localhost:3000/api/health > /dev/null; then
    echo "✅ API is healthy"
    API_STATUS=0
else
    echo "❌ API is down"
    API_STATUS=1
fi

# Verificar PostgreSQL
if psql "$DATABASE_URL" -c "SELECT 1" &> /dev/null; then
    echo "✅ Database is accessible"
    DB_STATUS=0
else
    echo "❌ Database is not accessible"
    DB_STATUS=1
fi

# Verificar espacio en disco
DISK_USAGE=$(df -h /var/recordings | tail -1 | awk '{print $5}' | sed 's/%//')
if [ "$DISK_USAGE" -lt 90 ]; then
    echo "✅ Disk space OK ($DISK_USAGE% used)"
    DISK_STATUS=0
else
    echo "⚠️  Disk space warning ($DISK_USAGE% used)"
    DISK_STATUS=1
fi

# Verificar memoria
MEM_USAGE=$(free | grep Mem | awk '{print int($3/$2 * 100.0)}')
if [ "$MEM_USAGE" -lt 90 ]; then
    echo "✅ Memory OK ($MEM_USAGE% used)"
    MEM_STATUS=0
else
    echo "⚠️  Memory warning ($MEM_USAGE% used)"
    MEM_STATUS=1
fi

# Verificar bots activos
if command -v psql &> /dev/null && [ ! -z "$DATABASE_URL" ]; then
    ACTIVE_BOTS=$(psql "$DATABASE_URL" -t -c "SELECT COUNT(*) FROM bots WHERE status NOT IN ('DONE', 'FATAL')")
    echo "📊 Active bots: $ACTIVE_BOTS"
fi

# Exit code (0 = todo OK, 1 = algún problema)
if [ $API_STATUS -eq 0 ] && [ $DB_STATUS -eq 0 ]; then
    exit 0
else
    exit 1
fi
```

### 2.4 Script de Renovación de Subscripciones de Graph

**Archivo**: `scripts/renew-graph-subscriptions.sh` (NUEVO)

```bash
#!/bin/bash

# Script para renovar suscripciones de Microsoft Graph que están por expirar
# Debe ejecutarse como cron job cada 12 horas

set -e

# Cargar variables de entorno
if [ -f "/opt/teams-recording-bot/src/server/.env" ]; then
    export $(cat /opt/teams-recording-bot/src/server/.env | grep -v '^#' | xargs)
fi

echo "🔄 Checking Graph API subscriptions for renewal..."

# Ejecutar script Node.js que renueva las suscripciones
cd /opt/teams-recording-bot/src/server
node -e "
const { graphService } = require('./dist/server/api/services/graph.service');
const { db } = require('./dist/server/db');
const { graphSubscriptions } = require('./dist/server/db/schema');
const { lt, eq } = require('drizzle-orm');

async function renewSubscriptions() {
    // Obtener suscripciones que expiran en las próximas 24 horas
    const expiringDate = new Date();
    expiringDate.setHours(expiringDate.getHours() + 24);

    const expiring = await db
        .select()
        .from(graphSubscriptions)
        .where(
            and(
                lt(graphSubscriptions.expirationDateTime, expiringDate),
                eq(graphSubscriptions.isActive, true)
            )
        );

    console.log(\`Found \${expiring.length} subscriptions to renew\`);

    for (const sub of expiring) {
        try {
            await graphService.renewSubscription(sub.subscriptionId);
            console.log(\`✅ Renewed subscription: \${sub.subscriptionId}\`);
        } catch (error) {
            console.error(\`❌ Failed to renew \${sub.subscriptionId}:\`, error.message);
        }
    }
}

renewSubscriptions().catch(console.error);
" 2>&1 | tee -a /var/log/graph-renewals.log

echo "✅ Subscription renewal check completed"
```

Agregar a crontab:

```bash
# Ejecutar cada 12 horas
0 */12 * * * /opt/teams-recording-bot/scripts/renew-graph-subscriptions.sh
```

## 3. CI/CD con GitHub Actions

**Archivo**: `.github/workflows/deploy.yml` (NUEVO)

```yaml
name: Deploy to Production

on:
  push:
    branches:
      - main
  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '20'

      - name: Install pnpm
        run: npm install -g pnpm

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Run tests
        run: pnpm test

      - name: Build
        run: |
          cd src/server
          pnpm run build

      - name: Deploy to VPS
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USERNAME }}
          key: ${{ secrets.VPS_SSH_KEY }}
          script: |
            cd /opt/teams-recording-bot
            git pull origin main
            bash scripts/deploy.sh

      - name: Health check
        run: |
          sleep 10
          curl -f https://your-domain.com/api/health || exit 1

      - name: Notify success
        if: success()
        run: echo "✅ Deployment successful!"

      - name: Notify failure
        if: failure()
        run: echo "❌ Deployment failed!"
```

## 4. Comandos Útiles

### Comandos de gestión diaria

```bash
# Ver logs en tiempo real
sudo journalctl -u teams-recording-server -f

# Reiniciar servicio
sudo systemctl restart teams-recording-server

# Ver estado del servicio
sudo systemctl status teams-recording-server

# Ver bots activos
psql $DATABASE_URL -c "SELECT id, status, meeting_title FROM bots WHERE status NOT IN ('DONE', 'FATAL')"

# Limpiar grabaciones antiguas (más de 90 días)
find /var/recordings -name "*.webm" -mtime +90 -exec rm {} \;

# Verificar espacio en disco
df -h /var/recordings

# Backup manual de base de datos
pg_dump $DATABASE_URL | gzip > backup_$(date +%Y%m%d).sql.gz

# Restaurar backup
gunzip -c backup_20241215.sql.gz | psql $DATABASE_URL
```

## Resumen

Esta configuración proporciona:

✅ Migraciones SQL completas para todas las tablas necesarias
✅ Scripts de deployment automatizados con verificaciones
✅ Health checks para monitoreo
✅ Scripts de renovación automática de Graph subscriptions
✅ CI/CD con GitHub Actions
✅ Comandos útiles para gestión diaria

### Próximos pasos

- Configurar alertas automáticas (email, Slack, etc.)
- Implementar backup incremental
- Configurar replicación de base de datos para HA
- Implementar queue system con Redis para mejor manejo de auto-join jobs
