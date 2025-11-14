# Componentes del Panel de Administración

Este documento contiene todos los componentes de React/Next.js para el panel de administración.

## 3.1 Página Principal de Administración

**Archivo**: `src/server/src/app/admin/page.tsx` (NUEVO)

```typescript
'use client';

import { useSession } from 'next-auth/react';
import { redirect } from 'next/navigation';
import Link from 'next/link';
import { Card, CardHeader, CardTitle, CardDescription, CardContent } from '~/components/ui/card';
import { Users, Shield, FileText, Settings, BarChart3, CheckCircle2 } from 'lucide-react';

export default function AdminPage() {
  const { data: session, status } = useSession();

  if (status === 'loading') {
    return <div>Loading...</div>;
  }

  if (!session) {
    redirect('/');
  }

  const adminSections = [
    {
      title: 'Usuarios para Grabar',
      description: 'Gestiona qué usuarios de Teams serán grabados automáticamente',
      icon: Users,
      href: '/admin/users',
      color: 'text-blue-500',
    },
    {
      title: 'Criterios de Exclusión',
      description: 'Configura reglas para excluir reuniones específicas de la grabación',
      icon: Shield,
      href: '/admin/exclusion',
      color: 'text-purple-500',
    },
    {
      title: 'Planes y Límites',
      description: 'Gestiona planes de suscripción y límites de grabaciones simultáneas',
      icon: BarChart3,
      href: '/admin/plans',
      color: 'text-green-500',
    },
    {
      title: 'Grabaciones',
      description: 'Visualiza y gestiona todas las grabaciones almacenadas',
      icon: FileText,
      href: '/admin/recordings',
      color: 'text-orange-500',
    },
    {
      title: 'Autorización de Grabación',
      description: 'Configura disclaimer y autorización para grabaciones sin aviso',
      icon: CheckCircle2,
      href: '/admin/authorization',
      color: 'text-red-500',
    },
    {
      title: 'Configuración',
      description: 'Configuración general del sistema de grabación',
      icon: Settings,
      href: '/admin/settings',
      color: 'text-gray-500',
    },
  ];

  return (
    <div className="container mx-auto py-10">
      <div className="mb-8">
        <h1 className="text-4xl font-bold mb-2">Panel de Administración</h1>
        <p className="text-gray-600">
          Gestiona el sistema de grabación automática de Microsoft Teams
        </p>
      </div>

      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
        {adminSections.map((section) => (
          <Link key={section.href} href={section.href}>
            <Card className="hover:shadow-lg transition-shadow cursor-pointer h-full">
              <CardHeader>
                <div className="flex items-center space-x-4">
                  <section.icon className={`h-8 w-8 ${section.color}`} />
                  <CardTitle>{section.title}</CardTitle>
                </div>
              </CardHeader>
              <CardContent>
                <CardDescription>{section.description}</CardDescription>
              </CardContent>
            </Card>
          </Link>
        ))}
      </div>
    </div>
  );
}
```

## 3.2 Gestión de Usuarios para Grabar

**Archivo**: `src/server/src/app/admin/users/page.tsx` (NUEVO)

```typescript
'use client';

import { useState } from 'react';
import { api } from '~/trpc/react';
import { Button } from '~/components/ui/button';
import { Input } from '~/components/ui/input';
import { Card, CardHeader, CardTitle, CardContent } from '~/components/ui/card';
import { DataTable } from '~/components/custom/DataTable';
import { Plus, Trash2, Edit, CheckCircle, XCircle } from 'lucide-react';
import {
  Dialog,
  DialogContent,
  DialogDescription,
  DialogFooter,
  DialogHeader,
  DialogTitle,
  DialogTrigger,
} from '~/components/ui/dialog';
import { Label } from '~/components/ui/label';
import { Switch } from '~/components/ui/switch';

export default function UsersPage() {
  const [isAddDialogOpen, setIsAddDialogOpen] = useState(false);
  const [newUserEmail, setNewUserEmail] = useState('');

  // Queries
  const { data: targets, isLoading, refetch } = api.users.getRecordingTargets.useQuery({});

  // Mutations
  const addTarget = api.users.addRecordingTarget.useMutation({
    onSuccess: () => {
      refetch();
      setIsAddDialogOpen(false);
      setNewUserEmail('');
    },
  });

  const updateTarget = api.users.updateRecordingTarget.useMutation({
    onSuccess: () => {
      refetch();
    },
  });

  const deleteTarget = api.users.deleteRecordingTarget.useMutation({
    onSuccess: () => {
      refetch();
    },
  });

  const handleAddUser = async () => {
    if (!newUserEmail) return;
    await addTarget.mutateAsync({ targetEmail: newUserEmail });
  };

  const handleToggleRecording = async (id: number, enabled: boolean) => {
    await updateTarget.mutateAsync({
      id,
      recordingEnabled: enabled,
    });
  };

  const handleDelete = async (id: number) => {
    if (confirm('¿Estás seguro de que deseas eliminar este usuario?')) {
      await deleteTarget.mutateAsync({ id });
    }
  };

  const columns = [
    {
      header: 'Email',
      accessorKey: 'targetEmail',
    },
    {
      header: 'Nombre',
      accessorKey: 'displayName',
    },
    {
      header: 'Estado',
      accessorKey: 'isActive',
      cell: ({ row }: any) => (
        <span className={row.original.isActive ? 'text-green-600' : 'text-gray-400'}>
          {row.original.isActive ? 'Activo' : 'Inactivo'}
        </span>
      ),
    },
    {
      header: 'Grabación',
      accessorKey: 'recordingEnabled',
      cell: ({ row }: any) => (
        <Switch
          checked={row.original.recordingEnabled}
          onCheckedChange={(enabled) => handleToggleRecording(row.original.id, enabled)}
        />
      ),
    },
    {
      header: 'Disclaimer',
      accessorKey: 'hasAcceptedDisclaimer',
      cell: ({ row }: any) =>
        row.original.hasAcceptedDisclaimer ? (
          <CheckCircle className="h-5 w-5 text-green-600" />
        ) : (
          <XCircle className="h-5 w-5 text-red-600" />
        ),
    },
    {
      header: 'Acciones',
      id: 'actions',
      cell: ({ row }: any) => (
        <div className="flex space-x-2">
          <Button
            variant="ghost"
            size="sm"
            onClick={() => {
              // TODO: Abrir modal de edición
            }}
          >
            <Edit className="h-4 w-4" />
          </Button>
          <Button
            variant="ghost"
            size="sm"
            onClick={() => handleDelete(row.original.id)}
          >
            <Trash2 className="h-4 w-4 text-red-600" />
          </Button>
        </div>
      ),
    },
  ];

  return (
    <div className="container mx-auto py-10">
      <Card>
        <CardHeader className="flex flex-row items-center justify-between">
          <div>
            <CardTitle>Usuarios para Grabar</CardTitle>
            <p className="text-sm text-gray-600 mt-1">
              Gestiona qué usuarios de Teams serán grabados automáticamente
            </p>
          </div>
          <Dialog open={isAddDialogOpen} onOpenChange={setIsAddDialogOpen}>
            <DialogTrigger asChild>
              <Button>
                <Plus className="mr-2 h-4 w-4" />
                Agregar Usuario
              </Button>
            </DialogTrigger>
            <DialogContent>
              <DialogHeader>
                <DialogTitle>Agregar Usuario para Grabar</DialogTitle>
                <DialogDescription>
                  Ingresa el email del usuario de Teams que deseas grabar automáticamente
                </DialogDescription>
              </DialogHeader>
              <div className="grid gap-4 py-4">
                <div className="grid grid-cols-4 items-center gap-4">
                  <Label htmlFor="email" className="text-right">
                    Email
                  </Label>
                  <Input
                    id="email"
                    type="email"
                    placeholder="usuario@tuempresa.com"
                    className="col-span-3"
                    value={newUserEmail}
                    onChange={(e) => setNewUserEmail(e.target.value)}
                  />
                </div>
              </div>
              <DialogFooter>
                <Button
                  onClick={handleAddUser}
                  disabled={addTarget.isPending || !newUserEmail}
                >
                  {addTarget.isPending ? 'Agregando...' : 'Agregar'}
                </Button>
              </DialogFooter>
            </DialogContent>
          </Dialog>
        </CardHeader>
        <CardContent>
          {isLoading ? (
            <div>Cargando...</div>
          ) : (
            <DataTable columns={columns} data={targets || []} />
          )}
        </CardContent>
      </Card>
    </div>
  );
}
```

## 3.3 Gestión de Criterios de Exclusión

**Archivo**: `src/server/src/app/admin/exclusion/page.tsx` (NUEVO)

```typescript
'use client';

import { useState } from 'react';
import { api } from '~/trpc/react';
import { Button } from '~/components/ui/button';
import { Card, CardHeader, CardTitle, CardContent } from '~/components/ui/card';
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from '~/components/ui/select';
import {
  Dialog,
  DialogContent,
  DialogDescription,
  DialogFooter,
  DialogHeader,
  DialogTitle,
  DialogTrigger,
} from '~/components/ui/dialog';
import { Label } from '~/components/ui/label';
import { Input } from '~/components/ui/input';
import { Textarea } from '~/components/ui/textarea';
import { Plus, Trash2, Edit2, Lightbulb } from 'lucide-react';

export default function ExclusionPage() {
  const [selectedUserId, setSelectedUserId] = useState<number | null>(null);
  const [isCreateDialogOpen, setIsCreateDialogOpen] = useState(false);
  const [isTemplatesDialogOpen, setIsTemplatesDialogOpen] = useState(false);

  // Form state
  const [criteriaType, setCriteriaType] = useState<string>('');
  const [criteriaValue, setCriteriaValue] = useState<string>('');
  const [description, setDescription] = useState<string>('');

  // Queries
  const { data: targets } = api.users.getRecordingTargets.useQuery({});
  const { data: criteria, refetch } = api.exclusion.getCriteria.useQuery(
    { recordingTargetId: selectedUserId! },
    { enabled: !!selectedUserId }
  );
  const { data: templates } = api.exclusion.getTemplates.useQuery({});

  // Mutations
  const createCriterion = api.exclusion.createCriterion.useMutation({
    onSuccess: () => {
      refetch();
      setIsCreateDialogOpen(false);
      resetForm();
    },
  });

  const deleteCriterion = api.exclusion.deleteCriterion.useMutation({
    onSuccess: () => {
      refetch();
    },
  });

  const resetForm = () => {
    setCriteriaType('');
    setCriteriaValue('');
    setDescription('');
  };

  const handleCreateCriterion = async () => {
    if (!selectedUserId || !criteriaType) return;

    let parsedValue: any = criteriaValue;

    // Parsear el valor según el tipo de criterio
    try {
      if (
        criteriaType === 'subject_contains' ||
        criteriaType === 'subject_not_contains' ||
        criteriaType === 'participant_email' ||
        criteriaType === 'participant_domain'
      ) {
        // Array de strings separados por coma
        parsedValue = criteriaValue.split(',').map((s) => s.trim());
      } else if (
        criteriaType === 'min_participants' ||
        criteriaType === 'max_participants' ||
        criteriaType === 'min_duration_minutes' ||
        criteriaType === 'max_duration_minutes'
      ) {
        // Número
        parsedValue = parseInt(criteriaValue, 10);
      } else if (criteriaType === 'day_of_week') {
        // Array de números
        parsedValue = criteriaValue.split(',').map((s) => parseInt(s.trim(), 10));
      } else if (criteriaType === 'time_of_day') {
        // Objeto { startHour, endHour }
        const [startHour, endHour] = criteriaValue.split(',').map((s) => parseInt(s.trim(), 10));
        parsedValue = { startHour, endHour };
      } else {
        // Intentar parsear como JSON, o usar como string
        try {
          parsedValue = JSON.parse(criteriaValue);
        } catch {
          parsedValue = criteriaValue;
        }
      }
    } catch (error) {
      alert('Error al parsear el valor del criterio');
      return;
    }

    await createCriterion.mutateAsync({
      recordingTargetId: selectedUserId,
      criteriaType: criteriaType as any,
      criteriaValue: parsedValue,
      description,
    });
  };

  const handleUseTemplate = (template: any) => {
    setCriteriaType(template.criteriaType);
    setCriteriaValue(JSON.stringify(template.exampleValue));
    setDescription(template.description);
    setIsTemplatesDialogOpen(false);
    setIsCreateDialogOpen(true);
  };

  const getCriteriaTypeLabel = (type: string): string => {
    const labels: Record<string, string> = {
      subject_contains: 'Asunto contiene',
      subject_not_contains: 'Asunto NO contiene',
      participant_email: 'Email de participante',
      participant_domain: 'Dominio de participante',
      min_participants: 'Mínimo de participantes',
      max_participants: 'Máximo de participantes',
      min_duration_minutes: 'Duración mínima (min)',
      max_duration_minutes: 'Duración máxima (min)',
      time_of_day: 'Horario del día',
      day_of_week: 'Día de la semana',
      meeting_type: 'Tipo de reunión',
      is_recurring: 'Es recurrente',
      organizer_email: 'Email del organizador',
      custom_keyword: 'Palabra clave personalizada',
    };
    return labels[type] || type;
  };

  return (
    <div className="container mx-auto py-10">
      <div className="grid grid-cols-1 lg:grid-cols-3 gap-6">
        {/* Panel izquierdo: Selección de usuario */}
        <Card className="lg:col-span-1">
          <CardHeader>
            <CardTitle>Seleccionar Usuario</CardTitle>
          </CardHeader>
          <CardContent>
            <Select
              value={selectedUserId?.toString() || ''}
              onValueChange={(value) => setSelectedUserId(parseInt(value, 10))}
            >
              <SelectTrigger>
                <SelectValue placeholder="Selecciona un usuario" />
              </SelectTrigger>
              <SelectContent>
                {targets?.map((target) => (
                  <SelectItem key={target.id} value={target.id.toString()}>
                    {target.displayName || target.targetEmail}
                  </SelectItem>
                ))}
              </SelectContent>
            </Select>
          </CardContent>
        </Card>

        {/* Panel derecho: Criterios de exclusión */}
        <Card className="lg:col-span-2">
          <CardHeader className="flex flex-row items-center justify-between">
            <div>
              <CardTitle>Criterios de Exclusión</CardTitle>
              <p className="text-sm text-gray-600 mt-1">
                {selectedUserId
                  ? 'Configura reglas para excluir reuniones específicas'
                  : 'Selecciona un usuario para ver sus criterios'}
              </p>
            </div>
            {selectedUserId && (
              <div className="flex space-x-2">
                <Dialog open={isTemplatesDialogOpen} onOpenChange={setIsTemplatesDialogOpen}>
                  <DialogTrigger asChild>
                    <Button variant="outline">
                      <Lightbulb className="mr-2 h-4 w-4" />
                      Plantillas
                    </Button>
                  </DialogTrigger>
                  <DialogContent className="max-w-2xl">
                    <DialogHeader>
                      <DialogTitle>Plantillas de Criterios</DialogTitle>
                      <DialogDescription>
                        Selecciona una plantilla para empezar rápidamente
                      </DialogDescription>
                    </DialogHeader>
                    <div className="grid gap-4">
                      {templates?.map((template, index) => (
                        <Card
                          key={index}
                          className="cursor-pointer hover:shadow-md transition-shadow"
                          onClick={() => handleUseTemplate(template)}
                        >
                          <CardHeader>
                            <CardTitle className="text-base">{template.name}</CardTitle>
                            <p className="text-sm text-gray-600">{template.description}</p>
                            <p className="text-xs text-gray-400 mt-2">
                              Tipo: {getCriteriaTypeLabel(template.criteriaType)}
                            </p>
                          </CardHeader>
                        </Card>
                      ))}
                    </div>
                  </DialogContent>
                </Dialog>

                <Dialog open={isCreateDialogOpen} onOpenChange={setIsCreateDialogOpen}>
                  <DialogTrigger asChild>
                    <Button>
                      <Plus className="mr-2 h-4 w-4" />
                      Nuevo Criterio
                    </Button>
                  </DialogTrigger>
                  <DialogContent>
                    <DialogHeader>
                      <DialogTitle>Crear Criterio de Exclusión</DialogTitle>
                      <DialogDescription>
                        Define una regla para excluir reuniones automáticamente
                      </DialogDescription>
                    </DialogHeader>
                    <div className="grid gap-4 py-4">
                      <div className="grid gap-2">
                        <Label>Tipo de Criterio</Label>
                        <Select value={criteriaType} onValueChange={setCriteriaType}>
                          <SelectTrigger>
                            <SelectValue placeholder="Selecciona un tipo" />
                          </SelectTrigger>
                          <SelectContent>
                            <SelectItem value="subject_contains">Asunto contiene</SelectItem>
                            <SelectItem value="subject_not_contains">Asunto NO contiene</SelectItem>
                            <SelectItem value="participant_email">Email de participante</SelectItem>
                            <SelectItem value="participant_domain">Dominio de participante</SelectItem>
                            <SelectItem value="min_participants">Mínimo de participantes</SelectItem>
                            <SelectItem value="max_participants">Máximo de participantes</SelectItem>
                            <SelectItem value="min_duration_minutes">Duración mínima (min)</SelectItem>
                            <SelectItem value="max_duration_minutes">Duración máxima (min)</SelectItem>
                            <SelectItem value="time_of_day">Horario del día</SelectItem>
                            <SelectItem value="day_of_week">Día de la semana</SelectItem>
                            <SelectItem value="meeting_type">Tipo de reunión</SelectItem>
                            <SelectItem value="organizer_email">Email del organizador</SelectItem>
                          </SelectContent>
                        </Select>
                      </div>

                      <div className="grid gap-2">
                        <Label>Valor</Label>
                        <Input
                          placeholder="Ej: confidencial, privado (separado por comas)"
                          value={criteriaValue}
                          onChange={(e) => setCriteriaValue(e.target.value)}
                        />
                        <p className="text-xs text-gray-500">
                          {criteriaType === 'subject_contains' && 'Palabras separadas por comas'}
                          {criteriaType === 'min_participants' && 'Número de participantes'}
                          {criteriaType === 'time_of_day' && 'Formato: horaInicio,horaFin (Ej: 18,9)'}
                          {criteriaType === 'day_of_week' && 'Días: 0=Domingo, 1=Lunes... (Ej: 0,6)'}
                        </p>
                      </div>

                      <div className="grid gap-2">
                        <Label>Descripción</Label>
                        <Textarea
                          placeholder="Descripción del criterio"
                          value={description}
                          onChange={(e) => setDescription(e.target.value)}
                        />
                      </div>
                    </div>
                    <DialogFooter>
                      <Button
                        onClick={handleCreateCriterion}
                        disabled={createCriterion.isPending || !criteriaType}
                      >
                        {createCriterion.isPending ? 'Creando...' : 'Crear'}
                      </Button>
                    </DialogFooter>
                  </DialogContent>
                </Dialog>
              </div>
            )}
          </CardHeader>
          <CardContent>
            {!selectedUserId ? (
              <div className="text-center text-gray-500 py-8">
                Selecciona un usuario para ver sus criterios de exclusión
              </div>
            ) : criteria?.length === 0 ? (
              <div className="text-center text-gray-500 py-8">
                No hay criterios configurados. Haz click en "Nuevo Criterio" para empezar.
              </div>
            ) : (
              <div className="space-y-4">
                {criteria?.map((criterion) => (
                  <Card key={criterion.id}>
                    <CardHeader className="flex flex-row items-center justify-between">
                      <div className="flex-1">
                        <div className="flex items-center space-x-2">
                          <span className="font-semibold">
                            {getCriteriaTypeLabel(criterion.criteriaType)}
                          </span>
                          <span
                            className={`px-2 py-1 rounded text-xs ${
                              criterion.isActive
                                ? 'bg-green-100 text-green-800'
                                : 'bg-gray-100 text-gray-800'
                            }`}
                          >
                            {criterion.isActive ? 'Activo' : 'Inactivo'}
                          </span>
                        </div>
                        <p className="text-sm text-gray-600 mt-1">
                          {criterion.description || 'Sin descripción'}
                        </p>
                        <p className="text-xs text-gray-400 mt-1">
                          Valor: {JSON.stringify(criterion.criteriaValue)}
                        </p>
                      </div>
                      <div className="flex space-x-2">
                        <Button
                          variant="ghost"
                          size="sm"
                          onClick={() => deleteCriterion.mutate({ id: criterion.id })}
                        >
                          <Trash2 className="h-4 w-4 text-red-600" />
                        </Button>
                      </div>
                    </CardHeader>
                  </Card>
                ))}
              </div>
            )}
          </CardContent>
        </Card>
      </div>
    </div>
  );
}
```

## 3.4 Gestión de Planes

**Archivo**: `src/server/src/app/admin/plans/page.tsx` (NUEVO)

```typescript
'use client';

import { api } from '~/trpc/react';
import { Card, CardHeader, CardTitle, CardContent } from '~/components/ui/card';
import { Button } from '~/components/ui/button';
import { CheckCircle2, XCircle, Crown, Zap, Rocket } from 'lucide-react';

export default function PlansPage() {
  const { data: currentPlan } = api.plans.getMyPlan.useQuery({});
  const { data: allPlans } = api.plans.getAllPlans.useQuery({});
  const subscribeToPlan = api.plans.subscribeToPlan.useMutation();

  const handleSubscribe = async (planId: number) => {
    if (confirm('¿Estás seguro de que deseas suscribirte a este plan?')) {
      await subscribeToPlan.mutateAsync({ planId });
      window.location.reload();
    }
  };

  const getPlanIcon = (planName: string) => {
    if (planName.toLowerCase().includes('basic')) return Crown;
    if (planName.toLowerCase().includes('pro')) return Zap;
    if (planName.toLowerCase().includes('enterprise')) return Rocket;
    return CheckCircle2;
  };

  return (
    <div className="container mx-auto py-10">
      {/* Plan actual */}
      {currentPlan?.plan && (
        <Card className="mb-8 bg-gradient-to-r from-blue-50 to-purple-50">
          <CardHeader>
            <CardTitle>Tu Plan Actual</CardTitle>
          </CardHeader>
          <CardContent>
            <div className="flex justify-between items-center">
              <div>
                <h3 className="text-2xl font-bold">{currentPlan.plan.name}</h3>
                <p className="text-gray-600">{currentPlan.plan.description}</p>
              </div>
              <div className="text-right">
                <p className="text-3xl font-bold">
                  ${(currentPlan.plan.priceMonthly / 100).toFixed(2)}
                  <span className="text-lg text-gray-600">/mes</span>
                </p>
                <p className="text-sm text-gray-600">
                  Grabaciones activas: {currentPlan.currentUsage.activeRecordings} /{' '}
                  {currentPlan.currentUsage.maxAllowed === 0
                    ? '∞'
                    : currentPlan.currentUsage.maxAllowed}
                </p>
              </div>
            </div>
          </CardContent>
        </Card>
      )}

      {/* Todos los planes disponibles */}
      <h2 className="text-2xl font-bold mb-6">Planes Disponibles</h2>
      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
        {allPlans?.map((plan) => {
          const Icon = getPlanIcon(plan.name);
          const isCurrentPlan = currentPlan?.plan?.id === plan.id;

          return (
            <Card
              key={plan.id}
              className={`${
                isCurrentPlan ? 'border-2 border-blue-500 shadow-lg' : ''
              } relative`}
            >
              {isCurrentPlan && (
                <div className="absolute top-0 right-0 bg-blue-500 text-white px-3 py-1 rounded-bl-lg text-sm">
                  Plan Actual
                </div>
              )}
              <CardHeader>
                <Icon className="h-12 w-12 mb-4 text-blue-500" />
                <CardTitle>{plan.name}</CardTitle>
                <p className="text-gray-600">{plan.description}</p>
              </CardHeader>
              <CardContent>
                <div className="mb-6">
                  <p className="text-4xl font-bold">
                    ${(plan.priceMonthly / 100).toFixed(2)}
                    <span className="text-lg text-gray-600">/mes</span>
                  </p>
                </div>

                <div className="space-y-3 mb-6">
                  <div className="flex items-center space-x-2">
                    <CheckCircle2 className="h-5 w-5 text-green-500" />
                    <span>
                      {plan.maxSimultaneousRecordings === 0
                        ? 'Grabaciones simultáneas ilimitadas'
                        : `Hasta ${plan.maxSimultaneousRecordings} grabaciones simultáneas`}
                    </span>
                  </div>

                  <div className="flex items-center space-x-2">
                    <CheckCircle2 className="h-5 w-5 text-green-500" />
                    <span>
                      Almacenamiento:{' '}
                      {plan.storageType === 'both'
                        ? 'Azure + VPS'
                        : plan.storageType === 'azure'
                        ? 'Azure Blob'
                        : 'VPS Local'}
                    </span>
                  </div>

                  {plan.features?.map((feature, index) => (
                    <div key={index} className="flex items-center space-x-2">
                      <CheckCircle2 className="h-5 w-5 text-green-500" />
                      <span>{feature}</span>
                    </div>
                  ))}
                </div>

                {!isCurrentPlan && (
                  <Button
                    className="w-full"
                    onClick={() => handleSubscribe(plan.id)}
                    disabled={subscribeToPlan.isPending}
                  >
                    {subscribeToPlan.isPending ? 'Procesando...' : 'Suscribirse'}
                  </Button>
                )}
              </CardContent>
            </Card>
          );
        })}
      </div>
    </div>
  );
}
```

## Resumen de Componentes

Los componentes del panel de administración incluyen:

1. **Dashboard principal** (`/admin/page.tsx`): Vista general con acceso a todas las secciones
2. **Gestión de usuarios** (`/admin/users/page.tsx`): CRUD de usuarios para grabar
3. **Criterios de exclusión** (`/admin/exclusion/page.tsx`): Configuración de reglas con plantillas
4. **Gestión de planes** (`/admin/plans/page.tsx`): Visualización y suscripción a planes

### Próximos pasos

1. Crear página de grabaciones (`/admin/recordings/page.tsx`)
2. Crear página de autorización (`/admin/authorization/page.tsx`)
3. Implementar configuración de almacenamiento
4. Agregar analytics y reportes
