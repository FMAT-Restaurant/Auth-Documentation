# Configuración, Despliegue y Variables de Entorno

## Variables de Entorno del Servicio Auth

El servicio **Auth** se configura de manera desacoplada mediante variables de entorno adhering a la metodología *Twelve-Factor App*.

| Variable | Tipo | Obligatoria | Valor por Defecto | Descripción |
| :--- | :--- | :--- | :--- | :--- |
| `NODE_ENV` / `APP_ENV` | String | Sí | `development` | Entorno de ejecución (`development`, `test`, `production`). |
| `PORT` | Integer | No | `4000` | Puerto TCP de escucha para la API de Auth. |
| `DATABASE_URL` | String | Sí | — | Cadena de conexión JDBC/PostgreSQL (`postgres://user:pass@host:5432/fmat_auth`). |
| `JWT_ALGORITHM` | String | No | `RS256` | Algoritmo de firma digital (`RS256` o `HS256`). |
| `JWT_PRIVATE_KEY_PATH` | String | Condicional | `./certs/jwt-private.pem` | Ruta al archivo con la llave privada RSA para firmar JWTs (si `RS256`). |
| `JWT_PUBLIC_KEY_PATH` | String | Condicional | `./certs/jwt-public.pem` | Ruta a la llave pública RSA para validar tokens emitidos. |
| `JWT_SECRET` | String | Condicional | — | Secreto simétrico de alta entropía (si `HS256`). |
| `JWT_ACCESS_TOKEN_TTL` | String | No | `3600s` | Tiempo de vida del Access Token (ej. `1h` o `3600s`). |
| `JWT_REFRESH_TOKEN_TTL`| String | No | `14d` | Tiempo de vida del Refresh Token (ej. `14d`). |
| `RABBITMQ_URL` | String | Sí | `amqp://guest:guest@localhost:5672` | URI de conexión al broker de mensajería RabbitMQ. |
| `RABBITMQ_STAFF_EXCHANGE`| String | No | `restaurant.staff.events` | Nombre del exchange topic para eventos de personal. |
| `CORS_ALLOWED_ORIGINS` | String | No | `http://localhost:3000,http://localhost:5173` | Orígenes web autorizados para interactuar con la API. |

---

## Configuración del API Gateway

El **API Gateway** gestiona la ruta perimetral y requiere acceso a la llave pública para verificar la autenticidad de los tokens:

| Variable | Tipo | Obligatoria | Valor por Defecto | Descripción |
| :--- | :--- | :--- | :--- | :--- |
| `GATEWAY_PORT` | Integer | No | `8080` | Puerto principal expuesto hacia clientes externos. |
| `AUTH_SERVICE_URL` | String | Sí | `http://auth-service:4000` | URL interna del servicio de autenticación. |
| `MENU_SERVICE_URL` | String | Sí | `http://menu-service:4001` | URL interna del servicio de catálogo y menú. |
| `ORDERS_SERVICE_URL` | String | Sí | `http://orders-service:4002`| URL interna del servicio de comandas y cocina (KDS). |
| `SALA_SERVICE_URL` | String | Sí | `http://sala-service:4003` | URL interna del servicio de sala y reservaciones. |
| `INVENTORY_SERVICE_URL`| String | Sí | `http://inventory-service:4004`| URL interna del servicio de almacén e inventario. |
| `BILLING_SERVICE_URL` | String | Sí | `http://billing-service:4005`| URL interna del servicio de cuentas y cobros. |
| `JWT_PUBLIC_KEY_PATH` | String | Sí | `./certs/jwt-public.pem` | Llave pública para validar firmas sin interrogar a Auth. |
| `RATE_LIMIT_MAX_REQ` | Integer | No | `100` | Límite máximo de peticiones por minuto por dirección IP. |

---

## Topología y Declaración de Infraestructura en RabbitMQ

El equipo de Auth es responsable de declarar y custodiar la topología base del broker de mensajería para el personal:

```mermaid
flowchart LR
    Producer["Microservicio Auth<br/>(Publisher)"]
    Exchange{{"Exchange: restaurant.staff.events<br/>(Type: topic, Durable: true)"}}
    QueueSala[("Queue: sala.staff-sync<br/>Binding: staff.*")]
    QueueAudit[("Queue: audit.staff-history<br/>Binding: staff.#")]
    ConsumerSala["Microservicio Sala / Host<br/>(Sincroniza Meseros)"]
    ConsumerAudit["Servicio Auditoría<br/>(Registros Históricos)"]

    Producer -->|"Publish: staff.created,<br/>staff.roles.updated"| Exchange
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
