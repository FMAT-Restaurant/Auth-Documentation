# Requisitos Funcionales del Servicio Auth

Cada bloque establece una obligación primaria formal del sistema. Las verificaciones siguen los cuatro métodos formales de la disciplina de *Verificación y Validación*: **Inspección**, **Demostración**, **Prueba** y **Análisis**.

---

<a id="req-auth-001"></a>

### REQ-AUTH-001 — Inicialización de Cuenta de Administrador Local

**Requisito:**
El servicio Auth deberá permitir la inicialización de la cuenta raíz del Gerente (`ADMINISTRADOR`) en la instalación local mediante `POST /api/v1/auth/setup-admin` capturando correo electrónico, contraseña, nombre y apellidos, bloqueando cualquier registro administrativo posterior si ya existe una cuenta de administrador en el sistema.

**Tipo:** Funcional  
**Fuente:** Especificación de Roles Softrestaurant FMAT  
**Verificación:** Prueba: Ejecutar la inicialización con datos válidos; comprobar que se genera el usuario administrador con el rol exclusivo `ADMINISTRADOR` y permisos universales (`*`). Comprobar que intentos posteriores de inicialización sean rechazados con error HTTP `409 Conflict` o `400 Bad Request`.  
**Estado:** Confirmado  
**Reglas Relacionadas:** BR-AUTH-001, BR-AUTH-002, INV-AUTH-002  

---

<a id="req-auth-002"></a>

### REQ-AUTH-002 — Autenticación del Gerente / Administrador

**Requisito:**
El servicio Auth deberá validar las credenciales de acceso del Administrador mediante su correo electrónico y contraseña maestra, retornando un par de tokens criptográficos (`accessToken` y `refreshToken`) con alcance total de administración (`permissions: ["*"]`).

**Tipo:** Funcional  
**Fuente:** Modelo de Acceso Administrativo FMAT  
**Verificación:** Prueba: Probar autenticación con credenciales correctas e incorrectas, validando que solo en caso de éxito se retorne un JWT firmado con tipo `ADMIN`, `permissions: ["*"]` y `mustChangePassword = false`.  
**Estado:** Confirmado  
**Reglas Relacionadas:** BR-AUTH-001, INV-AUTH-002, INV-AUTH-007  

---

<a id="req-auth-003"></a>

### REQ-AUTH-003 — Creación de Perfil de Personal con Roles y Permisos

**Requisito:**
El servicio Auth deberá permitir a un usuario con rol `ADMINISTRADOR` registrar a un nuevo miembro del equipo operativo (`StaffMember`), capturando nombres, apellidos y asignándole uno o más roles del sistema o personalizados con permisos granulares por microservicio.

**Tipo:** Funcional  
**Fuente:** Requisitos de Gestión de Personal Softrestaurant FMAT  
**Verificación:** Demostración: Desde una sesión de administrador, dar de alta a un miembro del personal y verificar que su perfil quede persistido en la base de datos local con sus roles y permisos asociados.  
**Estado:** Confirmado  
**Reglas Relacionadas:** BR-AUTH-003, BR-AUTH-010, INV-AUTH-001  

---

<a id="req-auth-004"></a>

### REQ-AUTH-004 — Generación Automática del Identificador `XYYYYYY`

**Requisito:**
El servicio Auth deberá asignar automáticamente a cada nuevo miembro del personal un identificador unívoco e inmutable con formato `XYYYYYY`, donde `X` es un carácter alfabético en mayúscula y `YYYYYY` es un correlativo numérico de seis dígitos, garantizando su unicidad en toda la base de datos local.

**Tipo:** Funcional  
**Fuente:** Especificación de Identificadores de Personal FMAT  
**Verificación:** Prueba: Registrar múltiples miembros de personal consecutivamente; inspeccionar que todos cumplan con la expresión regular `^[A-Z][0-9]{6}$` y que no se produzcan colisiones en el identificador.  
**Estado:** Confirmado  
**Reglas Relacionadas:** BR-AUTH-004, BR-AUTH-005, BR-AUTH-006, INV-AUTH-003, INV-AUTH-004  

---

<a id="req-auth-005"></a>

### REQ-AUTH-005 — Asignación y Estado de Contraseña Temporal

**Requisito:**
El servicio Auth deberá asignar a todo nuevo miembro del personal una contraseña temporal generada aleatoriamente por el sistema, marcando su credencial con el estado `passwordStatus = TEMPORARY` (`mustChangePassword = true`).

**Tipo:** Funcional  
**Fuente:** Especificación de Seguridad y Primer Ingreso FMAT  
**Verificación:** Prueba: Crear un nuevo miembro del personal, inspeccionar la base de datos para confirmar que `password_status == 'TEMPORARY'` y verificar que el hash almacenado corresponda a la contraseña temporal generada.  
**Estado:** Confirmado  
**Reglas Relacionadas:** BR-AUTH-007, BR-AUTH-008, INV-AUTH-005, INV-AUTH-007  

---

<a id="req-auth-006"></a>

### REQ-AUTH-006 — Asignación Obligatoria de Roles y Permisos

**Requisito:**
El servicio Auth deberá exigir la asignación de al menos un (1) rol válido (del sistema o personalizado) al crear o editar personal, rechazando solicitudes con colecciones de roles vacías o nulas.

**Tipo:** Funcional  
**Fuente:** Modelo de Control de Acceso PBAC FMAT  
**Verificación:** Prueba: Enviar peticiones de creación con lista de roles vacía (`[]`), validando que el servicio rechace la operación con código HTTP `422 Unprocessable Entity` o `400 Bad Request`.  
**Estado:** Confirmado  
**Reglas Relacionadas:** BR-AUTH-010, INV-AUTH-001  

---

<a id="req-auth-007"></a>

### REQ-AUTH-007 — Soporte para Creación de Roles Dinámicos y Permisos Granulares

**Requisito:**
El servicio Auth deberá permitir al Administrador crear roles personalizados configurando cualquier combinación de permisos granulares por microservicio (por ejemplo: agregar ingredientes, eliminar ingredientes y cambiar estado de stock en Inventario, y/o abrir comandas en Órdenes), otorgándole acceso exacto a las funciones y botones habilitados.

**Tipo:** Funcional  
**Fuente:** Requisitos Operativos y Flexibilidad Administrativa FMAT  
**Verificación:** Demostración: Crear un rol personalizado con permisos `inventory:view`, `inventory:ingredients:create`, `inventory:ingredients:delete`, `inventory:stock:update_status`, asignarlo a un empleado, autenticarlo y comprobar que el JWT retorne exactamente la lista de permisos configurada.  
**Estado:** Confirmado  
**Reglas Relacionadas:** BR-AUTH-011, BR-AUTH-012, INV-AUTH-001  

---

<a id="req-auth-008"></a>

### REQ-AUTH-008 — Exclusión de Facultades de Administrador en Personal

**Requisito:**
El servicio Auth deberá bloquear de forma terminante cualquier intento de asignar el rol `ADMINISTRADOR` o permisos de superusuario (`*`) a una cuenta de personal operativo a través de las APIs de gestión de personal.

**Tipo:** Funcional  
**Fuente:** Principio de Mínimo Privilegio y Seguridad FMAT  
**Verificación:** Prueba: Enviar una petición a los endpoints de creación/edición de personal incluyendo `ADMINISTRADOR` en la lista de roles; constatar que la API retorne un error HTTP `400 Bad Request` denegando la asignación.  
**Estado:** Confirmado  
**Reglas Relacionadas:** BR-AUTH-002, INV-AUTH-001, INV-AUTH-002  

---

<a id="req-auth-009"></a>

### REQ-AUTH-009 — Autenticación de Personal con Staff ID y Contraseña

**Requisito:**
El servicio Auth deberá permitir al personal operativo iniciar sesión proporcionando su identificador `staffId` (`XYYYYYY`) y su contraseña (temporal o personal), retornando los tokens criptográficos de sesión correspondientes.

**Tipo:** Funcional  
**Fuente:** Especificación de Terminales POS / KDS FMAT  
**Verificación:** Prueba: Enviar solicitudes de inicio de sesión con identificador `XYYYYYY` válido y contraseña correcta, verificando que se autentique exitosamente al usuario en el sistema.  
**Estado:** Confirmado  
**Reglas Relacionadas:** BR-AUTH-004, BR-AUTH-006, BR-AUTH-007, INV-AUTH-003  

---

<a id="req-auth-010"></a>

### REQ-AUTH-010 — Detección y Notificación de Contraseña Temporal

**Requisito:**
Al autenticarse un usuario cuyo estado de credencial sea `TEMPORARY`, el servicio Auth deberá emitir un token JWT con la alegación `"mustChangePassword": true` y una respuesta JSON indicando explícitamente la obligatoriedad de cambio de contraseña previo a la operación normal.

**Tipo:** Funcional  
**Fuente:** Flujo de Primer Ingreso Seguro FMAT  
**Verificación:** Prueba: Autenticar a un usuario con contraseña temporal recién asignada; inspeccionar la respuesta y el payload del JWT emitido confirmando que `mustChangePassword` se encuentre en `true`.  
**Estado:** Confirmado  
**Reglas Relacionadas:** BR-AUTH-007, BR-AUTH-008, INV-AUTH-005  

---

<a id="req-auth-011"></a>

### REQ-AUTH-011 — Cambio de Contraseña Inicial y Preservación de Identidad

**Requisito:**
El servicio Auth deberá proporcionar un endpoint para cambiar la contraseña temporal por una contraseña personal definitiva, validando la clave anterior, actualizando el hash criptográfico a `passwordStatus = ACTIVE`, emitiendo un nuevo JWT con `mustChangePassword = false` y preservando inmutable el identificador `staffId` asignado.

**Tipo:** Funcional  
**Fuente:** Flujo de Transición de Credencial FMAT  
**Verificación:** Prueba: Ejecutar el cambio de contraseña inicial exitoso; comprobar que el estado pase a `ACTIVE`, que la sesión admita operaciones normales y que el `staffId` permanezca idéntico al original.  
**Estado:** Confirmado  
**Reglas Relacionadas:** BR-AUTH-008, BR-AUTH-009, INV-AUTH-004, INV-AUTH-005  

---

<a id="req-auth-012"></a>

### REQ-AUTH-012 — Emisión y Firma Digital de JSON Web Tokens (JWT)

**Requisito:**
El servicio Auth deberá firmar digitalmente cada Access Token mediante un algoritmo criptográfico seguro (`HS256` con secreto de alta entropía), incluyendo en la carga útil las claims `sub`, `staffId`, `roles`, `permissions` y `mustChangePassword`.

**Tipo:** Funcional  
**Fuente:** Estándar de Integración de Tokens RFC 7519  
**Verificación:** Análisis: Decodificar y verificar la firma criptográfica de un token generado usando la clave correspondiente; verificar que ninguna claim obligatoria falte o sea nula.  
**Estado:** Confirmado  
**Reglas Relacionadas:** BR-AUTH-012, BR-AUTH-013, INV-AUTH-006  

---

<a id="req-auth-013"></a>

### REQ-AUTH-013 — Renovación de Sesión mediante Refresh Token

**Requisito:**
El servicio Auth deberá proveer un mecanismo de renovación de Access Tokens mediante `RefreshToken` rotativos almacenados con dispersión hash en base de datos, revocables de forma inmediata ante desvinculación o cierre de sesión explícito.

**Tipo:** Funcional  
**Fuente:** Gestión de Sesiones Seguras FMAT  
**Verificación:** Prueba: Presentar un `refreshToken` válido ante el endpoint de rotación; comprobar que se devuelva un nuevo par de tokens y que el `refreshToken` anterior sea invalidado.  
**Estado:** Confirmado  
**Reglas Relacionadas:** BR-AUTH-014, INV-AUTH-007  

---

<a id="req-auth-014"></a>

### REQ-AUTH-014 — Consulta de Identidad, Permisos y Vistas (`/auth/me`)

**Requisito:**
El servicio Auth deberá exponer un endpoint autenticado `GET /api/v1/auth/me` que retorne los datos del usuario en sesión, sus roles, el estado de su contraseña y la lista de permisos granulares activos para que el Frontend renderice dinámicamente las vistas y botones autorizados.

**Tipo:** Funcional  
**Fuente:** Consumo de Dashboard Frontend FMAT  
**Verificación:** Demostración: Consultar `/auth/me` con tokens de usuarios con distintos perfiles; contrastar los permisos y vistas retornadas contra la matriz de autorización configurada.  
**Estado:** Confirmado  
**Reglas Relacionadas:** BR-AUTH-012  

---

<a id="req-auth-015"></a>

### REQ-AUTH-015 — Modificación de Roles y Permisos de Personal

**Requisito:**
El servicio Auth deberá permitir al Administrador actualizar los datos, roles y permisos de cualquier empleado del restaurante desde el expediente lateral, garantizando que conserve al menos un rol activo y publicando el evento `StaffRolesUpdated`.

**Tipo:** Funcional  
**Fuente:** Operación Administrativa de Turnos y Roles Dinámicos FMAT  
**Verificación:** Prueba: Actualizar los roles/permisos de un empleado; verificar la actualización en base de datos y la emisión del evento a RabbitMQ.  
**Estado:** Confirmado  
**Reglas Relacionadas:** BR-AUTH-010, BR-AUTH-011, BR-AUTH-015, INV-AUTH-001  

---

<a id="req-auth-016"></a>

### REQ-AUTH-016 — Desactivación y Eliminación Protegida de Personal

**Requisito:**
El servicio Auth deberá permitir al Administrador cambiar el estado operativo de un miembro del personal a inactivo (`isActive = false`) o eliminarlo del sistema, provocando la revocación inmediata de sus sesiones activas e impidiendo nuevos inicios de sesión.

**Tipo:** Funcional  
**Fuente:** Gestión de Bajas de Personal FMAT  
**Verificación:** Prueba: Desactivar o eliminar a un empleado activo; comprobar que sus refresh tokens se revoquen y que un intento subsiguiente de login sea rechazado con `401 Unauthorized` o `403 Forbidden`.  
**Estado:** Confirmado  
**Reglas Relacionadas:** BR-AUTH-014, BR-AUTH-015  

---

<a id="req-auth-017"></a>

### REQ-AUTH-017 — Emisión de Eventos de Dominio en RabbitMQ

**Requisito:**
El servicio Auth deberá publicar eventos estructurados bajo especificación CloudEvents en el broker RabbitMQ ante la creación de personal (`StaffMemberCreated`), cambio de roles (`StaffRolesUpdated`) y cambio de estado (`StaffStatusChanged`), conteniendo el `staffId`, nombre y roles asociados (sin identificadores multi-tenant).

**Tipo:** Funcional  
**Fuente:** Integración Asíncrona de Ecosistema FMAT  
**Verificación:** Prueba: Crear un miembro del personal en Auth y consumir el mensaje desde una cola de prueba en RabbitMQ; verificar que el payload contenga la lista de roles con el routing key correspondiente.  
**Estado:** Confirmado  
**Reglas Relacionadas:** BR-AUTH-015  

---

<a id="req-auth-018"></a>

### REQ-AUTH-018 — Bloqueo Perimetral de Acceso en API Gateway por Contraseña Temporal

**Requisito:**
El API Gateway del sistema deberá interceptar cada petición HTTP entrante; si el token JWT contiene la marca `mustChangePassword: true`, denegará el acceso hacia cualquier microservicio operacional (`Menu`, `Orders`, `Sala`, `Inventory`, `Billing`) respondiendo HTTP `403 Forbidden` con código `PASSWORD_CHANGE_REQUIRED`.

**Tipo:** Funcional / Perimetral  
**Fuente:** Política de Seguridad y Contención Perimetral FMAT  
**Verificación:** Prueba: Emitir una petición con Bearer token temporal hacia `GET /api/v1/inventory/ingredients`; verificar que el API Gateway retorne HTTP 403 sin reenviar la petición al backend de inventario.  
**Estado:** Confirmado  
**Reglas Relacionadas:** BR-AUTH-008, INV-AUTH-005  

---

<a id="req-auth-019"></a>

### REQ-AUTH-019 — Sanitización e Inyección de Cabeceras de Contexto en API Gateway

**Requisito:**
El API Gateway deberá eliminar cualquier cabecera `X-User-*` proveniente del cliente público antes de enrutar peticiones a la red interna de microservicios, e inyectar cabeceras autenticadas confiables (`X-User-Id`, `X-Staff-Id`, `X-User-Roles`) extraídas del JWT verificado.

**Tipo:** Funcional / Perimetral  
**Fuente:** Prevención de Spoofing en Arquitecturas de Microservicios  
**Verificación:** Prueba: Enviar una petición HTTP inyectando deliberadamente `X-User-Roles: ADMINISTRADOR` con un token de empleado ordinario; inspeccionar en el microservicio destino que la cabecera recibida sea exclusivamente la lista de roles legítima del usuario.  
**Estado:** Confirmado  
**Reglas Relacionadas:** BR-AUTH-013, INV-AUTH-006  

---

<a id="req-auth-020"></a>

### REQ-AUTH-020 — Restablecimiento Administrativo de Contraseña Temporal

**Requisito:**
El servicio Auth deberá permitir al Administrador restablecer la contraseña de un empleado que la haya olvidado, asignando una nueva contraseña temporal generada por el backend y revirtiendo su credencial al estado `passwordStatus = TEMPORARY` con forzado de cambio en su siguiente ingreso.

**Tipo:** Funcional  
**Fuente:** Soporte Operativo FMAT  
**Verificación:** Demostración: Desde la consola de administrador, ejecutar "Restablecer contraseña" sobre un empleado con contraseña activa; comprobar que su estado retorne a `TEMPORARY` y que sus sesiones activas se revoquen.  
**Estado:** Confirmado  
**Reglas Relacionadas:** BR-AUTH-007, BR-AUTH-008, INV-AUTH-005  

---

<a id="req-auth-021"></a>

### REQ-AUTH-021 — Restricción de Actualización y Eliminación de Perfil por Tipo de Usuario

**Requisito:**
El servicio Auth deberá permitir únicamente a los usuarios con tipo `ADMIN` actualizar sus datos de perfil (nombre, apellido) o eliminar cuentas. Cualquier intento de actualización o eliminación de cuenta por parte de usuarios de tipo `STAFF` deberá ser rechazado con HTTP `403 Forbidden`, ya que los datos del personal operativo solo pueden ser gestionados administrativamente.

**Tipo:** Funcional  
**Fuente:** Regla de Negocio FMAT (BR-AUTH-016)  
**Verificación:** Prueba: Intentar actualizar o eliminar el perfil autenticado como `STAFF`; verificar respuesta HTTP 403. Autenticado como `ADMIN`, verificar actualización exitosa HTTP 200.  
**Estado:** Confirmado  
**Reglas Relacionadas:** BR-AUTH-016  

---

<a id="req-auth-022"></a>

### REQ-AUTH-022 — Endpoint de Verificación de Salud del Servicio (`/health`)

**Requisito:**
El servicio Auth deberá exponer un endpoint público `GET /health` que informe el estado de operatividad del microservicio (status `ok`, timestamp, uptime) para monitorización y healthchecks de orquestación (Docker, Kubernetes).

**Tipo:** No Funcional / Operacional  
**Fuente:** Requisitos de Observabilidad FMAT  
**Verificación:** Prueba: Realizar una petición `GET /health`; comprobar respuesta HTTP 200 con `{ status: "ok" }`.  
**Estado:** Confirmado  
**Reglas Relacionadas:** N/A  

---

<a id="req-auth-023"></a>

### REQ-AUTH-023 — Vista Unificada de Autenticación con Pestañas de Personal y Gerente

**Requisito:**
La interfaz de usuario de Auth (`Auth-Frontend`) deberá presentar una vista unificada con selector entre **Personal (Staff ID)** y **Gerente** (inicio de sesión y configuración inicial de administrador único si aún no se ha registrado).

**Tipo:** Funcional / UI  
**Fuente:** Especificación de Vistas de Acceso FMAT / Guía de Componentes v2.0  
**Verificación:** Demostración: Navegar a la pantalla de Auth; alternar entre las pestañas comprobando que cada una renderice su formulario correspondiente sin recargar la página.  
**Estado:** Confirmado  
**Reglas Relacionadas:** BR-AUTH-001, BR-AUTH-004  

---

<a id="req-auth-024"></a>

### REQ-AUTH-024 — Flujo Modal de Cambio Obligatorio de Contraseña en Primer Ingreso

**Requisito:**
Cuando un miembro del personal inicie sesión con su Staff ID y contraseña temporal (`passwordStatus = TEMPORARY`), la interfaz de usuario deberá desplegar inmediatamente un modal bloqueante que requiera al usuario introducir su nueva contraseña personal definitiva y su confirmación antes de permitir el acceso al dashboard.

**Tipo:** Funcional / UI  
**Fuente:** Flujo de Seguridad y Primer Ingreso FMAT  
**Verificación:** Demostración: Iniciar sesión con credenciales temporales; comprobar que se despliegue el modal de cambio de clave y que, tras el cambio exitoso, se cierre el modal y se redirija al Dashboard.  
**Estado:** Confirmado  
**Reglas Relacionadas:** BR-AUTH-007, BR-AUTH-008, BR-AUTH-009, INV-AUTH-005  

---

<a id="req-auth-025"></a>

### REQ-AUTH-025 — Adherencia al Sistema de Diseño FMAT UI Kit v2.0 y Modo Oscuro

**Requisito:**
Todas las pantallas, modales, drawers laterales, formularios y controles interactivos de `Auth-Frontend` deberán cumplir los tokens visuales del **FMAT UI Kit (Versión 2.0)** en Modo Claro y la paleta *Warm Obsidian Espresso & Burnt Copper* en Modo Oscuro.

**Tipo:** No Funcional / UI & Usabilidad  
**Fuente:** Guía Visual de Componentes FMAT Restaurant v2.0  
**Verificación:** Inspección: Contrastar las hojas de estilos CSS y componentes de `Auth-Frontend` con los tokens definidos en claro y oscuro.  
**Estado:** Confirmado  
**Reglas Relacionadas:** DEC-AUTH-002  
