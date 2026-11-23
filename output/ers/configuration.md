# Configuración, Despliegue y Variables de Entorno

## Variables de Entorno del Servicio Auth

El servicio **Auth** se configura de manera desacoplada mediante variables de entorno adhering a la metodología *Twelve-Factor App*.

| Variable | Tipo | Obligatoria | Valor por Defecto | Descripción |
| :--- | :--- | :--- | :--- | :--- |
| `NODE_ENV` / `APP_ENV` | String | Sí | `development` | Entorno de ejecución (`development`, `test`, `production`). |
| `PORT` | Integer | No | `4000` | Puerto TCP de escucha para la API de Auth. |
| `DATABASE_URL` | String | Sí | — | Cadena de conexión JDBC/PostgreSQL (`postgres://user:pass@host:5432/fmat_auth`). |
| `JWT_ALGORITHM` | String | No | `HS256` | Algoritmo de firma digital (`HS256` por defecto). |
| `JWT_SECRET` | String | Sí | — | Secreto simétrico de alta entropía (mínimo 32 caracteres, compartido con el API Gateway). |
| `JWT_ACCESS_TOKEN_TTL` | String | No | `15m` | Tiempo de vida del Access Token. |
| `REFRESH_TOKEN_TTL_DAYS` | Integer | No | `7` | Días de vigencia del Refresh Token (1 a 30). |
| `RABBITMQ_URL` | String | No | — (vacía = deshabilitado) | URI del broker RabbitMQ. Vacía o sin definir: no se publican eventos y `/health` reporta `rabbitmq: disabled`. Definida pero inaccesible: `/health` responde `degraded`. Ver OPEN-AUTH-007. |
| `RABBITMQ_STAFF_EXCHANGE`| String | No | `restaurant.staff.events` | Nombre del exchange topic para eventos de personal. |
| `CORS_ALLOWED_ORIGINS` | String | No | `http://localhost:3000,http://localhost:5173` | Orígenes web autorizados para interactuar con la API. |

---

## Configuración del API Gateway

El **API Gateway** gestiona la ruta perimetral y requiere acceso a la llave pública para verificar la autenticidad de los tokens:

| Variable | Tipo | Obligatoria | Valor por Defecto | Descripción |
| :--- | :--- | :--- | :--- | :--- |
| `KONG_PROXY_PORT` | Integer | No | `8000` | Puerto principal del API Gateway Kong expuesto hacia clientes. |
| `AUTH_SERVICE_URL` | String | Sí | `http://auth-backend:3000` | URL interna del servicio de autenticación. |
| `MENU_SERVICE_URL` | String | Sí | `http://menu-backend:8080` | URL interna del servicio de catálogo y menú. |
| `ORDERS_SERVICE_URL` | String | Sí | `http://orders-backend:8082`| URL interna del servicio de comandas (POS). |
| `KITCHEN_SERVICE_URL` | String | Sí | `http://kitchen-backend:8086`| URL interna del servicio de cocina (KDS). |
| `FLOOR_SERVICE_URL` | String | Sí | `http://floor-backend:8085` | URL interna del servicio de sala y reservaciones. |
| `INVENTORY_SERVICE_URL`| String | Sí | `http://inventory-backend:3000`| URL interna del servicio de almacén e inventario. |
| `PAYMENTS_SERVICE_URL` | String | Sí | `http://payments-backend:8084`| URL interna del servicio de caja y facturación. |
| `JWT_SECRET` | String | Sí | — | Secreto HS256 compartido con Auth-Backend para validar firmas en el Gateway. |
| `RATE_LIMIT_MAX_REQ` | Integer | No | `100` | Límite máximo de peticiones por minuto por dirección IP. |

---

## Topología y Declaración de Infraestructura en RabbitMQ

El equipo de Auth es responsable de declarar y custodiar la topología base del broker de mensajería para el personal.

> **Estado actual (OPEN-AUTH-007):** RabbitMQ es opcional y hoy ningún microservicio consume estos eventos; las colas `sala.staff-sync` y `audit.staff-history` de abajo son la topología **prevista**, no desplegada. Se declaran solo cuando un equipo solicite consumir los eventos.

```mermaid
flowchart LR
    Producer["Microservicio Auth<br/>(Publisher)"]
    Exchange{{"Exchange: restaurant.staff.events<br/>(Type: topic, Durable: true)"}}
    QueueSala[("Queue: sala.staff-sync<br/>Binding: staff.*")]
    QueueAudit[("Queue: audit.staff-history<br/>Binding: staff.#")]
    ConsumerSala["Microservicio Sala / Host<br/>(Sincroniza Meseros)"]
    ConsumerAudit["Servicio Auditoría<br/>(Registros Históricos)"]

    Producer -->|"Publish: staff.created,<br/>staff.permissions.updated"| Exchange
    Exchange -->|"Routing match: staff.*"| QueueSala
    Exchange -->|"Routing match: staff.#"| QueueAudit
    QueueSala --> ConsumerSala
    QueueAudit --> ConsumerAudit
```

### Script de Inicialización de RabbitMQ (`rabbitmq-init.sh`):
```bash
#!/bin/sh
# Declaración de Exchange principal
rabbitmqadmin declare exchange name=restaurant.staff.events type=topic durable=true

# Declaración de colas operacionales
rabbitmqadmin declare queue name=sala.staff-sync durable=true
rabbitmqadmin declare queue name=audit.staff-history durable=true

# Enlace de colas con routing keys
rabbitmqadmin declare binding source=restaurant.staff.events destination=sala.staff-sync routing_key="staff.*"
rabbitmqadmin declare binding source=restaurant.staff.events destination=audit.staff-history routing_key="staff.#"
```
