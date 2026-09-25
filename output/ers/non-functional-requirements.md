# Requisitos No Funcionales del Servicio Auth

Los requisitos no funcionales definen los atributos de calidad, seguridad perimetral, rendimiento y confiabilidad que debe cumplir el servicio **Auth** y sus componentes satélites (**API Gateway** y **RabbitMQ**).

---

## 1. Seguridad y Criptografía (NFR-AUTH-SEC)

<a id="nfr-auth-sec-001"></a>

### NFR-AUTH-SEC-001 — Algoritmo de Dispersión Criptográfica de Contraseñas

**Requisito:**
El servicio Auth deberá almacenar todas las contraseñas procesadas empleando una función de derivación de claves computacionalmente costosa y resistente a ataques de fuerza bruta en GPU/ASIC: **Argon2id** (con parámetros recomendados OWASP: $m=64\,\text{MB}$, $t=3$ iteraciones, $p=4$ hilos) o **bcrypt** (con factor de trabajo $cost \ge 12$).

**Tipo:** No Funcional / Seguridad  
**Fuente:** Guía OWASP Password Storage Cheat Sheet  
**Verificación:** Inspección: Analizar el código fuente del módulo de hashing y el formato de los registros almacenados en la columna `password_hash` para verificar el prefijo de algoritmo (`$argon2id$` o `$2b$12$`).  
**Estado:** Confirmado  

---

<a id="nfr-auth-sec-002"></a>

### NFR-AUTH-SEC-002 — Algoritmo y Robustez de Firma de Tokens JWT

**Requisito:**
Los Access Tokens emitidos deberán ser firmados digitalmente utilizando un par de llaves asimétricas mediante **RS256** (RSA de 2048 o 4096 bits) o **Ed25519** (o en su defecto **HS256** con un secreto compartido de al menos 256 bits de entropía pseudoaleatoria criptográficamente segura).

**Tipo:** No Funcional / Seguridad  
**Fuente:** RFC 7519 / NIST Special Publication 800-63B  
**Verificación:** Prueba: Inspeccionar la cabecera `alg` del token generado y verificar la firma con la clave pública configurada en el API Gateway.  
**Estado:** Confirmado  

---

<a id="nfr-auth-sec-003"></a>

### NFR-AUTH-SEC-003 — Política de Expiración de Tokens

**Requisito:**
Los Access Tokens deberán tener una vigencia temporal máxima de sesenta (60) minutos para limitar la ventana de exposición en caso de filtración de credenciales. Los Refresh Tokens tendrán una vigencia máxima configurable de catorce (14) días y estarán sujetos a rotación automática en cada uso.

**Tipo:** No Funcional / Seguridad  
**Fuente:** Mejores Prácticas de Arquitecturas OAuth2 / OIDC  
**Verificación:** Prueba: Intentar consumir un endpoint con un token expirado ($iat + 61\,\text{min}$); verificar que el Gateway retorne inmediatamente HTTP `401 Unauthorized` con código `TOKEN_EXPIRED`.  
**Estado:** Confirmado  

---

<a id="nfr-auth-sec-004"></a>

### NFR-AUTH-SEC-004 — Política de Complejidad de Contraseña Definitiva

**Requisito:**
La contraseña definitiva establecida por cualquier usuario deberá tener una longitud mínima de ocho (8) caracteres, incluyendo al menos una letra mayúscula, una letra minúscula, un dígito numérico y un símbolo especial, rechazando contraseñas que coincidan con el identificador de usuario o secuencias triviales.

**Tipo:** No Funcional / Seguridad  
**Fuente:** OWASP Authentication Guidelines  
**Verificación:** Prueba: Ejecutar pruebas automatizadas con un diccionario de contraseñas débiles y verificar que todas sean rechazadas con mensajes de validación explícitos.  
**Estado:** Confirmado  

---

## 2. Rendimiento y Latencia (NFR-AUTH-PERF)

<a id="nfr-auth-perf-001"></a>

### NFR-AUTH-PERF-001 — Sobrecarga Máxima de Validación Perimetral en API Gateway

**Requisito:**
El tiempo introducido por el API Gateway para validar la firma del token JWT, verificar el claim `mustChangePassword` e inyectar las cabeceras `X-User-*` no deberá exceder de diez (10) milisegundos en el percentil 99 ($P99 \le 10\,\text{ms}$) bajo condiciones normales de carga.

**Tipo:** No Funcional / Rendimiento  
**Fuente:** SLA de Comunicación Perimetral FMAT  
**Verificación:** Prueba: Medir los tiempos de respuesta del API Gateway en pruebas de carga sintéticas con 50 peticiones concurrentes y registrar la latencia de procesamiento perimetral.  
**Estado:** Confirmado  

---

<a id="nfr-auth-perf-002"></a>

### NFR-AUTH-PERF-002 — Tiempo de Respuesta en Autenticación

**Requisito:**
El tiempo de respuesta total del endpoint de inicio de sesión (`POST /api/v1/auth/login`) no deberá superar los seiscientos (600) milisegundos ($P95 \le 600\,\text{ms}$), considerando el cómputo del hash criptográfico defensivo.

**Tipo:** No Funcional / Rendimiento  
**Fuente:** Experiencia de Usuario en Terminales Táctiles FMAT  
**Verificación:** Prueba: Ejecutar 100 inicios de sesión secuenciales registrando el tiempo transcurrido desde la solicitud hasta la recepción de la respuesta.  
**Estado:** Confirmado  

---

## 3. Disponibilidad y Confiabilidad (NFR-AUTH-AVAIL)

<a id="nfr-auth-avail-001"></a>

### NFR-AUTH-AVAIL-001 — Resiliencia en Validación Desacoplada

**Requisito:**
El API Gateway deberá ser capaz de validar y autorizar peticiones hacia microservicios operacionales (Menu, Orders, Sala, Inventory) de manera completamente autónoma sin consultar de forma sincrónica la base de datos de Auth, utilizando exclusivamente la firma criptográfica y claims del JWT.

**Tipo:** No Funcional / Confiabilidad  
**Fuente:** Principios de Arquitectura de Microservicios Desacoplados  
**Verificación:** Demostración: Detener temporalmente el servicio Auth-Backend mientras el API Gateway permanece en ejecución; verificar que las peticiones hacia otros servicios con tokens válidos previamente emitidos continúen operando exitosamente.  
**Estado:** Confirmado  

---

<a id="nfr-auth-avail-002"></a>

### NFR-AUTH-AVAIL-002 — Persistencia y Garantía de Entrega de Eventos en RabbitMQ

**Requisito:**
El exchange `restaurant.staff.events` y las colas vinculadas en RabbitMQ deberán estar declaradas como durables (`durable: true`), y los mensajes emitidos por Auth deberán contar con modo de entrega persistente (`delivery_mode: 2`) para evitar pérdida de eventos de personal ante reinicios del broker.

**Tipo:** No Funcional / Confiabilidad  
**Fuente:** Guía de Confiabilidad de Integración FMAT Restaurant  
**Verificación:** Inspección: Comprobar en la consola de RabbitMQ la configuración de durabilidad del exchange y de los mensajes en tránsito.  
**Estado:** Confirmado  

---

## 4. Auditoría y Trazabilidad (NFR-AUTH-AUD)

<a id="nfr-auth-aud-001"></a>

### NFR-AUTH-AUD-001 — Registro Inmutable de Eventos Críticos de Seguridad

**Requisito:**
El servicio Auth deberá registrar de forma estructurada en su bitácora de auditoría todo intento fallido de autenticación, alta de personal, modificación de roles, revocación de credenciales y cambio de contraseñas, incluyendo marca temporal ISO 8601, dirección IP de origen y usuario ejecutor.

**Tipo:** No Funcional / Auditoría  
**Fuente:** Estándares de Trazabilidad y Verificación FMAT  
**Verificación:** Prueba: Realizar una secuencia de acciones críticas (login fallido, cambio de roles) e inspeccionar que las filas correspondientes en la tabla `AUDIT_LOG` se registren de forma íntegra.  
**Estado:** Confirmado  
