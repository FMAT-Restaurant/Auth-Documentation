# Prompt de Integración para Equipos (Microfrontends + API Gateway + PBAC)

Copia y pega el siguiente bloque en el agente de IA de tu equipo (Cursor, Antigravity, Claude Code, Copilot, etc.) dentro del repositorio de tu microservicio:

---

## Prompt para el Agente de IA de cada equipo

````markdown
Actúa como un ingeniero experto en React, TypeScript, Vite, Node/NestJS y arquitecturas de Microfrontends y Microservicios.
Necesito que configures y adaptes este repositorio para conectarse como un **Microfrontend Remoto (Remote)** dentro del **Shell principal (`Auth-Frontend`)** del proyecto **FMAT-Restaurant** usando `@originjs/vite-plugin-federation`, además de alinear las llamadas HTTP y los permisos PBAC con el **API Gateway (Kong)** y **`Auth-Backend`**.

Primero, inspecciona este repositorio para identificar a cuál de los 5 módulos pertenece y usa **estrictamente** los valores de su fila en esta tabla oficial de contratos del Shell:

| Módulo | `name` en Federation | Puerto Frontend (`server` y `preview`) | Ruta en API Gateway (`http://localhost:8000`) | `strip_path` en Kong | Puerto interno Backend |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Sala / Host** | `salaApp` | `5174` | `/api/v1/floor` | `false` (recibe `/api/v1/floor/...`) | `8085` |
| **Menú** | `menuApp` | `5173` (o `5175`) | `/api/v1/menu` | `false` (recibe `/api/v1/menu/...`) | `8080` |
| **Inventario** | `inventarioApp` | `5176` | `/api/v1/inventory` | `true` (recibe `/items`, `/batches`, etc.) | `8083` |
| **Órdenes y Cocina (KDS)** | `ordenesApp` | `5177` | `/api/v1/orders` | `false` (recibe `/api/v1/orders/...`) | `8082` |
| **Caja y Facturación** | `cajaApp` | `5178` | `/api/v1/payments` | `false` (recibe `/api/v1/payments/...`) | `8084` |

Realiza los siguientes cambios en el repositorio:

### 1. Instalar y configurar Module Federation en el Frontend
1. **Alinear la versión de React con el Shell:**
   El Shell (`Auth-Frontend`) utiliza **React 18** (`^18.3.1`). Como Module Federation comparte la misma instancia en memoria (`shared: ['react', 'react-dom']`), es requisito obligatorio que todos los microfrontends utilicen React 18 (si usan React 19, React 18 no reconoce sus elementos y arroja error de `Objects are not valid as a React child`):
   ```bash
   npm install react@^18.3.1 react-dom@^18.3.1
   npm install -D @types/react@^18.3.18 @types/react-dom@^18.3.5
   ```
2. Instala el plugin de federación en las dependencias de desarrollo:
   ```bash
   npm install -D @originjs/vite-plugin-federation
   ```
3. Actualiza `vite.config.ts` para exponer el componente raíz como `./App` en el archivo `remoteEntry.js` y fijar el puerto asignado tanto en `server` como en `preview` con `cors: true` y `host: true`:
   ```ts
   import { defineConfig } from 'vite';
   import react from '@vitejs/plugin-react';
   import federation from '@originjs/vite-plugin-federation';

   export default defineConfig({
     plugins: [
       react(),
       federation({
         name: '<NAME_SEGUN_TABLA>', // ej. 'menuApp', 'inventarioApp', 'salaApp', 'ordenesApp', 'cajaApp'
         filename: 'remoteEntry.js',
         exposes: {
           './App': './src/App.tsx',
         },
         shared: ['react', 'react-dom'],
       }),
     ],
     server: {
       port: <PUERTO_SEGUN_TABLA>,
       strictPort: true,
       cors: true,
       host: true,
     },
     preview: {
       port: <PUERTO_SEGUN_TABLA>,
       strictPort: true,
       cors: true,
       host: true,
     },
     build: {
       target: 'esnext',
       minify: false,
       cssCodeSplit: false,
     },
   });
   ```
   > **Nota crítica:** El Shell (`Auth-Frontend`) importa el módulo desde `http://<IP>:<PUERTO>/assets/remoteEntry.js` y monta el `export default` de `./src/App.tsx`. Asegúrate de que `src/App.tsx` tenga `export default App`.

4. En `package.json`, agrega o actualiza los scripts para levantar el microfrontend en modo federado exponiéndolo en red (`--host`), ya que `@originjs/vite-plugin-federation` **solo genera `assets/remoteEntry.js` al compilar (`vite build`), NO con `vite dev`**:
   ```json
   "scripts": {
     "dev:federation": "vite build && vite preview --port <PUERTO_SEGUN_TABLA> --host",
     "preview": "vite preview --port <PUERTO_SEGUN_TABLA> --host"
   }
   ```
   > **Si usan Docker / Nginx para el Frontend:** Cada vez que cambien código deben reconstruir con `docker compose build --no-cache && docker compose up -d`, asegurarse de incluir `add_header Access-Control-Allow-Origin *;` en `nginx.conf`, y verificar con `curl -I http://localhost:<PUERTO>/assets/remoteEntry.js` que devuelva `Content-Type: application/javascript` (si devuelve `text/html` aunque sea `200 OK`, es el fallback de Nginx y el archivo no se compiló).

---

### 2. Recibir la Sesión, Usuario, Permisos PBAC Exactos y Tema desde el Shell
El Shell (`Auth-Frontend`) le pasa a `<App />` las siguientes props al montarlo, y además guarda la sesión en `localStorage` bajo la clave `'fmat_restaurant_auth_session'` (que es compartida cuando se ejecuta dentro del Shell):

1. Define la interfaz de props en `src/App.tsx` (o en un archivo de tipos):
   ```ts
   export interface ShellUser {
     id: string;
     userType: 'ADMIN' | 'STAFF';
     staffId?: string;
     email?: string;
     displayName: string;
     roleLabel?: string;
     roles?: string[];      // Ej: ['ADMINISTRADOR'] o ['MESERO', 'HOST']
     permissions: string[]; // Ej: ['*'] para ADMIN, o permisos con ':'
   }

   export interface FederatedModuleProps {
     token?: string;
     currentUser?: ShellUser;
     theme?: 'light' | 'dark';
   }
   ```
2. Haz que `App` acepte `FederatedModuleProps` y tenga un fallback automático a `localStorage.getItem('fmat_restaurant_auth_session')` por si no recibe props o se recarga:
   ```tsx
   const getStoredSession = (): { accessToken?: string; user?: ShellUser } | null => {
     try {
       const raw = localStorage.getItem('fmat_restaurant_auth_session');
       return raw ? JSON.parse(raw) : null;
     } catch {
       return null;
     }
   };
   ```
   - El token efectivo será `const activeToken = token ?? getStoredSession()?.accessToken;`
   - El usuario efectivo será `const activeUser = currentUser ?? getStoredSession()?.user;`
   - Para verificar si el usuario tiene un permiso en la UI, crea o actualiza este helper (debe permitir acceso total a `ADMIN`, rol `ADMINISTRADOR` o comodín `*`):
     ```ts
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

3. **Catálogo Oficial de Permisos PBAC (¡USAR SIEMPRE DOS PUNTOS `:`, NUNCA PUNTOS `.`!):**
   `Auth-Backend` emite **únicamente** estos códigos de permisos separados por **dos puntos (`:`)**. Revisa todo el Frontend y el Backend de este repositorio y reemplaza cualquier permiso con puntos (ej. `inventory.items.read`) o nombres distintos por los de esta tabla oficial:

   | Módulo | Códigos Exactos de Permisos emitidos por Auth (`permissions`) | Roles por defecto |
   | :--- | :--- | :--- |
   | **Administrador** | `*` (Comodín de acceso total a todos los módulos y operaciones) | `ADMINISTRADOR` |
   | **Sala / Host** | `sala:tables:view`, `sala:tables:assign_diner`, `sala:tables:assign_waiter`, `staff:view` | `HOST` (todos), `MESERO` (`sala:tables:view`) |
   | **Menú** | `menu:view`, `menu:edit` | `MESERO` (`menu:view`), `ADMINISTRADOR` (`*`) |
   | **Inventario** | `inventory:items:read`, `inventory:items:write`, `inventory:batches:read`, `inventory:batches:write`, `inventory:suppliers:read`, `inventory:suppliers:write`, `inventory:recipes:read`, `inventory:reservations:read`, `inventory:movements:read` | `ALMACENISTA` (los 9) |
   | **Órdenes** | `orders:view`, `orders:create`, `orders:cancel` | `MESERO` (`orders:view`, `orders:create`), `CAJERO` (`orders:view`) |
   | **Cocina (KDS)** | `kitchen:kds:view`, `kitchen:order:start_preparation`, `kitchen:order:mark_ready` | `CHEF_MASTER` (los 3) |
   | **Caja / Pagos** | `billing:invoices:create`, `billing:reports:view` | `CAJERO` (los 2) |

4. **Soporte de Modo Oscuro (`[data-theme="dark"]`):**
   El Shell aplica el atributo `data-theme="dark"` en `document.documentElement` (`<html data-theme="dark">`). Asegúrate de que tu `src/index.css` incluya la paleta oscura de FMAT-Restaurant:
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
5. **Evitar duplicar el Sidebar principal:** El Shell ya provee el Sidebar lateral de navegación entre microservicios y el header móvil. Si este frontend tenía un Sidebar global de todo el restaurante o barra de login propia, ocúltalo cuando se monte desde el Shell para renderizar únicamente el contenido del módulo.

---

### 3. Conectar el Frontend y Backend con el API Gateway (Kong)
1. **En el Frontend (servicios HTTP / `fetch` / `axios`):**
   - Usa la variable de entorno `VITE_API_URL` con fallback al API Gateway: `http://localhost:8000<RUTA_SEGUN_TABLA>` (ej. `http://localhost:8000/api/v1/menu`).
   - En todas las peticiones incluye siempre el header:
     ```ts
     Authorization: `Bearer ${activeToken}`
     ```
2. **En el Backend (Guards / Middlewares de Autenticación y Permisos):**
   - El API Gateway (Kong + plugin `auth-perimeter`) ya valida la firma del JWT (`HS256`, `iss: "urn:fmat:restaurant:auth"`), bloquea contraseñas temporales y reenvía la petición al backend inyectando exactamente estos **headers HTTP**:
     - `x-user-id`: UUID del usuario autenticado (`sub` del JWT)
     - `x-staff-id`: Matrícula operativa del usuario (ej. `ADM000001`, `M000003`, `O000006`)
     - `x-user-roles`: Roles separados por comas (ej. `ADMINISTRADOR` o `MESERO,HOST`)
     - `x-user-permissions`: Permisos separados por comas usando **dos puntos `:`** (ej. `menu:view,menu:edit` o `*` para el Administrador)
     - `authorization`: `Bearer <accessToken>` (Kong también reenvía el JWT original)
   - **Regla crítica en el `PermissionsGuard` / Middleware del Backend:**
     1. Lee `x-user-permissions` y `x-user-roles` separándolos por comas (`.split(',').map(s => s.trim())`), o extráelos del payload del Bearer JWT como fallback.
     2. **Bypass obligatorio de Admin:** Si `permissions.includes('*')` o `roles.includes('ADMINISTRADOR')` o `roles.includes('ADMIN')`, **permite el acceso inmediatamente (`return true`)** sin rechazar con `403`.
     3. Asegúrate de que todos los decoradores/rutas del backend validen los permisos usando **dos puntos (`:`)** según la tabla de la Sección 2.3 (nunca puntos `.`).
   - Asegúrate de que el Backend escuche en `0.0.0.0` en el puerto interno indicado en la tabla y tenga CORS habilitado.

Por favor, aplica todos estos cambios, verifica que `npm run build` genere correctamente `dist/assets/remoteEntry.js` sin errores de TypeScript, y muéstrame un resumen de los archivos modificados.
````
