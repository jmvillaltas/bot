# Routers de API (tRPC)

Este documento contiene todos los routers de tRPC necesarios para el sistema.

## 2.4.1 Users Router (Gestión de Usuarios para Grabar)

**Archivo**: `src/server/src/server/api/routers/users.ts` (NUEVO)

```typescript
import { z } from "zod";
import { createTRPCRouter, protectedProcedure } from "~/server/api/trpc";
import {
  recordingTargets,
  insertRecordingTargetSchema,
  selectRecordingTargetSchema,
} from "../../db/schema";
import { eq, and } from "drizzle-orm";
import { graphService } from "../services/graph.service";
import { TRPCError } from "@trpc/server";

export const usersRouter = createTRPCRouter({
  /**
   * Obtener todos los usuarios configurados para grabar
   */
  getRecordingTargets: protectedProcedure
    .input(z.object({}))
    .output(z.array(selectRecordingTargetSchema))
    .query(async ({ ctx }) => {
      return await ctx.db
        .select()
        .from(recordingTargets)
        .where(eq(recordingTargets.tenantAdminId, ctx.session.user.id))
        .orderBy(recordingTargets.createdAt);
    }),

  /**
   * Agregar usuario para grabar
   */
  addRecordingTarget: protectedProcedure
    .input(
      z.object({
        targetEmail: z.string().email(),
      })
    )
    .output(selectRecordingTargetSchema)
    .mutation(async ({ input, ctx }) => {
      // Verificar que el usuario existe en Microsoft 365
      try {
        const user = await graphService.getUserByEmail(input.targetEmail);

        // Crear registro en la base de datos
        const result = await ctx.db
          .insert(recordingTargets)
          .values({
            tenantAdminId: ctx.session.user.id,
            targetEmail: input.targetEmail,
            targetUserId: user.id,
            displayName: user.displayName,
            isActive: true,
            recordingEnabled: true,
            hasAcceptedDisclaimer: false,
          })
          .returning();

        if (!result[0]) {
          throw new TRPCError({
            code: "INTERNAL_SERVER_ERROR",
            message: "Failed to create recording target",
          });
        }

        // Crear suscripción de Graph API para eventos de calendario
        const notificationUrl = `${process.env.NEXT_PUBLIC_APP_URL}/api/webhooks/graph`;
        const clientState = `user_${result[0].id}_${Date.now()}`;

        const subscription = await graphService.createCalendarSubscription(
          input.targetEmail,
          notificationUrl,
          clientState
        );

        // Guardar suscripción en la base de datos
        await ctx.db.insert(graphSubscriptions).values({
          subscriptionId: subscription.id,
          userId: ctx.session.user.id,
          targetEmail: input.targetEmail,
          resource: `/users/${input.targetEmail}/events`,
          changeType: "created,updated",
          expirationDateTime: new Date(subscription.expirationDateTime),
          clientState,
          isActive: true,
        });

        return result[0];
      } catch (error: any) {
        throw new TRPCError({
          code: "BAD_REQUEST",
          message: `Failed to add recording target: ${error.message}`,
        });
      }
    }),

  /**
   * Actualizar configuración de usuario
   */
  updateRecordingTarget: protectedProcedure
    .input(
      z.object({
        id: z.number(),
        recordingEnabled: z.boolean().optional(),
        hasAcceptedDisclaimer: z.boolean().optional(),
      })
    )
    .output(selectRecordingTargetSchema)
    .mutation(async ({ input, ctx }) => {
      const { id, ...updateData } = input;

      // Verificar que el usuario pertenece al admin actual
      const existing = await ctx.db
        .select()
        .from(recordingTargets)
        .where(eq(recordingTargets.id, id))
        .limit(1);

      if (!existing[0] || existing[0].tenantAdminId !== ctx.session.user.id) {
        throw new TRPCError({
          code: "NOT_FOUND",
          message: "Recording target not found",
        });
      }

      const result = await ctx.db
        .update(recordingTargets)
        .set({
          ...updateData,
          disclaimerAcceptedAt: input.hasAcceptedDisclaimer
            ? new Date()
            : existing[0].disclaimerAcceptedAt,
          updatedAt: new Date(),
        })
        .where(eq(recordingTargets.id, id))
        .returning();

      if (!result[0]) {
        throw new TRPCError({
          code: "INTERNAL_SERVER_ERROR",
          message: "Failed to update recording target",
        });
      }

      return result[0];
    }),

  /**
   * Eliminar usuario de la lista de grabación
   */
  deleteRecordingTarget: protectedProcedure
    .input(z.object({ id: z.number() }))
    .output(z.object({ message: z.string() }))
    .mutation(async ({ input, ctx }) => {
      // Verificar que el usuario pertenece al admin actual
      const existing = await ctx.db
        .select()
        .from(recordingTargets)
        .where(eq(recordingTargets.id, input.id))
        .limit(1);

      if (!existing[0] || existing[0].tenantAdminId !== ctx.session.user.id) {
        throw new TRPCError({
          code: "NOT_FOUND",
          message: "Recording target not found",
        });
      }

      // Eliminar suscripciones de Graph asociadas
      const subscriptions = await ctx.db
        .select()
        .from(graphSubscriptions)
        .where(
          and(
            eq(graphSubscriptions.targetEmail, existing[0].targetEmail),
            eq(graphSubscriptions.userId, ctx.session.user.id)
          )
        );

      for (const sub of subscriptions) {
        try {
          await graphService.deleteSubscription(sub.subscriptionId);
        } catch (error) {
          console.error(`Failed to delete subscription ${sub.subscriptionId}:`, error);
        }
      }

      // Eliminar el registro
      await ctx.db
        .delete(recordingTargets)
        .where(eq(recordingTargets.id, input.id));

      return { message: "Recording target deleted successfully" };
    }),

  /**
   * Bulk update - habilitar/deshabilitar grabación para múltiples usuarios
   */
  bulkUpdateRecording: protectedProcedure
    .input(
      z.object({
        ids: z.array(z.number()),
        recordingEnabled: z.boolean(),
      })
    )
    .output(z.object({ count: z.number() }))
    .mutation(async ({ input, ctx }) => {
      const result = await ctx.db
        .update(recordingTargets)
        .set({
          recordingEnabled: input.recordingEnabled,
          updatedAt: new Date(),
        })
        .where(
          and(
            inArray(recordingTargets.id, input.ids),
            eq(recordingTargets.tenantAdminId, ctx.session.user.id)
          )
        )
        .returning();

      return { count: result.length };
    }),
});
```

## 2.4.2 Plans Router (Gestión de Planes)

**Archivo**: `src/server/src/server/api/routers/plans.ts` (NUEVO)

```typescript
import { z } from "zod";
import { createTRPCRouter, protectedProcedure, publicProcedure } from "~/server/api/trpc";
import {
  plans,
  userPlans,
  insertPlanSchema,
  selectPlanSchema,
  insertUserPlanSchema,
  selectUserPlanSchema,
  bots,
  status,
} from "../../db/schema";
import { eq, and, notInArray, sql } from "drizzle-orm";
import { TRPCError } from "@trpc/server";

export const plansRouter = createTRPCRouter({
  /**
   * Obtener todos los planes disponibles
   */
  getAllPlans: publicProcedure
    .input(z.object({}))
    .output(z.array(selectPlanSchema))
    .query(async ({ ctx }) => {
      return await ctx.db
        .select()
        .from(plans)
        .where(eq(plans.isActive, true))
        .orderBy(plans.priceMonthly);
    }),

  /**
   * Crear nuevo plan (solo admins)
   */
  createPlan: protectedProcedure
    .input(insertPlanSchema)
    .output(selectPlanSchema)
    .mutation(async ({ input, ctx }) => {
      // TODO: Verificar que el usuario es admin del sistema

      const result = await ctx.db.insert(plans).values(input).returning();

      if (!result[0]) {
        throw new TRPCError({
          code: "INTERNAL_SERVER_ERROR",
          message: "Failed to create plan",
        });
      }

      return result[0];
    }),

  /**
   * Obtener plan actual del usuario
   */
  getMyPlan: protectedProcedure
    .input(z.object({}))
    .output(
      z.object({
        plan: selectPlanSchema.nullable(),
        userPlan: selectUserPlanSchema.nullable(),
        currentUsage: z.object({
          activeRecordings: z.number(),
          maxAllowed: z.number(),
          canStartNew: z.boolean(),
        }),
      })
    )
    .query(async ({ ctx }) => {
      // Obtener plan activo del usuario
      const userPlanResult = await ctx.db
        .select()
        .from(userPlans)
        .where(
          and(
            eq(userPlans.userId, ctx.session.user.id),
            eq(userPlans.isActive, true)
          )
        )
        .limit(1);

      if (!userPlanResult[0]) {
        return {
          plan: null,
          userPlan: null,
          currentUsage: {
            activeRecordings: 0,
            maxAllowed: 0,
            canStartNew: false,
          },
        };
      }

      const planResult = await ctx.db
        .select()
        .from(plans)
        .where(eq(plans.id, userPlanResult[0].planId))
        .limit(1);

      // Contar grabaciones activas
      const activeRecordingsResult = await ctx.db
        .select({ count: sql<number>`count(*)` })
        .from(bots)
        .where(
          and(
            eq(bots.userId, ctx.session.user.id),
            notInArray(bots.status, ["DONE", "FATAL"] as const)
          )
        );

      const activeRecordings = Number(activeRecordingsResult[0]?.count || 0);
      const maxAllowed = planResult[0]?.maxSimultaneousRecordings || 0;
      const canStartNew = maxAllowed === 0 || activeRecordings < maxAllowed;

      return {
        plan: planResult[0] || null,
        userPlan: userPlanResult[0],
        currentUsage: {
          activeRecordings,
          maxAllowed,
          canStartNew,
        },
      };
    }),

  /**
   * Suscribirse a un plan
   */
  subscribeToPlan: protectedProcedure
    .input(
      z.object({
        planId: z.number(),
      })
    )
    .output(selectUserPlanSchema)
    .mutation(async ({ input, ctx }) => {
      // Verificar que el plan existe y está activo
      const plan = await ctx.db
        .select()
        .from(plans)
        .where(and(eq(plans.id, input.planId), eq(plans.isActive, true)))
        .limit(1);

      if (!plan[0]) {
        throw new TRPCError({
          code: "NOT_FOUND",
          message: "Plan not found or inactive",
        });
      }

      // Desactivar planes anteriores
      await ctx.db
        .update(userPlans)
        .set({ isActive: false })
        .where(
          and(
            eq(userPlans.userId, ctx.session.user.id),
            eq(userPlans.isActive, true)
          )
        );

      // Crear nueva suscripción
      const result = await ctx.db
        .insert(userPlans)
        .values({
          userId: ctx.session.user.id,
          planId: input.planId,
          startDate: new Date(),
          isActive: true,
        })
        .returning();

      if (!result[0]) {
        throw new TRPCError({
          code: "INTERNAL_SERVER_ERROR",
          message: "Failed to subscribe to plan",
        });
      }

      return result[0];
    }),

  /**
   * Cancelar suscripción
   */
  cancelSubscription: protectedProcedure
    .input(z.object({}))
    .output(z.object({ message: z.string() }))
    .mutation(async ({ ctx }) => {
      await ctx.db
        .update(userPlans)
        .set({
          isActive: false,
          endDate: new Date(),
        })
        .where(
          and(
            eq(userPlans.userId, ctx.session.user.id),
            eq(userPlans.isActive, true)
          )
        );

      return { message: "Subscription cancelled successfully" };
    }),

  /**
   * Verificar si el usuario puede iniciar una nueva grabación
   */
  canStartRecording: protectedProcedure
    .input(z.object({}))
    .output(
      z.object({
        canStart: z.boolean(),
        reason: z.string().optional(),
        currentUsage: z.number(),
        maxAllowed: z.number(),
      })
    )
    .query(async ({ ctx }) => {
      // Obtener plan activo
      const userPlanResult = await ctx.db
        .select()
        .from(userPlans)
        .where(
          and(
            eq(userPlans.userId, ctx.session.user.id),
            eq(userPlans.isActive, true)
          )
        )
        .limit(1);

      if (!userPlanResult[0]) {
        return {
          canStart: false,
          reason: "No active plan",
          currentUsage: 0,
          maxAllowed: 0,
        };
      }

      const planResult = await ctx.db
        .select()
        .from(plans)
        .where(eq(plans.id, userPlanResult[0].planId))
        .limit(1);

      if (!planResult[0]) {
        return {
          canStart: false,
          reason: "Plan not found",
          currentUsage: 0,
          maxAllowed: 0,
        };
      }

      // Contar grabaciones activas
      const activeRecordingsResult = await ctx.db
        .select({ count: sql<number>`count(*)` })
        .from(bots)
        .where(
          and(
            eq(bots.userId, ctx.session.user.id),
            notInArray(bots.status, ["DONE", "FATAL"] as const)
          )
        );

      const currentUsage = Number(activeRecordingsResult[0]?.count || 0);
      const maxAllowed = planResult[0].maxSimultaneousRecordings;

      // maxAllowed === 0 significa ilimitado
      const canStart = maxAllowed === 0 || currentUsage < maxAllowed;

      return {
        canStart,
        reason: canStart ? undefined : "Maximum simultaneous recordings reached",
        currentUsage,
        maxAllowed,
      };
    }),
});
```

## 2.4.3 Exclusion Router (Criterios de Exclusión)

**Archivo**: `src/server/src/server/api/routers/exclusion.ts` (NUEVO)

```typescript
import { z } from "zod";
import { createTRPCRouter, protectedProcedure } from "~/server/api/trpc";
import {
  exclusionCriteria,
  insertExclusionCriteriaSchema,
  selectExclusionCriteriaSchema,
  exclusionCriteriaTypes,
  recordingTargets,
} from "../../db/schema";
import { eq, and } from "drizzle-orm";
import { TRPCError } from "@trpc/server";

export const exclusionRouter = createTRPCRouter({
  /**
   * Obtener criterios de exclusión para un usuario
   */
  getCriteria: protectedProcedure
    .input(z.object({ recordingTargetId: z.number() }))
    .output(z.array(selectExclusionCriteriaSchema))
    .query(async ({ input, ctx }) => {
      // Verificar que el recording target pertenece al usuario actual
      const target = await ctx.db
        .select()
        .from(recordingTargets)
        .where(eq(recordingTargets.id, input.recordingTargetId))
        .limit(1);

      if (!target[0] || target[0].tenantAdminId !== ctx.session.user.id) {
        throw new TRPCError({
          code: "FORBIDDEN",
          message: "Access denied",
        });
      }

      return await ctx.db
        .select()
        .from(exclusionCriteria)
        .where(eq(exclusionCriteria.recordingTargetId, input.recordingTargetId))
        .orderBy(exclusionCriteria.priority);
    }),

  /**
   * Crear nuevo criterio de exclusión
   */
  createCriterion: protectedProcedure
    .input(
      z.object({
        recordingTargetId: z.number(),
        criteriaType: exclusionCriteriaTypes,
        criteriaValue: z.any(),
        description: z.string().optional(),
        priority: z.number().optional(),
      })
    )
    .output(selectExclusionCriteriaSchema)
    .mutation(async ({ input, ctx }) => {
      // Verificar acceso
      const target = await ctx.db
        .select()
        .from(recordingTargets)
        .where(eq(recordingTargets.id, input.recordingTargetId))
        .limit(1);

      if (!target[0] || target[0].tenantAdminId !== ctx.session.user.id) {
        throw new TRPCError({
          code: "FORBIDDEN",
          message: "Access denied",
        });
      }

      const result = await ctx.db
        .insert(exclusionCriteria)
        .values({
          recordingTargetId: input.recordingTargetId,
          criteriaType: input.criteriaType,
          criteriaValue: input.criteriaValue,
          description: input.description,
          priority: input.priority || 0,
          isActive: true,
        })
        .returning();

      if (!result[0]) {
        throw new TRPCError({
          code: "INTERNAL_SERVER_ERROR",
          message: "Failed to create criterion",
        });
      }

      return result[0];
    }),

  /**
   * Actualizar criterio
   */
  updateCriterion: protectedProcedure
    .input(
      z.object({
        id: z.number(),
        criteriaValue: z.any().optional(),
        description: z.string().optional(),
        priority: z.number().optional(),
        isActive: z.boolean().optional(),
      })
    )
    .output(selectExclusionCriteriaSchema)
    .mutation(async ({ input, ctx }) => {
      const { id, ...updateData } = input;

      // Verificar acceso
      const criterion = await ctx.db
        .select()
        .from(exclusionCriteria)
        .leftJoin(
          recordingTargets,
          eq(exclusionCriteria.recordingTargetId, recordingTargets.id)
        )
        .where(eq(exclusionCriteria.id, id))
        .limit(1);

      if (
        !criterion[0] ||
        criterion[0].recording_targets?.tenantAdminId !== ctx.session.user.id
      ) {
        throw new TRPCError({
          code: "FORBIDDEN",
          message: "Access denied",
        });
      }

      const result = await ctx.db
        .update(exclusionCriteria)
        .set({
          ...updateData,
          updatedAt: new Date(),
        })
        .where(eq(exclusionCriteria.id, id))
        .returning();

      if (!result[0]) {
        throw new TRPCError({
          code: "INTERNAL_SERVER_ERROR",
          message: "Failed to update criterion",
        });
      }

      return result[0];
    }),

  /**
   * Eliminar criterio
   */
  deleteCriterion: protectedProcedure
    .input(z.object({ id: z.number() }))
    .output(z.object({ message: z.string() }))
    .mutation(async ({ input, ctx }) => {
      // Verificar acceso
      const criterion = await ctx.db
        .select()
        .from(exclusionCriteria)
        .leftJoin(
          recordingTargets,
          eq(exclusionCriteria.recordingTargetId, recordingTargets.id)
        )
        .where(eq(exclusionCriteria.id, input.id))
        .limit(1);

      if (
        !criterion[0] ||
        criterion[0].recording_targets?.tenantAdminId !== ctx.session.user.id
      ) {
        throw new TRPCError({
          code: "FORBIDDEN",
          message: "Access denied",
        });
      }

      await ctx.db
        .delete(exclusionCriteria)
        .where(eq(exclusionCriteria.id, input.id));

      return { message: "Criterion deleted successfully" };
    }),

  /**
   * Plantillas de criterios comunes
   */
  getTemplates: protectedProcedure
    .input(z.object({}))
    .output(
      z.array(
        z.object({
          name: z.string(),
          description: z.string(),
          criteriaType: exclusionCriteriaTypes,
          exampleValue: z.any(),
        })
      )
    )
    .query(async () => {
      return [
        {
          name: "Excluir reuniones confidenciales",
          description: "No grabar si el asunto contiene 'confidencial' o 'privado'",
          criteriaType: "subject_contains" as const,
          exampleValue: ["confidencial", "privado", "personal"],
        },
        {
          name: "Solo reuniones de equipo",
          description: "No grabar reuniones 1-a-1",
          criteriaType: "meeting_type" as const,
          exampleValue: "1-on-1",
        },
        {
          name: "Horario laboral",
          description: "No grabar fuera de 9AM - 6PM",
          criteriaType: "time_of_day" as const,
          exampleValue: {
            startHour: 18, // 6PM
            endHour: 9, // 9AM
          },
        },
        {
          name: "Reuniones cortas",
          description: "No grabar reuniones de menos de 15 minutos",
          criteriaType: "min_duration_minutes" as const,
          exampleValue: 15,
        },
        {
          name: "Dominio externo",
          description: "No grabar si hay participantes de dominios externos",
          criteriaType: "participant_domain" as const,
          exampleValue: ["external.com", "competitor.com"],
        },
        {
          name: "Días laborales",
          description: "No grabar fines de semana",
          criteriaType: "day_of_week" as const,
          exampleValue: [0, 6], // Domingo y Sábado
        },
      ];
    }),
});
```

## 2.4.4 Webhooks Router (Microsoft Graph Notifications)

**Archivo**: `src/server/src/app/api/webhooks/graph/route.ts` (NUEVO)

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { graphService } from '~/server/api/services/graph.service';
import { exclusionService } from '~/server/api/services/exclusion.service';
import { db } from '~/server/db';
import {
  graphSubscriptions,
  recordingTargets,
  autoJoinLogs,
  bots,
  insertBotSchema,
} from '~/server/db/schema';
import { eq, and } from 'drizzle-orm';

/**
 * Webhook endpoint para Microsoft Graph notifications
 *
 * Microsoft Graph envía notificaciones cuando hay cambios en calendarios
 */
export async function POST(request: NextRequest) {
  try {
    const body = await request.json();

    // Validación de Microsoft Graph
    // Cuando creas una suscripción, Graph envía un validationToken
    if (request.nextUrl.searchParams.has('validationToken')) {
      const validationToken = request.nextUrl.searchParams.get('validationToken');
      return new NextResponse(validationToken, {
        status: 200,
        headers: { 'Content-Type': 'text/plain' },
      });
    }

    // Procesar notificaciones
    const notifications = body.value || [];

    for (const notification of notifications) {
      await processNotification(notification);
    }

    return NextResponse.json({ success: true });
  } catch (error: any) {
    console.error('Webhook error:', error);
    return NextResponse.json(
      { error: error.message },
      { status: 500 }
    );
  }
}

async function processNotification(notification: any) {
  const { subscriptionId, clientState, resource, changeType } = notification;

  console.log('Processing notification:', {
    subscriptionId,
    changeType,
    resource,
  });

  // Obtener suscripción de la base de datos
  const subscription = await db
    .select()
    .from(graphSubscriptions)
    .where(eq(graphSubscriptions.subscriptionId, subscriptionId))
    .limit(1);

  if (!subscription[0]) {
    console.warn('Subscription not found:', subscriptionId);
    return;
  }

  // Verificar clientState para seguridad
  if (subscription[0].clientState !== clientState) {
    console.warn('Invalid clientState');
    return;
  }

  // Solo procesar eventos created y updated
  if (!['created', 'updated'].includes(changeType)) {
    return;
  }

  // Obtener recording target
  const target = await db
    .select()
    .from(recordingTargets)
    .where(
      and(
        eq(recordingTargets.targetEmail, subscription[0].targetEmail),
        eq(recordingTargets.isActive, true),
        eq(recordingTargets.recordingEnabled, true),
        eq(recordingTargets.hasAcceptedDisclaimer, true)
      )
    )
    .limit(1);

  if (!target[0]) {
    console.log('Recording target not active or not configured');
    return;
  }

  // Extraer event ID del resource
  // resource format: /users/{userId}/events/{eventId}
  const eventId = resource.split('/').pop();

  if (!eventId) {
    console.warn('Could not extract event ID from resource');
    return;
  }

  // Obtener detalles del evento
  const event = await graphService.getCalendarEvent(
    subscription[0].targetEmail,
    eventId
  );

  // Verificar si es una reunión online
  if (!event.isOnlineMeeting || !event.onlineMeeting) {
    console.log('Not an online meeting, skipping');
    return;
  }

  // Evaluar criterios de exclusión
  const evaluation = await exclusionService.evaluateMeeting(target[0].id, event);

  // Guardar log de decisión
  await db.insert(autoJoinLogs).values({
    recordingTargetId: target[0].id,
    meetingId: event.id,
    meetingSubject: event.subject,
    meetingStartTime: new Date(event.start.dateTime),
    decision: evaluation.shouldRecord ? 'join' : 'skip',
    reason: evaluation.reason,
    exclusionCriteriaMatched: evaluation.matchedCriteriaIds,
  });

  if (!evaluation.shouldRecord) {
    console.log('Meeting excluded based on criteria:', evaluation.reason);
    return;
  }

  // Verificar límites del plan del usuario
  const canStart = await checkPlanLimits(subscription[0].userId);

  if (!canStart) {
    console.log('User has reached plan limits');

    await db.insert(autoJoinLogs).values({
      recordingTargetId: target[0].id,
      meetingId: event.id,
      meetingSubject: event.subject,
      meetingStartTime: new Date(event.start.dateTime),
      decision: 'skip',
      reason: 'Plan limit reached',
      exclusionCriteriaMatched: [],
    });

    return;
  }

  // Crear bot para unirse a la reunión
  await createBotForMeeting(subscription[0].userId, target[0], event);
}

async function checkPlanLimits(userId: string): Promise<boolean> {
  // Implementar lógica de verificación de planes
  // Por ahora, retornar true
  // TODO: Implementar verificación real contra tabla userPlans
  return true;
}

async function createBotForMeeting(
  userId: string,
  target: typeof recordingTargets.$inferSelect,
  event: any
) {
  try {
    // Extraer información de Teams meeting
    const meetingInfo = await graphService.getOnlineMeetingByJoinUrl(
      event.onlineMeeting.joinUrl
    );

    // Calcular tiempo de inicio (5 minutos antes de la reunión)
    const startTime = new Date(event.start.dateTime);
    startTime.setMinutes(startTime.getMinutes() - 5);

    // Calcular tiempo de fin
    const endTime = new Date(event.end.dateTime);

    // Crear bot
    const botResult = await db.insert(bots).values({
      userId,
      botDisplayName: 'Meeting Recorder',
      meetingTitle: event.subject,
      meetingInfo: {
        meetingId: meetingInfo.meetingId,
        meetingUrl: event.onlineMeeting.joinUrl,
        platform: 'teams',
      },
      startTime,
      endTime,
      heartbeatInterval: 5000,
      automaticLeave: {
        waitingRoomTimeout: 300000, // 5 minutos
        noOneJoinedTimeout: 300000,
        everyoneLeftTimeout: 300000,
        inactivityTimeout: 300000,
      },
      status: 'READY_TO_DEPLOY',
    }).returning();

    console.log('Created bot for meeting:', {
      botId: botResult[0]?.id,
      meetingSubject: event.subject,
    });

    // Actualizar log de auto-join con bot ID
    await db
      .update(autoJoinLogs)
      .set({ botId: botResult[0]?.id })
      .where(
        and(
          eq(autoJoinLogs.meetingId, event.id),
          eq(autoJoinLogs.recordingTargetId, target.id)
        )
      );

    // TODO: Deployer el bot automáticamente si la reunión empieza pronto
    // Esto se puede hacer llamando a la función deployBot del servicio botDeployment

  } catch (error: any) {
    console.error('Error creating bot for meeting:', error);

    await db.insert(autoJoinLogs).values({
      recordingTargetId: target.id,
      meetingId: event.id,
      meetingSubject: event.subject,
      meetingStartTime: new Date(event.start.dateTime),
      decision: 'skip',
      reason: `Error: ${error.message}`,
      exclusionCriteriaMatched: [],
    });
  }
}

// GET method para health check
export async function GET() {
  return NextResponse.json({ status: 'ok' });
}
```

## Resumen

Estos routers proporcionan:

1. **users.ts**: Gestión de usuarios para grabar con Graph API subscriptions
2. **plans.ts**: Sistema de planes con límites de grabaciones simultáneas
3. **exclusion.ts**: CRUD completo de criterios de exclusión con plantillas
4. **webhooks/graph**: Auto-join automático basado en eventos de calendario

### Próximos pasos

- Implementar componentes de UI del panel de administración
- Crear lógica de auto-deployment de bots
- Configurar renovación automática de subscripciones de Graph
- Implementar sistema de notificaciones
