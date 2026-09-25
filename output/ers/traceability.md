# Matriz de Trazabilidad de Requisitos y Verificación

Esta matriz vincula cada requisito funcional con sus reglas de negocio asociadas, las interfaces de implementación (endpoints HTTP y eventos RabbitMQ) y los métodos formales de verificación de la disciplina de *Verificación y Validación*.

| ID Requisito | Título del Requisito | Reglas / Invariantes Asociadas | Endpoint / Operación Técnica | Evento Asíncrono (RabbitMQ) | Método de V&V | Criterio de Aceptación / Verificación |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **REQ-AUTH-001** | Registro de Restaurante y Admin | BR-001, BR-002, INV-002, INV-006 | `POST /api/v1/auth/register-restaurant` | — | **Prueba** | Creación de tenant y usuario con rol `ADMINISTRADOR` garantizado. |
| **REQ-AUTH-002** | Autenticación del Administrador | BR-001, INV-002, INV-007 | `POST /api/v1/auth/login` | — | **Prueba** | Retorno de JWT con rol `ADMINISTRADOR` y `mustChangePassword=false`. |
| **REQ-AUTH-003** | Creación de Perfil de Personal | BR-003, BR-010, INV-001, INV-006 | `POST /api/v1/staff` | `staff.created` | **Demostración** | Persistencia de empleado vinculado al tenant del administrador. |
| **REQ-AUTH-004** | Generación de ID `XYYYYYY` | BR-004, BR-005, BR-006, INV-003, INV-004 | `POST /api/v1/staff` (Lógica interna) | `staff.created` | **Prueba** | Comprobación de formato `^[A-Z][0-9]{6}$` y ausencia de colisiones. |
| **REQ-AUTH-005** | Asignación Contraseña Temporal | BR-007, BR-008, INV-005, INV-007 | `POST /api/v1/staff` | — | **Prueba** | Verificación de estado `password_status = 'TEMPORARY'`. |
| **REQ-AUTH-006** | Cardinalidad Mínima de Roles ($\ge 1$) | BR-010, INV-001 | `POST /api/v1/staff`, `PUT /api/v1/staff/:id/roles` | — | **Prueba** | Rechazo HTTP 422 ante listas de roles vacías o nulas. |
| **REQ-AUTH-007** | Soporte Multi-Rol | BR-011, BR-012, INV-001 | `POST /api/v1/staff` | `staff.roles.updated` | **Demostración** | JWT emitido contiene arreglo con múltiples roles operativos. |
| **REQ-AUTH-008** | Exclusión de Rol Administrador | BR-002, INV-001, INV-002 | `POST /api/v1/staff` | — | **Prueba** | Rechazo HTTP 400 ante intento de asignar `ADMINISTRADOR` a personal. |
| **REQ-AUTH-009** | Login de Personal con Staff ID | BR-004, BR-006, BR-007, INV-003 | `POST /api/v1/auth/login` | — | **Prueba** | Inicio de sesión exitoso resolviendo usuario por `staffId`. |
| **REQ-AUTH-010** | Detección Contraseña Temporal | BR-007, BR-008, INV-005 | `POST /api/v1/auth/login` | — | **Prueba** | JWT emitido incluye claim `"mustChangePassword": true`. |
| **REQ-AUTH-011** | Cambio de Contraseña Inicial | BR-008, BR-009, INV-004, INV-005 | `POST /api/v1/auth/change-initial-password` | — | **Prueba** | Estado pasa a `ACTIVE`, se emite nuevo JWT y se preserva `staffId`. |
| **REQ-AUTH-012** | Emisión y Firma de Tokens JWT | BR-013, INV-006 | Servicio de Tokenización JWT | — | **Análisis** | Verificación de firma criptográfica con clave pública y validación de claims. |
| **REQ-AUTH-013** | Renovación con Refresh Token | BR-014, INV-007 | `POST /api/v1/auth/refresh` | — | **Prueba** | Rotación exitosa de Refresh Token e invalidación del token previo. |
| **REQ-AUTH-014** | Consulta de Vistas (`/auth/me`) | BR-012 | `GET /api/v1/auth/me` | — | **Demostración** | Retorno consolidado de vistas UI asociadas a los roles activos. |
| **REQ-AUTH-015** | Modificación de Roles | BR-010, BR-011, BR-015, INV-001 | `PUT /api/v1/staff/:id/roles` | `staff.roles.updated` | **Prueba** | Actualización efectiva en BD y despacho del evento a RabbitMQ. |
| **REQ-AUTH-016** | Desactivación de Personal | BR-014, BR-015 | `PATCH /api/v1/staff/:id/status` | `staff.status.changed` | **Prueba** | Revocación inmediata de refresh tokens y bloqueo de logins futuros. |
| **REQ-AUTH-017** | Emisión Evento Staff Creado | BR-015 | RabbitMQ Producer | `staff.created` | **Prueba** | Verificación de recepción de CloudEvent en cola `sala.staff-sync`. |
| **REQ-AUTH-018** | Bloqueo Perimetral Gateway | BR-008, INV-005 | Filtro Interceptor en API Gateway | — | **Prueba** | Retorno HTTP 403 al intentar consumir endpoints de cocina/menú con clave temporal. |
| **REQ-AUTH-019** | Sanitización de Cabeceras | BR-013, INV-006 | Inyección de Cabeceras en Gateway | — | **Prueba** | Eliminación de `X-User-*` de origen público y sustitución por claims del token. |
| **REQ-AUTH-020** | Reset de Contraseña por Admin | BR-007, BR-008, INV-005 | `POST /api/v1/staff/:id/reset-password` | — | **Demostración** | Retorno forzoso de la cuenta de personal a estado `TEMPORARY`. |
