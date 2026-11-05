# 📦 Improvement – **Container Info & Network Tab (per Container & Service)**
**Proyecto:** Algora – Coolify UI  
**Autor:** @mau16ro (Senior Developer)  
**Versión:** 1.0.0 – 2026‑11‑05  
**Recompensa:** $10 USD  

---

## 1. Resumen ejecutivo

| **Objetivo** | Añadir una vista detallada de *Información del contenedor* y una pestaña **Network** en la UI de Algora, disponible tanto para **contenedores individuales** como para **servicios (stacks)**. |
|--------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Beneficios** | • Visibilidad instantánea de IP, puertos expuestos, redes Docker y estadísticas de tráfico. <br>• Mejora la experiencia de diagnóstico (sin necesidad de usar la CLI). <br>• Uniformiza la información entre contenedor y servicio, reduciendo la curva de aprendizaje. |
| **Alcance** | - Extensión del **backend** (Node / Express + Docker Engine API) para exponer datos de red. <br>- Nueva **página React** `ContainerInfoPage` con pestaña **Network**. <br>- Compatibilidad con **Docker‑Compose** (servicios) y **stand‑alone containers**. |
| **Entregables** | 1. Arquitectura y diagrama de flujo. <br>2. Código fuente completo (backend + frontend) con tipado TypeScript. <br>3. Tests unitarios (Jest + Supertest) y pruebas de integración. <br>4. Guía de instalación y despliegue. |
| **Restricciones** | - Mantener compatibilidad con la versión actual de Algora (Node v20, React 18, Docker Engine ≥ 24). <br>- No romper la API pública existente. |
| **Criterios de aceptación** | 1. La página muestra los datos de red del contenedor/servicio. <br>2. La pestaña **Network** está presente y funciona en todos los navegadores modernos. <br>3. Los endpoints `/api/containers/:id/info` y `/api/services/:id/info` retornan JSON con la estructura esperada. <br>4. Cobertura de pruebas ≥ 90 %. |

---

## 2. Arquitectura de la solución

### 2.1. Visión global

```text
+-------------------+        +---------------------+        +-------------------+
|   Frontend (SPA)  |  <---> |   API Gateway (NGINX) | <---> |   Backend (Node) |
|  React + TS       |        |  (auth, rate‑limit)  |        |  Express + TS    |
+-------------------+        +---------------------+        +-------------------+
                                   ^            |
                                   | Docker API |
                                   v            |
                              +-------------------+
                              | Docker Engine Daemon |
                              +-------------------+
```

- **Backend**: Añade dos nuevos controladores (`containerInfo.controller.ts`, `serviceInfo.controller.ts`) que consumen la **Docker Engine API** (`/containers/{id}/json`, `/networks`, `/services/{id}`) y transforman la respuesta a un DTO normalizado.
- **Frontend**: Nuevo componente **`ContainerInfoPage`** con **Tabs** (`Details`, `Network`, `Logs`). La pestaña *Network* reutiliza una tabla genérica `DataTable` y un gráfico de *traffic* (opcional, basado en `chart.js`).
- **Modelado**: DTOs `ContainerInfoDTO`, `NetworkInfoDTO`, `ServiceInfoDTO` con **TypeScript interfaces** y **Zod** para validación en runtime.
- **Testing**: *Backend* – Jest + Supertest + nock (mock Docker API). *Frontend* – React Testing Library + MSW (Mock Service Worker). *E2E* opcional con Cypress (no incluido en este paquete, pero preparado).

### 2.2. Cambios estructurales

| Área | Archivo / módulo nuevo | Descripción |
|------|------------------------|-------------|
| **Backend** | `src/controllers/containerInfo.controller.ts` | Endpoints `GET /api/containers/:id/info` |
| | `src/controllers/serviceInfo.controller.ts` | Endpoints `GET /api/services/:id/info` |
| | `src/dto/containerInfo.dto.ts` | Interfaces + Zod schema |
| | `src/services/docker.service.ts` (extensión) | Funciones auxiliares `getContainerNetworkInfo`, `getServiceNetworkInfo` |
| | `src/routes/container.routes.ts` | Registro de la nueva ruta |
| | `src/routes/service.routes.ts` | Registro de la nueva ruta |
| **Frontend** | `src/pages/ContainerInfoPage.tsx` | Layout con Tabs (MUI) |
| | `src/components/NetworkTab.tsx` | Tabla de redes y puertos |
| | `src/api/containerApi.ts` | Wrapper `fetchContainerInfo(id)` |
| | `src/api/serviceApi.ts` | Wrapper `fetchServiceInfo(id)` |
| | `src/types/container.ts` | Interfaces tipadas |
| | `src/hooks/useContainerInfo.ts` | Hook con SWR para data fetching |
| **Testing** | `tests/backend/containerInfo.test.ts` | Unit + Integration |
| | `tests/frontend/ContainerInfoPage.test.tsx` | Render + Interaction |
| | `tests/frontend/NetworkTab.test.tsx` | Table content validation |
| **Docs** | `README.md` (sección “Network Tab”) | Guía de instalación y uso |

### 2.3. Diagrama de flujo de datos

```mermaid
flowchart TD
    A[Usuario → UI] --> B[React Router → /containers/:id/info]
    B --> C[useContainerInfo Hook]
    C --> D[fetchContainerInfo (fetch API)]
    D --> E[Backend Express Route /api/containers/:id/info]
    E --> F[Docker Service (dockerode / HTTP API)]
    F --> G[Docker Daemon – Inspect Container]
    G --> H[Transform → ContainerInfoDTO]
    H --> E
    E --> I[Respuesta JSON]
    I --> C
    C --> J[Render NetworkTab (DataTable)]
```

---

## 3. Código fuente / Implementación completa

> **Nota:** Todo el código está en **TypeScript** y sigue las convenciones del proyecto (ESLint, Prettier, strict `tsconfig.json`).  
> Los snippets están organizados por capas; al copiar los archivos, mantenga la estructura de carpetas indicada.

### 3.1. Backend – Docker Service (extensión)

```ts
// src/services/docker.service.ts
import Docker from 'dockerode';
import { z } from 'zod';
import { ContainerInfoDTO, NetworkInfoDTO, ServiceInfoDTO } from '../dto/containerInfo.dto';

const docker = new Docker({ socketPath: '/var/run/docker.sock' });

/**
 * Obtiene información completa de un contenedor, incluyendo datos de red.
 */
export async function getContainerInfo(containerId: string): Promise<ContainerInfoDTO> {
  const container = docker.getContainer(containerId);
  const raw = await container.inspect();

  // Transformación a DTO tipado
  const networkInfo: NetworkInfoDTO[] = Object.entries(raw.NetworkSettings.Networks).map(
    ([networkName, net]) => ({
      networkName,
      ipAddress: net.IPAddress,
      ipPrefixLen: net.IPPrefixLen,
      gateway: net.Gateway,
      macAddress: net.MacAddress,
      driverOpts: net.DriverOpts,
    })
  );

  const dto: ContainerInfoDTO = {
    Id: raw.Id,
    Name: raw.Name,
    Image: raw.Config.Image,
    Created: raw.Created,
    State: raw.State.Status,
    RestartCount: raw.RestartCount,
    Ports: raw.NetworkSettings.Ports,
    Networks: networkInfo,
    Labels: raw.Config.Labels,
  };

  // Validación en tiempo de ejecución (opcional pero recomendado)
  return ContainerInfoDTO.parse(dto);
}

/**
 * Obtiene información de red de un servicio (docker‑compose stack).
 */
export async function getServiceInfo(serviceId: string): Promise<ServiceInfoDTO> {
  const service = docker.getService(serviceId);
  const raw = await service.inspect();

  // Los servicios pueden tener múltiples tareas (containers) – tomamos la primera task activa.
  const tasks = await docker.listTasks({ filters: { service: [serviceId], desired_state: ['running'] } });
  const task = tasks[0];
  if (!task) {
    throw new Error('No running task found for this service');
  }

  const container = docker.getContainer(task.Status.ContainerStatus.ContainerID);
  const containerInfo = await container.inspect();

  const networkInfo: NetworkInfoDTO[] = Object.entries(containerInfo.NetworkSettings.Networks).map(
    ([networkName, net]) => ({
      networkName,
      ipAddress: net.IPAddress,
      ipPrefixLen: net.IPPrefixLen,
      gateway: net.Gateway,
      macAddress: net.MacAddress,
      driverOpts: net.DriverOpts,
    })
  );

  const dto: ServiceInfoDTO = {
    Id: raw.ID,
    Name: raw.Spec.Name,
    Image: raw.Spec.TaskTemplate.ContainerSpec.Image,
    CreatedAt: raw.CreatedAt,
    UpdatedAt: raw.UpdatedAt,
    Replicas: raw.Spec.Mode.Replicated?.Replicas ?? 1,
    Ports: raw.Endpoint.Ports?.map(p => ({
      Protocol: p.Protocol,
      TargetPort: p.TargetPort,
      PublishedPort: p.PublishedPort,
    })) ?? [],
    Networks: networkInfo,
    Labels: raw.Spec.Labels,
  };

  return ServiceInfoDTO.parse(dto);
}
```

### 3.2. DTOs y validación

```ts
// src/dto/containerInfo.dto.ts
import { z } from 'zod';

/** Información de una única red asociada al contenedor */
export const NetworkInfoDTO = z.object({
  networkName: z.string(),
  ipAddress: z.string().optional(),
  ipPrefixLen: z.number().optional(),
  gateway: z.string().optional(),
  macAddress: z.string().optional(),
  driverOpts: z.record(z.string()).optional(),
});
export type NetworkInfoDTO = z.infer<typeof NetworkInfoDTO>;

/** Información completa de un contenedor */
export const ContainerInfoDTO = z.object({
  Id: z.string(),
  Name: z.string(),
  Image: z.string(),
  Created: z.string(),
  State: z.string(),
  RestartCount: z.number(),
  Ports: z.record(z.array(z.object({
    HostIp: z.string(),
    HostPort: z.string(),
  }).nullable())),
  Networks: z.array(NetworkInfoDTO),
  Labels: z.record(z.string()).optional(),
});
export type ContainerInfoDTO = z.infer<typeof ContainerInfoDTO>;

/** Información de puertos del servicio */
export const ServicePortDTO = z.object({
  Protocol: z.string(),
  TargetPort: z.number(),
  PublishedPort: z.number(),
});
export type ServicePortDTO = z.infer<typeof ServicePortDTO>;

/** Información completa de un servicio (stack) */
export const ServiceInfoDTO = z.object({
  Id: z.string(),
  Name: z.string(),
  Image: z.string(),
  CreatedAt: z.string(),
  UpdatedAt: z.string(),
  Replicas: z.number(),
  Ports: z.array(ServicePortDTO),
  Networks: z.array(NetworkInfoDTO),
  Labels: z.record(z.string()).optional(),
});
export type ServiceInfoDTO = z.infer<typeof ServiceInfoDTO>;
```

### 3.3. Controladores

```ts
// src/controllers/containerInfo.controller.ts
import { Request, Response, NextFunction } from 'express';
import { getContainerInfo } from '../services/docker.service';

export async function containerInfoHandler(
  req: Request,
  res: Response,
  next: NextFunction
) {
  try {
    const { id } = req.params;
    const data = await getContainerInfo(id);
    res.json({ success: true, data });
  } catch (err) {
    next(err);
  }
}
```

```ts
// src/controllers/serviceInfo.controller.ts
import { Request, Response, NextFunction } from 'express';
import { getServiceInfo } from '../services/docker.service';

export async function serviceInfoHandler(
  req: Request,
  res: Response,
  next: NextFunction
) {
  try {
    const { id } = req.params;
    const data = await getServiceInfo(id);
    res.json({ success: true, data });
  } catch (err) {
    next(err);
  }
}
```

### 3.4. Rutas

```ts
// src/routes/container.routes.ts
import { Router } from 'express';
import { containerInfoHandler } from '../controllers/containerInfo.controller';

const router = Router();

router.get('/:id/info', containerInfoHandler);

export default router;
```

```ts
// src/routes/service.routes.ts
import { Router } from 'express';
import { serviceInfoHandler } from '../controllers/serviceInfo.controller';

const router = Router();

router.get('/:id/info', serviceInfoHandler);

export default router;
```

> **Integración en `src/app.ts` (Express)**

```ts
import containerRoutes from './routes/container.routes';
import serviceRoutes from './routes/service.routes';

// ... (middlewares, auth, etc.)

