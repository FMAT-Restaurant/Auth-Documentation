# Matriz de Trazabilidad de Requisitos y Verificación

Esta matriz vincula cada requisito funcional con sus reglas de negocio asociadas, las interfaces de implementación (endpoints HTTP y eventos RabbitMQ) y los métodos formales de verificación de la disciplina de *Verificación y Validación*.

| ID Requisito | Título del Requisito | Reglas / Invariantes Asociadas | Endpoint / Operación Técnica | Evento Asíncrono (RabbitMQ) | Método de V&V | Criterio de Aceptación / Verificación |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **REQ-AUTH-001** | Registro de Restaurante y Admin | BR-001, BR-002, INV-002, INV-006 | `POST /api/v1/auth/register-restaurant` | — | **Prueba** | Creación de tenant y usuario con tipo `ADMIN` y permisos universales. |
| **REQ-AUTH-002** | Autenticación del Administrador | BR-001, INV-002, INV-007 | `POST /api/v1/auth/login` | — | **Prueba** | Retorno de JWT con permisos universales `["*"]` y `mustChangePassword=false`. |
| **REQ-AUTH-003** | Creación de Personal con Permisos | BR-003, BR-010, INV-001, INV-006 | `POST /api/v1/staff` | `staff.created` | **Demostración** | Persistencia de empleado con etiqueta de rol libre y permisos vinculados al tenant. |
| **REQ-AUTH-004** | Generación de ID `XYYYYYY` | BR-004, BR-005, BR-006, INV-003, INV-004 | `POST /api/v1/staff` (Lógica interna) | `staff.created` | **Prueba** | Formato canónico `E000001` y ausencia de colisiones dentro del restaurante. |
| **REQ-AUTH-005** | Asignación Contraseña Temporal | BR-007, BR-008, INV-005, INV-007 | `POST /api/v1/staff` | — | **Prueba** | Verificación de estado `password_status = 'TEMPORARY'`. |
| **REQ-AUTH-006** | Cardinalidad Mínima de Permisos | BR-010, INV-001 | `POST /api/v1/staff`, `PUT /api/v1/staff/:id/permissions` | — | **Prueba** | Rechazo HTTP 422 ante listas de permisos vacías o falta de etiqueta de rol. |
| **REQ-AUTH-007** | Permisos Granulares y Transversales | BR-011, BR-012, INV-001 | `POST /api/v1/staff` | `staff.permissions.updated` | **Demostración** | JWT contiene arreglo de permisos específicos (ej. ingredientes, stock). |
| **REQ-AUTH-008** | Exclusión de Facultades Admin | BR-002, INV-001, INV-002 | `POST /api/v1/staff` | — | **Prueba** | Rechazo HTTP 400 ante intento de asignar `*` o tipo `ADMIN` a personal operativo. |
| **REQ-AUTH-009** | Login de Personal con Staff ID | BR-004, BR-006, BR-007, INV-003 | `POST /api/v1/auth/login` | — | **Prueba** | Inicio de sesión exitoso resolviendo usuario por `staffId` (`E000104`). |
| **REQ-AUTH-010** | Detección Contraseña Temporal | BR-007, BR-008, INV-005 | `POST /api/v1/auth/login` | — | **Prueba** | JWT emitido incluye claim `"mustChangePassword": true`. |
| **REQ-AUTH-011** | Cambio de Contraseña Inicial | BR-008, BR-009, INV-004, INV-005 | `POST /api/v1/auth/change-initial-password` | — | **Prueba** | Estado pasa a `ACTIVE`, se emite nuevo JWT y se preserva `staffId`. |
| **REQ-AUTH-012** | Emisión y Firma de Tokens JWT | BR-012, BR-013, INV-006 | Servicio de Tokenización JWT | — | **Análisis** | Verificación de firma criptográfica y claims `permissions` y `roleLabel`. |
| **REQ-AUTH-013** | Renovación con Refresh Token | BR-014, INV-007 | `POST /api/v1/auth/refresh` | — | **Prueba** | Rotación exitosa de Refresh Token e invalidación del token previo. |
| **REQ-AUTH-014** | Consulta de Identidad (`/auth/me`) | BR-012 | `GET /api/v1/auth/me` | — | **Demostración** | Retorno consolidado de permisos asignados, etiqueta de rol y estado. |
| **REQ-AUTH-015** | Modificación de Permisos y Rol | BR-010, BR-011, BR-015, INV-001 | `PUT /api/v1/staff/:id/permissions` | `staff.permissions.updated` | **Prueba** | Actualización efectiva en BD y despacho del evento a RabbitMQ. |
| **REQ-AUTH-016** | Desactivación de Personal | BR-014, BR-015 | `PATCH /api/v1/staff/:id/status` | `staff.status.changed` | **Prueba** | Revocación inmediata de refresh tokens y bloqueo de logins futuros. |
| **REQ-AUTH-017** | Emisión Eventos en RabbitMQ | BR-015 | RabbitMQ Producer | `staff.*` | **Prueba** | Verificación de recepción de CloudEvent con permissions en `sala.staff-sync`. |
| **REQ-AUTH-018** | Bloqueo Perimetral Gateway | BR-008, INV-005 | Filtro Interceptor en API Gateway | — | **Prueba** | Retorno HTTP 403 al intentar consumir endpoints de inventario/órdenes con clave temporal. |
| **REQ-AUTH-019** | Sanitización e Inyección de Headers | BR-013, INV-006 | Inyección de Cabeceras en Gateway | — | **Prueba** | Inyección fidedigna de `X-User-Permissions` y `X-User-Role-Label`. |
| **REQ-AUTH-020** | Reset de Contraseña por Admin | BR-007, BR-008, INV-005 | `POST /api/v1/staff/:id/reset-password` | — | **Demostración** | Retorno forzoso de la cuenta de personal a estado `TEMPORARY`. |
| **REQ-AUTH-021** | Vista Unificada (Registro / Login) | BR-001, BR-004 | Frontend Router / UI Tab Switcher | — | **Demostración** | Renderizado dinámico de pestañas Registro y Login en una vista. |
| **REQ-AUTH-022** | Conmutador "¿Eres un admin?" | BR-001, BR-004 | Frontend Auth Form Controller | — | **Demostración** | Alternancia fluida entre entrada Staff ID y Correo corporativo. |
| **REQ-AUTH-023** | Modal Cambio de Clave Inicial | BR-007, BR-008, INV-005 | Frontend Password Modal Component | — | **Demostración** | Despliegue de modal bloqueante al detectar `mustChangePassword=true`. |
| **REQ-AUTH-024** | Adherencia a FMAT UI Kit v2.0 | DEC-002 | Componentes UI & Tokens CSS | — | **Inspección** | Cumplimiento del 100% de paleta `#C2410C`, radios 8/12px y regla 70/20/10. |
