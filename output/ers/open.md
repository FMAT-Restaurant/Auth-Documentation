# Decisiones Abiertas, Supuestos y Extensiones Futuras (Auth)

Este documento registra los puntos de diseño complementarios, supuestos operativos, directrices de arquitectura para la implementación y la especificación del sistema de diseño visual del microservicio **Auth**, del **API Gateway** y del esquema de eventos en **RabbitMQ**.

---

## 1. Decisiones Consolidadas

### DEC-AUTH-001 — Formato de Staff ID (`XYYYYYY`) por Rol Inicial
- **Resolución:** Cada miembro del personal operativo recibe un identificador único e inmutable con máscara `XYYYYYY`, donde `X` representa la inicial del rol principal asignado al momento del alta (ej. `M000001` para Mesero, `C000001` para Chef, `H000001` para Host, `A000001` para Almacenista, `G000001` para Gestor de Menú, `J000001` para Cajero, o `'E'` como prefijo genérico para roles personalizados) seguido de un correlativo de seis dígitos. El identificador es inmutable aunque posteriormente se modifiquen sus roles.

### DEC-AUTH-002 — Adopción del FMAT UI Kit v2.0 para Interfaces de Auth
- **Resolución:** Toda la interfaz de usuario de `Auth-Frontend` (Pantalla de Autenticación, Selector de Pestañas, Modales de Cambio de Clave y Panel de Gestión de Personal) se implementa siguiendo estrictamente la **Guía visual de componentes (Versión 2.0)** del proyecto FMAT Restaurant:
  - **Paleta de Color:**
    - `Primary`: `#C2410C` (Botones de acción principal y selección activa)
    - `Hover`: `#9A3412` (Estados interactivos pressed)
    - `Accent`: `#F97316` (Detalles, iconos de énfasis)
    - `Soft`: `#FFF7ED` (Fondo de selección suave)
    - `Ink`: `#111827` (Títulos y tipografía de alto contraste)
    - `Text`: `#374151` (Cuerpo de texto principal)
    - `Muted`: `#6B7280` (Textos secundarios y placeholders)
    - `Border`: `#E5E7EB` (Bordes de inputs y separadores)
    - `Canvas`: `#F8FAFC` (Fondo de pantalla completo)
    - `Surface`: `#FFFFFF` (Superficie de tarjetas y modales)
  - **Regla de Distribución Cromática:** 70% blanco y fondos neutros (`#F8FAFC`, `#FFFFFF`), 20% texto, bordes y estructura (`#111827`, `#E5E7EB`), 10% naranja y estados (`#C2410C`, `#F97316`).
  - **Tokens de Espaciado y Forma:**
    - Radios: `--radius-control: 8px` (inputs, botones), `--radius-card: 12px` (contenedores principales y modales).
    - Altura de controles: `40px` en escritorio, `48px` para interacción táctil (POS).
    - Tipografía: Familia `Inter` con escala `H1` (28/36px 700), `H2` (22/30px 700), `Body` (16/24px 400), `Small` (14/20px 400).
    - Botones: Primario a la derecha (`#C2410C`), texto con fórmula "verbo + objeto" (*"Iniciar sesión"*, *"Crear restaurante"*), estado `loading` para evitar doble clic accidental.

---

## 2. Directrices de Arquitectura y Mitigación de Riesgos de Implementación

A partir del análisis crítico de dependencias entre los equipos del salón, se establecen las siguientes directrices operativas:

### GUIDE-AUTH-001 — Entorno de Desarrollo Local Unificado (`docker-compose.yml`)
- **Problema:** El equipo de Auth es responsable de Auth MS, API Gateway y RabbitMQ. Si los demás equipos no pueden levantar estos servicios fácilmente, el equipo de Auth se convertirá en un cuello de botella para todo el salón.
- **Directriz:** Proveer un archivo `docker-compose.yml` en la raíz de la documentación/infraestructura que levante con un solo comando:
  1. Base de datos PostgreSQL para Auth con script de inicialización (`seed.sql`).
  2. RabbitMQ con el exchange `restaurant.staff.events` y colas declaradas.
  3. Microservicio Auth-Backend preconfigurado.
  4. API Gateway enrutando peticiones locales.
  5. Datos semilla (Seed): Un restaurante de prueba (`Restaurante Demo FMAT`), una cuenta de Administrador lista (`admin@fmat.edu.mx` / `Admin123!`) y cuentas de personal iniciales con contraseñas conocidas.

### GUIDE-AUTH-002 — Modo Mock / Bypass en API Gateway para DX
- **Problema:** En fases tempranas, los equipos de Menú, Inventario o Sala pueden requerir probar sus microservicios sin que el servicio de Auth esté necesariamente corriendo en sus máquinas.
- **Directriz:** Implementar una variable de entorno en el API Gateway `GATEWAY_DEV_BYPASS=true` (activa únicamente en desarrollo) que permita aceptar peticiones con una cabecera simulada (ej. `X-Mock-Role: ADMIN` o `X-Mock-Permissions: *`), inyectando las cabeceras internas sin validar la firma criptográfica del JWT.

### GUIDE-AUTH-003 — Estrategia de Mitigación de "Token Bloat" en JWT
- **Problema:** Al utilizar PBAC con permisos granulares (`<servicio>:<recurso>:<accion>`), si un empleado polivalente recibe decenas de permisos, el tamaño del JWT puede exceder los límites de cabeceras HTTP de proxies inversos (error HTTP 431).
- **Directriz:** Mantener identificadores de permisos canónicos y concisos. Si el catálogo crece a más de 30 permisos por usuario, se utilizarán alias compactados en el JWT o comodines de nivel de servicio (`inventory:*` para usuarios con todos los permisos del módulo).

### GUIDE-AUTH-004 — Estabilidad y Congelamiento de Contratos de Eventos
- **Problema:** Modificaciones intempestivas en los nombres de las colas o payloads de RabbitMQ rompen los consumidores dependientes de otros equipos (como la sincronización de meseros en Sala).
- **Directriz:** El exchange `restaurant.staff.events` y los eventos `staff.created`, `staff.roles.updated`, `staff.status.changed` y `staff.deleted` se mantendrán estrictamente versionados y comunicados en el repositorio central `Documentation`.
- **Nota (22/11/2026):** Hoy ningún microservicio consume estos eventos y RabbitMQ es opcional en Auth. Ver [OPEN-AUTH-007](#open-auth-007--rabbitmq-opcional-en-auth-eventos-de-personal-sin-consumidores).

---

## 3. Puntos de Diseño y Decisiones Abiertas

### OPEN-AUTH-001 — Modo de Acceso Rápido por PIN para Terminales Táctiles (POS / KDS)
- **Contexto:** En el entorno dinámico de un restaurante físico, escribir una contraseña alfanumérica compleja en cada cambio de mesero o comanda en una tablet táctil puede restar agilidad operativa.
- **Propuesta:** Introducir en una segunda iteración el soporte para un **PIN numérico de 4 a 6 dígitos** ligado al `staffId`, que permita un desbloqueo rápido de sesión en terminales locales asignadas previamente al restaurante, manteniendo la contraseña completa para configuraciones administrativas o inicio de turno.
- **Estado:** Abierto para discusión con los equipos de Frontend y Orders.

---

### OPEN-AUTH-002 — Identificación de Personal en Turno Activo (Clock-In / Clock-Out)
- **Contexto:** El microservicio `Sala/Host` necesita la lista de personal disponible para asignar mesas. Con el modelo PBAC, Sala filtra a los empleados que posean el permiso `orders:create`. No obstante, no todos los meseros dados de alta en nómina están físicamente presentes en el restaurante en un momento dado.
- **Propuesta:** Definir si la lista proyectada por Sala debe incluir a *todos los empleados con permiso `orders:create` dados de alta*, o si Auth / Sala debe manejar un estado adicional de "En turno / Activo hoy" (`clockedIn: boolean`).
- **Estado:** Pendiente de coordinación con el equipo de Sala/Host.

---

### OPEN-AUTH-003 — Autenticación Multifactor (MFA / 2FA) para Administradores
- **Contexto:** Las cuentas de tipo `ADMIN` tienen acceso integral a nóminas, configuraciones comerciales y métricas financieras de todos los microservicios.
- **Propuesta:** Evaluar la implementación de MFA basado en TOTP (Google Authenticator) para los gerentes de restaurante, aumentando la resistencia ante vulnerabilidades de credenciales.
- **Estado:** Previsto como mejora de seguridad post-MVP.

---

### OPEN-AUTH-004 — Rotación de Claves Criptográficas (JWKS - JSON Web Key Set)
- **Contexto:** Si se emplea `RS256`, el API Gateway necesita conocer la clave pública de Auth.
- **Propuesta:** Exponer un endpoint estándar `GET /.well-known/jwks.json` en Auth para que el API Gateway pueda consultar y refrescar en caché la clave pública de verificación de forma transparente, permitiendo la rotación de claves sin reiniciar los servicios.
- **Estado:** Propuesto para la implementación de backend del Gateway.

---

### OPEN-AUTH-005 — Mecanismo de Evaluación de Permisos: Gateway vs. Microservicio
- **Contexto:** En arquitecturas de microservicios con PBAC, la validación de permisos puede ejecutarse en dos niveles:
  1. *A nivel perimetral (Gateway):* El Gateway cuenta con una tabla de enrutamiento que mapea endpoints a permisos requeridos.
  2. *A nivel de microservicio interno:* El Gateway valida autenticación y contraseña temporal, inyectando `X-User-Permissions`, para que cada microservicio verifique internamente con middlewares sus propios permisos.
- **Propuesta:** Adoptar el enfoque híbrido donde el Gateway protege el perímetro y propaga cabeceras sanitizadas para autorización fina downstream.
- **Estado:** En consenso con los demás equipos de backend.

---

### OPEN-AUTH-006 — Hardening de Resolución Dinámica de Hostname (`auto`) y CORS en Desarrollo LAN
- **Contexto:** Para facilitar pruebas en redes locales cambiantes sin editar `.env`, `Auth-Frontend` soporta `VITE_API_BASE_URL=auto` (resolviendo `window.location.hostname:4000`) y `VITE_GATEWAY_URL=auto` (reescribiendo `http://localhost:808x` en el middleware de desarrollo de `vite.config.ts` según `req.headers.host`).
- **Riesgos identificados:**
  1. *Host Header Injection (solo `vite dev`):* Un `Host` manipulado en la petición hacia `/remote/*` podría reescribir la URL destino de los microfrontends hacia un servidor externo y filtrar el `Authorization: Bearer <token>`.
  2. *Exposición en producción:* Usar `auto` fuera de desarrollo provocaría peticiones hacia puertos `:4000`/`:8000` sobre el dominio público o facilitaría *DNS Rebinding*.
  3. *Acumulación de IPs en `CORS_ORIGINS` (Kong):* Mantener IPs temporales de redes compartidas en `API-Gateway/.env` amplía innecesariamente la superficie permitida si otra máquina hereda esa IP por DHCP.
- **Mitigación recomendada:** Validar `reqHostname` en `vite.config.ts` contra `localhost`, `127.0.0.1` o rangos RFC 1918 (con fallback a `localhost`), restringir `auto` a `import.meta.env.DEV` y limpiar `CORS_ORIGINS` antes de producción.
- **Actualización (23/11/2026) — API del shell por el Gateway en el mismo origen:** `VITE_API_BASE_URL=auto` ya no resuelve `hostname:4000`; ahora es `/api/v1` en el mismo origen de la app. El servidor de Vite (desarrollo, `VITE_GATEWAY_TARGET`, por defecto `http://localhost:8000`) y nginx (producción, `GATEWAY_UPSTREAM`, por defecto `http://fmat-api-gateway:8000`) reenvían `/api/` al API Gateway, que valida el JWT antes de llegar a Auth-Backend. Efectos:
  - El shell **nunca** llama directo al backend en el puerto 4000; toda su API pasa por Kong (`iss`, `aud`, `exp`).
  - Funciona con cualquier IP o hostname (p. ej., la red de la facultad) sin agregar orígenes a CORS, y la cookie del refresh token es de primera parte.
  - El destino del proxy se resuelve en el servidor, no desde la cabecera `Host`, así que el riesgo 1 ya no aplica a la API del shell. El riesgo 2 desaparece para `VITE_API_BASE_URL`.
  - **Sigue pendiente** para los microfrontends: `VITE_GATEWAY_URL=auto` aún usa `req.headers.host:8000` (riesgos 1 y 3), y al abrir la app por IP hay que agregar `http://<IP>:3000` a `CORS_ORIGINS` del Gateway. No se cambió porque el Gateway expone rutas de Inventario en la raíz (`/items`, `/batches`…), que chocarían con las rutas del SPA si se sirvieran en el mismo origen.
  - Al pasar por el proxy, el backend ve la IP del Gateway para todas las peticiones; el límite por IP del login se comparte entre los usuarios de una misma instalación (el límite por cuenta no cambia).
- **Estado:** Parcialmente mitigado (API del shell). Pendiente para microfrontends.

---

### OPEN-AUTH-007 — RabbitMQ opcional en Auth: eventos de personal sin consumidores
- **Contexto (22/11/2026):** Auth publica `staff.created`, `staff.roles.updated`, `staff.status.changed` y `staff.deleted` en el exchange `restaurant.staff.events`, pero una búsqueda en todos los repositorios de la organización no encontró ningún consumidor. La arquitectura de Sala (`Floor-Module/agents/ARCHITECTURE.md`) indica que Sala no consume eventos externos, y desde Auth-Backend #28 los demás servicios pueden consultar `GET /api/v1/staff` por PBAC para obtener al personal. Mantener el broker obligatorio solo generaba errores en el arranque y un `/health` en `degraded`.
- **Decisión:** RabbitMQ pasa a ser **opcional** en Auth. Se conserva el código de publicación por si algún equipo necesita los eventos más adelante:
  - `RABBITMQ_URL` vacía o sin definir: Auth no se conecta, no publica eventos ni registra errores, y `GET /health` responde `status: healthy` con `dependencies.rabbitmq: disabled`.
  - `RABBITMQ_URL` definida: Auth publica los eventos. Si el broker no está disponible, `/health` responde `degraded` con `rabbitmq: disconnected` y en producción se registra como error.
- **Para consumir los eventos:** el equipo interesado lo comunica al equipo de Auth, que configura `RABBITMQ_URL` en el entorno compartido y declara la cola correspondiente (ver `configuration.md`).
- **Alternativa descartada por ahora:** eliminar por completo los eventos y la dependencia `amqplib`. Se reconsiderará si ningún equipo los solicita.
- **Estado:** Implementado (`feat/rabbitmq-opcional` en Auth-Backend).

---

### OPEN-AUTH-008 — Access token solo en memoria en el shell (Auth-Frontend #6)
- **Contexto (23/11/2026):** El shell guardaba la sesión (access token y usuario) en `localStorage` (`fmat_restaurant_auth_session`). Cualquier XSS en el shell o en un microfrontend federado podía leer el token. El refresh token ya vivía en una cookie HttpOnly (`SameSite=Lax`, `path=/api/v1/auth`, 7 días).
- **Decisión:** El access token y el usuario viven **solo en memoria** del shell:
  - Al arrancar, el shell borra la clave `fmat_restaurant_auth_session` que dejaron versiones anteriores.
  - Al recargar, muestra "Cargando sesión…" y recupera la sesión: primero se la pide a otra pestaña abierta (`BroadcastChannel`, mismo origen) y, si no hay ninguna, llama a `POST /api/v1/auth/refresh` (cookie) y `GET /api/v1/auth/me`.
  - Una sola restauración a la vez: el refresh token rota en cada uso y dos renovaciones simultáneas con la misma cookie harían fallar la segunda.
  - Cerrar sesión en una pestaña la cierra en todas.
- **Contrato con los microfrontends:** el token llega **solo** por la prop `token` de `FederatedModuleProps` (y el usuario por `currentUser`); el shell la actualiza cada vez que renueva el token (cada ≤ 15 min). **Ya no se debe leer `localStorage`**: en Menú, Órdenes/KDS y Sala ese acceso queda solo como respaldo para ejecutar el microfrontend por separado en desarrollo. Inventario no lo usa.
- **Pruebas:** recargar la página recupera la sesión sin volver al login; una segunda pestaña la obtiene de la primera sin llamar a la API; `localStorage` queda vacío.
- **Estado:** Implementado (Auth-Frontend, `feat/token-en-memoria`).
