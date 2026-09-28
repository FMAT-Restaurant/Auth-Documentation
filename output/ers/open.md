# Decisiones Abiertas, Supuestos y Extensiones Futuras (Auth)

Este documento registra los puntos de diseño complementarios, supuestos operativos, directrices de arquitectura para la implementación y la especificación del sistema de diseño visual del microservicio **Auth**, del **API Gateway** y del esquema de eventos en **RabbitMQ**.

---

## 1. Decisiones Consolidadas

### DEC-AUTH-001 — Desacoplamiento de Staff ID (`XYYYYYY`) y Adopción del Prefijo Canónico `'E'`
- **Resolución:** Dado que los roles son etiquetas descriptivas libres asignadas por el Administrador (ej. *"Líder de inventario"*, *"Mesero de terraza"*) y no enums fijos, el carácter `X` en el identificador `XYYYYYY` adopta de manera canónica el prefijo `'E'` (Empleado: `E000001` a `E999999`). Esto garantiza que el identificador sea estrictamente inmutable y permanezca neutral ante cualquier modificación de cargo o funciones del personal.

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
- **Directriz:** El exchange `restaurant.staff.events` y los eventos `staff.created`, `staff.permissions.updated` y `staff.status.changed` se mantendrán estrictamente versionados y comunicados en el repositorio central `Documentation`.

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
