# Reglas de Negocio e Invariantes del Dominio (Auth)

## Reglas de Negocio (BR-AUTH)

- **BR-AUTH-001 (Registro y Creación de Tenant por Administrador):** La creación de un nuevo restaurante (tenant) se efectúa simultáneamente con el registro de la cuenta de su Gerente/Administrador. La cuenta adquiere de forma inmediata e irrevocable para ese tenant el rol `ADMINISTRADOR`.
- **BR-AUTH-002 (No Asignabilidad Externa del Rol Administrador):** El rol `ADMINISTRADOR` no es seleccionable ni asignable dentro del formulario de alta o edición de miembros del personal operativo. Únicamente puede existir una cuenta titular de administración o delegados explícitos mediante flujos de gobierno de tenant independientes.
- **BR-AUTH-003 (Autoridad de Creación de Personal):** Únicamente usuarios con sesión activa autenticada y rol `ADMINISTRADOR` pueden dar de alta, modificar roles o desactivar miembros del personal pertenecientes a su mismo `restaurantId`.
- **BR-AUTH-004 (Estructura Canónica del Identificador de Personal):** Todo miembro del personal debe contar con un identificador unívoco denominado `staffId`, cuya máscara sintáctica responde estrictamente a la expresión regular:
  $$^[A-Z][0-9]{6}\$$
  donde el primer carácter es una letra alfabética en mayúscula (`X`) y los seis caracteres subsiguientes son dígitos numéricos (`YYYYYY`) del `000000` al `999999`.
- **BR-AUTH-005 (Convención de Letra Prefijo en Staff ID):** La letra prefijo `X` en el `staffId` se asigna al momento de la creación tomando como base el rol primario de contratación (`H` para Host, `A` para Almacenista, `M` para Mesero, `C` para Chef Master, o `E` para personal general/multirol). Una vez generado y asignado al usuario, el `staffId` es estrictamente inmutable durante todo su ciclo de vida en el restaurante.
- **BR-AUTH-006 (Unicidad Compuesta del Staff ID):** El identificador `staffId` es estrictamente único dentro del ámbito de un mismo restaurante (`restaurantId`). No pueden coexistir dos usuarios con el mismo `staffId` en el mismo establecimiento.
- **BR-AUTH-007 (Credencial Inicial y Contraseña Temporal):** Al dar de alta un miembro del personal, el sistema o el administrador debe fijar una contraseña temporal inicial, asignando el estado de credencial `passwordStatus = TEMPORARY` (equivalente a la bandera lógica `mustChangePassword = true`).
- **BR-AUTH-008 (Contención Perimetral por Contraseña Temporal):** Todo usuario autenticado cuyo token o registro contenga `passwordStatus = TEMPORARY` queda perimetralmente contenido: el API Gateway y el Dashboard bloquearán cualquier acceso a microservicios operacionales (Menú, Órdenes, Sala, Inventario, Caja), restringiendo sus operaciones permitidas exclusivamente al cambio de contraseña (`POST /api/v1/auth/change-initial-password`) y al cierre de sesión (`POST /api/v1/auth/logout`).
- **BR-AUTH-009 (Preservación de Identidad tras Cambio de Contraseña):** Al completarse exitosamente el cambio obligatorio de contraseña:
  1. El estado de la credencial pasa irreversiblemente a `passwordStatus = ACTIVE`.
  2. El identificador `staffId` (`XYYYYYY`) original permanece exactamente igual.
  3. Se invalida el token temporal previo y se emite un nuevo token con acceso pleno a los módulos autorizados por sus roles.
- **BR-AUTH-010 (Cardinalidad Mínima de Roles en Personal):** Todo miembro del personal debe poseer en todo momento al menos un (1) rol operativo asignado perteneciente al conjunto:
  $$\mathcal{R}_{\text{operativo}} = \{\text{HOST}, \text{ALMACENISTA}, \text{MESERO}, \text{CHEF\_MASTER}\}$$
  Se prohíbe persistir o actualizar un perfil de personal con cero roles asociados.
- **BR-AUTH-011 (Membresía Multi-Rol Plena):** Un miembro del personal puede tener asignado simultáneamente cualquier subconjunto no vacío de roles operativos $\mathcal{S} \subseteq \mathcal{R}_{\text{operativo}}$, incluyendo la totalidad de los mismos en caso de requerimientos operativos especiales.
- **BR-AUTH-012 (Determinación de Vistas por Composición de Roles):** El conjunto de vistas y módulos interactivos accesibles por un usuario en el Dashboard corresponde a la unión lógica de las vistas autorizadas para cada uno de los roles que ostenta activamente:
  $$\text{VistasPermitidas}(u) = \bigcup_{r \in u.\text{roles}} \text{VistasDelRol}(r)$$
- **BR-AUTH-013 (Aislamiento Estricto Multi-Tenant):** Ningún usuario autenticado, independientemente de sus roles, puede consultar, modificar ni autenticarse en el contexto de un `restaurantId` distinto al vinculado a su cuenta en la base de datos de Auth.
- **BR-AUTH-014 (Desactivación de Personal e Invalidación de Sesiones):** Cuando el Administrador cambia el estado de un empleado a `isActive = false`:
  1. Se revocan inmediatamente todos los `RefreshToken` asociados a dicho usuario.
  2. El API Gateway deniega futuras peticiones y el usuario queda impedido para autenticarse nuevamente hasta su reactivación.
- **BR-AUTH-015 (Publicación Obligatoria de Eventos de Personal):** Toda creación de personal, cambio en la asignación de roles o alteración de estado activo/inactivo debe publicar de forma transaccional un evento de dominio hacia el exchange `restaurant.staff.events` de RabbitMQ para sincronizar a los microservicios dependientes.

---

## Invariantes de Integridad del Dominio (INV-AUTH)

- **INV-AUTH-001 (Invariante de Pertenencia Mínima a Rol en Personal):**
  $$\forall \, s \in \text{StaffMember}, \quad |s.\text{roles}| \ge 1 \quad \land \quad s.\text{roles} \subseteq \{\text{HOST}, \text{ALMACENISTA}, \text{MESERO}, \text{CHEF\_MASTER}\}$$
  Ningún empleado del personal operativo puede existir en la base de datos sin al menos un rol operativo asignado, ni puede ostentar el rol `ADMINISTRADOR`.

- **INV-AUTH-002 (Invariante de Exclusividad del Administrador de Tenant):**
  $$\forall \, u \in \text{User}, \quad \text{ADMINISTRADOR} \in u.\text{roles} \implies u.\text{userType} = \text{ADMIN} \land u.\text{roles} = \{\text{ADMINISTRADOR}\}$$
  Una cuenta administradora no se mezcla en la misma tupla con roles operativos ordinarios de personal.

- **INV-AUTH-003 (Invariante de Sintaxis y Unicidad del Staff ID):**
  $$\forall \, s_1, s_2 \in \text{StaffMember}:$$
  $$\text{regex\_match}(s_1.\text{staffId}, \text{"^[A-Z][0-9]{6}\$"}) = \text{true}$$
  $$(s_1.\text{restaurantId} = s_2.\text{restaurantId} \land s_1.\text{staffId} = s_2.\text{staffId}) \implies s_1.\text{id} = s_2.\text{id}$$
  El formato cumple estrictamente 1 letra y 6 números, y la unicidad está matemáticamente garantizada por restaurante.

- **INV-AUTH-004 (Invariante de Inmutabilidad del Staff ID):**
  $$\forall \, s \in \text{StaffMember}, \quad \text{Old}(s.\text{staffId}) \ne \emptyset \implies \text{New}(s.\text{staffId}) = \text{Old}(s.\text{staffId})$$
  El identificador asignado no puede mutar por ninguna operación de actualización de perfil, cambio de contraseña o cambio de roles.

- **INV-AUTH-005 (Invariante de Bloqueo por Contraseña Temporal):**
  $$\forall \, \text{req} \in \text{APIRequests}: \quad \text{Token}(\text{req}).\text{mustChangePassword} = \text{true} \implies \text{TargetEndpoint}(\text{req}) \in \{\text{POST /auth/change-initial-password}, \text{POST /auth/logout}, \text{GET /auth/me}\}$$
  Cualquier petición hacia un endpoint distinto a los de remediación de credencial o cierre de sesión es indefectiblemente rechazada por el perímetro.

- **INV-AUTH-006 (Invariante de Integridad Multi-Tenant):**
  $$\forall \, u \in \text{User}, \forall \, e \in \text{EntitiesCreatedBy}(u): \quad e.\text{restaurantId} = u.\text{restaurantId}$$
  Ninguna entidad o personal generado puede tener un `restaurantId` discordante con el restaurante del usuario creador.

- **INV-AUTH-007 (Invariante de No Legibilidad de Secretos en Reposo):**
  $$\forall \, u \in \text{User}, \quad \text{Length}(u.\text{passwordHash}) > 0 \land u.\text{passwordHash} \ne u.\text{rawPassword}$$
  Bajo ninguna circunstancia se persiste en base de datos texto plano de contraseñas, ni en tablas principales ni en bitácoras de auditoría.
