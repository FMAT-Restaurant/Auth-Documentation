# Prompt de Integración Oficial para Equipos (Microfrontends + API Gateway + PBAC)

Copia y pega el siguiente bloque directamente en el agente de IA de tu equipo (Cursor, Antigravity, Claude Code, Copilot, Windsurf, etc.) dentro del repositorio de tu microservicio (**Frontend** y/o **Backend**):

---

## Prompt Maestro para el Agente de IA

````markdown
# PROMPT PARA AGENTE IA: Integración Oficial con Auth Shell (Module Federation) y API Gateway Kong (PBAC)

Actúa como un Ingeniero de Software Senior experto en TypeScript, React 18, Vite, Node/NestJS y arquitecturas de Microfrontends y Microservicios.
Necesito que audites, configures y adaptes este repositorio para integrarse sin errores al ecosistema **FMAT-Restaurant**:
1. Como un **Microfrontend Remoto (Remote)** consumido por el Shell principal (`Auth-Frontend`) usando `@originjs/vite-plugin-federation`.
2. Como un **Backend seguro** detrás del **API Gateway (Kong)** consumiendo headers confiables y aplicando el catálogo PBAC de `Auth-Backend`.

Primero, inspecciona este repositorio para identificar a cuál de los 6 módulos pertenece y usa **estrictamente** los valores de su fila en esta tabla oficial de contratos:

### Matriz Maestra de Contratos por Microservicio

| Módulo / Dominio | `name` en Federation | Puerto Frontend (`server` y `preview`) | Ruta en API Gateway (`http://localhost:8000`) | `strip_path` en Kong | Puerto interno Backend | Rol de Sistema en Auth |
| :--- | :--- | :---: | :--- | :---: | :---: | :--- |
| **Sala / Host** | `salaApp` | `5174` | `/api/v1/floor` | `false` (recibe `/api/v1/floor/...`) | `8085` | `HOST` |
| **Menú** | `menuApp` | `5173` | `/api/v1/menu` | `false` (recibe `/api/v1/menu/...`) | `8080` | `GESTOR_MENU` |
| **Inventario** | `inventarioApp` | `5176` | `/api/v1/inventory` | `true` (recibe `/items`, `/batches`, etc.) | `8083` | `ALMACENISTA` |
| **Órdenes (POS)** | `ordenesApp` | `5177` | `/api/v1/orders` | `false` (recibe `/api/v1/orders/...`) | `8082` | `MESERO` |
| **Cocina (KDS)** | `cocinaApp` | `5179` | `/api/v1/kitchen` (o `/api/v1/orders/kds`) | `false` (recibe `/api/v1/kitchen/...`) | `8086` | `CHEF_MASTER` |
| **Caja y Facturación** | `cajaApp` | `5178` | `/api/v1/payments` | `false` (recibe `/api/v1/payments/...`) | `8084` | `CAJERO` |

---

## 1. CONTRATO DE FRONTEND (VITE MODULE FEDERATION)

Si este repositorio contiene código Frontend, realiza las siguientes adecuaciones:

### 1.1 Requisito Crítico: Usar React 18 (¡NO React 19!)
El Shell (`Auth-Frontend`) corre en **React 18** (`^18.3.1`). Como Module Federation comparte la instancia de React en memoria (`shared: ['react', 'react-dom']`), si este microfrontend usa React 19 romperá el Shell con el error: `Objects are not valid as a React child`.
Asegura que `package.json` tenga:
```bash
npm install react@^18.3.1 react-dom@^18.3.1
npm install -D @types/react@^18.3.18 @types/react-dom@^18.3.5
```

### 1.2 Configuración en `vite.config.ts`
Instala `@originjs/vite-plugin-federation`:
```bash
npm install -D @originjs/vite-plugin-federation
```
Configura `vite.config.ts` para exponer `./App` en `assets/remoteEntry.js` con el puerto asignado en la Matriz Maestra y escucha en `0.0.0.0`:
```typescript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import federation from '@originjs/vite-plugin-federation';

export default defineConfig({
  plugins: [
    react(),
    federation({
      name: '<NAME_SEGUN_MATRIZ>', // ej. 'menuApp', 'inventarioApp', 'salaApp', 'ordenesApp', 'cocinaApp', 'cajaApp'
      filename: 'remoteEntry.js',
      exposes: {
        './App': './src/App.tsx', // Debe tener export default del componente raíz
      },
      shared: ['react', 'react-dom'],
    }),
  ],
  server: {
    port: <PUERTO_SEGUN_MATRIZ>,
    strictPort: true,
    cors: true,
    host: '0.0.0.0',
  },
  preview: {
    port: <PUERTO_SEGUN_MATRIZ>,
    strictPort: true,
    cors: true,
    host: '0.0.0.0',
  },
  build: {
    target: 'esnext',
    minify: false,
    cssCodeSplit: false,
  },
});
```
> **Nota obligatoria:** `./src/App.tsx` DEBE tener `export default App`.

### 1.3 Props de Sesión y Fallback a `localStorage`
El Shell monta tu `<App />` pasándole props de usuario y token. Además, guarda la sesión activa en `localStorage` bajo `'fmat_restaurant_auth_session'`.
Declara los tipos y el soporte en `./src/App.tsx`:

```typescript
export interface ShellUser {
  id: string;
  userType: 'ADMIN' | 'STAFF';
  staffId?: string;
  email?: string;
  displayName?: string;
  firstName?: string;
  lastName?: string;
  roles?: string[];      // ej. ['ADMINISTRADOR'] o ['MESERO']
  permissions: string[]; // ej. ['*'] o ['menu:entries:view']
}

export interface FederatedModuleProps {
  token?: string;
  currentUser?: ShellUser;
  theme?: 'light' | 'dark';
}
```

Implementa la extracción del token y usuario activo:
```tsx
const getStoredSession = (): { accessToken?: string; user?: ShellUser } | null => {
  try {
    const raw = localStorage.getItem('fmat_restaurant_auth_session');
    return raw ? JSON.parse(raw) : null;
  } catch {
    return null;
  }
};

export default function App({ token, currentUser, theme }: FederatedModuleProps) {
  const activeToken = token ?? getStoredSession()?.accessToken;
  const activeUser = currentUser ?? getStoredSession()?.user;

  // Usa activeToken en todas tus peticiones HTTP hacia el API Gateway (Kong):
  // headers: { Authorization: `Bearer ${activeToken}` }
  // ...
}
```

### 1.4 Helper de Permisos en la UI (`hasPermission`)
Usa esta función para proteger botones, menús o vistas. Otorga acceso total inmediato a `ADMIN`, `ADMINISTRADOR` o `*`:
```typescript
export const hasPermission = (user: ShellUser | undefined, requiredPermission: string): boolean => {
  if (!user) return false;
  if (
    user.userType === 'ADMIN' ||
    user.roles?.includes('ADMINISTRADOR') ||
    user.roles?.includes('ADMIN') ||
    user.permissions?.includes('*')
  ) {
    return true;
  }
  return Boolean(user.permissions?.includes(requiredPermission));
};
```

### 1.5 Soporte de Tema Oscuro y Layout del Shell
1. **Modo Oscuro:** El Shell conmuta el atributo `<html data-theme="dark">`. Agrega en tu `src/index.css`:
   ```css
   [data-theme="dark"] {
     --color-primary: #E06A3B;
     --color-primary-hover: #F07C48;
     --color-accent: #FB923C;
     --color-primary-soft: rgba(224, 106, 59, 0.16);
     --color-ink: #FFFFFF;
     --color-text: #E8E3DF;
     --color-muted: #9C9288;
     --color-border: #2C2420;
     --color-canvas: #0A0908;
     --color-surface: #161311;
     --color-success: #22C55E;
     --bg-success: rgba(34, 197, 94, 0.15);
     --color-warning: #F59E0B;
     --bg-warning: rgba(245, 158, 11, 0.15);
     --color-danger: #EF4444;
     --bg-danger: rgba(239, 68, 68, 0.15);
   }
   ```
2. **Evitar Sidebars Duplicados:** El Shell ya cuenta con Sidebar de navegación global y header. Si tu frontend tiene un sidebar propio de todo el restaurante o pantalla de login, ocúltalo cuando esté montado en modo federado.

### 1.6 ¡Advertencia Crítica de Ejecución con Vite y Nginx!
1. **`vite dev` NO expone `remoteEntry.js`:**
   `@originjs/vite-plugin-federation` solo genera el bundle remoto al compilar. En `package.json` define:
   ```json
   "scripts": {
     "dev:federation": "vite build && vite preview --port <PUERTO_SEGUN_MATRIZ> --host 0.0.0.0",
     "preview": "vite preview --port <PUERTO_SEGUN_MATRIZ> --host 0.0.0.0"
   }
   ```
2. **Si usas Docker con Nginx:**
   - Reconstruye sin caché ante cualquier cambio: `docker compose build --no-cache && docker compose up -d`.
   - Asegura encabezados CORS en `nginx.conf`: `add_header Access-Control-Allow-Origin *;`.
   - **Prueba obligatoria:** Ejecuta `curl -I http://localhost:<PUERTO>/assets/remoteEntry.js`. El header `Content-Type` **DEBE** ser `application/javascript`. Si devuelve `text/html` (aunque sea `200 OK`), Nginx está cayendo en el fallback de `index.html` porque el build falló o la ruta no existe.

---

## 2. CONTRATO DE BACKEND (API GATEWAY KONG + PBAC)

Si este repositorio contiene código Backend, aplica estas reglas obligatorias:

### 2.1 Formato Estricto de Permisos: DOS PUNTOS (`:`), NUNCA PUNTOS (`.`)
- **TODOS** los permisos emitidos por `Auth-Backend` usan **dos puntos (`:`)** bajo el esquema `modulo:recurso:accion`.
  - ✅ Correcto: `inventory:items:read`, `menu:entries:manage`, `orders:orders:create`, `kitchen:queues:view`, `sala:tables:view`
  - ❌ Incorrecto: `inventory.items.read`, `menu.entries.manage`, `orders.create`, `sala.tables.view`
- Revisa decoradores (`@RequirePermissions`), guards, constantes y reemplaza cualquier punto (`.`) por dos puntos (`:`).

### 2.2 Bypass Obligatorio para Administrador (`*` y `ADMINISTRADOR`)
El Administrador del restaurante (`ADM000001`) tiene rol `ADMINISTRADOR` y permisos `["*"]`.
En **cualquier Guard o Middleware**, otorga acceso total de inmediato (`return true`) si:
- `permissions.includes('*')`
- `roles.includes('ADMINISTRADOR') || roles.includes('ADMIN')`

### 2.3 Catálogo Oficial de Permisos PBAC por Microservicio
Usa **únicamente** los siguientes códigos exactos registrados en `Auth-Backend`:

#### 1. Administrador General (`ADMINISTRADOR`)
- Permiso: `*` (Acceso irrestricto a todos los endpoints, vistas y acciones de todos los microservicios).

#### 2. Inventario y Almacén (`inventory`) — Rol por defecto: `ALMACENISTA`
- `inventory:items:read` — Ver insumos y existencias
- `inventory:items:write` — Gestionar insumos (crear, editar, baja)
- `inventory:batches:read` — Ver lotes y caducidades (FEFO)
- `inventory:batches:write` — Gestionar lotes (entradas, ajustes, mermas)
- `inventory:suppliers:read` — Ver proveedores
- `inventory:suppliers:write` — Gestionar proveedores
- `inventory:recipes:read` — Ver recetas, costos y disponibilidad
- `inventory:reservations:read` — Ver reservas de stock por órdenes
- `inventory:movements:read` — Ver bitácora de movimientos de almacén

#### 3. Menú y Carta (`menu`) — Rol por defecto: `GESTOR_MENU`
- `menu:entries:view` — Ver platillos, carta y precios
- `menu:entries:manage` — Gestionar platillos (crear, editar, disponibilidad)
- `menu:categories:view` — Ver categorías del menú
- `menu:categories:manage` — Gestionar categorías (crear, editar, ordenar)
- `menu:recipes:view` — Ver fichas técnicas y recetas vinculadas al menú
- `menu:recipes:manage` — Gestionar recetas de platillos
- `menu:offers:view` — Ver ofertas y promociones vigentes
- `menu:offers:manage` — Gestionar ofertas y promociones

#### 4. Sala y Mesas (`sala`) — Rol por defecto: `HOST`
- `sala:tables:view` — Ver catálogo de mesas y plano en tiempo real
- `sala:tables:occupy` — Sentar comensales e iniciar sesión de consumo
- `sala:tables:update_status` — Registrar salida de comensales y fin de limpieza
- `sala:tables:assign_waiter` — Asignar o reasignar mesero a mesa activa
- `sala:assignments:view` — Ver asignaciones de meseros a mesas
- `sala:table_unions:view` — Ver uniones de mesas
- `sala:table_unions:create` — Unir mesas libres para grupos grandes
- `sala:table_unions:release` — Disolver uniones de mesas
- `sala:reservations:view` — Ver agenda de reservas y disponibilidad
- `sala:reservations:create` — Registrar nueva reservación confirmada
- `sala:reservations:update` — Reprogramar o modificar reservaciones
- `sala:reservations:cancel` — Cancelar reservaciones
- `sala:reservations:mark_no_show` — Registrar No-show tras expirar tolerancia
- `sala:waitlist_entries:view` — Ver lista de espera presencial (FIFO)
- `sala:waitlist_entries:create` — Registrar clientes en lista de espera
- `sala:waitlist_entries:update_status` — Gestionar turnos en espera (sentar, cancelar)
> *Permisos de administración física de sala (reservados para Administrador):*
> - `sala:tables:create` — Crear nuevas mesas físicas
> - `sala:tables:update` — Editar número, capacidad o zona de mesas
> - `sala:tables:delete` — Dar de baja mesas físicas

#### 5. Órdenes y Pedidos (`orders`) — Rol por defecto: `MESERO`
- `orders:orders:view` — Consultar pedidos activos, historial y detalle
- `orders:orders:create` — Abrir nueva comanda (mesa o para llevar)
- `orders:orders:update` — Modificar comanda en borrador
- `orders:orders:confirm` — Confirmar comanda a cocina y reservar stock
- `orders:orders:cancel` — Cancelar comanda por cliente o desistimiento
- `orders:orders:transfer` — Reasignar mesa o comensal a comanda en curso
- `orders:items:redo` — Solicitar rehacer platillo a cocina por incidencia
- `orders:orders:request_bill` — Solicitar cuenta a caja bloqueando adiciones
> *Permiso de anulación especial (reservado para Administrador):*
> - `orders:orders:void_admin` — Anulación administrativa de comandas en preparación (merma justificada)

#### 6. Cocina y Pantalla KDS (`kitchen`) — Rol por defecto: `CHEF_MASTER`
- `kitchen:queues:view` — Ver cola KDS de preparación y filtrar por estación
- `kitchen:orders:start_preparation` — Iniciar preparación de comanda (consume stock)
- `kitchen:items:ready` — Marcar platillo individual como listo (notifica al mesero)
- `kitchen:orders:ready` — Marcar comanda completa lista para entrega
- `kitchen:items:reject` — Rechazar platillo por falta física de ingredientes
- `kitchen:items:cancel` — Cancelar platillo desde cocina antes de cocción
- `kitchen:orders:void_error` — Anular comanda por desperfecto en cocina (sin cargo al cliente)
- `orders:orders:view` — Consultar detalle de comanda para preparación

#### 7. Caja y Facturación (`billing`) — Rol por defecto: `CAJERO`
- `billing:invoices:create` — Emitir comprobantes y tickets de pago
- `billing:reports:view` — Consultar arqueo y reportes de caja

> **Regla de roles independientes**: Los roles de sistema no combinan microservicios. Si un empleado requiere operar en múltiples áreas (por ejemplo, un mesero que consulta el menú o el estado de mesas), se le asignan roles adicionales o se crea un rol personalizado desde la UI de Auth.

### 2.4 Headers Confiables Inyectados por Kong
El API Gateway Kong (`http://localhost:8000`) valida la firma del JWT (`HS256`, `iss: "urn:fmat:restaurant:auth"`), limpia headers falsificados por clientes externos e inyecta los siguientes headers hacia tu backend:
- `x-user-id`: UUID del usuario autenticado (`sub` del JWT).
- `x-staff-id`: Matrícula del colaborador (ej. `ADM000001`, `M000003`, `G000001`).
- `x-user-roles`: Cadena separada por comas (ej. `ADMINISTRADOR` o `MESERO`).
- `x-user-permissions`: Cadena separada por comas usando dos puntos `:` (ej. `orders:orders:view,orders:orders:create` o `*`).
- `authorization`: `Bearer <accessToken>` (Kong también reenvía el JWT original).

### 2.5 Implementación del `PermissionsGuard` (Ejemplo NestJS / Express)
```typescript
import { Injectable, CanActivate, ExecutionContext, ForbiddenException } from '@nestjs/common';
import { Reflector } from '@nestjs/core';

@Injectable()
export class PermissionsGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredPermissions = this.reflector.get<string[]>('permissions', context.getHandler());
    if (!requiredPermissions || requiredPermissions.length === 0) {
      return true;
    }

    const req = context.switchToHttp().getRequest();
    const rawRoles = (req.headers['x-user-roles'] as string) || '';
    const rawPermissions = (req.headers['x-user-permissions'] as string) || '';

    const roles: string[] = rawRoles
      ? rawRoles.split(',').map((r) => r.trim()).filter(Boolean)
      : (req.user?.roles ?? []);

    const permissions: string[] = rawPermissions
      ? rawPermissions.split(',').map((p) => p.trim()).filter(Boolean)
      : (req.user?.permissions ?? []);

    // 1. Bypass obligatorio para Administrador / Wildcard:
    if (
      permissions.includes('*') ||
      roles.includes('ADMINISTRADOR') ||
      roles.includes('ADMIN')
    ) {
      return true;
    }

    // 2. Validación de permisos requeridos (usando dos puntos ':'):
    const hasRequired = requiredPermissions.every((perm) => permissions.includes(perm));
    if (!hasRequired) {
      throw new ForbiddenException('No cuentas con los permisos necesarios para realizar esta acción');
    }

    return true;
  }
}
```

### 2.6 Red, CORS y Rutas
- Tu backend DEBE escuchar en host `0.0.0.0` en el puerto interno asignado en la Matriz Maestra.
- Habilita CORS para aceptar peticiones reenviadas.
- Verifica el comportamiento de `strip_path`:
  - En **Inventario** (`strip_path: true`), Kong recibe `/api/v1/inventory/items` y envía `/items` al backend.
  - En los demás (`strip_path: false`), Kong envía la ruta completa con el prefijo `/api/v1/...`.

---

## 3. CHECKLIST FINAL DE VERIFICACIÓN PARA EL AGENTE DE IA
Antes de reportar tu trabajo como completado, valida uno a uno los siguientes puntos:
1. [ ] **Permisos:** Todos los decoradores, guards y helpers usan **dos puntos (`:`)** y coinciden con la tabla de PBAC.
2. [ ] **Bypass Admin:** Los endpoints y vistas permiten acceso total si `permissions.includes('*')` o `roles.includes('ADMINISTRADOR')`.
3. [ ] **Frontend:** Si aplica, `react` y `react-dom` están en `^18.3.1`.
4. [ ] **Federation:** `vite.config.ts` expone `./App` con el nombre y puerto exactos de la Matriz Maestra y host `0.0.0.0`.
5. [ ] **Build:** `npm run build` compila limpiamente sin errores de TypeScript y genera `remoteEntry.js`.
6. [ ] **Backend:** Escucha en `0.0.0.0`, tiene CORS habilitado y lee los headers `x-user-permissions` / `x-user-roles` inyectados por Kong.
7. [ ] **Consumo API:** El Frontend apunta al API Gateway en `http://localhost:8000` con `Authorization: Bearer <token>`.
````
