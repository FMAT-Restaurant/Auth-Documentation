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

Primero, inspecciona este repositorio para identificar a cuál de los 5 módulos pertenece y usa **estrictamente** los valores de su fila en esta tabla oficial de contratos:

### Matriz Maestra de Contratos por Microservicio

| Módulo / Dominio | `name` en Federation | Puerto Frontend (`server` y `preview`) | Ruta en API Gateway (`http://localhost:8000`) | `strip_path` en Kong | Puerto interno Backend | Rol de Sistema en Auth |
| :--- | :--- | :---: | :--- | :---: | :---: | :--- |
| **Sala / Host** | `salaApp` | `5174` | `/api/v1/floor` | `false` (recibe `/api/v1/floor/...`) | `8085` | `HOST` |
| **Menú** | `menuApp` | `5173` | `/api/v1/menu` | `false` (recibe `/api/v1/menu/...`) | `8080` | `GESTOR_MENU` |
| **Inventario** | `inventarioApp` | `5176` | `/api/v1/inventory` | `true` (recibe `/items`, `/batches`, etc.) | `8083` | `ALMACENISTA` |
| **Órdenes (POS)** | `ordenesApp` | `5177` | `/api/v1/orders` | `false` (recibe `/api/v1/orders/...`) | `8082` | `MESERO` |
| **Cocina (KDS)** | `ordenesApp` | `5177` | `/api/v1/orders` | `false` (recibe `/api/v1/orders/...`) | `8082` | `CHEF_MASTER` |
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
      name: '<NAME_SEGUN_MATRIZ>', // ej. 'menuApp', 'inventarioApp', 'salaApp', 'ordenesApp', 'cajaApp'
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
- **TODOS** los permisos emitidos por `Auth-Backend` usan **dos puntos (`:`)**.
  - ✅ Correcto: `inventory:items:read`, `menu:entries:manage`, `orders:create`, `kitchen:kds:view`
  - ❌ Incorrecto: `inventory.items.read`, `menu.entries.manage`, `orders.create`
- Revisa decoradores (`@RequirePermissions`), guards, constantes y reemplaza cualquier punto (`.`) por dos puntos (`:`).

### 2.2 Bypass Obligatorio para Administrador (`*` y `ADMINISTRADOR`)
El Administrador del restaurante (`ADM000001`) tiene rol `ADMINISTRADOR` y permisos `["*"]`.
En **cualquier Guard o Middleware**, otorga acceso total de inmediato (`return true`) si:
- `permissions.includes('*')`
- `roles.includes('ADMINISTRADOR') || roles.includes('ADMIN')`

### 2.3 Catálogo Oficial de Permisos PBAC
Usa **únicamente** los siguientes códigos exactos:

| Dominio | Rol de Sistema en Auth | Códigos Exactos de Permisos (`permissions`) |
| :--- | :--- | :--- |
| **Administrador** | `ADMINISTRADOR` | `*` (Acceso irrestricto) |
| **Inventario** | `ALMACENISTA` | `inventory:items:read`, `inventory:items:write`, `inventory:batches:read`, `inventory:batches:write`, `inventory:suppliers:read`, `inventory:suppliers:write`, `inventory:recipes:read`, `inventory:reservations:read`, `inventory:movements:read` |
| **Menú** | `GESTOR_MENU` | `menu:entries:view`, `menu:entries:manage`, `menu:categories:view`, `menu:categories:manage`, `menu:recipes:view`, `menu:recipes:manage`, `menu:offers:view`, `menu:offers:manage` |
| **Sala / Host** | `HOST` | `sala:tables:view`, `sala:tables:assign_diner`, `sala:tables:assign_waiter` |
| **Órdenes (POS)** | `MESERO` | `orders:view`, `orders:create`, `orders:cancel` |
| **Cocina (KDS)** | `CHEF_MASTER` | `kitchen:kds:view`, `kitchen:order:start_preparation`, `kitchen:order:mark_ready` |
| **Caja / Facturación**| `CAJERO` | `billing:invoices:create`, `billing:reports:view` |

> *Nota:* Los roles de sistema no combinan microservicios. Si un puesto requiere operar en múltiples áreas (por ejemplo, un mesero que consulta el menú), se le asignan roles adicionales o un rol personalizado desde la interfaz de administración de Auth.

### 2.4 Headers Confiables Inyectados por Kong
El API Gateway Kong (`http://localhost:8000`) valida la firma del JWT (`HS256`, `iss: "urn:fmat:restaurant:auth"`), limpia headers falsificados por clientes externos e inyecta los siguientes headers hacia tu backend:
- `x-user-id`: UUID del usuario autenticado (`sub` del JWT).
- `x-staff-id`: Matrícula del colaborador (ej. `ADM000001`, `M000003`, `G000001`).
- `x-user-roles`: Cadena separada por comas (ej. `ADMINISTRADOR` o `MESERO`).
- `x-user-permissions`: Cadena separada por comas usando dos puntos `:` (ej. `menu:entries:view,menu:entries:manage` o `*`).
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
