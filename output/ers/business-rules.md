# Reglas de Negocio e Invariantes del Dominio (Auth)

## Reglas de Negocio (BR-AUTH)

- **BR-AUTH-001 (Registro y Creación de Tenant por Administrador):** La creación de un nuevo restaurante (tenant) se efectúa simultáneamente con el registro de la cuenta de su Gerente/Administrador. La cuenta adquiere de forma inmediata el tipo de usuario `ADMIN` con facultades de superusuario (`*`) para dicho tenant.
- **BR-AUTH-002 (No Asignabilidad Externa de Facultades Administrativas Raíz):** La cuenta titular `ADMIN` no es seleccionable ni asignable dentro del formulario de personal operativo. Únicamente puede existir una cuenta administradora de tenant o delegados explícitos mediante flujos de gobierno de tenant independientes.
- **BR-AUTH-003 (Autoridad de Creación y Configuración de Personal):** Únicamente usuarios con sesión activa autenticada y tipo `ADMIN` pueden dar de alta empleados, asignar o modificar su etiqueta descriptiva de rol, configurar sus permisos granulares o desactivar miembros del personal pertenecientes a su mismo `restaurantId`.
- **BR-AUTH-004 (Estructura Canónica del Identificador de Personal):** Todo miembro del personal debe contar con un identificador unívoco denominado `staffId`, cuya máscara sintáctica responde estrictamente a la expresión regular:
  $$^[A-Z][0-9]{6}\$$
  donde el primer carácter es una letra alfabética en mayúscula (`X`) y los seis caracteres subsiguientes son dígitos numéricos (`YYYYYY`) del `000000` al `999999`.
- **BR-AUTH-005 (Convención de Letra Prefijo en Staff ID):** La letra prefijo `X` en el `staffId` se fija de manera canónica con el carácter `'E'` (Empleado: `E000001` a `E999999`), manteniendo el identificador completamente neutral e independiente de las etiquetas de rol u ocupación del empleado. Una vez generado y asignado al usuario, el `staffId` es estrictamente inmutable durante todo su ciclo de vida en el establecimiento.
- **BR-AUTH-006 (Unicidad Compuesta del Staff ID):** El identificador `staffId` es estrictamente único dentro del ámbito de un mismo restaurante (`restaurantId`). No pueden coexistir dos usuarios con el mismo `staffId` en el mismo establecimiento.
- **BR-AUTH-007 (Credencial Inicial y Contraseña Temporal):** Al dar de alta un miembro del personal, el sistema o el administrador debe fijar una contraseña temporal inicial, asignando el estado de credencial `passwordStatus = TEMPORARY` (equivalente a la bandera lógica `mustChangePassword = true`).
- **BR-AUTH-008 (Contención Perimetral por Contraseña Temporal):** Todo usuario autenticado cuyo token o registro contenga `passwordStatus = TEMPORARY` queda perimetralmente contenido: el API Gateway y el Dashboard bloquearán cualquier acceso a microservicios operacionales (Menú, Órdenes, Sala, Inventario, Facturación), restringiendo sus operaciones permitidas exclusivamente al cambio de contraseña (`POST /api/v1/auth/change-initial-password`), a la consulta de identidad (`GET /api/v1/auth/me`) y al cierre de sesión (`POST /api/v1/auth/logout`).
- **BR-AUTH-009 (Preservación de Identidad tras Cambio de Contraseña):** Al completarse exitosamente el cambio obligatorio de contraseña:
  1. El estado de la credencial pasa irreversiblemente a `passwordStatus = ACTIVE`.
  2. El identificador `staffId` (`XYYYYYY`) original permanece exactamente igual.
  3. Se invalida el token temporal previo y se emite un nuevo token con acceso pleno a los módulos autorizados por sus permisos.
- **BR-AUTH-010 (Cardinalidad Mínima de Permisos y Etiqueta de Rol):** Todo miembro del personal debe poseer en todo momento una etiqueta descriptiva de rol no vacía (ej. *"Líder de inventario"*, *"Mesero de piso"*) y al menos un (1) permiso granular operativo asignado del catálogo del sistema $\mathcal{P}_{\text{sistema}}$:
  $$|s.\text{permissions}| \ge 1 \quad \land \quad \text{Length}(s.\text{roleLabel}) \ge 1$$
  Se prohíbe persistir o actualizar un perfil de personal con cero permisos asociados.
- **BR-AUTH-011 (Composición Libre y Transversal de Permisos):** El Administrador puede asignar a cualquier miembro del personal cualquier combinación no vacía de permisos granulares $\mathcal{S} \subseteq \mathcal{P}_{\text{sistema}}$ provenientes de uno o varios microservicios (por ejemplo: permisos de inventario + permisos de comandas), sin limitaciones impuestas por nomenclaturas fijas o roles predefinidos.
- **BR-AUTH-012 (Determinación de Vistas y Acciones por Permisos PBAC):** El conjunto de vistas, menús y botones de acción accesibles por un usuario en el Dashboard corresponde a la unión lógica de las capacidades autorizadas por cada uno de los permisos que ostenta activamente:
  $$\text{CapacidadesPermitidas}(u) = \bigcup_{p \in u.\text{permissions}} \text{CapacidadesDelPermiso}(p)$$
- **BR-AUTH-013 (Aislamiento Estricto Multi-Tenant):** Ningún usuario autenticado, independientemente de sus permisos, puede consultar, modificar ni autenticarse en el contexto de un `restaurantId` distinto al vinculado a su cuenta en la base de datos de Auth.
- **BR-AUTH-014 (Desactivación de Personal e Invalidación de Sesiones):** Cuando el Administrador cambia el estado de un empleado a `isActive = false`:
  1. Se revocan inmediatamente todos los `RefreshToken` asociados a dicho usuario.
  2. El API Gateway deniega futuras peticiones y el usuario queda impedido para autenticarse nuevamente hasta su reactivación.
- **BR-AUTH-015 (Publicación Obligatoria de Eventos de Personal):** Toda creación de personal, cambio en la etiqueta de rol, actualización en la matriz de permisos o alteración de estado activo/inactivo debe publicar de forma transaccional un evento de dominio hacia el exchange `restaurant.staff.events` de RabbitMQ para sincronizar a los microservicios dependientes.

---

## Invariantes de Integridad del Dominio (INV-AUTH)

- **INV-AUTH-001 (Invariante de Pertenencia Mínima a Permisos en Personal):**
  $$\forall \, s \in \text{StaffMember}, \quad |s.\text{permissions}| \ge 1 \quad \land \quad \forall \, p \in s.\text{permissions}: p \in \mathcal{P}_{\text{sistema}} \setminus \{*\}$$
  Ningún empleado del personal operativo puede existir en la base de datos sin al menos un permiso granular operativo válido asignado, ni puede ostentar el permiso universal comodín (`*`).

- **INV-AUTH-002 (Invariante de Exclusividad del Administrador de Tenant):**
  $$\forall \, u \in \text{User}, \quad u.\text{userType} = \text{ADMIN} \implies u.\text{permissions} = \{*\} \land u.\text{email} \ne \emptyset$$
  Una cuenta administradora de restaurante posee inherentemente permisos universales sobre su tenant y requiere un correo corporativo formal.

- **INV-AUTH-003 (Invariante de Sintaxis y Unicidad del Staff ID):**
  $$\forall \, s_1, s_2 \in \text{StaffMember}:$$
  $$\text{regex\_match}(s_1.\text{staffId}, \text{"^[A-Z][0-9]{6}\$"}) = \text{true}$$
  $$(s_1.\text{restaurantId} = s_2.\text{restaurantId} \land s_1.\text{staffId} = s_2.\text{staffId}) \implies s_1.\text{id} = s_2.\text{id}$$
  El formato cumple estrictamente 1 letra y 6 números, y la unicidad está matemáticamente garantizada por restaurante.

- **INV-AUTH-004 (Invariante de Inmutabilidad del Staff ID):**
  $$\forall \, s \in \text{StaffMember}, \quad \text{Old}(s.\text{staffId}) \ne \emptyset \implies \text{New}(s.\text{staffId}) = \text{Old}(s.\text{staffId})$$
  El identificador asignado no puede mutar por ninguna operación de actualización de perfil, cambio de contraseña, modificación de etiqueta de rol o reasignación de permisos.

- **INV-AUTH-005 (Invariante de Bloqueo por Contraseña Temporal):**
  $$\forall \, \text{req} \in \text{APIRequests}: \quad \text{Token}(\text{req}).\text{mustChangePassword} = \text{true} \implies \text{TargetEndpoint}(\text{req}) \in \{\text{POST /auth/change-initial-password}, \text{POST /auth/logout}, \text{GET /auth/me}\}$$
  Cualquier petición hacia un endpoint distinto a los de remediación de credencial o cierre de sesión es indefectiblemente rechazada por el perímetro.

- **INV-AUTH-006 (Invariante de Integridad Multi-Tenant):**
  $$\forall \, u \in \text{User}, \forall \, e \in \text{EntitiesCreatedBy}(u): \quad e.\text{restaurantId} = u.\text{restaurantId}$$
  Ninguna entidad o personal generado puede tener un `restaurantId` discordante con el restaurante del usuario creador.

- **INV-AUTH-007 (Invariante de No Legibilidad de Secretos en Reposo):**
  $$\forall \, u \in \text{User}, \quad \text{Length}(u.\text{passwordHash}) > 0 \land u.\text{passwordHash} \ne u.\text{rawPassword}$$
  Bajo ninguna circunstancia se persiste en base de datos texto plano de contraseñas, ni en tablas principales ni en bitácoras de auditoría.
