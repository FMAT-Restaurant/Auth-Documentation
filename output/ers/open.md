# Decisiones Abiertas, Supuestos y Extensiones Futuras (Auth)

Este documento registra los puntos de diseño complementarios, supuestos operativos y posibles extensiones del microservicio **Auth**, del **API Gateway** y del esquema de eventos en **RabbitMQ** que podrán ser abordados en fases posteriores del proyecto.

---

## 1. Puntos de Diseño y Decisiones Abiertas

### OPEN-AUTH-001 — Modo de Acceso Rápido por PIN para Terminales Táctiles (POS / KDS)
- **Contexto:** En el entorno dinámico de un restaurante físico, escribir una contraseña alfanumérica compleja en cada cambio de mesero o comanda en una tablet táctil puede restar agilidad operativa.
- **Propuesta:** Introducir en una segunda iteración el soporte para un **PIN numérico de 4 a 6 dígitos** ligado al `staffId`, que permita un desbloqueo rápido de sesión en terminales locales asignadas previamente al restaurante, manteniendo la contraseña completa para configuraciones administrativas o inicio de turno.
- **Estado:** Abierto para discusión con el equipo de Frontend y Orders.

---

### OPEN-AUTH-002 — Asignación de Meseros a Turnos Activos (Clock-In / Clock-Out)
- **Contexto:** El microservicio `Sala/Host` necesita la lista de meseros para asignar mesas. Sin embargo, no todos los meseros dados de alta en nómina están físicamente presentes en el restaurante en un momento dado.
- **Propuesta:** Definir si la lista consumida por Sala debe incluir a *todos los empleados con rol Mesero dados de alta*, o si Auth / Sala debe manejar un estado adicional de "En turno / Activo hoy" (`clockedIn: boolean`).
- **Estado:** Pendiente de coordinación con el equipo de Sala/Host.

---

### OPEN-AUTH-003 — Autenticación Multifactor (MFA / 2FA) para Administradores
- **Contexto:** Las cuentas con rol `ADMINISTRADOR` tienen acceso integral a nóminas, configuraciones comerciales y métricas financieras de todos los microservicios.
- **Propuesta:** Evaluar la implementación de MFA basado en TOTP (Google Authenticator) para los gerentes de restaurante, aumentando la resistencia ante vulnerabilidades de credenciales.
- **Estado:** Previsto como mejora de seguridad post-MVP.

---

### OPEN-AUTH-004 — Rotación de Claves Criptográficas (JWKS - JSON Web Key Set)
- **Contexto:** Si se emplea `RS256`, el API Gateway necesita conocer la clave pública de Auth.
- **Propuesta:** Exponer un endpoint estándar `GET /.well-known/jwks.json` en Auth para que el API Gateway pueda consultar y refrescar en caché la clave pública de verificación de forma transparente, permitiendo la rotación de claves sin reiniciar los servicios.
- **Estado:** Propuesto para la implementación de backend del Gateway.
