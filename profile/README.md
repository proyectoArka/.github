# Arquitectura general del proyecto Arka

Este documento describe de forma general cómo está compuesto el ecosistema de microservicios `Arka`, qué tecnologías utiliza y cómo interactúan los servicios y las bases de datos.

---

## Servicios principales

- **MSEureka**: Servidor de descubrimiento (Eureka). Registra y permite localizar microservicios.
- **MSConfigServer**: Servidor de configuración central, apunta al directorio `config_server_files` con archivos YAML por servicio.
- **MSApiGateway**: Gateway que expone rutas públicas y protegidas, reescribe paths y aplica filtros de autorización.
- **MSAuthentication**: Gestiona usuarios, roles, permisos, login y refresh de tokens JWT.
- **MSCart**: Gestión de carritos (reactivo con WebFlux en algunos módulos). Detecta carritos abandonados y puede orquestar la creación de órdenes.
- **MSInventario**: CRUD de productos, gestión de stock.
- **MSOrden**: Gestión y creación de órdenes de compra; comunica con inventario y envía notificaciones (email) tras confirmación cuando pasa una orden de un estado a otro.
- **LambdaEmail**: Función serverless (AWS Lambda) que envía correos utilizando Brevo y guarda logs en DynamoDB.

---

## Stack tecnológico (por capa)

- Lenguaje: Java 21
- Framework: Spring Boot 3.x (mix WebFlux y MVC según servicio)
- Observabilidad: Actuator (endpoints expuestos en varios servicios)
- Documentación API: SpringDoc OpenAPI / Swagger UI (algunos servicios)
- Mensajería: RabbitMQ + Spring Cloud Bus (refresh de configuración)
- Persistencia: PostgreSQL (JDBC/JPA y R2DBC para servicios reactivos)
- Registro de servicios: Netflix Eureka
- Configuración central: Spring Cloud Config Server (repo: `config_server_files`)
- Autenticación: JWT (jjwt)
- Serverless: AWS Lambda (Java 21), DynamoDB para logs
---

## 🏗️ Diagrama de Arquitectura del Ecosistema Arka

![diagrama](https://github.com/user-attachments/assets/77143360-19ba-4eb3-8f36-201ae33dcd02)

## 🔒 Seguridad en Arka

La plataforma Arka implementa varias capas de seguridad para proteger los datos y las operaciones:

- **Autenticación JWT:** Todos los microservicios protegidos requieren un token JWT válido, emitido por el microservicio MSAuthentication tras login exitoso.
- **Roles y permisos:** El sistema gestiona roles (admin, cliente, etc.) y permisos granulares para cada endpoint. Los roles se asignan al usuario y se validan en cada petición.
- **Spring Security:** Cada microservicio utiliza Spring Security para proteger rutas, validar tokens y aplicar reglas de acceso.
- **Encriptación de contraseñas:** Las contraseñas de los usuarios se almacenan en la base de datos usando algoritmos de hash seguros (BCrypt).
- **Validación de datos:** Todos los datos recibidos por los endpoints se validan para evitar inyecciones y errores de formato.
- **CORS y HTTPS:** El gateway y los microservicios están configurados para aceptar peticiones solo desde orígenes permitidos y bajo HTTPS.
- **Refresh tokens:** El sistema soporta refresh tokens para renovar sesiones de usuario de forma segura sin exponer credenciales.

## 📬 Mensajería y eventos

- **RabbitMQ:** Utilizado como bus de eventos para comunicación asíncrona entre microservicios y para el refresh dinámico de configuración vía Spring Cloud Bus.
- **Spring Cloud Bus:** Permite que los cambios en la configuración se propaguen automáticamente a todos los servicios conectados.
- **Eventos de negocio:** Los microservicios pueden publicar y consumir eventos (por ejemplo, creación de orden, actualización de stock, envío de email) para desacoplar procesos y mejorar la escalabilidad.
- **Notificaciones:** El sistema puede enviar notificaciones internas y externas (por ejemplo, emails mediante LambdaEmail y Brevo) en respuesta a eventos clave.