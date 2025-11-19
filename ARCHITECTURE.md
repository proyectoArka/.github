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

## Diagrama de alto nivel

Copiar este bloque en un renderizador Mermaid (GitHub o VSCode con extensión Mermaid) para verlo interactivo.

```mermaid
flowchart LR
  subgraph CLIENTES
    A[Usuario / Frontend]
  end

  subgraph GATEWAY
    G[MSApiGateway]
  end

  subgraph DISCOVERY
    E[Eureka]
    C[Config Server]
  end

  subgraph BACKEND
    Auth[MSAuthentication]
    Cart[MSCart]
    Inv[MSInventario]
    Orden[MSOrden]
    Lambda[LambdaEmail]
  end

  subgraph INFRA
    PG[(PostgreSQL)]
    RMQ[(RabbitMQ)]
    Dynamo[(DynamoDB)]
    Brevo[Brevo (Email API)]
  end

  A -->|HTTP: /api/*| G
  G -->|Discovery (Eureka) & routing| Auth
  G --> Cart
  G --> Inv
  G --> Orden

  Auth -->|users, tokens| PG
  Cart -->|carritos, estado| PG
  Inv -->|productos, stock| PG
  Orden -->|órdenes| PG

  C ---|proporciona YAML| Auth
  C --- Cart
  C --- Inv
  C --- Orden

  RMQ ---|Spring Cloud Bus| C
  RMQ ---|bus| G

  Orden -->|solicita envío email| Lambda
  Lambda -->|guarda logs| Dynamo
  Lambda -->|usando API| Brevo

  Auth --- E
  Cart --- E
  Inv --- E
  Orden --- E
  G --- E

  click G "#MSApiGateway" "Ver configuración de rutas en `config_server_files/MSApiGateway.yml`"
  click Auth "#MSAuthentication" "Ver `config_server_files/MSAuthentication.yml`"
``` 

> Nota: Los `click` en Mermaid funcionan en plataformas que soporten enlaces; GitHub renderiza Mermaid sin clicks, pero VSCode con extensiones sí.

---

## Flujo de ejemplo: Login y uso de token

1. El cliente hace POST `/api/v1/gateway/auth/login` al `MSApiGateway`.
2. El gateway reescribe y enruta la petición a `MSAuthentication`.
3. `MSAuthentication` valida credenciales contra PostgreSQL y devuelve JWT.
4. El cliente incluye `Authorization: Bearer <token>` en llamadas siguientes al gateway.
5. El gateway aplica un filtro de autorización y, si procede, reenvía la petición al microservicio destino.

## Flujo de ejemplo: Checkout (carrito → orden)

1. Cliente solicita checkout en `MSCart`.
2. `MSCart` valida stock solicitando `/stockprice/{id}` a `MSInventario` (vía carga/`lb://MSInventario` o a través del gateway según configuración).
3. Si hay stock suficiente, `MSCart` solicita la creación de orden a `MSOrden`.
4. `MSOrden` crea la orden en PostgreSQL, reduce stock en `MSInventario`.
5. `MSOrden` manda evento para notificar (interno o llamado a `LambdaEmail`) que dispara el envío de correo.

---

## Archivos de configuración clave (en este repo `config_server_files`)

- `MSApiGateway.yml` — Rutas públicas y protegidas del gateway.
- `MSAuthentication.yml` — Secretos JWT y puerto del servicio de auth.
- `MScart.yml`, `MSInventario.yml`, `MSOrden.yml` — Configs de cada servicio (endpoints externos, tiempos para carrito abandonado, etc.).
- `application.yml` — Valores generales (por ejemplo: `spring.datasource`, `rabbitmq`) usados por varios servicios.

Revisa la carpeta `config_server_files` para ver cada archivo YAML y adaptarlos a tu entorno.

---

## Cómo ver el diagrama Mermaid localmente

- En GitHub: crea un PR/commit con este archivo y GitHub renderizará Mermaid automáticamente (si está activado en tu organización). Si no, usa la extensión VSCode.
- En VSCode: instala la extensión `vstirbu.vscode-mermaid-preview` o `mermaid-preview` y abre `ARCHITECTURE.md`, luego activa la vista previa.

---

## Notas para levantar el entorno local mínimo

1. PostgreSQL con la base `arka_db` (usuario `postgres`, contraseña `123` según `application.yml` dentro de `config_server_files`).
2. RabbitMQ en `localhost:5672` (usuario `admin` / `admin123`) para Spring Cloud Bus.
3. Levantar `MSEureka` (registro), `MSConfigServer` (apunta a este repo `config_server_files`), luego los microservicios (Gateway, Authentication, Inventario, Cart, Orden).
4. Opcional: desplegar `LambdaEmail` en AWS (serverless) o simular la llamada HTTP a la URL configurada en `MScart.yml` / `MSOrden.yml`.

Comandos rápidos (ejecución en Windows PowerShell, desde cada microservicio):

```powershell
# Ejemplo: desde la carpeta de un microservicio Gradle
./gradlew bootRun

# Para Maven (MSCart):
mvn spring-boot:run
```

---

## Próximos pasos que puedo hacer por ti

- Añadir diagramas de secuencia (Mermaid) para los flujos: login, checkout y carrito abandonado.
- Extraer y documentar endpoints importantes por servicio y ejemplos de request/response.
- Generar README por cada microservicio (si quieres que lo cree automáticamente desde los controladores/existing README).

Si quieres que continúe con alguno de los puntos anteriores, dime cuál prefieres y lo desarrollo a continuación.
