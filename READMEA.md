# BodegaPro – Control de inventario para bodegas - grupo7

Proyecto grupal – **Cibertec**.

BodegaPro es un sistema web de gestión de inventario para bodegas que permite registrar productos, categorías y proveedores, controlar entradas y salidas de mercadería (kardex), recibir alertas asíncronas de stock bajo, exportar reportes en Excel y PDF, y visualizar indicadores clave en un panel de control (valor del inventario, stock crítico y productos próximos a vencer).

El backend está construido con una arquitectura de **microservicios en Spring Boot 3.2.5** (Java 17, Spring Cloud 2023.0.1, Gradle) y el frontend con **Angular 21**.

---

## 1. Arquitectura

```
Angular :4200  ──►  api-gateway :8080  ──►  auth-service         :8081
                    (valida JWT, CORS)  ──►  catalog-service      :8082
                          │             ──►  inventory-service    :8083
                          │             ──►  notification-service :8084
                    config-server :8888 (configuración centralizada)
```

| Carpeta / servicio | Puerto | Responsabilidad |
|---|---|---|
| `backend/config-server` | 8888 | Servidor de configuración centralizada (Spring Cloud Config con perfil `native`). |
| `backend/api-gateway` | 8080 | Punto de entrada único (Spring Cloud Gateway). Valida el JWT (HS384), aplica CORS y propaga las cabeceras `X-User` y `X-Role` a los servicios internos. |
| `backend/auth-service` | 8081 | Autenticación y registro de usuarios. Contraseñas cifradas con BCrypt y emisión de tokens JWT (expiración de 120 min). |
| `backend/catalog-service` | 8082 | Gestión de productos, categorías, proveedores y stock actual. Publica eventos `StockAlertEvent` en RabbitMQ cuando el stock llega al mínimo. |
| `backend/inventory-service` | 8083 | Registro de movimientos de stock (kardex), dashboard ejecutivo y exportación de reportes en Excel (Apache POI) y PDF (PDFBox). Consulta y actualiza el catálogo vía OpenFeign con Resilience4j (Retry + Circuit Breaker). |
| `backend/notification-service` | 8084 | Consume eventos de stock bajo desde RabbitMQ, almacena en memoria las 50 alertas más recientes y registra métricas en Micrometer (`bodegapro.eventos.recibidos`, `bodegapro.alertas.stock_bajo`). |
| `backend/common-dto` | – | Librería compartida de DTOs (`record`) y eventos (`StockAlertEvent`) entre los microservicios. |
| `frontend/inventario-web` | 4200 | Aplicación SPA en Angular 21. |
| `k8s/` | – | Manifiestos declarativos de Kubernetes (Namespace, Secrets, ConfigMaps, PVC, Deployments y Services). |
| `docker/` | – | Configuraciones de Prometheus (`prometheus.yml`) y Logstash (`logstash.conf`). |
| `database/bodegapro.sql` | – | Script SQL montado al iniciar el contenedor de MySQL; crea las tablas y precarga datos iniciales de prueba (5 categorías, 3 proveedores y 8 productos). Hibernate también gestiona el esquema (`ddl-auto=update`). |
| `docs/` | – | Colección de Postman (`BodegaPro_Postman_Collection.json`). |

### Comunicación entre servicios

- **Síncrona (REST / OpenFeign):** `inventory-service` → `catalog-service` para consultar productos, categorías y proveedores, y actualizar el stock al registrar un movimiento, protegido con **Circuit Breaker** (`catalogServiceCB`) y **Retry** (`catalogServiceRetry`).
- **Asíncrona (RabbitMQ):** `catalog-service` publica `StockAlertEvent` en el `DirectExchange` `bodegapro.alerts.exchange` con la routing key `bodegapro.alerts.stock`. Tanto `inventory-service` (cola `bodegapro.stock.alert.queue`) como `notification-service` (cola `bodegapro.notification.alert.queue`) consumen el evento de forma independiente.

### Persistencia

- **MySQL 8.0** expuesto en el puerto `5510` del host (`3306` en el contenedor), base de datos `appdb` (usuario `app` / contraseña `password`).

---

## 2. Requisitos

- **JDK 17** (`JAVA_HOME` configurado)
- **Node.js 20 LTS** o superior (con `npm`   10+)
- **Docker Desktop** (para MySQL, RabbitMQ y el stack opcional de observabilidad)

*(No es necesario instalar Gradle globalmente; el proyecto incluye Gradle Wrapper `gradlew` / `gradlew.bat` 8.7).*

---

## 3. Cómo ejecutar

La guía detallada se encuentra en [`ejecucion_paso_a_paso.md`](ejecucion_paso_a_paso.md). Resumen para desarrollo local (modo híbrido: infraestructura en Docker y microservicios desde consola o IDE):

### 1) Infraestructura con Docker

Solo base de datos y broker de mensajería:
```bash
docker compose up -d mysql rabbitmq
```

O incluyendo el stack completo de observabilidad  (Prometheus, Grafana y ELK):
```bash
docker compose up -d mysql rabbitmq elasticsearch logstash kibana prometheus grafana
```

### 2) Backend ( desde la carpeta `backend/`, un terminal por servicio y en este orden)

En **Windows (PowerShell / CMD)**:
```powershell
cd backend
.\gradlew.bat :config-server:bootRun
.\gradlew.bat :auth-service:bootRun
.\gradlew.bat :catalog-service:bootRun
.\gradlew.bat :inventory-service:bootRun
.\gradlew.bat :notification-service:bootRun
.\gradlew.bat :api-gateway:bootRun
```

En **Linux / macOS / Git Bash**:
```bash
cd backend
./gradlew :config-server:bootRun
./gradlew :auth-service:bootRun
./gradlew :catalog-service:bootRun
./gradlew :inventory-service:bootRun
./gradlew :notification-service:bootRun
./gradlew :api-gateway:bootRun
```

> **Nota sobre `docker-compose.yml`:** En el archivo `docker-compose.yml` los contenedores de los microservicios no publican sus puertos al host a propósito para permitir el desarrollo híbrido sin conflictos de puertos con el IDE. Si se levantan todos los contenedores con `docker compose up -d --build`, debe generarse antes el `.jar` de cada módulo (`.\gradlew.bat bootJar -x test`).

### 3) Frontend Angular

```bash
cd frontend/inventario-web
npm install
npm start          # disponible en http://localhost:4200
```

### Usuarios y datos de prueba

`auth-service` crea automaticamente al iniciar los siguientes usuarios de prueba:

| Usuario | Contraseña | Rol |
|---|---|---|
| `admin` | `admin123` | `ADMIN` |
| `vendedor` | `vendedor123` | `VENDEDOR` |

El Contenedor de MySQL precarga automáticamente 5 categorías, 3 proveedores y 8 productos desde `database/bodegapro.sql` en su primera ejecución.

---

## 4. Endpoints (vía API Gateway en `http://localhost:8080`)

| Método | Ruta | Descripción | Acceso |
|---|---|---|---|
| `POST` | `/auth/login` | Inicia sesión y devuelve el token JWT y datos del usuario | Público |
| `POST` | `/auth/register` | Registra un nuevo usuario (asigna rol `VENDEDOR`) | Público |
| `GET` · `POST` · `PUT` · `DELETE` | `/categorias`, `/categorias/{id}` | CRUD de categorías | JWT |
| `GET` · `POST` · `PUT` · `DELETE` | `/proveedores`, `/proveedores/{id}` | CRUD de proveedores | JWT |
| `GET` · `POST` · `PUT` · `DELETE` | `/productos`, `/productos/{id}` | CRUD de productos | JWT |
| `PUT` | `/productos/{id}/stock` | Actualiza el stock de un producto (usado por `inventory-service`) | JWT |
| `GET` · `POST` | `/movimientos` | Historial de movimientos / registrar `ENTRADA` o `SALIDA` | JWT |
| `GET` | `/dashboard` | Totales, alertas de stock bajo, productos por vencer y valor del inventario | JWT |
| `GET` | `/reportes/movimientos/excel` · `/reportes/movimientos/pdf` | Descarga el kardex en `.xlsx` o `.pdf`. Filtros opcionales: `desde`, `hasta` (`AAAA-MM-DD`) y `tipo` (`ENTRADA` / `SALIDA`) | JWT |
| `GET` | `/alertas` (o `/notificaciones`) | Lista las últimas alertas de stock bajo (máx. 50) | JWT |
| `GET` | `/actuator/health`, `/actuator/prometheus` | Estado de salud y métricas Prometheus | Público |

- Las rutas protegidas requieren la cabecera `Authorization: Bearer <token>`.
- Las respuestas de error devuelven formato estándar JSON: `{ "status": 409, "message": "...", "errors": [] }`.

---

## 5. Reglas de negocio implementadas

- **Control de stock por movimientos:** Una `ENTRADA` suma al stock actual y una `SALIDA` resta. Si una salida supera el stock disponible, devuelve `409 Conflict` y no realiza cambios.
- **Trazabilidad de usuario:** El Gateway extrae el usuario del JWT y lo envía en la cabecera `X-User` para registrar quién ejecutó cada movimiento.
- **Validación de precios:** El precio de venta no puede ser menor al precio de compra (`400 Bad Request`).
- **Unicidad de registros:** No se permiten códigos de producto duplicados, nombres de categoría repetidos ni proveedores con el mismo RUC (`409 Conflict`). El RUC debe tener exactamente 11 dígitos numéricos (`400 Bad Request`).
- **Integridad referencial en catálogo:** No se permite eliminar una categoría ni un proveedor si tienen productos asociados (`409 Conflict`).
- **Alertas automáticas de stock crítico:** Si tras actualizar el stock `stockActual <= stockMinimo`, `catalog-service` emite un evento asíncrono por RabbitMQ hacia `inventory-service` y `notification-service`.
- **Control de vencimientos:** El dashboard marca como "por vencer" los productos cuya fecha de vencimiento ocurre dentro de los próximos **30 días** (o ya vencieron).
- **Tolerancia a fallos (Circuit Breaker + Retry):** Si `catalog-service` no está disponible, el panel de control degrada elegantemente devolviendo listas vacías, mientras que el registro de movimientos responde `503 Service Unavailable` sin guardar movimientos inconsistentes.
- **Control de rol en interfaz:** La interfaz Angular restringe las acciones de eliminación exclusivamente al rol `ADMIN`.

---

## 6. Pruebas unitarias

```bash
cd backend
./gradlew test        # En Windows: .\gradlew.bat test
```

El proyecto cuenta con **14 pruebas unitarias** implementadas con JUnit 5 y Mockito sobre la capa de negocio:

| Servicio | Clase de prueba | Pruebas | Casos verificados |
|---|---|---|---|
| `auth-service` | `AuthServiceTest` | 5 | Login exitoso con JWT, contraseña incorrecta (401), usuario inexistente (401), registro con hash BCrypt y rol `VENDEDOR`, y usuario duplicado (409). |
| `catalog-service` | `ProductoServiceTest` | 2 | Rechazo de precio de venta menor al de compra (400) y código de producto duplicado (409). |
| `inventory-service` | `MovimientoServiceTest` | 4 | Entrada suma stock, salida resta stock, salida con stock insuficiente (409 sin modificar datos) y producto inexistente (404). |
| `notification-service` | `AlertaServiceTest` | 3 | Generación de alerta por stock bajo + métrica, ignorar evento con stock normal y límite máximo de 50 alertas en memoria. |

---

## 7. Monitoreo y logs (Docker Compose)

| Herramienta | URL | Credenciales / Detalle |
|---|---|---|
| **RabbitMQ Management** | http://localhost:15672 | `guest` / `guest` |
| **Prometheus** | http://localhost:9090 | Scraping cada 15 s sobre `/actuator/prometheus` de los 6 servicios Spring Boot |
| **Grafana** | http://localhost:3000 | `admin` / `admin` |
| **Kibana** | http://localhost:5601 | Crear data view `bodegapro-logs-*` |
| **Elasticsearch** | http://localhost:9200 | Recibe logs estructurados desde Logstash (`:5044`) |

Cada servicio expone métricas con Micrometer + Actuator (`/actuator/prometheus`) y envía sus logs en JSON a Logstash mediante `logstash-logback-encoder`.

---

## 8. Kubernetes

La carpeta `k8s/` contiene los manifiestos numerados (`00-namespace.yaml` a `09-api-gateway.yaml`) y el consolidado `k8s-all-in-one.yaml` para desplegar en el namespace `bodegapro`: MySQL (con PVC), RabbitMQ, `config-server`, `auth-service`, `catalog-service`, `inventory-service` y `api-gateway` (expuesto por `LoadBalancer` en el puerto `8080`). Consultar [`k8s/README.md`](k8s/README.md).

---

## 9. Limitaciones conocidas

- La restricción de eliminación por rol (`ADMIN`) se aplica en el frontend; aunque el Gateway propaga la cabecera `X-Role`, los controladores REST no bloquean `DELETE` por rol en el servidor.
- Si `catalog-service` no responde al registrar un movimiento, el Circuit Breaker devuelve `503 Service Unavailable`; no se encolan movimientos pendientes.
- No se restringe en base de datos la eliminación de un producto que ya tenga movimientos históricos registrados en `movimientos_stock`.
- Las alertas de `notification-service` se almacenan en memoria (máximo 50) y se reinician al detener el servicio.
- `notification-service` y el stack de observabilidad (Prometheus, Grafana, ELK) están configurados en `docker-compose.yml`, pero no tienen manifiestos en `k8s/`.
- En `docker-compose.yml` los microservicios no exponen puertos al host para facilitar el modo de desarrollo híbrido.

---

## 10. Colección de Postman

Importar [`docs/BodegaPro_Postman_Collection.json`](docs/BodegaPro_Postman_Collection.json) en Postman. Ejecutar primero la petición **`AUTH - Login admin`** (o **`AUTH - Login vendedor`**); el script de prueba guarda automáticamente el JWT en la variable de colección `{{token}}` para todas las demás peticiones.
