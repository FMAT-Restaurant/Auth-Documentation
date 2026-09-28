# Requisitos Funcionales del Servicio Auth

Cada bloque establece una obligación primaria formal del sistema. Las verificaciones siguen los cuatro métodos formales de la disciplina de *Verificación y Validación*: **Inspección**, **Demostración**, **Prueba** y **Análisis**.

---

<a id="req-auth-001"></a>

### REQ-AUTH-001 — Registro de Restaurante y Cuenta de Administrador

**Requisito:**
El servicio Auth deberá permitir el registro de un nuevo establecimiento gastronómico (`Restaurant`) proporcionando nombre comercial, razón social, dirección y correo electrónico corporativo, creando de forma atómica la cuenta raíz del Gerente con tipo de usuario `ADMIN` y facultades plenas de superusuario (`*`).

**Tipo:** Funcional  
**Fuente:** Especificación de Roles Softrestaurant FMAT  
**Verificación:** Prueba: Ejecutar el registro de un nuevo restaurante con datos válidos; comprobar que se genera el registro del tenant, que el usuario resultante posee el tipo `ADMIN` y permisos universales, y rechazar intentos con correo duplicado.  
**Estado:** Confirmado  
**Reglas Relacionadas:** BR-AUTH-001, BR-AUTH-002, INV-AUTH-002, INV-AUTH-006  

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

### REQ-AUTH-003 — Creación de Perfil de Personal con Etiqueta y Permisos

**Requisito:**
El servicio Auth deberá permitir a un usuario con rol `ADMIN` registrar a un nuevo miembro del equipo operativo (`StaffMember`), capturando nombres, apellidos, teléfono de contacto, una etiqueta descriptiva de rol no precodificada (ej. *"Líder de inventario"*, *"Mesero terraza"*) y la selección explícita de sus permisos granulares por microservicio.

**Tipo:** Funcional  
**Fuente:** Requisitos de Gestión de Personal Softrestaurant FMAT  
**Verificación:** Demostración: Desde una sesión de administrador, dar de alta a un miembro del personal asignándole la etiqueta `"Líder de inventario"` y permisos de ingredientes y stock; verificar que su perfil quede persistido en la base de datos vinculado al `restaurantId` del administrador ejecutor.  
**Estado:** Confirmado  
**Reglas Relacionadas:** BR-AUTH-003, BR-AUTH-010, INV-AUTH-001, INV-AUTH-006  

---

<a id="req-auth-004"></a>

### REQ-AUTH-004 — Generación Automática del Identificador `XYYYYYY`

**Requisito:**
El servicio Auth deberá asignar automáticamente a cada nuevo miembro del personal un identificador unívoco e inmutable con formato `XYYYYYY` con prefijo canónico `E` (`E000001` a `E999999`), garantizando su unicidad dentro del restaurante y manteniéndolo desacoplado de las etiquetas de rol.

**Tipo:** Funcional  
**Fuente:** Especificación de Identificadores de Personal FMAT  
**Verificación:** Prueba: Registrar múltiples miembros de personal consecutivamente; inspeccionar que todos cumplan con la expresión regular `^[A-Z][0-9]{6}$` con prefijo `E` y que no se produzcan colisiones en el identificador dentro del mismo restaurante.  
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

### REQ-AUTH-006 — Asignación Obligatoria de Permisos y Etiqueta de Rol

**Requisito:**
El servicio Auth deberá exigir la asignación de al menos un (1) permiso granular válido del catálogo del sistema y una etiqueta descriptiva no vacía al crear o editar personal, rechazando solicitudes con colecciones de permisos vacías o nulas.

**Tipo:** Funcional  
**Fuente:** Modelo de Control de Acceso PBAC FMAT  
**Verificación:** Prueba: Enviar peticiones de creación con lista de permisos vacía (`[]`) o sin etiqueta de rol, validando que el servicio rechace la operación con código HTTP `422 Unprocessable Entity` o `400 Bad Request`.  
**Estado:** Confirmado  
**Reglas Relacionadas:** BR-AUTH-010, INV-AUTH-001  

---

<a id="req-auth-007"></a>

### REQ-AUTH-007 — Soporte para Asignación Transversal y Granular de Permisos

**Requisito:**
El servicio Auth deberá permitir al Administrador configurar cualquier combinación de permisos granulares por microservicio para un miembro del personal (por ejemplo: agregar ingredientes, eliminar ingredientes y cambiar estado de stock en Inventario, y/o abrir comandas en Órdenes), otorgándole acceso exacto a las funciones y botones habilitados.

**Tipo:** Funcional  
**Fuente:** Requisitos Operativos y Flexibilidad Administrativa FMAT  
**Verificación:** Demostración: Crear al empleado Juan Pérez con rol *"Líder de inventario"* y permisos `inventory:view`, `inventory:ingredients:create`, `inventory:ingredients:delete`, `inventory:stock:update_status`; autenticarlo y comprobar que el JWT retorne exactamente la lista de permisos configurada.  
**Estado:** Confirmado  
**Reglas Relacionadas:** BR-AUTH-011, BR-AUTH-012, INV-AUTH-001  

---

<a id="req-auth-008"></a>

### REQ-AUTH-008 — Exclusión de Facultades de Administrador en Personal

**Requisito:**
El servicio Auth deberá bloquear de forma terminante cualquier intento de asignar permisos de superusuario (`*`) o tipo `ADMIN` a una cuenta de personal operativo a través de las APIs de gestión de personal.

**Tipo:** Funcional  
**Fuente:** Principio de Mínimo Privilegio y Seguridad FMAT  
**Verificación:** Prueba: Enviar una petición a los endpoints de creación/edición de personal incluyendo `"*"` en la lista de permisos; constatar que la API retorne un error HTTP `400 Bad Request` denegando la asignación.  
**Estado:** Confirmado  
**Reglas Relacionadas:** BR-AUTH-002, INV-AUTH-001, INV-AUTH-002  

---

<a id="req-auth-009"></a>

### REQ-AUTH-009 — Autenticación de Personal con Staff ID y Contraseña

**Requisito:**
El servicio Auth deberá permitir al personal operativo iniciar sesión proporcionando su identificador `staffId` (`XYYYYYY`) y su contraseña (temporal o personal), retornando los tokens criptográficos de sesión correspondientes.

**Tipo:** Funcional  
**Fuente:** Especificación de Terminales POS / KDS / Tablets FMAT  
**Verificación:** Prueba: Enviar solicitudes de inicio de sesión con identificador `XYYYYYY` válido y contraseña correcta, verificando que se autentique exitosamente al usuario asociado al restaurante correspondiente.  
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
El servicio Auth deberá firmar digitalmente cada Access Token mediante un algoritmo criptográfico seguro (`RS256` con par de llaves asimétricas o `HS256` con secreto de alta entropía), incluyendo en la carga útil las claims `sub`, `restaurantId`, `staffId`, `roleLabel`, `permissions` y `mustChangePassword`.

**Tipo:** Funcional  
**Fuente:** Estándar de Integración de Tokens RFC 7519  
**Verificación:** Análisis: Decodificar y verificar la firma criptográfica de un token generado usando la clave pública correspondiente; verificar que contenga el arreglo `permissions` y el texto de `roleLabel`.  
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
El servicio Auth deberá exponer un endpoint autenticado `GET /api/v1/auth/me` que retorne los datos del usuario en sesión, su etiqueta de rol (`roleLabel`), el estado de su contraseña y la lista de permisos granulares activos para que el Frontend renderice dinámicamente las vistas y botones autorizados.

**Tipo:** Funcional  
**Fuente:** Consumo de Dashboard Frontend FMAT  
**Verificación:** Demostración: Consultar `/auth/me` con tokens de usuarios con distintos perfiles (ej. Líder de inventario vs. Mesero); contrastar los permisos y vistas retornadas contra la matriz de autorización configurada.  
**Estado:** Confirmado  
**Reglas Relacionadas:** BR-AUTH-012  

---

<a id="req-auth-015"></a>

### REQ-AUTH-015 — Modificación de Permisos y Etiqueta de Rol de Personal

**Requisito:**
El servicio Auth deberá permitir al Administrador actualizar la etiqueta de rol y la matriz de permisos de cualquier empleado de su restaurante, garantizando que conserve al menos un permiso activo y publicando de inmediato el evento `StaffPermissionsUpdated`.

**Tipo:** Funcional  
**Fuente:** Operación Administrativa de Turnos y Roles Dinámicos FMAT  
**Verificación:** Prueba: Actualizar los permisos de un empleado para concederle permisos adicionales de inventario; verificar la actualización en base de datos y la emisión del evento a RabbitMQ.  
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
El servicio Auth deberá publicar eventos estructurados bajo especificación CloudEvents en el broker RabbitMQ ante la creación de personal (`StaffMemberCreated`), cambio de permisos/etiqueta (`StaffPermissionsUpdated`) y cambio de estado (`StaffStatusChanged`), conteniendo el `restaurantId`, `staffId`, nombre, `roleLabel` y el arreglo de `permissions`.

**Tipo:** Funcional  
**Fuente:** Integración Asíncrona de Ecosistema FMAT  
**Verificación:** Prueba: Crear un miembro del personal en Auth y consumir el mensaje desde una cola de prueba en RabbitMQ; verificar que el payload contenga la etiqueta de rol y la lista de permisos con el routing key correspondiente.  
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
El API Gateway deberá eliminar cualquier cabecera `X-User-*` proveniente del cliente público antes de enrutar peticiones a la red interna, e inyectar cabeceras autenticadas confiables (`X-User-Id`, `X-Restaurant-Id`, `X-Staff-Id`, `X-User-Role-Label`, `X-User-Permissions`) extraídas del JWT verificado.

**Tipo:** Funcional / Perimetral  
**Fuente:** Prevención de Spoofing en Arquitecturas de Microservicios  
**Verificación:** Prueba: Enviar una petición HTTP inyectando deliberadamente `X-User-Permissions: *` con un token de empleado ordinario; inspeccionar en el microservicio destino que la cabecera recibida sea exclusivamente la lista de permisos legítima del usuario.  
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

---

<a id="req-auth-021"></a>

### REQ-AUTH-021 — Vista Unificada de Autenticación con Pestañas de Registro e Inicio de Sesión

**Requisito:**
La interfaz de usuario de Auth (`Auth-Frontend`) deberá presentar una vista unificada con un selector de dos secciones o pestañas principales:
1. **Registro:** Destinada a que un nuevo Gerente dé de alta su restaurante (nombre del restaurante, razón social, dirección) y cree de forma atómica su cuenta administrativa (nombre, correo corporativo, contraseña maestra).
2. **Inicio de Sesión:** Destinada al acceso cotidiano del sistema, mostrando por defecto el formulario de acceso para personal operativo mediante su Staff ID (`E000001` a `E999999`) y contraseña.

**Tipo:** Funcional / UI  
**Fuente:** Especificación de Vistas de Acceso FMAT / Guía de Componentes v2.0  
**Verificación:** Demostración: Navegar a la pantalla de Auth; alternar entre las pestañas "Iniciar Sesión" y "Registrar Restaurante", comprobando que cada una renderice su formulario correspondiente sin recargar la página.  
**Estado:** Confirmado  
**Reglas Relacionadas:** BR-AUTH-001, BR-AUTH-004  

---

<a id="req-auth-022"></a>

### REQ-AUTH-022 — Conmutador de Identidad para Administradores en Inicio de Sesión

**Requisito:**
Dentro de la pestaña de **Inicio de Sesión**, la interfaz deberá incorporar un enlace o acción secundaria con el texto *"¿Eres un administrador? Inicia sesión aquí"*. Al ser accionado, la interfaz conmutará dinámicamente el campo de entrada principal para permitir el ingreso mediante **Correo Electrónico Corporativo** en lugar del Staff ID, permitiendo al Administrador acceder con su correo y facilitando un enlace de retorno (*"¿Eres personal operativo? Ingresa con tu ID"*).

**Tipo:** Funcional / UI  
**Fuente:** Especificación de Experiencia de Usuario FMAT  
**Verificación:** Demostración: En la vista de login, hacer clic en "¿Eres un administrador?"; verificar que el label y validación del campo de entrada cambien de máscara `XYYYYYY` a formato de email corporativo (`usuario@restaurante.com`) y permitan autenticar satisfactoriamente a la cuenta administrativa.  
**Estado:** Confirmado  
**Reglas Relacionadas:** BR-AUTH-001, BR-AUTH-004  

---

<a id="req-auth-023"></a>

### REQ-AUTH-023 — Flujo Modal de Cambio Obligatorio de Contraseña en Primer Ingreso

**Requisito:**
Cuando un miembro del personal inicie sesión con su Staff ID y contraseña temporal (estado `passwordStatus = TEMPORARY`), la interfaz de usuario deberá contener perimetralmente la navegación y desplegar inmediatamente un modal de diálogo bloqueante (overlay translúcido y card blanca con radio de 12px) que requiera al usuario introducir: (1) Contraseña temporal provista por el gerente, (2) Nueva contraseña personal definitiva, y (3) Confirmación de la nueva contraseña, aplicando la validación de complejidad requerida antes de permitir el acceso al dashboard.

**Tipo:** Funcional / UI  
**Fuente:** Flujo de Seguridad y Primer Ingreso FMAT  
**Verificación:** Demostración: Iniciar sesión con credenciales temporales; comprobar que no se permita interactuar con el fondo, que se despliegue el modal de cambio de clave y que, tras el cambio exitoso, se cierre el modal y se redirija al Dashboard correspondiente a sus permisos.  
**Estado:** Confirmado  
**Reglas Relacionadas:** BR-AUTH-007, BR-AUTH-008, BR-AUTH-009, INV-AUTH-005  

---

<a id="req-auth-024"></a>

### REQ-AUTH-024 — Adherencia Estricta al Sistema de Diseño FMAT UI Kit v2.0

**Requisito:**
Todas las pantallas, modales, formularios y controles interactivos de `Auth-Frontend` deberán ser implementados cumpliendo de manera estricta los tokens visuales y directrices del **FMAT UI Kit (Versión 2.0)**:
1. **Paleta Cromática:** Botones de acción principal en `#C2410C` (hover `#9A3412`), acentos en `#F97316`, selección suave en `#FFF7ED`, fondo de pantalla en `#F8FAFC`, superficie de contenedores en `#FFFFFF`, bordes en `#E5E7EB` y textos en `#111827` / `#374151`.
2. **Proporción Visual:** 70% blanco y fondos neutros, 20% texto, bordes y estructura, y 10% naranja y estados.
3. **Inputs y Formularios:** Etiquetas fijas visibles sobre el campo (sin utilizar placeholders como etiquetas), mensajes de error explícitos adyacentes al campo con borde rojo, y estados de foco con borde primario.
4. **Radios y Dimensiones:** `--radius-control: 8px`, `--radius-card: 12px`, altura de controles de 40px (MD) y 48px para entornos táctiles (LG).

**Tipo:** No Funcional / UI & Usabilidad  
**Fuente:** Guía Visual de Componentes FMAT Restaurant v2.0  
**Verificación:** Inspección: Contrastar las hojas de estilos CSS/Tailwind y componentes de `Auth-Frontend` con la tabla de tokens del FMAT UI Kit v2.0; verificar que ningún color ni radio arbitrario haya sido introducido.  
**Estado:** Confirmado  
**Reglas Relacionadas:** DEC-AUTH-002  
