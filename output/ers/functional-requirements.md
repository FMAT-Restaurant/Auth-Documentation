# Requisitos Funcionales del Servicio Auth

Cada bloque establece una obligación primaria formal del sistema. Las verificaciones siguen los cuatro métodos formales de la disciplina de *Verificación y Validación*: **Inspección**, **Demostración**, **Prueba** y **Análisis**.

---

<a id="req-auth-001"></a>

### REQ-AUTH-001 — Registro de Restaurante y Cuenta de Administrador

**Requisito:**
El servicio Auth deberá permitir el registro de un nuevo establecimiento gastronómico (`Restaurant`) proporcionando nombre comercial, razón social, dirección y correo electrónico corporativo, creando de forma atómica la cuenta raíz del Gerente con el rol exclusivo `ADMINISTRADOR`.

**Tipo:** Funcional  
**Fuente:** Especificación de Roles Softrestaurant FMAT  
**Verificación:** Prueba: Ejecutar el registro de un nuevo restaurante con datos válidos; comprobar que se genera el registro del tenant, que el usuario resultante posee el rol `ADMINISTRADOR`, y rechazar intentos con correo duplicado.  
**Estado:** Confirmado  
**Reglas Relacionadas:** BR-AUTH-001, BR-AUTH-002, INV-AUTH-002, INV-AUTH-006  

---

<a id="req-auth-002"></a>

### REQ-AUTH-002 — Autenticación del Gerente / Administrador

**Requisito:**
El servicio Auth deberá validar las credenciales de acceso del Administrador mediante su correo electrónico y contraseña maestra, retornando un par de tokens criptográficos (`accessToken` y `refreshToken`) con alcance total de administración.

**Tipo:** Funcional  
**Fuente:** Modelo de Acceso Administrativo FMAT  
**Verificación:** Prueba: Probar autenticación con credenciales correctas e incorrectas, validando que solo en caso de éxito se retorne un JWT firmado con el rol `ADMINISTRADOR` y `mustChangePassword = false`.  
**Estado:** Confirmado  
**Reglas Relacionadas:** BR-AUTH-001, INV-AUTH-002, INV-AUTH-007  

---

<a id="req-auth-003"></a>

### REQ-AUTH-003 — Creación de Perfil de Personal

**Requisito:**
El servicio Auth deberá permitir a un usuario con rol `ADMINISTRADOR` registrar a un nuevo miembro del equipo operativo (`StaffMember`), capturando nombres, apellidos, teléfono de contacto y al menos un rol operativo inicial.

**Tipo:** Funcional  
**Fuente:** Requisitos de Gestión de Personal Softrestaurant  
**Verificación:** Demostración: Desde una sesión de administrador, dar de alta a un miembro del personal y verificar que su perfil quede persistido en la base de datos vinculado al `restaurantId` del administrador ejecutor.  
**Estado:** Confirmado  
**Reglas Relacionadas:** BR-AUTH-003, BR-AUTH-010, INV-AUTH-001, INV-AUTH-006  

---

<a id="req-auth-004"></a>

### REQ-AUTH-004 — Generación Automática del Identificador `XYYYYYY`

**Requisito:**
El servicio Auth deberá asignar automáticamente a cada nuevo miembro del personal un identificador unívoco e inmutable con formato `XYYYYYY`, donde `X` es un carácter alfabético en mayúscula y `YYYYYY` es un correlativo numérico de seis dígitos, garantizando su unicidad dentro del restaurante.

**Tipo:** Funcional  
**Fuente:** Especificación de Identificadores de Personal FMAT  
**Verificación:** Prueba: Registrar múltiples miembros de personal consecutivamente; inspeccionar que todos cumplan con la expresión regular `^[A-Z][0-9]{6}$` y que no se produzcan colisiones en el identificador dentro del mismo restaurante.  
**Estado:** Confirmado  
**Reglas Relacionadas:** BR-AUTH-004, BR-AUTH-005, BR-AUTH-006, INV-AUTH-003, INV-AUTH-004  

---

<a id="req-auth-005"></a>

### REQ-AUTH-005 — Asignación y Estado de Contraseña Temporal

**Requisito:**
El servicio Auth deberá asignar a todo nuevo miembro del personal una contraseña temporal configurada por el administrador o generada automáticamente por el sistema, marcando su credencial con el estado `passwordStatus = TEMPORARY` (`mustChangePassword = true`).

**Tipo:** Funcional  
**Fuente:** Especificación de Seguridad y Primer Ingreso FMAT  
**Verificación:** Prueba: Crear un nuevo miembro del personal, inspeccionar la base de datos para confirmar que `password_status == 'TEMPORARY'` y verificar que el hash almacenado corresponda a la contraseña temporal provista.  
**Estado:** Confirmado  
**Reglas Relacionadas:** BR-AUTH-007, BR-AUTH-008, INV-AUTH-005, INV-AUTH-007  

---

<a id="req-auth-006"></a>

### REQ-AUTH-006 — Asignación Obligatoria de Roles al Personal

**Requisito:**
El servicio Auth deberá exigir la asignación de al menos un rol operativo válido (`HOST`, `ALMACENISTA`, `MESERO`, `CHEF_MASTER`) al momento de crear o editar a un miembro del personal, rechazando solicitudes con colecciones de roles vacías.

**Tipo:** Funcional  
**Fuente:** Modelo de Control de Acceso RBAC FMAT  
**Verificación:** Prueba: Enviar peticiones de creación con lista de roles vacía (`[]`) o nula, validando que el servicio rechace la operación con código HTTP `422 Unprocessable Entity` o `400 Bad Request`.  
**Estado:** Confirmado  
**Reglas Relacionadas:** BR-AUTH-010, INV-AUTH-001  

---

<a id="req-auth-007"></a>

### REQ-AUTH-007 — Soporte para Asignación Multi-Rol

**Requisito:**
El servicio Auth deberá permitir la asignación simultánea de múltiples roles operativos a un mismo miembro del personal (por ejemplo, `MESERO` y `HOST` simultáneamente), otorgándole acceso acumulativo a las vistas y operaciones correspondientes a todos sus roles.

**Tipo:** Funcional  
**Fuente:** Requisitos Operativos de Flexibilidad FMAT  
**Verificación:** Demostración: Crear un empleado asignándole los roles `MESERO` y `HOST`; autenticarlo y comprobar que el JWT retornado contenga ambos roles en el arreglo `roles: ["MESERO", "HOST"]`.  
**Estado:** Confirmado  
**Reglas Relacionadas:** BR-AUTH-011, BR-AUTH-012, INV-AUTH-001  

---

<a id="req-auth-008"></a>

### REQ-AUTH-008 — Exclusión del Rol Administrador en Nómina Operativa

**Requisito:**
El servicio Auth deberá bloquear de forma terminante cualquier intento de asignar el rol `ADMINISTRADOR` a una cuenta de personal operativo a través de las APIs de gestión de miembros de equipo.

**Tipo:** Funcional  
**Fuente:** Principio de Mínimo Privilegio y Seguridad FMAT  
**Verificación:** Prueba: Enviar una petición a los endpoints de creación/edición de personal incluyendo `ADMINISTRADOR` en el payload; constatar que la API retorne un error HTTP `400 Bad Request` denegando la asignación.  
**Estado:** Confirmado  
**Reglas Relacionadas:** BR-AUTH-002, INV-AUTH-001, INV-AUTH-002  

---

<a id="req-auth-009"></a>

### REQ-AUTH-009 — Autenticación de Personal con Staff ID y Contraseña

**Requisito:**
El servicio Auth deberá permitir al personal operativo iniciar sesión proporcionando su identificador `staffId` (`XYYYYYY`) y su contraseña (temporal o personal), retornando los tokens criptográficos de sesión correspondientes.

**Tipo:** Funcional  
**Fuente:** Especificación de Terminales POS / KDS FMAT  
**Verificación:** Prueba: Enviar solicitudes de inicio de sesión con identificador `XYYYYYY` válido y contraseña correcta, verificando que se autentique exitosamente al usuario asociado al restaurante correspondiente.  
**Estado:** Confirmado  
**Reglas Relacionadas:** BR-AUTH-004, BR-AUTH-006, BR-AUTH-007, INV-AUTH-003  

---

<a id="req-auth-010"></a>

### REQ-AUTH-010 — Detección y Notificación de Contraseña Temporal

**Requisito:**
Al autenticarse un usuario cuyo estado de credencial sea `TEMPORARY`, el servicio Auth deberá emitir un token JWT con la alegación `"mustChangePassword": true` y una respuesta JSON indicando explícitamente la obligatoriedad de cambio de contraseña previo a la operación.

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
El servicio Auth deberá firmar digitalmente cada Access Token mediante un algoritmo criptográfico seguro (`RS256` con par de llaves asimétricas o `HS256` con secreto de alta entropía), incluyendo en la carga útil las claims `sub`, `restaurantId`, `staffId`, `roles` y `mustChangePassword`.

**Tipo:** Funcional  
**Fuente:** Estándar de Integración de Tokens RFC 7519  
**Verificación:** Análisis: Decodificar y verificar la firma criptográfica de un token generado usando la clave pública correspondiente; verificar que ninguna claim obligatoria falte o sea nula.  
**Estado:** Confirmado  
**Reglas Relacionadas:** BR-AUTH-013, INV-AUTH-006  

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

### REQ-AUTH-014 — Consulta de Identidad y Vistas Autorizadas (`/auth/me`)

**Requisito:**
El servicio Auth deberá exponer un endpoint autenticado `GET /api/v1/auth/me` que retorne los datos del usuario en sesión, su lista de roles asignados, el estado de su contraseña y la lista consolidada de identificadores de vista habilitados para renderizar el Dashboard.

**Tipo:** Funcional  
**Fuente:** Consumo de Dashboard Frontend FMAT  
**Verificación:** Demostración: Consultar `/auth/me` con tokens de usuarios con distintos roles (`ADMINISTRADOR`, `MESERO`, `HOST + MESERO`); contrastar las vistas retornadas contra la matriz de autorización establecida.  
**Estado:** Confirmado  
**Reglas Relacionadas:** BR-AUTH-012  

---

<a id="req-auth-015"></a>

### REQ-AUTH-015 — Modificación de Roles de Personal Existente

**Requisito:**
El servicio Auth deberá permitir al Administrador modificar la lista de roles asignados a un miembro de su personal, garantizando que el usuario conserve al menos un rol operativo y publicando de inmediato el evento correspondiente.

**Tipo:** Funcional  
**Fuente:** Operación Administrativa de Turnos FMAT  
**Verificación:** Prueba: Actualizar los roles de un mesero para asignarle también el rol `HOST`; verificar la actualización en base de datos y la emisión del evento `StaffRolesUpdated`.  
**Estado:** Confirmado  
**Reglas Relacionadas:** BR-AUTH-010, BR-AUTH-011, BR-AUTH-015, INV-AUTH-001  

---

<a id="req-auth-016"></a>

### REQ-AUTH-016 — Desactivación de Personal e Invalidación Inmediata

**Requisito:**
El servicio Auth deberá permitir al Administrador cambiar el estado operativo de un miembro del personal a inactivo (`isActive = false`), provocando la revocación inmediata de sus sesiones activas e impidiendo nuevos inicios de sesión.

**Tipo:** Funcional  
**Fuente:** Gestión de Bajas de Personal FMAT  
**Verificación:** Prueba: Desactivar a un empleado activo; comprobar que sus refresh tokens se marquen como revocados y que un intento subsiguiente de login sea rechazado con `401 Unauthorized` o `403 Forbidden`.  
**Estado:** Confirmado  
**Reglas Relacionadas:** BR-AUTH-014, BR-AUTH-015  

---

<a id="req-auth-017"></a>

### REQ-AUTH-017 — Emisión de Eventos de Dominio en RabbitMQ

**Requisito:**
El servicio Auth deberá publicar eventos estructurados bajo especificación CloudEvents en el broker RabbitMQ ante la creación de personal (`StaffMemberCreated`), cambio de roles (`StaffRolesUpdated`) y cambio de estado (`StaffStatusChanged`), conteniendo el `restaurantId`, `staffId`, nombre y roles asociados.

**Tipo:** Funcional  
**Fuente:** Integración Asíncrona de Ecosistema FMAT  
**Verificación:** Prueba: Crear un miembro del personal en Auth y consumir el mensaje desde una cola de prueba en RabbitMQ; verificar que el payload contenga todos los atributos esperados y el routing key correcto.  
**Estado:** Confirmado  
**Reglas Relacionadas:** BR-AUTH-015  

---

<a id="req-auth-018"></a>

### REQ-AUTH-018 — Bloqueo Perimetral de Acceso en API Gateway por Contraseña Temporal

**Requisito:**
El API Gateway del sistema deberá interceptar cada petición HTTP entrante; si el token JWT contiene la marca `mustChangePassword: true`, denegará el acceso hacia cualquier microservicio operacional (`Menu`, `Orders`, `Sala`, `Inventory`, `Billing`) respondiendo HTTP `403 Forbidden` con código `PASSWORD_CHANGE_REQUIRED`.

**Tipo:** Funcional / Perimetral  
**Fuente:** Política de Seguridad y Contención Perimetral FMAT  
**Verificación:** Prueba: Emitir una petición con Bearer token temporal hacia `GET /api/v1/orders`; verificar que el API Gateway retorne HTTP 403 sin reenviar la petición al backend de órdenes.  
**Estado:** Confirmado  
**Reglas Relacionadas:** BR-AUTH-008, INV-AUTH-005  

---

<a id="req-auth-019"></a>

### REQ-AUTH-019 — Sanitización e Inyección de Cabeceras de Contexto en API Gateway

**Requisito:**
El API Gateway deberá eliminar cualquier cabecera `X-User-*` proveniente del cliente público antes de enrutar peticiones a la red interna de microservicios, e inyectar cabeceras autenticadas confiables (`X-User-Id`, `X-Restaurant-Id`, `X-Staff-Id`, `X-User-Roles`) extraídas del JWT verificado.

**Tipo:** Funcional / Perimetral  
**Fuente:** Prevención de Spoofing en Arquitecturas de Microservicios  
**Verificación:** Prueba: Enviar una petición HTTP inyectando deliberadamente `X-User-Roles: ADMINISTRADOR` con un token de mesero; inspeccionar en el microservicio destino que la cabecera recibida sea exclusivamente `X-User-Roles: MESERO`.  
**Estado:** Confirmado  
**Reglas Relacionadas:** BR-AUTH-013, INV-AUTH-006  

---

<a id="req-auth-020"></a>

### REQ-AUTH-020 — Restablecimiento Administrativo de Contraseña Temporal

**Requisito:**
El servicio Auth deberá permitir al Administrador restablecer la contraseña de un empleado que la haya olvidado, asignando una nueva contraseña temporal y revirtiendo su credencial al estado `passwordStatus = TEMPORARY` con forzado de cambio en su siguiente ingreso.

**Tipo:** Funcional  
**Fuente:** Soporte Operativo FMAT  
**Verificación:** Demostración: Desde la consola de administrador, ejecutar "Restablecer contraseña" sobre un empleado con contraseña activa; comprobar que su estado retorne a `TEMPORARY` y que sus sesiones activas se revoquen.  
**Estado:** Confirmado  
**Reglas Relacionadas:** BR-AUTH-007, BR-AUTH-008, INV-AUTH-005  
