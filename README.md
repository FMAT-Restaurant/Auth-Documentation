<div align="center">

# Auth Documentation · FMAT Restaurant

![Tipo](https://img.shields.io/badge/tipo-especificaci%C3%B3n%20y%20dise%C3%B1o-111111)
![Materia](https://img.shields.io/badge/materia-Verificaci%C3%B3n%20y%20Validaci%C3%B3n-blue)
![Idioma](https://img.shields.io/badge/idioma-espa%C3%B1ol-444444)
![Estado](https://img.shields.io/badge/estado-especificaci%C3%B3n%20ERS%20aprobada-brightgreen)

</div>

## Propósito del Repositorio

Este repositorio contiene la definición documental, formal y verificable del bounded context **Auth** (Autenticación, Autorización y Gestión de Personal) del ecosistema distribuido **FMAT Restaurant**, desarrollado como proyecto integrador para la materia de *Verificación y Validación* en la Facultad de Matemáticas (UADY).

Adicionalmente, este repositorio centraliza las especificaciones operativas de las responsabilidades transversales asignadas al equipo de Auth:
1. **Microservicio de Autenticación y Autorización (Auth)**: Gestión autoritativa de restaurantes (tenants), credenciales, ciclo de vida del personal y emisión de tokens criptográficos (JWT).
2. **API Gateway**: Perímetro de seguridad, validación de firmas digitales de acceso, autorización RBAC perimetral y propagación segura de contexto de usuario hacia microservices internos.
3. **Broker de Eventos (RabbitMQ)**: Topología, exchanges y contratos de eventos emitidos ante cambios en el personal (como la sincronización de meseros activos hacia el microservicio de Sala/Host).

---

## Estructura Documental (ERS Modular)

Siguiendo la metodología de especificación formal y trazabilidad de requisitos del proyecto, la documentación se organiza modularmente en `output/ers/`:

```text
Auth-Documentation/
├── output/
│   └── ers/
│       ├── context.md                    # Contexto, límites de bounded context, actores y lenguaje ubicuo
│       ├── architecture.md               # Arquitectura del servicio, API Gateway, RabbitMQ y modelo de datos
│       ├── business-rules.md             # Reglas de negocio (BR-AUTH) e invariantes formales (INV-AUTH)
│       ├── functional-requirements.md    # Requisitos funcionales formales (REQ-AUTH) con métodos de verificación
│       ├── non-functional-requirements.md# Requisitos no funcionales (NFR-AUTH): seguridad, rendimiento y auditoría
│       ├── configuration.md              # Variables de entorno, topología de colas RabbitMQ y perfiles
│       ├── traceability.md               # Matriz de trazabilidad (Requisitos, Reglas, Endpoints, Eventos, Pruebas)
│       └── open.md                       # Decisiones abiertas, supuestos y extensiones futuras
└── README.md                             # Guía general de la especificación
```

---

## Índice Rápido de la Especificación

| Documento | Enlace | Resumen de Contenido |
| :--- | :--- | :--- |
| **01. Contexto y Dominio** | [context.md](output/ers/context.md) | Bounded context, actores (Gerente/Admin, Host, Almacenista, Mesero, Chef Master) y ownership de identidad. |
| **02. Arquitectura y Componentes** | [architecture.md](output/ers/architecture.md) | Flujo perimetral API Gateway, esquema JWT, topología RabbitMQ y modelo entidad-relación. |
| **03. Reglas de Negocio e Invariantes** | [business-rules.md](output/ers/business-rules.md) | Reglas de asignación de roles, invariantes formales de unicidad de ID `XYYYYYY`, obligatoriedad de roles y control de contraseñas temporales. |
| **04. Requisitos Funcionales** | [functional-requirements.md](output/ers/functional-requirements.md) | Catálogo formal de requisitos (`REQ-AUTH-001` al `REQ-AUTH-020`) con criterio y método de verificación formal de V&V. |
| **05. Requisitos No Funcionales** | [non-functional-requirements.md](output/ers/non-functional-requirements.md) | Políticas criptográficas (Argon2id/bcrypt, RS256/Ed25519), latencias perimetrales y seguridad en descanso y tránsito. |
| **06. Configuración y Despliegue** | [configuration.md](output/ers/configuration.md) | Configuración de entorno, llaves de firma, tópicos/exchanges de RabbitMQ y perfiles de ejecución. |
| **07. Trazabilidad** | [traceability.md](output/ers/traceability.md) | Mapeo bidireccional entre requisitos, reglas de negocio, endpoints REST, eventos asíncronos y casos de prueba. |
| **08. Puntos Abiertos** | [open.md](output/ers/open.md) | Consideraciones de evolución, rotación de claves e integración con proveedores de identidad externos. |

---

## Repositorios Relacionados del Ecosistema

- **Backend de Auth**: [`Auth-Backend`](https://github.com/FMAT-Restaurant/Auth-Backend)
- **Frontend de Auth**: [`Auth-Frontend`](https://github.com/FMAT-Restaurant/Auth-Frontend)
- **Documentación Central / Contratos**: [`Documentation`](https://github.com/FMAT-Restaurant/Documentation)
- **Servicio de Menú**: [`Menu-Documentation`](https://github.com/FMAT-Restaurant/Menu-Documentation)
- **Servicio de Órdenes y Cocina**: [`OrdersKDS-Documentation`](https://github.com/FMAT-Restaurant/OrdersKDS-Documentation)
- **Servicio de Inventario**: [`Inventory-Documentation`](https://github.com/FMAT-Restaurant/Inventory-Documentation)
