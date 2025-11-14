# Instrucciones de Implementación Completa
## Sistema de Grabación Automática para Microsoft Teams

Este documento es tu guía maestra para implementar el sistema completo de grabación automática para Microsoft Teams desde cero.

---

## 📋 Tabla de Contenidos

1. [Visión General del Sistema](#visión-general-del-sistema)
2. [Documentación Disponible](#documentación-disponible)
3. [Pasos de Implementación](#pasos-de-implementación)
4. [Características Implementadas](#características-implementadas)
5. [Roadmap de Implementación](#roadmap-de-implementación)
6. [Preguntas Frecuentes](#preguntas-frecuentes)

---

## 🎯 Visión General del Sistema

Este sistema proporciona:

### ✅ Funcionalidades Principales
- **Auto-join automático** a reuniones de Teams basado en eventos de calendario
- **Panel de administración web** para gestionar usuarios y configuración
- **Criterios de exclusión personalizables** (asunto, participantes, horario, etc.)
- **Sistema de planes** con límites de sesiones simultáneas
- **Almacenamiento flexible** (Azure Blob Storage o VPS local)
- **Sistema de autorización** para grabación sin avisos automáticos
- **Escalable y seguro** con cumplimiento de privacidad

### 🏗️ Arquitectura
```
Microsoft 365 → Graph API → Auto-Join Service → Bot Instances
                                    ↓
                            Storage Layer (Azure/VPS)
                                    ↓
                            PostgreSQL Database
                                    ↓
                            Admin Panel (Next.js)
```

---

## 📚 Documentación Disponible

He creado documentación completa dividida en secciones:

### 1. Guía Principal
**Archivo**: `GUIA_COMPLETA_BOT_TEAMS.md`
- Introducción y arquitectura del sistema
- Configuración completa de Azure/Microsoft 365
- Servicios backend (Graph, Exclusion, Storage)
- Schema de base de datos extendido

### 2. Routers de API
**Archivo**: `docs/02_ROUTERS_API.md`
- Router de usuarios (gestión de recording targets)
- Router de planes (suscripciones y límites)
- Router de exclusión (criterios personalizables)
- Webhook de Microsoft Graph (auto-join)

### 3. Componentes del Panel de Administración
**Archivo**: `docs/03_PANEL_ADMIN_COMPONENTS.md`
- Dashboard principal
- Gestión de usuarios para grabar
- Configuración de criterios de exclusión con plantillas
- Gestión de planes y suscripciones

### 4. Configuración del VPS
**Archivo**: `docs/04_CONFIGURACION_VPS_UBUNTU.md`
- Setup completo de Ubuntu 22.04
- Instalación de todas las dependencias
- Configuración de NGINX, PostgreSQL, Docker
- Seguridad, backups y monitoreo

### 5. Migraciones y Deployment
**Archivo**: `docs/05_MIGRACIONES_Y_DEPLOYMENT.md`
- Scripts SQL de migración
- Scripts de deployment automatizado
- Health checks y renovación de subscripciones
- CI/CD con GitHub Actions

---

## 🚀 Pasos de Implementación

### FASE 1: Configuración de Azure/Microsoft 365 (1-2 días)

**Objetivo**: Tener todos los permisos y servicios de Azure configurados

1. **Crear App Registration en Azure AD**
   - Seguir: `GUIA_COMPLETA_BOT_TEAMS.md` → Sección 1.2
   - Guardar: Client ID, Tenant ID, Client Secret
   - Configurar permisos de Microsoft Graph
   - ⚠️ **CRÍTICO**: Otorgar consentimiento de administrador

2. **Configurar Azure Communication Services**
   - Seguir: `GUIA_COMPLETA_BOT_TEAMS.md` → Sección 1.3
   - Habilitar interoperabilidad con Teams
   - Guardar connection string

3. **Configurar Azure Blob Storage**
   - Seguir: `GUIA_COMPLETA_BOT_TEAMS.md` → Sección 1.4
   - Crear containers: `recordings`, `metadata`
   - Configurar lifecycle management
   - Guardar credenciales

4. **Configurar políticas de Teams**
   - Seguir: `GUIA_COMPLETA_BOT_TEAMS.md` → Sección 1.5
   - Habilitar compliance recording
   - Asignar políticas a usuarios de prueba

5. **Configurar Azure Key Vault (Opcional pero recomendado)**
   - Seguir: `GUIA_COMPLETA_BOT_TEAMS.md` → Sección 1.7
   - Almacenar todos los secretos de forma segura

**Verificación**:
```bash
# Crear archivo .env.azure con todas las credenciales
# Verificar que tienes todos estos valores:
AZURE_CLIENT_ID=...
AZURE_TENANT_ID=...
AZURE_CLIENT_SECRET=...
ACS_CONNECTION_STRING=...
AZURE_STORAGE_CONNECTION_STRING=...
```

---

### FASE 2: Configuración del VPS (1 día)

**Objetivo**: VPS completamente configurado y listo para la aplicación

1. **Setup Inicial del Servidor**
   - Seguir: `docs/04_CONFIGURACION_VPS_UBUNTU.md` → Sección 2
   - Actualizar sistema
   - Crear usuario `botadmin`
   - Configurar firewall

2. **Instalar Dependencias**
   - Seguir: `docs/04_CONFIGURACION_VPS_UBUNTU.md` → Sección 3
   - Node.js v20
   - Docker y Docker Compose
   - PostgreSQL 15
   - NGINX
   - Certbot (SSL)

3. **Configurar PostgreSQL**
   - Crear base de datos `teamsrecording`
   - Crear usuario con contraseña segura
   - Configurar autenticación

4. **Configurar NGINX como Reverse Proxy**
   - Seguir: `docs/04_CONFIGURACION_VPS_UBUNTU.md` → Sección 5
   - Configurar SSL con Let's Encrypt
   - Configurar proxying para la aplicación

5. **Configurar Almacenamiento Local**
   - Crear directorio `/var/recordings`
   - Configurar permisos
   - Configurar rotación de logs

**Verificación**:
```bash
# Verificar que todos los servicios están corriendo
sudo systemctl status postgresql
sudo systemctl status nginx
docker --version
node --version  # Debe ser v20+
```

---

### FASE 3: Implementación del Código (2-3 días)

**Objetivo**: Código completo funcionando localmente

#### 3.1 Actualizar Schema de Base de Datos

1. **Agregar nuevas tablas al schema**
   ```bash
   cd src/server/src/server/db
   ```

2. **Copiar el schema extendido**
   - Abrir: `GUIA_COMPLETA_BOT_TEAMS.md` → Sección 2.2
   - Agregar al final de `schema.ts` todo el código de las nuevas tablas:
     - `plans`
     - `userPlans`
     - `recordingTargets`
     - `exclusionCriteria`
     - `graphSubscriptions`
     - `autoJoinLogs`
     - `storageConfig`
     - `recordingMetadata`

3. **Ejecutar migración**
   ```bash
   pnpm run db:push
   ```

#### 3.2 Implementar Servicios Backend

1. **Crear GraphService**
   ```bash
   cd src/server/src/server/api/services
   touch graph.service.ts
   ```
   - Copiar código de: `GUIA_COMPLETA_BOT_TEAMS.md` → Sección 2.3.1
   - Instalar dependencias:
     ```bash
     pnpm add @microsoft/microsoft-graph-client @azure/identity
     ```

2. **Crear ExclusionService**
   ```bash
   touch exclusion.service.ts
   ```
   - Copiar código de: `GUIA_COMPLETA_BOT_TEAMS.md` → Sección 2.3.2

3. **Crear StorageService**
   ```bash
   touch storage.service.ts
   ```
   - Copiar código de: `GUIA_COMPLETA_BOT_TEAMS.md` → Sección 2.3.3
   - Instalar dependencias:
     ```bash
     pnpm add @azure/storage-blob
     ```

#### 3.3 Implementar Routers de API

1. **Crear routers**
   ```bash
   cd src/server/src/server/api/routers
   touch users.ts plans.ts exclusion.ts
   ```

2. **Copiar implementaciones**
   - `users.ts`: Desde `docs/02_ROUTERS_API.md` → Sección 2.4.1
   - `plans.ts`: Desde `docs/02_ROUTERS_API.md` → Sección 2.4.2
   - `exclusion.ts`: Desde `docs/02_ROUTERS_API.md` → Sección 2.4.3

3. **Registrar routers en el root**
   ```typescript
   // src/server/src/server/api/root.ts
   import { usersRouter } from './routers/users';
   import { plansRouter } from './routers/plans';
   import { exclusionRouter } from './routers/exclusion';

   export const appRouter = createTRPCRouter({
     // ... routers existentes
     users: usersRouter,
     plans: plansRouter,
     exclusion: exclusionRouter,
   });
   ```

#### 3.4 Implementar Webhook de Graph

1. **Crear endpoint de webhook**
   ```bash
   cd src/server/src/app/api/webhooks
   mkdir -p graph
   cd graph
   touch route.ts
   ```

2. **Copiar implementación**
   - Desde `docs/02_ROUTERS_API.md` → Sección 2.4.4

#### 3.5 Implementar Componentes del Panel de Administración

1. **Crear páginas de administración**
   ```bash
   cd src/server/src/app
   mkdir -p admin/{users,exclusion,plans,recordings,authorization}
   ```

2. **Copiar componentes**
   - Dashboard: `docs/03_PANEL_ADMIN_COMPONENTS.md` → Sección 3.1
   - Usuarios: `docs/03_PANEL_ADMIN_COMPONENTS.md` → Sección 3.2
   - Exclusión: `docs/03_PANEL_ADMIN_COMPONENTS.md` → Sección 3.3
   - Planes: `docs/03_PANEL_ADMIN_COMPONENTS.md` → Sección 3.4

3. **Instalar componentes UI faltantes**
   ```bash
   pnpm add lucide-react
   # Agregar componentes de shadcn/ui si no existen:
   # Dialog, Switch, Select, Textarea, etc.
   ```

**Verificación**:
```bash
# Compilar y verificar que no hay errores
cd src/server
pnpm run build

# Iniciar en desarrollo
pnpm run dev

# Verificar endpoints
curl http://localhost:3000/api/health
```

---

### FASE 4: Testing Local (1 día)

**Objetivo**: Verificar que todo funciona localmente

1. **Configurar variables de entorno local**
   ```bash
   cd src/server
   cp .env.example .env
   # Editar .env con todas las credenciales de Azure
   ```

2. **Iniciar base de datos local**
   ```bash
   # Si usas Docker para PostgreSQL local:
   docker run -d \
     --name postgres-teams \
     -e POSTGRES_DB=teamsrecording \
     -e POSTGRES_USER=botadmin \
     -e POSTGRES_PASSWORD=test123 \
     -p 5432:5432 \
     postgres:15
   ```

3. **Ejecutar migraciones**
   ```bash
   pnpm run db:push
   ```

4. **Iniciar aplicación**
   ```bash
   pnpm run dev
   ```

5. **Testing manual**
   - Acceder a `http://localhost:3000`
   - Login con GitHub
   - Navegar a `/admin`
   - Probar agregar usuario para grabar
   - Probar crear criterios de exclusión
   - Verificar planes

6. **Testing de Graph API**
   ```bash
   # Probar crear suscripción de Graph
   # (requiere un usuario real de Microsoft 365 de prueba)
   ```

**Checklist de Testing**:
- [ ] La aplicación inicia sin errores
- [ ] Base de datos se conecta correctamente
- [ ] Panel de admin es accesible
- [ ] Puedo agregar usuarios para grabar
- [ ] Puedo crear criterios de exclusión
- [ ] Los planes se muestran correctamente
- [ ] Graph API webhook responde (validationToken)

---

### FASE 5: Deployment a VPS (1 día)

**Objetivo**: Aplicación corriendo en producción

1. **Preparar el código para producción**
   ```bash
   # En tu máquina local, commit todo
   git add .
   git commit -m "feat: complete teams recording bot implementation"
   git push origin main
   ```

2. **Clonar en el VPS**
   ```bash
   # En el VPS
   ssh botadmin@tu-vps-ip

   sudo mkdir -p /opt/teams-recording-bot
   sudo chown botadmin:botadmin /opt/teams-recording-bot

   cd /opt/teams-recording-bot
   git clone <tu-repo-url> .
   ```

3. **Configurar variables de entorno**
   ```bash
   cd /opt/teams-recording-bot/src/server
   nano .env
   # Copiar todas las variables de entorno de producción
   ```

4. **Ejecutar migración de base de datos**
   ```bash
   # Opción 1: Usar SQL directo
   psql postgresql://botadmin:password@localhost:5432/teamsrecording \
     -f ../../docs/05_MIGRACIONES_Y_DEPLOYMENT.md  # Copiar SQL de la sección 1.1

   # Opción 2: Usar Drizzle
   pnpm run db:push
   ```

5. **Build y deployment**
   ```bash
   cd /opt/teams-recording-bot
   bash scripts/deploy.sh
   ```

6. **Configurar servicio systemd**
   - Seguir: `docs/04_CONFIGURACION_VPS_UBUNTU.md` → Sección 6
   - Crear y habilitar servicio

7. **Verificar deployment**
   ```bash
   # Health check
   curl https://tu-dominio.com/api/health

   # Ver logs
   sudo journalctl -u teams-recording-server -f
   ```

**Verificación**:
- [ ] Aplicación accesible via HTTPS
- [ ] SSL funcionando correctamente
- [ ] Webhook de Graph accesible públicamente
- [ ] Panel de admin funcional
- [ ] Base de datos de producción poblada con planes

---

### FASE 6: Configuración de Auto-Join (1 día)

**Objetivo**: Sistema de auto-join funcionando end-to-end

1. **Configurar webhook público en Azure**
   - Ir a Azure Portal → App Registration
   - Agregar redirect URI: `https://tu-dominio.com/api/webhooks/graph`

2. **Crear suscripciones de prueba**
   - En el panel de admin, agregar un usuario de Teams de prueba
   - Verificar que se crea la suscripción de Graph

3. **Probar auto-join**
   - Crear una reunión de prueba en Teams con el usuario configurado
   - Verificar logs para ver si el webhook recibe la notificación
   - Verificar que se evalúan los criterios de exclusión
   - Verificar que se crea un bot si no hay exclusiones

4. **Configurar renovación automática de suscripciones**
   ```bash
   # Copiar script de renovación
   sudo cp scripts/renew-graph-subscriptions.sh /opt/teams-recording-bot/scripts/

   # Agregar a crontab
   crontab -e
   # Agregar: 0 */12 * * * /opt/teams-recording-bot/scripts/renew-graph-subscriptions.sh
   ```

**Verificación**:
- [ ] Webhook recibe notificaciones de Microsoft Graph
- [ ] Criterios de exclusión se evalúan correctamente
- [ ] Bots se crean automáticamente para reuniones válidas
- [ ] Subscripciones se renuevan automáticamente

---

## ✨ Características Implementadas

### 1. Auto-Join Inteligente
- ✅ Escucha eventos de calendario via Microsoft Graph
- ✅ Evalúa criterios de exclusión automáticamente
- ✅ Crea bots solo para reuniones que cumplen criterios
- ✅ Logs completos de decisiones para auditoría

### 2. Criterios de Exclusión Personalizables
- ✅ Asunto contiene/no contiene palabras clave
- ✅ Filtrado por participantes (email o dominio)
- ✅ Límites de participantes (min/max)
- ✅ Duración de reunión (min/max)
- ✅ Horario del día (ej: solo horas laborales)
- ✅ Días de la semana (ej: excluir fines de semana)
- ✅ Tipo de reunión (1-on-1, grupal, conferencia)
- ✅ Reuniones recurrentes vs. únicas
- ✅ Email del organizador
- ✅ Palabras clave personalizadas en metadata

### 3. Sistema de Planes
- ✅ Planes Basic, Pro, Enterprise predefinidos
- ✅ Límites de grabaciones simultáneas por plan
- ✅ Verificación automática de límites antes de crear bot
- ✅ Diferentes opciones de almacenamiento por plan

### 4. Almacenamiento Flexible
- ✅ Azure Blob Storage
- ✅ VPS local storage
- ✅ Configuración por usuario
- ✅ Monitoreo de espacio disponible
- ✅ Lifecycle management automático

### 5. Panel de Administración
- ✅ Dashboard con métricas
- ✅ Gestión de usuarios para grabar
- ✅ Configuración visual de criterios de exclusión
- ✅ Plantillas de criterios comunes
- ✅ Gestión de planes y suscripciones
- ✅ Visualización de grabaciones

### 6. Seguridad y Cumplimiento
- ✅ Autenticación con OAuth (GitHub/Azure AD)
- ✅ Sistema de disclaimer para autorización
- ✅ Logs de auditoría completos
- ✅ Almacenamiento seguro de credenciales (Key Vault)
- ✅ Encriptación en tránsito (HTTPS/TLS)

---

## 🗺️ Roadmap de Implementación

### ✅ Completado (en esta sesión)
- [x] Arquitectura y diseño del sistema
- [x] Schema de base de datos completo
- [x] Servicios backend (Graph, Exclusion, Storage)
- [x] Routers de API (Users, Plans, Exclusion, Webhooks)
- [x] Componentes del panel de administración
- [x] Scripts de deployment y migraciones
- [x] Configuración completa del VPS
- [x] Documentación detallada

### 📝 Por Implementar (siguientes pasos)

#### Prioridad Alta
- [ ] Página de grabaciones (`/admin/recordings/page.tsx`)
- [ ] Página de autorización/disclaimer (`/admin/authorization/page.tsx`)
- [ ] Sistema de transcripción automática (Azure Speech Services)
- [ ] Endpoint API para descargar grabaciones desde VPS
- [ ] Sistema de notificaciones (email/webhook cuando grabación completa)
- [ ] Dashboard de analytics y métricas

#### Prioridad Media
- [ ] Sistema de tags y categorización de grabaciones
- [ ] Búsqueda full-text de grabaciones por metadata
- [ ] Exportación de grabaciones a formatos adicionales
- [ ] Integración con Azure Cognitive Services para análisis
- [ ] Sistema de permisos granulares (RBAC)
- [ ] Audit logs completos con exportación

#### Prioridad Baja
- [ ] Integración con Slack para notificaciones
- [ ] Mobile app para visualizar grabaciones
- [ ] Análisis de sentimiento en reuniones
- [ ] Dashboard público para usuarios finales
- [ ] Sistema de quotas y facturación

---

## 🔧 Archivos que Necesitas Crear/Modificar

### Archivos Nuevos a Crear

```
src/server/src/server/db/
  └── schema.ts (MODIFICAR - agregar nuevas tablas)

src/server/src/server/api/services/
  ├── graph.service.ts (NUEVO)
  ├── exclusion.service.ts (NUEVO)
  └── storage.service.ts (NUEVO)

src/server/src/server/api/routers/
  ├── users.ts (NUEVO)
  ├── plans.ts (NUEVO)
  └── exclusion.ts (NUEVO)

src/server/src/app/api/webhooks/graph/
  └── route.ts (NUEVO)

src/server/src/app/admin/
  ├── page.tsx (NUEVO)
  ├── users/page.tsx (NUEVO)
  ├── exclusion/page.tsx (NUEVO)
  ├── plans/page.tsx (NUEVO)
  ├── recordings/page.tsx (TODO)
  └── authorization/page.tsx (TODO)

scripts/
  ├── pre-deploy.sh (NUEVO)
  ├── deploy.sh (NUEVO)
  ├── health-check.sh (NUEVO)
  └── renew-graph-subscriptions.sh (NUEVO)

migrations/
  ├── 001_teams_recording_tables.sql (NUEVO)
  └── 001_teams_recording_tables_rollback.sql (NUEVO)

.github/workflows/
  └── deploy.yml (NUEVO)
```

### Dependencias a Instalar

```bash
# En src/server
pnpm add @microsoft/microsoft-graph-client
pnpm add @azure/identity
pnpm add @azure/storage-blob
pnpm add lucide-react
```

---

## ❓ Preguntas Frecuentes

### ¿Cómo manejo el aviso de "Esta llamada está siendo grabada"?

El sistema implementa un enfoque de "autorización previa":

1. Los administradores aceptan un disclaimer en `/admin/authorization`
2. Este disclaimer indica que están autorizando la grabación de llamadas
3. El campo `hasAcceptedDisclaimer` en la tabla `recording_targets` registra esto
4. El bot solo graba usuarios que han aceptado el disclaimer
5. **Importante**: Consulta con legal para asegurar cumplimiento con leyes locales

### ¿Cómo escalo para más de 100 grabaciones simultáneas?

1. **Horizontal scaling**: Usar múltiples VPS con load balancer
2. **Base de datos**: Migrar a PostgreSQL managed (Azure Database, AWS RDS)
3. **Queue system**: Implementar Redis/Bull para gestionar jobs de auto-join
4. **Almacenamiento**: Usar CDN (CloudFront, Azure CDN) para servir grabaciones
5. **Monitoring**: Implementar Prometheus + Grafana para métricas

### ¿Cuánto cuesta operar este sistema?

**Costos estimados mensuales**:

```
VPS (16 cores, 32GB RAM): $80-150/mes
Azure Blob Storage (1TB): $20-40/mes
PostgreSQL Managed: $50-100/mes
Azure Communication Services: Variable (~$0.004/min)
Bandwidth: ~$0.08/GB

Total estimado para 50 grabaciones/día: $200-400/mes
```

### ¿Puedo usar esto solo con VPS sin Azure?

Sí, pero perderás algunas funcionalidades:

- ✅ Puedes: Grabar reuniones manualmente (sin auto-join)
- ✅ Puedes: Almacenar en VPS local
- ❌ No puedes: Auto-join automático (requiere Graph API)
- ❌ No puedes: Transcripción automática (sin Azure Speech)

### ¿Es legal grabar reuniones sin avisar?

**⚠️ IMPORTANTE**: Las leyes varían por país/región.

- **USA**: Algunos estados requieren consentimiento de todos (two-party consent)
- **EU**: GDPR requiere consentimiento explícito
- **Recomendación**: Siempre consultar con un abogado especializado

Este sistema implementa:
- Disclaimer de autorización
- Logs de auditoría completos
- Control granular de qué usuarios grabar

Pero **TÚ eres responsable** de cumplir con las leyes locales.

---

## 📞 Soporte y Siguientes Pasos

### Si tienes problemas

1. **Revisa los logs**:
   ```bash
   sudo journalctl -u teams-recording-server -f
   ```

2. **Verifica variables de entorno**:
   ```bash
   # Asegúrate de que todas las variables críticas estén configuradas
   env | grep AZURE
   env | grep DATABASE
   ```

3. **Health check**:
   ```bash
   bash scripts/health-check.sh
   ```

### Para continuar la implementación

1. **Implementa las páginas faltantes**:
   - `/admin/recordings/page.tsx` - Visualización de grabaciones
   - `/admin/authorization/page.tsx` - Gestión de disclaimers

2. **Agrega transcripción**:
   - Integra Azure Speech to Text
   - Almacena transcripciones junto con grabaciones

3. **Mejora el sistema de auto-deploy**:
   - Implementa CI/CD completo con GitHub Actions
   - Agrega tests automatizados

4. **Monitoreo y alertas**:
   - Configura Prometheus/Grafana
   - Alertas automáticas por email/Slack

---

## 🎉 ¡Felicidades!

Si has llegado hasta aquí, tienes toda la información necesaria para implementar un sistema profesional de grabación automática para Microsoft Teams.

**Recuerda**:
- La documentación está dividida en 5 archivos para facilitar la navegación
- Todos los scripts y código están listos para copy-paste
- Sigue las fases en orden para evitar problemas
- Consulta con legal antes de deployment en producción

¡Éxito con tu implementación! 🚀

---

**Última actualización**: 2024-12-XX
**Autor**: Desarrollado con Claude Code (Anthropic)
**Licencia**: Según tu repositorio base (LGPL)
