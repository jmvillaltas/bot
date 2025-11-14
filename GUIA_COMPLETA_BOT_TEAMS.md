# Guía Completa: Bot de Grabación Automática para Microsoft Teams

## Tabla de Contenidos
1. [Introducción](#introducción)
2. [Arquitectura del Sistema](#arquitectura-del-sistema)
3. [SECCIÓN 1: Configuración de Microsoft/Azure/Teams](#sección-1-configuración-de-microsoftazureteams)
4. [SECCIÓN 2: Desarrollo del Código](#sección-2-desarrollo-del-código)
5. [SECCIÓN 3: Configuración VPS Ubuntu 22.04](#sección-3-configuración-vps-ubuntu-2204)
6. [Despliegue y Pruebas](#despliegue-y-pruebas)
7. [Consideraciones de Seguridad y Cumplimiento](#consideraciones-de-seguridad-y-cumplimiento)

---

## Introducción

Este documento proporciona una guía completa para desarrollar un sistema de bot de grabación automática para Microsoft Teams que incluye:

- ✅ Auto-join a llamadas 1-a-1, reuniones grupales y conferencias
- ✅ Panel de administración web para gestión de usuarios y criterios
- ✅ Criterios de exclusión personalizables (asunto, participantes, horario, etc.)
- ✅ Sistema de planes con límites de sesiones simultáneas
- ✅ Almacenamiento flexible (Azure Blob Storage o VPS local)
- ✅ Sistema de autorización de grabación sin avisos automáticos
- ✅ Escalable, seguro y compatible con regulaciones de privacidad

---

## Arquitectura del Sistema

```
┌─────────────────────────────────────────────────────────────────┐
│                     MICROSOFT 365 / TEAMS                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   Calendar   │  │   Teams      │  │   Graph API  │          │
│  │   Events     │  │   Meetings   │  │   Webhooks   │          │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘          │
└─────────┼──────────────────┼──────────────────┼─────────────────┘
          │                  │                  │
          │                  │                  │
┌─────────▼──────────────────▼──────────────────▼─────────────────┐
│                   AZURE APP REGISTRATION                         │
│  • OnlineMeetings.ReadWrite.All                                 │
│  • Calendars.Read.All                                           │
│  • User.Read.All                                                │
└─────────┬────────────────────────────────────────────────────────┘
          │
          │ Graph API Calls
          │
┌─────────▼─────────────────────────────────────────────────────┐
│                  AUTO-JOIN SERVICE (Node.js)                   │
│  ┌────────────────────────────────────────────────────────┐   │
│  │  • Escucha eventos de calendario via Microsoft Graph   │   │
│  │  • Evalúa criterios de exclusión                       │   │
│  │  • Crea bots para reuniones que cumplen criterios      │   │
│  └────────────────────────────────────────────────────────┘   │
└─────────┬─────────────────────────────────────────────────────┘
          │
          │ Crea Bot Instance
          │
┌─────────▼─────────────────────────────────────────────────────┐
│              BOT INSTANCES (Puppeteer + Teams)                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │
│  │  Bot 1       │  │  Bot 2       │  │  Bot N       │        │
│  │  Recording   │  │  Recording   │  │  Recording   │        │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘        │
└─────────┼──────────────────┼──────────────────┼───────────────┘
          │                  │                  │
          │ Streams          │                  │
          │                  │                  │
┌─────────▼──────────────────▼──────────────────▼───────────────┐
│                   STORAGE LAYER                                │
│  ┌─────────────────────┐      ┌─────────────────────┐        │
│  │  Azure Blob Storage │  OR  │  VPS Local Storage  │        │
│  └─────────────────────┘      └─────────────────────┘        │
└────────────────────────────────────────────────────────────────┘
          │                                      │
          │                                      │
┌─────────▼──────────────────────────────────────▼───────────────┐
│                   DATABASE (PostgreSQL)                         │
│  • Usuarios y permisos de grabación                            │
│  • Criterios de exclusión personalizados                       │
│  • Metadata de grabaciones                                     │
│  • Planes y límites de sesiones simultáneas                    │
│  • Logs de eventos                                             │
└─────────┬──────────────────────────────────────────────────────┘
          │
          │
┌─────────▼──────────────────────────────────────────────────────┐
│             PANEL DE ADMINISTRACIÓN (Next.js)                   │
│  • Gestión de usuarios para grabar                             │
│  • Configuración de criterios de exclusión                     │
│  • Gestión de planes y límites                                 │
│  • Autorización de grabación (disclaimer)                      │
│  • Visualización de grabaciones                                │
│  • Analytics y reportes                                        │
└─────────────────────────────────────────────────────────────────┘
```

---

## SECCIÓN 1: Configuración de Microsoft/Azure/Teams

### 1.1 Requisitos Previos

Antes de comenzar, necesitas:
- ✅ Una cuenta de Microsoft 365 con permisos de administrador global
- ✅ Una suscripción de Azure (puede ser la misma cuenta)
- ✅ Acceso al Centro de Administración de Teams
- ✅ Conocimientos básicos de Azure Portal

### 1.2 Registrar Aplicación en Azure Active Directory

#### Paso 1: Crear App Registration

1. **Navegar a Azure Portal**
   - Ir a https://portal.azure.com
   - Iniciar sesión con tu cuenta de administrador

2. **Crear nueva aplicación**
   ```
   Azure Portal → Azure Active Directory → App registrations → New registration
   ```

3. **Configurar detalles de la aplicación**
   - **Nombre**: `TeamsRecordingBot` (o el nombre que prefieras)
   - **Supported account types**:
     - Seleccionar "Accounts in this organizational directory only (Single tenant)"
   - **Redirect URI**:
     - Tipo: Web
     - URL: `https://tu-dominio.com/api/auth/callback` (reemplazar con tu dominio)
   - Click en **Register**

4. **Guardar información importante**
   Después de crear la app, guarda estos valores (los necesitarás después):
   ```
   Application (client) ID: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
   Directory (tenant) ID: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
   ```

#### Paso 2: Configurar Client Secret

1. **Crear secreto**
   ```
   Tu App → Certificates & secrets → New client secret
   ```

2. **Configurar secreto**
   - **Description**: `TeamsRecordingBotSecret`
   - **Expires**: 24 months (o según tus políticas de seguridad)
   - Click en **Add**

3. **IMPORTANTE: Copiar el valor del secreto INMEDIATAMENTE**
   ```
   Client Secret Value: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
   ```
   ⚠️ **ADVERTENCIA**: Este valor solo se muestra una vez. Guárdalo en un lugar seguro.

#### Paso 3: Configurar Permisos de API (MUY IMPORTANTE)

Esta es la parte más crítica. Necesitas configurar permisos específicos para que el bot funcione.

1. **Navegar a API permissions**
   ```
   Tu App → API permissions → Add a permission
   ```

2. **Agregar permisos de Microsoft Graph**

   Selecciona **Microsoft Graph** → **Application permissions** y agrega:

   **Para leer calendarios y reuniones:**
   ```
   ✓ Calendars.Read            - Leer calendarios de todos los usuarios
   ✓ OnlineMeetings.Read.All   - Leer información de reuniones online
   ```

   **Para unirse a reuniones y grabar:**
   ```
   ✓ OnlineMeetings.ReadWrite.All  - Gestionar reuniones online
   ✓ Calls.JoinGroupCallAsGuest.All - Unirse a llamadas grupales como invitado
   ✓ Calls.AccessMedia.All          - Acceder a streams de medios en llamadas
   ```

   **Para obtener información de usuarios:**
   ```
   ✓ User.Read.All             - Leer perfiles de usuarios
   ✓ Directory.Read.All        - Leer información del directorio
   ```

   **Para webhooks y notificaciones:**
   ```
   ✓ ChannelMessage.Read.All   - Leer mensajes de canales (opcional)
   ```

3. **CRÍTICO: Otorgar consentimiento de administrador**

   Después de agregar todos los permisos:
   ```
   Click en "Grant admin consent for [Tu organización]"
   ```

   ⚠️ **Sin este paso, la aplicación NO funcionará**

4. **Verificar permisos**

   Asegúrate de que todos los permisos muestren:
   ```
   Status: Granted for [Tu organización]
   Type: Application
   ```

#### Paso 4: Configurar Authentication

1. **Configurar Platform settings**
   ```
   Tu App → Authentication → Add a platform → Web
   ```

2. **Agregar Redirect URIs**
   ```
   https://tu-dominio.com/api/auth/callback
   https://tu-dominio.com/api/auth/signin
   ```

3. **Configurar tokens**

   En la sección "Implicit grant and hybrid flows":
   - ☑️ **Access tokens** (used for implicit flows)
   - ☑️ **ID tokens** (used for implicit and hybrid flows)

4. **Guardar cambios**

### 1.3 Configurar Azure Communication Services (ACS)

Azure Communication Services es necesario para unirse a llamadas de Teams programáticamente.

#### Paso 1: Crear recurso ACS

1. **Crear recurso**
   ```
   Azure Portal → Create a resource → Search "Communication Services"
   → Create
   ```

2. **Configurar recurso**
   - **Subscription**: Tu suscripción
   - **Resource group**: Crear nuevo o usar existente (ej: `rg-teams-recording-bot`)
   - **Resource name**: `acs-teams-recording-bot`
   - **Data location**: Elegir región cercana (ej: East US)
   - Click **Review + create** → **Create**

3. **Obtener Connection String**
   ```
   Tu recurso ACS → Keys → Copy "Connection string"
   ```

   Guarda:
   ```
   ACS_CONNECTION_STRING=endpoint=https://acs-teams-recording-bot.communication.azure.com/;accesskey=xxxxxx
   ```

#### Paso 2: Habilitar Interoperabilidad con Teams

1. **Navegar a configuración**
   ```
   Tu recurso ACS → Teams Interop (en el menú izquierdo)
   ```

2. **Habilitar Teams interoperability**
   - Toggle: **Enabled**
   - **Application ID**: Pegar tu Application (client) ID de la App Registration
   - Click **Save**

3. **Verificar estado**

   Debería mostrar:
   ```
   Status: Enabled
   Application ID: tu-client-id
   ```

### 1.4 Configurar Almacenamiento en Azure Blob Storage

#### Paso 1: Crear Storage Account

1. **Crear cuenta de almacenamiento**
   ```
   Azure Portal → Create a resource → Storage account → Create
   ```

2. **Configurar cuenta**
   - **Subscription**: Tu suscripción
   - **Resource group**: `rg-teams-recording-bot`
   - **Storage account name**: `stteamsrecordings` (debe ser único globalmente)
   - **Region**: La misma que ACS (ej: East US)
   - **Performance**: Standard
   - **Redundancy**: LRS (Locally-redundant storage) para desarrollo, GRS para producción

3. **Configurar opciones avanzadas**

   En la pestaña **Advanced**:
   - **Require secure transfer**: Enabled
   - **Allow Blob public access**: Disabled (por seguridad)
   - **Minimum TLS version**: Version 1.2

4. **Crear**

   Click **Review + create** → **Create**

#### Paso 2: Crear Container para grabaciones

1. **Navegar a Containers**
   ```
   Tu Storage Account → Containers → + Container
   ```

2. **Crear container**
   - **Name**: `recordings`
   - **Public access level**: Private
   - Click **Create**

3. **Crear container para metadata**
   - **Name**: `metadata`
   - **Public access level**: Private
   - Click **Create**

#### Paso 3: Obtener credenciales de acceso

1. **Obtener Access Keys**
   ```
   Tu Storage Account → Access keys → Show keys
   ```

   Guarda:
   ```
   AZURE_STORAGE_ACCOUNT_NAME=stteamsrecordings
   AZURE_STORAGE_ACCESS_KEY=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
   ```

2. **Obtener Connection String**
   ```
   AZURE_STORAGE_CONNECTION_STRING=DefaultEndpointsProtocol=https;AccountName=stteamsrecordings;AccountKey=xxxxx;EndpointSuffix=core.windows.net
   ```

#### Paso 4: Configurar Lifecycle Management (Opcional pero recomendado)

Para gestión automática de almacenamiento:

1. **Navegar a Lifecycle management**
   ```
   Tu Storage Account → Lifecycle Management → Add rule
   ```

2. **Crear regla de retención**
   ```
   Rule name: DeleteOldRecordings
   Rule type: Delete blobs
   Blob type: Block blobs

   Conditions:
   - Blob was last modified: more than 90 days ago

   Actions:
   - Delete the blob
   ```

### 1.5 Configurar Políticas de Teams

#### Paso 1: Habilitar Cloud Recording (Si no está habilitado)

1. **Teams Admin Center**
   ```
   https://admin.teams.microsoft.com → Meetings → Meeting policies
   ```

2. **Configurar política**
   - Seleccionar política (Global o crear nueva)
   - **Cloud recording**: On
   - **Save**

#### Paso 2: Configurar Compliance Recording (Importante para bots)

1. **Navegar a Compliance recording policies**
   ```
   Teams Admin Center → Voice → Compliance recording policies
   ```

2. **Crear nueva política**
   ```
   + Add → Nombre: "BotRecordingPolicy"
   ```

3. **Configurar política**
   - **Compliance recording**: On
   - **Application ID**: Tu Application (client) ID
   - **Save**

4. **Asignar política a usuarios**
   ```
   Seleccionar usuarios → Actions → Assign policies → Compliance recording policy → BotRecordingPolicy
   ```

#### Paso 3: Configurar permisos para aplicaciones de terceros

1. **Navegar a App setup policies**
   ```
   Teams Admin Center → Teams apps → Setup policies
   ```

2. **Verificar que "Allow user pinning" está habilitado**

### 1.6 Configurar Microsoft Graph Webhooks/Subscriptions

Para auto-join, necesitas recibir notificaciones de nuevas reuniones.

#### Paso 1: Preparar endpoint público

Tu servidor necesita un endpoint HTTPS público accesible:
```
https://tu-dominio.com/api/webhooks/graph
```

#### Paso 2: Crear suscripción de Graph (esto se hará vía código)

Necesitarás crear suscripciones vía Microsoft Graph API. Esto se implementará en el código, pero aquí está la estructura:

```javascript
// Ejemplo de suscripción para eventos de calendario
POST https://graph.microsoft.com/v1.0/subscriptions
{
  "changeType": "created,updated",
  "notificationUrl": "https://tu-dominio.com/api/webhooks/graph",
  "resource": "/users/{userId}/events",
  "expirationDateTime": "2024-12-31T18:23:45.9356000Z",
  "clientState": "secretClientValue"
}
```

### 1.7 Configurar Azure Key Vault (Recomendado para producción)

Para almacenar secretos de forma segura:

#### Paso 1: Crear Key Vault

1. **Crear recurso**
   ```
   Azure Portal → Create a resource → Key Vault → Create
   ```

2. **Configurar**
   - **Name**: `kv-teams-recording-bot`
   - **Region**: La misma región
   - **Pricing tier**: Standard
   - Click **Review + create** → **Create**

#### Paso 2: Agregar secretos

1. **Navegar a Secrets**
   ```
   Tu Key Vault → Secrets → + Generate/Import
   ```

2. **Agregar cada secreto**
   ```
   Name: AZURE-CLIENT-SECRET
   Value: tu-client-secret

   Name: ACS-CONNECTION-STRING
   Value: tu-acs-connection-string

   Name: AZURE-STORAGE-ACCESS-KEY
   Value: tu-storage-access-key
   ```

#### Paso 3: Configurar Access Policy

1. **Agregar access policy**
   ```
   Tu Key Vault → Access policies → + Add Access Policy
   ```

2. **Configurar permisos**
   - **Secret permissions**: Get, List
   - **Select principal**: Seleccionar tu App Registration
   - Click **Add** → **Save**

### 1.8 Resumen de Credenciales Necesarias

Al final de esta sección, deberías tener estas credenciales guardadas:

```bash
# Azure AD App Registration
AZURE_CLIENT_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
AZURE_TENANT_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
AZURE_CLIENT_SECRET=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# Azure Communication Services
ACS_CONNECTION_STRING=endpoint=https://...;accesskey=...

# Azure Blob Storage
AZURE_STORAGE_ACCOUNT_NAME=stteamsrecordings
AZURE_STORAGE_ACCESS_KEY=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
AZURE_STORAGE_CONNECTION_STRING=DefaultEndpointsProtocol=https;...

# Microsoft Graph API
GRAPH_API_ENDPOINT=https://graph.microsoft.com/v1.0

# Webhook
WEBHOOK_URL=https://tu-dominio.com/api/webhooks/graph
WEBHOOK_SECRET=generate-a-random-secret-here

# Key Vault (si se usa)
KEY_VAULT_URL=https://kv-teams-recording-bot.vault.azure.net/
```

⚠️ **IMPORTANTE**: Nunca commits estos valores a Git. Usa variables de entorno o Azure Key Vault.

---

## SECCIÓN 2: Desarrollo del Código

Esta sección detalla todo el código necesario para implementar el sistema completo.

### 2.1 Estructura del Proyecto

El proyecto ya tiene una base sólida. Ampliaremos la estructura existente:

```
/home/user/bot/
├── src/
│   ├── bots/
│   │   ├── teams/
│   │   │   └── src/
│   │   │       ├── bot.ts                    # ✅ Ya existe
│   │   │       └── auto-join.ts              # ⭐ NUEVO
│   │   └── src/
│   │       ├── bot.ts
│   │       ├── types.ts
│   │       └── graph-client.ts               # ⭐ NUEVO
│   ├── server/
│   │   └── src/
│   │       ├── server/
│   │       │   ├── api/
│   │       │   │   ├── routers/
│   │       │   │   │   ├── bots.ts           # ✅ Ya existe
│   │       │   │   │   ├── users.ts          # ⭐ NUEVO
│   │       │   │   │   ├── plans.ts          # ⭐ NUEVO
│   │       │   │   │   ├── exclusion.ts      # ⭐ NUEVO
│   │       │   │   │   └── webhooks.ts       # ⭐ NUEVO
│   │       │   │   └── services/
│   │       │   │       ├── graph.service.ts  # ⭐ NUEVO
│   │       │   │       ├── exclusion.service.ts # ⭐ NUEVO
│   │       │   │       └── storage.service.ts   # ⭐ NUEVO
│   │       │   ├── db/
│   │       │   │   └── schema.ts             # ✅ Modificar
│   │       │   └── middleware/
│   │       │       └── plan-limits.ts        # ⭐ NUEVO
│   │       ├── app/
│   │       │   ├── admin/
│   │       │   │   ├── users/
│   │       │   │   │   └── page.tsx          # ⭐ NUEVO - Gestión usuarios
│   │       │   │   ├── exclusion/
│   │       │   │   │   └── page.tsx          # ⭐ NUEVO - Criterios exclusión
│   │       │   │   ├── plans/
│   │       │   │   │   └── page.tsx          # ⭐ NUEVO - Gestión planes
│   │       │   │   ├── recordings/
│   │       │   │   │   └── page.tsx          # ⭐ NUEVO - Ver grabaciones
│   │       │   │   └── authorization/
│   │       │   │       └── page.tsx          # ⭐ NUEVO - Disclaimer/Auth
│   │       │   └── api/
│   │       │       └── webhooks/
│   │       │           └── graph/
│   │       │               └── route.ts      # ⭐ NUEVO
│   │       └── components/
│   │           ├── admin/
│   │           │   ├── UserSelector.tsx      # ⭐ NUEVO
│   │           │   ├── ExclusionRules.tsx    # ⭐ NUEVO
│   │           │   └── PlanManager.tsx       # ⭐ NUEVO
│   └── terraform/                            # ✅ Ya existe
└── docs/
    └── GUIA_COMPLETA_BOT_TEAMS.md            # Este archivo
```

### 2.2 Actualizar Schema de Base de Datos

Primero, ampliaremos el schema de la base de datos para soportar todas las nuevas características.

**Archivo**: `src/server/src/server/db/schema.ts` (agregar al final)

```typescript
// ========================================
// NUEVAS TABLAS PARA SISTEMA DE GRABACIÓN AUTOMÁTICA
// ========================================

/** PLANES DE SUSCRIPCIÓN */
export const plans = pgTable("plans", {
  id: serial("id").primaryKey(),
  name: varchar("name", { length: 100 }).notNull(),
  description: text("description"),
  maxSimultaneousRecordings: integer("max_simultaneous_recordings").notNull(), // 0 = ilimitado
  priceMonthly: integer("price_monthly").notNull(), // en centavos
  storageType: varchar("storage_type", { length: 50 }).$type<'azure' | 'vps' | 'both'>().notNull().default('azure'),
  features: json("features").$type<string[]>().notNull().default([]),
  isActive: boolean("is_active").notNull().default(true),
  createdAt: timestamp("created_at").defaultNow(),
  updatedAt: timestamp("updated_at").defaultNow(),
});

export const insertPlanSchema = createInsertSchema(plans);
export const selectPlanSchema = createSelectSchema(plans);

/** USUARIOS CON PLANES */
export const userPlans = pgTable("user_plans", {
  id: serial("id").primaryKey(),
  userId: uuid("user_id")
    .references(() => users.id, { onDelete: "cascade" })
    .notNull(),
  planId: integer("plan_id")
    .references(() => plans.id)
    .notNull(),
  startDate: timestamp("start_date").notNull().defaultNow(),
  endDate: timestamp("end_date"), // null = activo indefinidamente
  isActive: boolean("is_active").notNull().default(true),
  createdAt: timestamp("created_at").defaultNow(),
});

export const insertUserPlanSchema = createInsertSchema(userPlans);
export const selectUserPlanSchema = createSelectSchema(userPlans);

/** USUARIOS PARA GRABAR */
export const recordingTargets = pgTable("recording_targets", {
  id: serial("id").primaryKey(),
  tenantAdminId: uuid("tenant_admin_id")
    .references(() => users.id, { onDelete: "cascade" })
    .notNull(),
  targetEmail: varchar("target_email", { length: 255 }).notNull(),
  targetUserId: varchar("target_user_id", { length: 255 }), // Microsoft Graph User ID
  displayName: varchar("display_name", { length: 255 }),
  isActive: boolean("is_active").notNull().default(true),
  recordingEnabled: boolean("recording_enabled").notNull().default(true),
  hasAcceptedDisclaimer: boolean("has_accepted_disclaimer").notNull().default(false),
  disclaimerAcceptedAt: timestamp("disclaimer_accepted_at"),
  createdAt: timestamp("created_at").defaultNow(),
  updatedAt: timestamp("updated_at").defaultNow(),
});

export const insertRecordingTargetSchema = createInsertSchema(recordingTargets);
export const selectRecordingTargetSchema = createSelectSchema(recordingTargets);

/** CRITERIOS DE EXCLUSIÓN */
export const exclusionCriteriaTypes = z.enum([
  'subject_contains',        // Asunto contiene texto
  'subject_not_contains',    // Asunto NO contiene texto
  'participant_email',       // Email de participante específico
  'participant_domain',      // Dominio de participante
  'min_participants',        // Número mínimo de participantes
  'max_participants',        // Número máximo de participantes
  'min_duration_minutes',    // Duración mínima en minutos
  'max_duration_minutes',    // Duración máxima en minutos
  'time_of_day',            // Horario del día (rango)
  'day_of_week',            // Día de la semana
  'meeting_type',           // Tipo de reunión (1-on-1, group, conference)
  'is_recurring',           // Si es reunión recurrente
  'organizer_email',        // Email del organizador
  'custom_keyword',         // Palabra clave personalizada en metadata
]);
export type ExclusionCriteriaType = z.infer<typeof exclusionCriteriaTypes>;

export const exclusionCriteria = pgTable("exclusion_criteria", {
  id: serial("id").primaryKey(),
  recordingTargetId: integer("recording_target_id")
    .references(() => recordingTargets.id, { onDelete: "cascade" })
    .notNull(),
  criteriaType: varchar("criteria_type", { length: 100 })
    .$type<ExclusionCriteriaType>()
    .notNull(),
  criteriaValue: json("criteria_value").$type<any>().notNull(), // Valor del criterio (flexible)
  description: text("description"),
  isActive: boolean("is_active").notNull().default(true),
  priority: integer("priority").notNull().default(0), // Para ordenar evaluación
  createdAt: timestamp("created_at").defaultNow(),
  updatedAt: timestamp("updated_at").defaultNow(),
});

export const insertExclusionCriteriaSchema = createInsertSchema(exclusionCriteria);
export const selectExclusionCriteriaSchema = createSelectSchema(exclusionCriteria);

/** SUBSCRIPCIONES DE GRAPH API */
export const graphSubscriptions = pgTable("graph_subscriptions", {
  id: serial("id").primaryKey(),
  subscriptionId: varchar("subscription_id", { length: 255 }).notNull().unique(), // ID de Microsoft Graph
  userId: uuid("user_id")
    .references(() => users.id, { onDelete: "cascade" })
    .notNull(),
  targetEmail: varchar("target_email", { length: 255 }).notNull(),
  resource: varchar("resource", { length: 500 }).notNull(), // Resource path de Graph
  changeType: varchar("change_type", { length: 100 }).notNull(), // created,updated,deleted
  expirationDateTime: timestamp("expiration_date_time").notNull(),
  clientState: varchar("client_state", { length: 255 }).notNull(),
  isActive: boolean("is_active").notNull().default(true),
  createdAt: timestamp("created_at").defaultNow(),
  updatedAt: timestamp("updated_at").defaultNow(),
});

export const insertGraphSubscriptionSchema = createInsertSchema(graphSubscriptions);
export const selectGraphSubscriptionSchema = createSelectSchema(graphSubscriptions);

/** LOGS DE AUTO-JOIN */
export const autoJoinLogs = pgTable("auto_join_logs", {
  id: serial("id").primaryKey(),
  recordingTargetId: integer("recording_target_id")
    .references(() => recordingTargets.id, { onDelete: "set null" }),
  meetingId: varchar("meeting_id", { length: 255 }).notNull(),
  meetingSubject: varchar("meeting_subject", { length: 500 }),
  meetingStartTime: timestamp("meeting_start_time").notNull(),
  decision: varchar("decision", { length: 50 }).$type<'join' | 'skip'>().notNull(),
  reason: text("reason"), // Por qué se decidió join o skip
  exclusionCriteriaMatched: json("exclusion_criteria_matched").$type<number[]>(), // IDs de criterios que hicieron match
  botId: integer("bot_id").references(() => bots.id, { onDelete: "set null" }),
  createdAt: timestamp("created_at").defaultNow(),
});

export const insertAutoJoinLogSchema = createInsertSchema(autoJoinLogs);
export const selectAutoJoinLogSchema = createSelectSchema(autoJoinLogs);

/** CONFIGURACIÓN DE ALMACENAMIENTO */
export const storageConfig = pgTable("storage_config", {
  id: serial("id").primaryKey(),
  userId: uuid("user_id")
    .references(() => users.id, { onDelete: "cascade" })
    .notNull()
    .unique(),
  storageType: varchar("storage_type", { length: 50 }).$type<'azure' | 'vps'>().notNull().default('azure'),
  azureConfig: json("azure_config").$type<{
    containerName?: string;
    customEndpoint?: string;
  }>(),
  vpsConfig: json("vps_config").$type<{
    basePath?: string;
    maxStorageGB?: number;
  }>(),
  createdAt: timestamp("created_at").defaultNow(),
  updatedAt: timestamp("updated_at").defaultNow(),
});

export const insertStorageConfigSchema = createInsertSchema(storageConfig);
export const selectStorageConfigSchema = createSelectSchema(storageConfig);

/** METADATA DE GRABACIONES EXTENDIDA */
export const recordingMetadata = pgTable("recording_metadata", {
  id: serial("id").primaryKey(),
  botId: integer("bot_id")
    .references(() => bots.id, { onDelete: "cascade" })
    .notNull(),
  recordingTargetId: integer("recording_target_id")
    .references(() => recordingTargets.id, { onDelete: "set null" }),
  meetingId: varchar("meeting_id", { length: 255 }),
  meetingSubject: varchar("meeting_subject", { length: 500 }),
  organizer: varchar("organizer", { length: 255 }),
  participants: json("participants").$type<string[]>(),
  startTime: timestamp("start_time").notNull(),
  endTime: timestamp("end_time").notNull(),
  durationMinutes: integer("duration_minutes"),
  storageLocation: varchar("storage_location", { length: 1000 }), // Path o URL
  storageType: varchar("storage_type", { length: 50 }).$type<'azure' | 'vps'>(),
  fileSize: integer("file_size"), // en bytes
  transcriptionAvailable: boolean("transcription_available").default(false),
  transcriptionPath: varchar("transcription_path", { length: 1000 }),
  tags: json("tags").$type<string[]>(),
  isDeleted: boolean("is_deleted").default(false),
  deletedAt: timestamp("deleted_at"),
  createdAt: timestamp("created_at").defaultNow(),
});

export const insertRecordingMetadataSchema = createInsertSchema(recordingMetadata);
export const selectRecordingMetadataSchema = createSelectSchema(recordingMetadata);
```

### 2.3 Servicios Backend

#### 2.3.1 Microsoft Graph Service

**Archivo**: `src/server/src/server/api/services/graph.service.ts` (NUEVO)

```typescript
import { Client } from '@microsoft/microsoft-graph-client';
import { ClientSecretCredential } from '@azure/identity';
import { z } from 'zod';

// Esquema para eventos de calendario
const calendarEventSchema = z.object({
  id: z.string(),
  subject: z.string(),
  start: z.object({
    dateTime: z.string(),
    timeZone: z.string(),
  }),
  end: z.object({
    dateTime: z.string(),
    timeZone: z.string(),
  }),
  organizer: z.object({
    emailAddress: z.object({
      name: z.string().optional(),
      address: z.string(),
    }),
  }),
  attendees: z.array(z.object({
    type: z.string().optional(),
    status: z.object({
      response: z.string(),
      time: z.string(),
    }).optional(),
    emailAddress: z.object({
      name: z.string().optional(),
      address: z.string(),
    }),
  })).optional(),
  isOnlineMeeting: z.boolean().optional(),
  onlineMeeting: z.object({
    joinUrl: z.string(),
    conferenceId: z.string().optional(),
    tollNumber: z.string().optional(),
  }).optional(),
  recurrence: z.any().optional(),
});

export type CalendarEvent = z.infer<typeof calendarEventSchema>;

export class GraphService {
  private client: Client;
  private credential: ClientSecretCredential;

  constructor() {
    const clientId = process.env.AZURE_CLIENT_ID!;
    const tenantId = process.env.AZURE_TENANT_ID!;
    const clientSecret = process.env.AZURE_CLIENT_SECRET!;

    if (!clientId || !tenantId || !clientSecret) {
      throw new Error('Azure AD credentials not configured');
    }

    this.credential = new ClientSecretCredential(
      tenantId,
      clientId,
      clientSecret
    );

    this.client = Client.initWithMiddleware({
      authProvider: {
        getAccessToken: async () => {
          const token = await this.credential.getToken(
            'https://graph.microsoft.com/.default'
          );
          return token?.token || '';
        },
      },
    });
  }

  /**
   * Crear suscripción para eventos de calendario de un usuario
   */
  async createCalendarSubscription(
    userEmail: string,
    notificationUrl: string,
    clientState: string
  ) {
    const expirationDateTime = new Date();
    expirationDateTime.setHours(expirationDateTime.getHours() + 24); // 24 horas

    const subscription = {
      changeType: 'created,updated',
      notificationUrl,
      resource: `/users/${userEmail}/events`,
      expirationDateTime: expirationDateTime.toISOString(),
      clientState,
    };

    try {
      const result = await this.client
        .api('/subscriptions')
        .post(subscription);

      console.log('Created subscription:', result.id);
      return result;
    } catch (error: any) {
      console.error('Error creating subscription:', error);
      throw new Error(`Failed to create subscription: ${error.message}`);
    }
  }

  /**
   * Renovar suscripción existente
   */
  async renewSubscription(subscriptionId: string) {
    const expirationDateTime = new Date();
    expirationDateTime.setHours(expirationDateTime.getHours() + 24);

    try {
      const result = await this.client
        .api(`/subscriptions/${subscriptionId}`)
        .patch({
          expirationDateTime: expirationDateTime.toISOString(),
        });

      console.log('Renewed subscription:', subscriptionId);
      return result;
    } catch (error: any) {
      console.error('Error renewing subscription:', error);
      throw new Error(`Failed to renew subscription: ${error.message}`);
    }
  }

  /**
   * Eliminar suscripción
   */
  async deleteSubscription(subscriptionId: string) {
    try {
      await this.client
        .api(`/subscriptions/${subscriptionId}`)
        .delete();

      console.log('Deleted subscription:', subscriptionId);
    } catch (error: any) {
      console.error('Error deleting subscription:', error);
      throw new Error(`Failed to delete subscription: ${error.message}`);
    }
  }

  /**
   * Obtener evento de calendario por ID
   */
  async getCalendarEvent(userEmail: string, eventId: string): Promise<CalendarEvent> {
    try {
      const event = await this.client
        .api(`/users/${userEmail}/events/${eventId}`)
        .select('id,subject,start,end,organizer,attendees,isOnlineMeeting,onlineMeeting,recurrence')
        .get();

      return calendarEventSchema.parse(event);
    } catch (error: any) {
      console.error('Error getting calendar event:', error);
      throw new Error(`Failed to get calendar event: ${error.message}`);
    }
  }

  /**
   * Obtener reunión online por join URL
   */
  async getOnlineMeetingByJoinUrl(joinUrl: string) {
    try {
      // Extraer meeting ID de la URL
      const urlObj = new URL(joinUrl);
      const pathParts = urlObj.pathname.split('/');
      const meetingId = pathParts[pathParts.indexOf('meetup-join') + 1]?.split('/')[0];

      if (!meetingId) {
        throw new Error('Could not extract meeting ID from join URL');
      }

      return {
        meetingId,
        joinUrl,
      };
    } catch (error: any) {
      console.error('Error parsing join URL:', error);
      throw new Error(`Failed to parse join URL: ${error.message}`);
    }
  }

  /**
   * Obtener información de usuario por email
   */
  async getUserByEmail(email: string) {
    try {
      const user = await this.client
        .api(`/users/${email}`)
        .select('id,displayName,mail,userPrincipalName')
        .get();

      return user;
    } catch (error: any) {
      console.error('Error getting user:', error);
      throw new Error(`Failed to get user: ${error.message}`);
    }
  }

  /**
   * Obtener eventos de calendario para un rango de fechas
   */
  async getCalendarEvents(
    userEmail: string,
    startDateTime: Date,
    endDateTime: Date
  ): Promise<CalendarEvent[]> {
    try {
      const events = await this.client
        .api(`/users/${userEmail}/calendarview`)
        .query({
          startDateTime: startDateTime.toISOString(),
          endDateTime: endDateTime.toISOString(),
        })
        .select('id,subject,start,end,organizer,attendees,isOnlineMeeting,onlineMeeting,recurrence')
        .top(100)
        .get();

      return events.value.map((e: any) => calendarEventSchema.parse(e));
    } catch (error: any) {
      console.error('Error getting calendar events:', error);
      throw new Error(`Failed to get calendar events: ${error.message}`);
    }
  }
}

// Singleton instance
export const graphService = new GraphService();
```

#### 2.3.2 Exclusion Criteria Service

**Archivo**: `src/server/src/server/api/services/exclusion.service.ts` (NUEVO)

```typescript
import { CalendarEvent } from './graph.service';
import { db } from '~/server/db';
import { exclusionCriteria, recordingTargets } from '~/server/db/schema';
import { eq, and } from 'drizzle-orm';

export interface EvaluationResult {
  shouldRecord: boolean;
  reason: string;
  matchedCriteriaIds: number[];
}

export class ExclusionService {
  /**
   * Evaluar si una reunión debe ser grabada basándose en criterios de exclusión
   */
  async evaluateMeeting(
    recordingTargetId: number,
    event: CalendarEvent
  ): Promise<EvaluationResult> {
    // Obtener todos los criterios activos para este usuario
    const criteria = await db
      .select()
      .from(exclusionCriteria)
      .where(
        and(
          eq(exclusionCriteria.recordingTargetId, recordingTargetId),
          eq(exclusionCriteria.isActive, true)
        )
      )
      .orderBy(exclusionCriteria.priority);

    const matchedCriteria: number[] = [];
    let shouldExclude = false;
    let exclusionReason = '';

    for (const criterion of criteria) {
      const matches = this.evaluateCriterion(criterion, event);

      if (matches) {
        matchedCriteria.push(criterion.id);
        shouldExclude = true;
        exclusionReason = criterion.description || `Matched criterion: ${criterion.criteriaType}`;
        // Si un criterio hace match, excluimos la reunión
        break;
      }
    }

    return {
      shouldRecord: !shouldExclude,
      reason: shouldExclude ? exclusionReason : 'No exclusion criteria matched',
      matchedCriteriaIds: matchedCriteria,
    };
  }

  /**
   * Evaluar un criterio individual
   */
  private evaluateCriterion(
    criterion: typeof exclusionCriteria.$inferSelect,
    event: CalendarEvent
  ): boolean {
    const { criteriaType, criteriaValue } = criterion;

    try {
      switch (criteriaType) {
        case 'subject_contains':
          return this.evaluateSubjectContains(event.subject, criteriaValue);

        case 'subject_not_contains':
          return !this.evaluateSubjectContains(event.subject, criteriaValue);

        case 'participant_email':
          return this.evaluateParticipantEmail(event, criteriaValue);

        case 'participant_domain':
          return this.evaluateParticipantDomain(event, criteriaValue);

        case 'min_participants':
          return this.evaluateMinParticipants(event, criteriaValue);

        case 'max_participants':
          return this.evaluateMaxParticipants(event, criteriaValue);

        case 'min_duration_minutes':
          return this.evaluateMinDuration(event, criteriaValue);

        case 'max_duration_minutes':
          return this.evaluateMaxDuration(event, criteriaValue);

        case 'time_of_day':
          return this.evaluateTimeOfDay(event, criteriaValue);

        case 'day_of_week':
          return this.evaluateDayOfWeek(event, criteriaValue);

        case 'meeting_type':
          return this.evaluateMeetingType(event, criteriaValue);

        case 'is_recurring':
          return this.evaluateIsRecurring(event, criteriaValue);

        case 'organizer_email':
          return this.evaluateOrganizerEmail(event, criteriaValue);

        case 'custom_keyword':
          return this.evaluateCustomKeyword(event, criteriaValue);

        default:
          console.warn(`Unknown criteria type: ${criteriaType}`);
          return false;
      }
    } catch (error) {
      console.error(`Error evaluating criterion ${criterion.id}:`, error);
      return false;
    }
  }

  // ==================== EVALUADORES INDIVIDUALES ====================

  private evaluateSubjectContains(subject: string, value: any): boolean {
    const keywords = Array.isArray(value) ? value : [value];
    const lowerSubject = subject.toLowerCase();

    return keywords.some((keyword: string) =>
      lowerSubject.includes(keyword.toLowerCase())
    );
  }

  private evaluateParticipantEmail(event: CalendarEvent, value: any): boolean {
    const targetEmails = Array.isArray(value) ? value : [value];
    const attendeeEmails = event.attendees?.map(a => a.emailAddress.address.toLowerCase()) || [];

    return targetEmails.some((email: string) =>
      attendeeEmails.includes(email.toLowerCase())
    );
  }

  private evaluateParticipantDomain(event: CalendarEvent, value: any): boolean {
    const targetDomains = Array.isArray(value) ? value : [value];
    const attendeeEmails = event.attendees?.map(a => a.emailAddress.address) || [];

    return attendeeEmails.some(email => {
      const domain = email.split('@')[1]?.toLowerCase();
      return targetDomains.some((targetDomain: string) =>
        domain === targetDomain.toLowerCase()
      );
    });
  }

  private evaluateMinParticipants(event: CalendarEvent, value: any): boolean {
    const minCount = typeof value === 'object' ? value.count : value;
    const participantCount = (event.attendees?.length || 0) + 1; // +1 for organizer

    return participantCount < minCount; // Excluir si hay MENOS que el mínimo
  }

  private evaluateMaxParticipants(event: CalendarEvent, value: any): boolean {
    const maxCount = typeof value === 'object' ? value.count : value;
    const participantCount = (event.attendees?.length || 0) + 1;

    return participantCount > maxCount; // Excluir si hay MÁS que el máximo
  }

  private evaluateMinDuration(event: CalendarEvent, value: any): boolean {
    const minMinutes = typeof value === 'object' ? value.minutes : value;
    const duration = this.calculateDurationMinutes(event);

    return duration < minMinutes; // Excluir si es MÁS CORTA que el mínimo
  }

  private evaluateMaxDuration(event: CalendarEvent, value: any): boolean {
    const maxMinutes = typeof value === 'object' ? value.minutes : value;
    const duration = this.calculateDurationMinutes(event);

    return duration > maxMinutes; // Excluir si es MÁS LARGA que el máximo
  }

  private evaluateTimeOfDay(event: CalendarEvent, value: any): boolean {
    // value: { startHour: 18, endHour: 8 } - Excluir fuera de horas laborales
    const { startHour, endHour } = value;
    const startTime = new Date(event.start.dateTime);
    const hour = startTime.getHours();

    if (startHour < endHour) {
      // Normal range (e.g., 9-17)
      return hour < startHour || hour >= endHour;
    } else {
      // Overnight range (e.g., 18-8)
      return hour >= endHour && hour < startHour;
    }
  }

  private evaluateDayOfWeek(event: CalendarEvent, value: any): boolean {
    // value: [0, 6] - Excluir domingos (0) y sábados (6)
    const excludedDays = Array.isArray(value) ? value : [value];
    const startTime = new Date(event.start.dateTime);
    const dayOfWeek = startTime.getDay();

    return excludedDays.includes(dayOfWeek);
  }

  private evaluateMeetingType(event: CalendarEvent, value: any): boolean {
    // value: '1-on-1' | 'group' | 'conference'
    const participantCount = (event.attendees?.length || 0) + 1;
    const targetType = value;

    let actualType: string;
    if (participantCount === 2) {
      actualType = '1-on-1';
    } else if (participantCount <= 10) {
      actualType = 'group';
    } else {
      actualType = 'conference';
    }

    return actualType === targetType;
  }

  private evaluateIsRecurring(event: CalendarEvent, value: any): boolean {
    const shouldBeRecurring = value;
    const isRecurring = !!event.recurrence;

    return isRecurring === shouldBeRecurring;
  }

  private evaluateOrganizerEmail(event: CalendarEvent, value: any): boolean {
    const targetEmails = Array.isArray(value) ? value : [value];
    const organizerEmail = event.organizer.emailAddress.address.toLowerCase();

    return targetEmails.some((email: string) =>
      organizerEmail === email.toLowerCase()
    );
  }

  private evaluateCustomKeyword(event: CalendarEvent, value: any): boolean {
    // Buscar en subject y en metadata (si está disponible)
    const { keyword, location } = value;
    const searchText = location === 'subject' ? event.subject : '';

    return searchText.toLowerCase().includes(keyword.toLowerCase());
  }

  // ==================== HELPERS ====================

  private calculateDurationMinutes(event: CalendarEvent): number {
    const start = new Date(event.start.dateTime);
    const end = new Date(event.end.dateTime);
    return (end.getTime() - start.getTime()) / 1000 / 60;
  }
}

// Singleton instance
export const exclusionService = new ExclusionService();
```

#### 2.3.3 Storage Service (Almacenamiento Flexible)

**Archivo**: `src/server/src/server/api/services/storage.service.ts` (NUEVO)

```typescript
import { BlobServiceClient, ContainerClient } from '@azure/storage-blob';
import fs from 'fs/promises';
import path from 'path';
import { db } from '~/server/db';
import { storageConfig } from '~/server/db/schema';
import { eq } from 'drizzle-orm';

export type StorageType = 'azure' | 'vps';

export interface UploadResult {
  success: boolean;
  path: string;
  storageType: StorageType;
  fileSize?: number;
  error?: string;
}

export class StorageService {
  private azureBlobService?: BlobServiceClient;
  private vpsBasePath: string;

  constructor() {
    // Inicializar Azure Blob Storage
    const azureConnectionString = process.env.AZURE_STORAGE_CONNECTION_STRING;
    if (azureConnectionString) {
      this.azureBlobService = BlobServiceClient.fromConnectionString(
        azureConnectionString
      );
    }

    // Inicializar VPS Storage
    this.vpsBasePath = process.env.VPS_STORAGE_PATH || '/var/recordings';
  }

  /**
   * Subir archivo de grabación según configuración del usuario
   */
  async uploadRecording(
    userId: string,
    botId: number,
    filePath: string,
    fileName: string
  ): Promise<UploadResult> {
    // Obtener configuración de almacenamiento del usuario
    const config = await db
      .select()
      .from(storageConfig)
      .where(eq(storageConfig.userId, userId))
      .limit(1);

    const storageType: StorageType = config[0]?.storageType || 'azure';

    try {
      if (storageType === 'azure') {
        return await this.uploadToAzure(userId, botId, filePath, fileName, config[0]);
      } else {
        return await this.uploadToVPS(userId, botId, filePath, fileName, config[0]);
      }
    } catch (error: any) {
      console.error('Error uploading recording:', error);
      return {
        success: false,
        path: '',
        storageType,
        error: error.message,
      };
    }
  }

  /**
   * Subir a Azure Blob Storage
   */
  private async uploadToAzure(
    userId: string,
    botId: number,
    filePath: string,
    fileName: string,
    config: any
  ): Promise<UploadResult> {
    if (!this.azureBlobService) {
      throw new Error('Azure Blob Storage not configured');
    }

    const containerName = config?.azureConfig?.containerName || 'recordings';
    const containerClient = this.azureBlobService.getContainerClient(containerName);

    // Asegurar que el container existe
    await containerClient.createIfNotExists();

    // Crear path en blob: users/{userId}/bots/{botId}/{fileName}
    const blobName = `users/${userId}/bots/${botId}/${fileName}`;
    const blockBlobClient = containerClient.getBlockBlobClient(blobName);

    // Subir archivo
    const fileStats = await fs.stat(filePath);
    await blockBlobClient.uploadFile(filePath);

    console.log(`Uploaded to Azure Blob: ${blobName}`);

    return {
      success: true,
      path: blobName,
      storageType: 'azure',
      fileSize: fileStats.size,
    };
  }

  /**
   * Subir a VPS local storage
   */
  private async uploadToVPS(
    userId: string,
    botId: number,
    filePath: string,
    fileName: string,
    config: any
  ): Promise<UploadResult> {
    const basePath = config?.vpsConfig?.basePath || this.vpsBasePath;

    // Crear path: {basePath}/users/{userId}/bots/{botId}/{fileName}
    const userDir = path.join(basePath, 'users', userId, 'bots', botId.toString());
    await fs.mkdir(userDir, { recursive: true });

    const destinationPath = path.join(userDir, fileName);

    // Copiar archivo
    await fs.copyFile(filePath, destinationPath);

    const fileStats = await fs.stat(destinationPath);

    console.log(`Uploaded to VPS: ${destinationPath}`);

    return {
      success: true,
      path: destinationPath,
      storageType: 'vps',
      fileSize: fileStats.size,
    };
  }

  /**
   * Obtener URL de descarga (signed URL para Azure, path para VPS)
   */
  async getDownloadUrl(
    storagePath: string,
    storageType: StorageType
  ): Promise<string> {
    if (storageType === 'azure') {
      return await this.getAzureSasUrl(storagePath);
    } else {
      // Para VPS, retornar un endpoint de API que sirva el archivo
      return `/api/recordings/download?path=${encodeURIComponent(storagePath)}`;
    }
  }

  /**
   * Generar SAS URL para Azure Blob
   */
  private async getAzureSasUrl(blobName: string): Promise<string> {
    if (!this.azureBlobService) {
      throw new Error('Azure Blob Storage not configured');
    }

    const containerName = 'recordings';
    const containerClient = this.azureBlobService.getContainerClient(containerName);
    const blobClient = containerClient.getBlobClient(blobName);

    // Generar SAS token con expiración de 1 hora
    const sasUrl = await blobClient.generateSasUrl({
      expiresOn: new Date(Date.now() + 3600 * 1000), // 1 hora
      permissions: 'r', // read only
    });

    return sasUrl;
  }

  /**
   * Verificar espacio disponible en VPS
   */
  async checkVPSSpace(userId: string): Promise<{
    totalGB: number;
    usedGB: number;
    availableGB: number;
  }> {
    const config = await db
      .select()
      .from(storageConfig)
      .where(eq(storageConfig.userId, userId))
      .limit(1);

    const maxStorageGB = config[0]?.vpsConfig?.maxStorageGB || 100;
    const basePath = config[0]?.vpsConfig?.basePath || this.vpsBasePath;
    const userDir = path.join(basePath, 'users', userId);

    try {
      const usedBytes = await this.getDirectorySize(userDir);
      const usedGB = usedBytes / 1024 / 1024 / 1024;

      return {
        totalGB: maxStorageGB,
        usedGB: parseFloat(usedGB.toFixed(2)),
        availableGB: parseFloat((maxStorageGB - usedGB).toFixed(2)),
      };
    } catch (error) {
      return {
        totalGB: maxStorageGB,
        usedGB: 0,
        availableGB: maxStorageGB,
      };
    }
  }

  /**
   * Calcular tamaño de directorio recursivamente
   */
  private async getDirectorySize(dirPath: string): Promise<number> {
    try {
      const entries = await fs.readdir(dirPath, { withFileTypes: true });
      let size = 0;

      for (const entry of entries) {
        const fullPath = path.join(dirPath, entry.name);

        if (entry.isDirectory()) {
          size += await this.getDirectorySize(fullPath);
        } else {
          const stats = await fs.stat(fullPath);
          size += stats.size;
        }
      }

      return size;
    } catch (error) {
      return 0;
    }
  }

  /**
   * Eliminar grabación
   */
  async deleteRecording(
    storagePath: string,
    storageType: StorageType
  ): Promise<boolean> {
    try {
      if (storageType === 'azure') {
        if (!this.azureBlobService) {
          throw new Error('Azure Blob Storage not configured');
        }

        const containerClient = this.azureBlobService.getContainerClient('recordings');
        const blobClient = containerClient.getBlobClient(storagePath);
        await blobClient.delete();

        console.log(`Deleted from Azure: ${storagePath}`);
      } else {
        await fs.unlink(storagePath);
        console.log(`Deleted from VPS: ${storagePath}`);
      }

      return true;
    } catch (error: any) {
      console.error('Error deleting recording:', error);
      return false;
    }
  }
}

// Singleton instance
export const storageService = new StorageService();
```

**Continuará en el siguiente mensaje debido a la longitud...**

### 2.4 Rutas de API (tRPC Routers)

Ahora voy a documentar los routers de tRPC necesarios. Debido a la extensión del documento, voy a resumir las partes clave.

¿Deseas que continúe con:
1. Los routers de API completos (users, plans, exclusion, webhooks)
2. La lógica de auto-join
3. Los componentes del panel de administración
4. La configuración del VPS Ubuntu 22.04
5. Scripts de deployment y testing

¿O prefieres que me enfoque en alguna parte específica primero?
